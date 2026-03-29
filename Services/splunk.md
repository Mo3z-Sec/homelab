# Splunk

## Overview

SIEM platform for log ingestion, search, and alerting across all
homelab services. Receives logs from vmbr0 services, vmbr1 lab VMs
via Universal Forwarder, and vmbr2 Cowrie honeypot.

---

## Container Details

| Field    | Value             |
|----------|-------------------|
| CT ID    | 107               |
| IP       | 192.168.0.59      |
| RAM      | 4096MB            |
| Storage  | 40GB              |
| CPU      | 2 cores           |
| Bridge   | vmbr0             |
| OS       | Debian 12         |
| Status   | ✅ Running         |

---

## Deployment

```bash
apt update && apt upgrade -y
apt install curl wget -y

# Download Splunk
wget -O splunk.deb "https://download.splunk.com/products/splunk/releases/9.4.0/linux/splunk-9.4.0-6b4ebe426ca6-linux-amd64.deb"

# Install
dpkg -i splunk.deb

# Start and accept license
/opt/splunk/bin/splunk start --accept-license

# Enable on boot
/opt/splunk/bin/splunk enable boot-start
```

---

## Access

| Interface | URL |
|-----------|-----|
| Web UI    | `http://192.168.0.59:8000` |

---

## Data Inputs Configured

| Type | Port | Source Type | Purpose |
|------|------|-------------|---------|
| UDP  | 514  | syslog      | Receive syslog from containers and VMs |

---

## Log Sources — Planned

| Source | Method | Status |
|--------|--------|--------|
| Proxmox host | Syslog → UDP 514 | ⬜ |
| Pi-hole | Syslog → UDP 514 | ⬜ |
| WireGuard | Syslog → UDP 514 | ⬜ |
| Cowrie honeypot | Universal Forwarder | ⬜ |
| Windows Server 2022 | Universal Forwarder | ⬜ |
| Windows 10 WS01/02 | Universal Forwarder | ⬜ |
| OPNsense | Syslog → UDP 514 | ⬜ |

---

## Network Access

Lab VMs on vmbr1 (`10.10.10.0/24`) need a firewall exception to
forward logs to Splunk on vmbr0. Only port `9997` (Universal Forwarder)
is allowed — all other vmbr1 → vmbr0 traffic remains blocked.

---

## Notes

- Free tier limited to 500MB/day log ingestion — sufficient for homelab
- Splunk is the heaviest service in the stack at 4GB RAM
- RAM upgrade (second 8GB DDR4 stick) recommended before deploying
  lab VMs and OPNsense simultaneously
- Security warning about `allowedDomainList` is non-critical —
  configure under Settings → Email Settings when alerting is set up

---

## Issues

See [issues-log.md](../proxmox/issues-log.md) for any deployment issues.