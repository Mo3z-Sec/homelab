# Pi-hole

## Overview

Network-wide DNS ad blocker running as an LXC container on vmbr0.
All devices on the home network use Pi-hole as their DNS resolver.
Upstream DNS queries are forwarded to Cloudflare (1.1.1.1).

---

## Container Details

| Field     | Value              |
|-----------|--------------------|
| CT ID     | 100                |
| IP        | 192.168.0.51        |
| RAM       | 256MB              |
| Storage   | 2GB                |
| Bridge    | vmbr0              |
| OS        | Debian 12          |
| Status    | ✅ Running          |

---

## Deployment

### 1. Create LXC Container

In Proxmox web UI:
- Template: Debian 12
- RAM: 256MB
- Storage: 2GB
- Network: vmbr0, static IP 192.168.0.X
- Unprivileged: yes

### 2. Install Pi-hole

```bash
apt update && apt upgrade -y
apt install curl -y
curl -sSL https://install.pi-hole.net | bash
```

### 3. Install Settings

| Setting          | Value              |
|------------------|--------------------|
| Interface        | eth0               |
| Upstream DNS     | Cloudflare (1.1.1.1 / 1.0.0.1) |
| Block lists      | Default (StevenBlack) |
| Admin web UI     | Enabled            |
| Query logging    | Enabled            |

### 4. Set Web UI Password

```bash
pihole -a -p
```

---

## Configuration

### Router DNS Setting

Point the home router's DNS to Pi-hole so all devices use it
automatically:

- Primary DNS: `192.168.0.X` (Pi-hole)
- Secondary DNS: `1.1.1.1` (fallback)

### Upstream DNS

| Provider    | Primary   | Secondary |
|-------------|-----------|-----------|
| Cloudflare  | 1.1.1.1   | 1.0.0.1   |

### Block Lists

| List                        | Entries   |
|-----------------------------|-----------|
| StevenBlack Unified Hosts   | ~100k+    |


---

## Access

| Interface | URL                                   |
|-----------|---------------------------------------|
| Web UI    | `http://192.168.0.X/admin`            |
| SSH       | `ssh root@192.168.0.X`                |

---

## Hardening

- [x] Change default admin password
- [ ] Disable SSH password auth (key only)
- [ ] Restrict web UI access to LAN only
- [ ] Enable HTTPS for web UI via Nginx Proxy Mgr

---

## Maintenance

### Update Pi-hole

```bash
pihole -up
```

### Update Gravity (block lists)

```bash
pihole -g
```

### Check Status

```bash
pihole status
```

---

## Integration

- **Nginx Proxy Manager** — will proxy the admin UI with SSL once deployed
- **Heimdall** — will be added to the dashboard once deployed
- **Grafana** — Pi-hole has a Grafana integration via FTL API for DNS
  query dashboards

---

## Issues

See [issues-log.md](../proxmox/issues-log.md) for any deployment
issues encountered.