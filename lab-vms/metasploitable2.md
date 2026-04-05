# Metasploitable 2

## Overview

Intentionally vulnerable Linux VM running on vmbr1. Built by Rapid7
as a training target packed with outdated and misconfigured services.


---

## VM Details

| Field    | Value            |
|----------|------------------|
| VM ID    | 204              |
| IP       | 10.10.10.20      |
| RAM      | 512MB            |
| Storage  | ~8GB             |
| Bridge   | vmbr1            |
| OS       | Ubuntu 8.04      |
| Status   | ✅ Running        |

---

## Setup

Metasploitable 2 comes as a pre-built VMDK — no installation needed.
Just import and run.

### Import Process
- Downloaded zip from SourceForge on Kali
- Transferred VMDK to Proxmox via SCP
- Imported disk using `qm importdisk`
- Attached as IDE device in VM hardware
- Set boot order to ide0

### VM Settings
- Machine type: i440fx (not q35 — old image works better with legacy hardware)
- BIOS: SeaBIOS
- Network model: e1000
- No QEMU guest agent — not supported on this OS

---

## Network Configuration

vmbr1 has no DHCP server so the IP had to be set manually inside
the VM. Without this it boots with no IP and is completely unreachable.

```bash
sudo nano /etc/network/interfaces
```

```
auto eth0
iface eth0 inet static
address 10.10.10.20
netmask 255.255.255.0
gateway 10.10.10.1
```

```bash
sudo /etc/init.d/networking restart
```

---

## Access

| Method | Details |
|--------|---------|
| Console | Proxmox web UI console |
| SSH | `msfadmin@10.10.10.20` |

---

## Open Services

A quick nmap scan shows what's available to attack:

```bash
nmap -sV 10.10.10.20
```

Key services running:

| Port | Service |
|------|---------|
| 21 | FTP (vsftpd 2.3.4) | 
| 22 | SSH (OpenSSH 4.7) | 
| 23 | Telnet |
| 25 | SMTP | |
| 80 | HTTP (Apache) 
| 139/445 | Samba 
| 3306 | MySQL 
| 5432 | PostgreSQL 
| 5900 | VNC
---


## Notes

- Never move this VM to vmbr0 — it's dangerously vulnerable
- Always keep it on vmbr1 with no internet access
- Kali reaches it via static route through Proxmox (10.10.10.0/24 via 192.168.0.169)
- Shut down when not in use to free RAM for other VMs

---

## Issues

See [issues-log.md](../proxmox/issues-log.md) for deployment issues
including the static IP fix for vmbr1.