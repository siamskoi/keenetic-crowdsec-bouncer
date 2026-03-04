# CrowdSec Firewall Bouncer on Keenetic Ultra (ARM)

This guide explains how to build and deploy the CrowdSec Firewall Bouncer on a Keenetic Ultra router running **Entware**.

---

## 0. Prerequisites
- Install crowdsec: https://docs.crowdsec.net/u/getting_started/installation/linux/
- Install entware: [ENG](https://help.keenetic.com/hc/en-us/articles/360021214160-Installing-the-Entware-repository-package-system-on-a-USB-drive) [RUS](https://help.keenetic.com/hc/ru/articles/360021214160-%D0%A3%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-%D1%81%D0%B8%D1%81%D1%82%D0%B5%D0%BC%D1%8B-%D0%BF%D0%B0%D0%BA%D0%B5%D1%82%D0%BE%D0%B2-%D1%80%D0%B5%D0%BF%D0%BE%D0%B7%D0%B8%D1%82%D0%BE%D1%80%D0%B8%D1%8F-Entware-%D0%BD%D0%B0-USB-%D0%BD%D0%B0%D0%BA%D0%BE%D0%BF%D0%B8%D1%82%D0%B5%D0%BB%D1%8C)

---

## 1. Clone the Repository

```sh
git clone https://github.com/crowdsecurity/cs-firewall-bouncer.git
cd cs-firewall-bouncer
```

---

## 2. Build the Bouncer for ARM

```sh
# Download Go modules
docker run --rm -v $PWD:/app -w /app golang:1.25-alpine go mod download

# Build the binary for Linux ARM64
docker run --rm -v $PWD:/app -w /app golang:1.25-alpine sh -c "GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -ldflags='-s -w' -o cs-firewall-bouncer"
```

---

## 3. Copy Binary to Router

The router must have **Entware** installed:

```sh
cp cs-firewall-bouncer /opt/bin/cs-firewall-bouncer
```

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
  enabled: false
  listen_addr: 127.0.0.1
  listen_port: 60601
```

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

## 10. Test Blocking IPs

Add a test IP to block:

```sh
cscli decisions add --ip 1.2.3.4 --duration 1h
```

Check on the router:

```sh
ipset list | grep 1.2.3.4
```
