# CrowdSec Firewall Bouncer on Keenetic Ultra (ARM)

This guide explains how to build and deploy the CrowdSec Firewall Bouncer on a Keenetic Ultra router running **Entware**.

---

## 0. Prerequisites
- Install crowdsec: https://docs.crowdsec.net/u/getting_started/installation/linux/
- Install entware: [ENG](https://help.keenetic.com/hc/en-us/articles/360021214160-Installing-the-Entware-repository-package-system-on-a-USB-drive) [RUS](https://help.keenetic.com/hc/ru/articles/360021214160-%D0%A3%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-%D1%81%D0%B8%D1%81%D1%82%D0%B5%D0%BC%D1%8B-%D0%BF%D0%B0%D0%BA%D0%B5%D1%82%D0%BE%D0%B2-%D1%80%D0%B5%D0%BF%D0%BE%D0%B7%D0%B8%D1%82%D0%BE%D1%80%D0%B8%D1%8F-Entware-%D0%BD%D0%B0-USB-%D0%BD%D0%B0%D0%BA%D0%BE%D0%BF%D0%B8%D1%82%D0%B5%D0%BB%D1%8C)
- Golang or docker installed
---

## 1. Clone the Repository

```sh
git clone https://github.com/crowdsecurity/cs-firewall-bouncer.git
cd cs-firewall-bouncer
```

---

## 2. Build the Bouncer for ARM

Local:
```sh
# Download Go modules
go mod download
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -ldflags='-s -w' -o cs-firewall-bouncer
```

Or docker:
```sh
# Download Go modules
docker run --rm -v $PWD:/app -w /app golang:1.25-alpine go mod download

# Build the binary for Linux ARM64
docker run --rm -v $PWD:/app -w /app golang:1.25-alpine sh -c "GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -ldflags='-s -w' -o cs-firewall-bouncer"
```

---

## 3. Copy Binary to Router

The router must have **Entware** installed:
copy `cs-firewall-bouncer` to `/opt/bin/cs-firewall-bouncer` on the router

---

## 4. Install Dependencies on Entware

```sh
opkg update
opkg install ipset iptables
```

---

## 5. Configure the Bouncer

Create `/opt/etc/crowdsec/crowdsec-firewall-bouncer.yaml`:

```yaml
mode: iptables
update_frequency: 10s
log_mode: stdout
log_level: info

api_url: https://crowdsec-lapi.acme.corp
api_key: api-key-here

insecure_skip_verify: false
disable_ipv6: true
deny_action: DROP
deny_log: false

supported_decisions_types:
  - ban

blacklists_ipv4: crowdsec-blacklists
blacklists_ipv6: crowdsec6-blacklists
ipset_type: nethash

iptables_chains:
  - FORWARD
iptables_add_rule_comments: false

prometheus:
  enabled: true            # expose metrics for Prometheus to scrape
  listen_addr: 0.0.0.0     # bind to LAN (NOT 127.0.0.1) so Prometheus can reach it
  listen_port: 60601
```

> ℹ️ `fw_bouncer_banned_ips` caveat: **upstream** cs-firewall-bouncer reports `0` here on
> Keenetic. `IPSet.Len()` reads the `Number of entries:` line of `ipset list`, which the
> router's kernel (set protocol v6) does not emit — so the count silently falls back to 0.
> Fixed in our fork (`pkg/ipsetcmd/ipset.go`: count the members listed after `Members:` when
> that header line is missing). Build from the fork to get a working gauge; the other metrics
> (`fw_bouncer_dropped_*`, `lapi_requests_*`) are unaffected.

> ⚠️ Replace `api_url` and `api_key` with your CrowdSec LAPI URL and API key.
> You can generate an API key using:
>
> ```sh
> sudo cscli bouncers add <BouncerName>
> ```

---

## 6. Create Init Script for Entware

Create `/opt/etc/init.d/S99csfirewall`:

```sh
#!/bin/sh

# Wait for router and firewall to load
sleep 40

ENABLED=yes
PROCS=cs-firewall-bouncer
ARGS="-c /opt/etc/crowdsec/crowdsec-firewall-bouncer.yaml"
PREARGS=""
DESC="CrowdSec Firewall Bouncer"
PATH=/opt/sbin:/opt/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

. /opt/etc/init.d/rc.func
```

Make it executable:

```sh
chmod +x /opt/etc/init.d/S99csfirewall
```

---

## 7. Start / Stop / Check Status

```sh
# Start the bouncer
/opt/etc/init.d/S99csfirewall start

# Stop the bouncer
/opt/etc/init.d/S99csfirewall stop

# Check status
/opt/etc/init.d/S99csfirewall status
```

> Scripts starting with `S` in `/opt/etc/init.d/` are automatically executed during Entware boot.

---

## 8. Verify Bouncer in CrowdSec

```sh
sudo cscli bouncers list
```

---

## 9. Persist the firewall rules across Keenetic reloads (REQUIRED)

> ⚠️ Without this step the bouncer silently stops blocking after a while.

Keenetic's firewall manager (**NDM**) periodically rebuilds netfilter (on WAN reconnect,
config change, or a routine refresh) and **wipes the bouncer's iptables rules** — it deletes
`CROWDSEC_CHAIN` and its jump from `FORWARD`. The ipsets stay populated but become orphaned
(nothing references them), so no traffic is filtered. The bouncer only re-creates its rules
on startup, so blocking stays dead until the next restart.

