# DuckDNS

## Overview

Dynamic DNS service that keeps `mo3z-homelab.duckdns.org` pointing to
the current home public IP. Home ISPs change your public IP periodically
— DuckDNS solves this by automatically updating the domain every 5
minutes via a cron job. Required for WireGuard to always find the home
network from anywhere in the world.

---

## Container Details

| Field    | Value             |
|----------|-------------------|
| CT ID    | 102              |
| IP       | 192.168.0.52      |
| RAM      | 128MB             |
| Storage  | 2GB               |
| Bridge   | vmbr0             |
| OS       | Debian 12         |
| Status   | ✅ Running         |

---

## Deployment

### 1. Create LXC Container

In Proxmox web UI:
- Template: Debian 12
- RAM: 128MB
- Storage: 2GB
- Network: vmbr0, static IP `192.168.0.52/24`
- Gateway: `192.168.0.1`
- DNS: `192.168.0.51` (Pi-hole)
- Unprivileged: yes

### 2. Install curl

```bash
apt update && apt upgrade -y
apt install curl -y
```

### 3. Create Update Script

```bash
mkdir -p /opt/duckdns
nano /opt/duckdns/duck.sh
```

Paste the following, replacing `YOUR_TOKEN_HERE` with the token from
the DuckDNS dashboard:

```bash
#!/bin/bash
curl -s "https://www.duckdns.org/update?domains=mo3z-homelab&token=YOUR_TOKEN_HERE&ip=" \
  -o /opt/duckdns/duck.log
```

> The `ip=` is intentionally left blank — DuckDNS auto-detects the
> real public IP. Filling it in manually would send the local LXC IP
> instead.

### 4. Make Script Executable

```bash
chmod +x /opt/duckdns/duck.sh
```

### 5. Test Manually

```bash
bash /opt/duckdns/duck.sh
cat /opt/duckdns/duck.log
```

Expected output: `OK`
If output is `KO` the token is wrong — double check against the
DuckDNS dashboard.

### 6. Set Up Cron Job

```bash
crontab -e
```

Add this line:

```
*/5 * * * * /opt/duckdns/duck.sh >/dev/null 2>&1
```

Runs every 5 minutes automatically.

### 7. Verify Cron is Running

```bash
systemctl status cron
```

If inactive:

```bash
apt install cron -y
systemctl enable cron
systemctl start cron
```

---

## Verification

Check the log after 5 minutes to confirm it's updating:

```bash
cat /opt/duckdns/duck.log
```

Confirm the domain resolves to your real public IP:

```bash
curl ifconfig.me          # your real public IP
nslookup mo3z-homelab.duckdns.org   # should match
```

---

## How It Works

1. Cron fires every 5 minutes and runs `duck.sh`
2. Script sends an HTTP request to the DuckDNS API with the domain and token
3. DuckDNS detects the source IP of the request and updates the DNS record
4. `mo3z-homelab.duckdns.org` now resolves to the current home public IP
5. WireGuard clients use this domain as the endpoint — always finds home

---

## Maintenance

No ongoing maintenance required. The cron job runs silently.
To manually force an update:

```bash
bash /opt/duckdns/duck.sh && cat /opt/duckdns/duck.log
```

---

## Issues

See [issues-log.md](../proxmox/issues-log.md) for any deployment
issues encountered.