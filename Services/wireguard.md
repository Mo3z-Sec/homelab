# WireGuard

## Overview

VPN server providing secure remote access to the homelab from anywhere
in the world. Once connected, the client device behaves as if it is
physically on the home network (`192.168.0.0/24`), with full access to
all services and Proxmox itself.

Only one port is exposed to the internet (UDP 51820). All other services
remain completely internal.

---

## Container Details

| Field    | Value             |
|----------|-------------------|
| CT ID    | 103              |
| IP       | 192.168.0.53      |
| RAM      | 256MB             |
| Storage  | 2GB               |
| Bridge   | vmbr0             |
| OS       | Debian 12         |
| Status   | ✅ Running         |

> Container must be **privileged** and have **Nesting** enabled under
> Options → Features.

---

## Network

| Network       | Subnet          | Purpose                        |
|---------------|-----------------|--------------------------------|
| Home LAN      | 192.168.0.0/24  | Homelab services               |
| VPN tunnel    | 10.8.0.0/24     | WireGuard clients              |

| Host          | VPN IP          |
|---------------|-----------------|
| WireGuard LXC | 10.8.0.1        |
| MacBook       | 10.8.0.2        |

---

## Router Port Forward

| Field         | Value           |
|---------------|-----------------|
| Protocol      | UDP             |
| External port | 51820           |
| Internal IP   | 192.168.0.53    |
| Internal port | 51820           |

This is the only port exposed to the internet.

---

## Deployment

### 1. Create LXC Container

In Proxmox web UI:
- Template: Debian 12
- RAM: 256MB
- Storage: 2GB
- Network: vmbr0, static IP `192.168.0.53/24`
- Gateway: `192.168.0.1`
- DNS: `192.168.0.51` (Pi-hole)
- Unprivileged: **No** (must be privileged)
- Features: enable **Nesting**

### 2. Install WireGuard

```bash
apt update && apt upgrade -y
apt install wireguard iptables -y
```

### 3. Generate Server Keys

```bash
cd /etc/wireguard
wg genkey | tee server_private.key | wg pubkey > server_public.key
chmod 600 server_private.key
```

### 4. Enable IP Forwarding

```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

### 5. Create Server Config

```bash
nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# MacBook
PublicKey = CLIENT_PUBLIC_KEY
AllowedIPs = 10.8.0.2/32
```

### 6. Start WireGuard

```bash
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
wg show
```

---

## Adding a New Client

### 1. Generate Client Keys (on WireGuard container)

```bash
cd /etc/wireguard
wg genkey | tee [device]_private.key | wg pubkey > [device]_public.key
```

### 2. Add Peer to Server Config

```bash
nano /etc/wireguard/wg0.conf
```

Add a new `[Peer]` block at the bottom:

```ini
[Peer]
# Device name
PublicKey = DEVICE_PUBLIC_KEY
AllowedIPs = 10.8.0.X/32
```

Increment the last octet for each new device (`.2`, `.3`, `.4`...).

### 3. Reload Server

```bash
systemctl restart wg-quick@wg0
```

### 4. Create Client Config

```ini
[Interface]
PrivateKey = DEVICE_PRIVATE_KEY
Address = 10.8.0.X/24
DNS = 192.168.0.51

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = mo3z-homelab.duckdns.org:51820
AllowedIPs = 192.168.0.0/24
PersistentKeepalive = 25
```

> `AllowedIPs = 192.168.0.0/24` enables split tunneling — only homelab
> traffic goes through the VPN. Regular browsing uses the client's
> normal internet connection.

### 5. Import on Client Device

| Device  | App                                      |
|---------|------------------------------------------|
| MacBook | WireGuard — Mac App Store                |
| iPhone  | WireGuard — App Store                    |
| Kali    | `apt install wireguard`                  |

---

## Registered Clients

| Device  | VPN IP    | Public Key                                   |
|---------|-----------|----------------------------------------------|
| MacBook | 10.8.0.2  | cYA0o3IVMD... |

---

## Verification

On the WireGuard container:

```bash
wg show
```

From the connected client, ping a homelab service:

```bash
ping 192.168.0.51   # Pi-hole — should reply if tunnel is working
```

---

## Notes

- Client source port will appear random on each connection — this is
  normal UDP behaviour. Only the server's destination port (51820) is fixed.
- DuckDNS keeps the endpoint domain updated as the home public IP
  changes — WireGuard clients always use `mo3z-homelab.duckdns.org`
  as the endpoint, never a raw IP.

---

## Hardening

- [ ] SSH port hardening on WireGuard container
- [ ] Fail2ban
- [ ] Rotate client keys periodically

---

## Issues

See [issues-log.md](../proxmox/issues-log.md) for any deployment
issues encountered.