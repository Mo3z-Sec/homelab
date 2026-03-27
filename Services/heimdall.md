# Heimdall

## Overview

Application dashboard providing a single page with icons linking to
all homelab services. Accessible locally at `http://heimdall.home`
or `http://192.168.0.58` instead of memorizing individual IPs and ports.

---

## Container Details

| Field    | Value             |
|----------|-------------------|
| CT ID    | 105               |
| IP       | 192.168.0.58      |
| RAM      | 256MB             |
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
mkdir -p /opt/heimdall
```

`/opt/heimdall/docker-compose.yml`:

```yaml
services:
  heimdall:
    image: lscr.io/linuxserver/heimdall:latest
    container_name: heimdall
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Riyadh
    volumes:
      - ./config:/config
    ports:
      - "80:80"
      - "443:443"
    restart: unless-stopped
```

```bash
cd /opt/heimdall
docker compose up -d
```

---

## Access

| Interface | URL |
|-----------|-----|
| Dashboard | `http://192.168.0.58` |
| Dashboard | `http://heimdall.home` (via Pi-hole local DNS) |

---

## Services Added

| Service     | URL                        |
|-------------|----------------------------|
| Proxmox     | https://192.168.0.169:8006 |
| Pi-hole     | http://192.168.0.51/admin  |
| Nextcloud   | http://192.168.0.56        |
| WireGuard   | http://192.168.0.53        |

---

## Notes

- Ports 80 and 443 are used internally only — not exposed to the internet
- Accessible remotely via WireGuard VPN
- Pi-hole local DNS record `heimdall.home → 192.168.0.58` set up for
  easy access without memorizing IPs

---

## Issues

See [issues-log.md](../proxmox/issues-log.md) for any deployment issues.