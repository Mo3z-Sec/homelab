# Uptime Kuma

## Overview

Internal service uptime monitor providing a live dashboard of all
homelab services. Alerts when individual services go down while
Proxmox is still running.

**Limitation:** If Proxmox itself goes down, Uptime Kuma goes down
with it — no external alerts. An external monitor is planned to cover
this gap. See notes below.

---

## Container Details

| Field    | Value             |
|----------|-------------------|
| CT ID    | 106              |
| IP       | 192.168.0.57      |
| RAM      | 256 MB             |
| Storage  | 4GB               |
| Bridge   | vmbr0             |
| OS       | Debian 12         |
| Status   | ✅ Running         |

---

## Deployment

```bash
apt update && apt upgrade -y
apt install curl -y
curl -fsSL https://get.docker.com | sh
mkdir -p /etc/docker
```

`/etc/docker/daemon.json`:

```json
{
  "dns": ["1.1.1.1", "8.8.8.8"],
  "max-concurrent-downloads": 1
}
```

```bash
systemctl restart docker
mkdir -p /opt/uptime-kuma
```

`/opt/uptime-kuma/docker-compose.yml`:

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    volumes:
      - ./data:/app/data
    ports:
      - "3001:3001"
    restart: unless-stopped
```

```bash
cd /opt/uptime-kuma
docker pull louislam/uptime-kuma:1
docker compose up -d
```

---

## Access

| Interface | URL |
|-----------|-----|
| Dashboard | `http://192.168.0.57:3001` |

---

## Monitors Configured

| Service     | Type      | Host/URL                        | Port  |
|-------------|-----------|---------------------------------|-------|
| Proxmox     | HTTP(s)   | https://192.168.0.169:8006      | —     |
| Pi-hole     | TCP Port  | 192.168.0.51                    | 80    |
| Nextcloud   | HTTP(s)   | http://192.168.0.56             | —     |
| WireGuard   | TCP Port  | 192.168.0.53                    | 22    |
| Heimdall    | HTTP(s)   | http://192.168.0.58             | —     |
| DuckDNS     | HTTP(s)   | http://192.168.0.52             | —     |

---

## Notes

- WireGuard monitored via SSH port 22 (TCP) — WireGuard itself runs
  on UDP 51820 which cannot be monitored via TCP checks
- Pi-hole monitored via TCP port 80 instead of HTTP due to redirect
  behaviour causing false down readings
- Uptime Kuma only monitors internal service health — it cannot alert
  if the entire Proxmox host goes down

## External Monitoring — Planned

Attempted to set up UptimeRobot as an external monitor but it is not
viable with the current network setup:
- ISP blocks inbound ports 80 and 443
- Router does not expose WAN ICMP/ping settings
- WireGuard runs on UDP which external TCP monitors cannot check

**Planned fix:** Set up an external monitoring solution once Cloudflare
Tunnel is configured — this will expose a reachable HTTP endpoint that
external monitors can ping to confirm the server is alive.

---

## Issues

See [issues-log.md](../proxmox/issues-log.md) for any deployment issues.