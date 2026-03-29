# Proxmox Host Setup

## Overview

Proxmox VE 9.1 installed on a budget Dell i5-6500 desktop PC as the
primary homelab hypervisor. Replaces Windows as the host OS. Managed
entirely through the web UI from MacBook browser at
`https://192.168.0.169:8006`.

---

## Hardware

| Component | Details                    |
|-----------|----------------------------|
| CPU       | Intel Core i5-6500         |
| RAM       | 8GB DDR4                   |
| Storage   | 931GB HDD (single disk)    |
| NICs      | nic0 (active), nic1        |
| OS        | Proxmox VE 9.1 (Debian 12) |

---

## Installation

### Boot Fix

Added `nomodeset` to boot parameters to resolve GPU hang during
installation. See [issues-log.md](issues-log.md) for full details.

### Install Settings

| Setting  | Value                  |
|----------|------------------------|
| Disk     | sda (931GB HDD)        |
| FS       | ext4                   |
| Hostname | pve.homelab.local      |
| IP       | 192.168.0.169 (static) |
| Gateway  | 192.168.0.1            |
| DNS      | 192.168.0.1            |

---

## Storage Layout

| Volume   | Size   | Purpose                |
|----------|--------|------------------------|
| sda1     | 1007KB | BIOS boot              |
| sda2     | 1GB    | EFI / boot             |
| sda3     | 930GB  | LVM physical volume    |
| pve-swap | 7.7GB  | Swap                   |
| pve-root | 96GB   | Proxmox OS             |
| pve-data | 794GB  | VM and container disks |

---

## Repository Configuration

### Problem

Proxmox VE 9 ships with enterprise repo files requiring a paid
subscription. Causes 401 errors on `apt update` without fixing.

### Fix — Files Disabled

- `/etc/apt/sources.list.d/pve-enterprise.list`
- `/etc/apt/sources.list.d/ceph.list`
- `/etc/apt/sources.list.d/pve-enterprise.sources`
- `/etc/apt/sources.list.d/ceph.sources`

### Free Repo Added

```bash
echo "deb http://download.proxmox.com/debian/pve trixie \
pve-no-subscription" > \
/etc/apt/sources.list.d/pve-no-subscription.list
```

### System Updated

```bash
apt update && apt full-upgrade -y
```

---

## Subscription Nag Removal

Created reusable script at `/usr/local/bin/fix-sub-nag.sh` to re-run
after each system update.

---

## Virtual Networks

### vmbr0 — Services Network

- Auto-created during installation
- Bridged to nic0 (physical ethernet to router)
- All service LXC containers use this bridge
- Full internet access through router
- Subnet: `192.168.0.0/24`

### vmbr1 — Isolated Lab Network

- Manually created post-installation
- **No bridge ports** — not connected to any physical NIC
- All vulnerable lab VMs use this bridge
- Zero internet access by design — vmbr1 has no path to the router
- Zero access to vmbr0 services
- Subnet: `10.10.10.0/24`
- Kali laptop reaches lab VMs via SSH on port 22
- SSH port hardening planned — see hardening checklist
- Exception: port 9997 allowed from vmbr1 → Splunk (192.168.0.59)
  for Universal Forwarder log shipping only

### vmbr2 — DMZ Network (planned)

- Will be created when Cowrie honeypot is deployed
- Internet-facing but completely isolated from vmbr0 and vmbr1
- Cowrie honeypot will be the only service on this bridge
- Real attack traffic captured without any path to internal services
- Subnet: `10.20.20.0/24` (planned)

---

## Firewall Configuration

See [firewall.md](firewall.md) for full rules and configuration.

---

## LXC Containers

| Container   | IP            | RAM    | Purpose          | Status |
|-------------|---------------|--------|------------------|--------|
| Pi-hole     | 192.168.0.51  | 256MB  | DNS / ad block   | ✅     |
| DuckDNS     | 192.168.0.52  | 128MB  | Dynamic DNS      | ✅     |
| WireGuard   | 192.168.0.53  | 256MB  | VPN              | ✅     |
| Nginx PM    | 192.168.0.54  | 512MB  | Reverse proxy    | ⏸     |
| CrowdSec    | 192.168.0.55  | 256MB  | Threat blocking  | ⬜     |
| Nextcloud   | 192.168.0.56  | 512MB  | File storage     | ✅     |
| Uptime Kuma | 192.168.0.57  | 128MB  | Uptime monitor   | ✅     |
| Heimdall    | 192.168.0.58  | 256MB  | Dashboard        | ✅     |
| Splunk      | 192.168.0.59  | 4096MB | SIEM             | ✅     |
| Grafana     | 192.168.0.60  | 512MB  | Dashboards       | ⬜     |
| Cowrie      | vmbr2         | 256MB  | SSH honeypot     | ⬜     |

---

## OPNsense (planned)

Will be deployed as a VM directly on Proxmox with two virtual NICs —
one on vmbr0 and one on vmbr1. Acts as the primary firewall and router
replacing the current Proxmox firewall rules. Includes Suricata IDS/IPS
built in for full network visibility across both bridges.

Planned deployment after RAM upgrade (second 8GB DDR4 stick).

---

## Lab VMs

| VM                  | IP           | RAM   | Purpose             | Status |
|---------------------|--------------|-------|---------------------|--------|
| Windows Server 2022 | 10.10.10.X   | 3GB   | Domain controller   | ⬜     |
| Windows 10 WS01     | 10.10.10.X   | 2GB   | Domain workstation  | ⬜     |
| Windows 10 WS02     | 10.10.10.X   | 2GB   | Misconfigured admin | ⬜     |
| Metasploitable 2    | 10.10.10.X   | 512MB | Vulnerable Linux    | ⬜     |
| DVWA                | 10.10.10.X   | 512MB | Vulnerable web app  | ⬜     |

---

## Checklist

- [x] Proxmox VE 9.1 installed
- [x] Enterprise repos disabled
- [x] Free repo added
- [x] System updated
- [x] Subscription nag removed
- [x] vmbr0 verified
- [x] vmbr1 created
- [x] Firewall configured
- [x] WireGuard VPN configured
- [x] LXC containers deployed (Pi-hole, DuckDNS, WireGuard, Nextcloud, Heimdall, Uptime Kuma, Splunk, Nginx PM)
- [ ] vmbr2 DMZ created
- [ ] Grafana deployed
- [ ] CrowdSec deployed
- [ ] Cowrie honeypot deployed
- [ ] OPNsense deployed
- [ ] Lab VMs deployed
- [ ] SSH hardening
- [ ] 2FA enabled
- [ ] Fail2ban installed
- [ ] Security assessment completed