Keenetic exposes a hook directory, `/opt/etc/ndm/netfilter.d/`, whose scripts run after every
netfilter rebuild. Drop in a script that re-asserts the bouncer's plumbing. NDM does **not**
touch ipsets, so we only need to restore the iptables rules that point at the existing sets.

Create `/opt/etc/ndm/netfilter.d/crowdsec.sh`:

```sh
#!/bin/sh
# Re-assert CrowdSec firewall-bouncer rules after Keenetic NDM rebuilds netfilter.
#
# NDM calls this with env vars: $type (iptables|ip6tables) and $table (filter|nat|...).

[ "$type" = "ip6tables" ] && exit 0    # bouncer runs with disable_ipv6: true
[ "$table" != "filter" ] && exit 0     # we only touch the filter table

CHAIN="CROWDSEC_CHAIN"
HOOK_CHAIN="FORWARD"                    # must match iptables_chains in the bouncer config

# Act only once the bouncer has created its sets at least once.
ipset list -n 2>/dev/null | grep -q "^crowdsec-blacklists-" || exit 0

# (Re)create the chain and a DROP rule per crowdsec ipv4 set.
iptables -n -L "$CHAIN" >/dev/null 2>&1 || iptables -N "$CHAIN"
iptables -F "$CHAIN"
for set in $(ipset list -n | grep "^crowdsec-blacklists-"); do
    iptables -A "$CHAIN" -m set --match-set "$set" src -j DROP
done

# Ensure FORWARD jumps to our chain as the very first rule.
iptables -C "$HOOK_CHAIN" -j "$CHAIN" 2>/dev/null || iptables -I "$HOOK_CHAIN" 1 -j "$CHAIN"

logger -t crowdsec-netfilter "re-asserted $CHAIN in $HOOK_CHAIN ($(ipset list -n | grep -c "^crowdsec-blacklists-") set(s))"
exit 0
```

Make it executable:

```sh
chmod +x /opt/etc/ndm/netfilter.d/crowdsec.sh
```

Verify the hook by simulating an NDM flush and re-running it:

```sh
# break it like NDM does
iptables -D FORWARD -j CROWDSEC_CHAIN; iptables -F CROWDSEC_CHAIN; iptables -X CROWDSEC_CHAIN

# run the hook the way NDM does
type=iptables table=filter /opt/etc/ndm/netfilter.d/crowdsec.sh

# chain + jump must be back, References on the set must be 1
iptables -L CROWDSEC_CHAIN -n -v
ipset list crowdsec-blacklists-0 | grep References
```

The script is idempotent — running it repeatedly keeps exactly one `FORWARD` jump and one
DROP rule per set.

---

## 10. Verify it is actually blocking

**a) The bouncer is registered and pulling** (run on the CrowdSec host):

```sh
cscli bouncers list   # your bouncer must show Valid=✔️ with a recent "Last API pull"
```

**b) The ipsets are populated and wired into iptables** (run on the router):

```sh
ipset list crowdsec-blacklists-0 | grep "Number of entries"   # should be > 0
iptables -L FORWARD -n --line-numbers | grep CROWDSEC_CHAIN    # jump must exist (rule #1)
iptables -L CROWDSEC_CHAIN -n -v                              # DROP rules with pkts counters
```

**c) Prove a real DROP with a manual decision** (simplest) — ban an IP you control, e.g. your
phone's current public IP (check it at https://ifconfig.me), from the CrowdSec host:

```sh
cscli decisions add --ip <your-public-ip> --duration 5m -t ban
```

Within ~10s the bouncer pulls it and that source IP is dropped at `FORWARD` — the device loses
access to anything published through the router. On the router you can confirm it:

```sh
iptables -L CROWDSEC_CHAIN -n -v        # DROP counters climb
ipset test crowdsec-blacklists-2 <ip>   # cscli-origin decisions land in the -2 set
```

Remove it again (or wait for it to expire):

```sh
cscli decisions delete --ip <your-public-ip>
```

**d) Alternative — drop straight at the ipset level** (no LAPI involved), handy to block a LAN
device for a quick test; auto-expires in 120s:

```sh
# on the router — replace with the target's IP
ipset add crowdsec-blacklists-1 192.168.1.50 timeout 120
```

> ℹ️ The bouncer keeps **one ipset per decision origin**, all referenced from `CROWDSEC_CHAIN`:
> `CAPI` (community blocklist) → `crowdsec-blacklists-0`, `crowdsec` (local scenarios) →
> `crowdsec-blacklists-1`, `cscli` (manual `cscli decisions add`) → `crowdsec-blacklists-2`.
> The numeric suffix is assigned in the order the bouncer first encounters each origin, so it
> can differ between runs — that is why the netfilter.d hook above enumerates
> `crowdsec-blacklists-*` dynamically instead of hard-coding set names.
