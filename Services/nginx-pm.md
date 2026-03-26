# Nginx Proxy Manager

## Overview

Reverse proxy and SSL manager for accessing internal services via
clean subdomains with HTTPS. Currently suspended due to ISP blocking
ports 80 and 443 on residential connections.

**Status: ⏸ Stopped — pending Cloudflare Tunnel setup**

---

## Container Details

| Field    | Value             |
|----------|-------------------|
| CT ID    | 104             |
| IP       | 192.168.0.54      |
| RAM      | 512MB             |
| Bridge   | vmbr0             |
| Status   | ⏸ Stopped         |

---

## Deployment

```bash
apt update && apt upgrade -y
apt install curl -y
curl -fsSL https://get.docker.com | sh
systemctl restart docker
docker pull jlesage/nginx-proxy-manager
mkdir -p /opt/nginx-pm
```

`/opt/nginx-pm/docker-compose.yml`:

```yaml
services:
  app:
    image: jlesage/nginx-proxy-manager
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "8181:8181"
    volumes:
      - ./data:/config
```

```bash
docker compose up -d
```

Admin UI available at `http://192.168.0.54:8181`.

---

## Planned Proxy Hosts

| Subdomain                          | Forward IP    | Port |
|------------------------------------|---------------|------|
| nextcloud.mo3z-homelab.duckdns.org | 192.168.0.56  | 80   |
| pihole.mo3z-homelab.duckdns.org    | 192.168.0.51  | 80   |
| heimdall.mo3z-homelab.duckdns.org  | 192.168.0.58  | 80   |

---

## Notes

- Use `jlesage/nginx-proxy-manager` image — the official `jc21` image
  caused repeated download timeouts on the home connection
- Use **DNS Challenge** for SSL certificates — ISP blocks port 80 so
  HTTP challenge always fails. Set DNS provider to DuckDNS and enter token
- Cloudflare Tunnel planned to bypass ISP port blocking

---

## Issues

See [issues-log.md](../proxmox/issues-log.md) for deployment issues.