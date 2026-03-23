# Proxmox Host Setup

## Overview
Proxmox VE 9.1 installed on a budget desktop PC I found laying around
 my home as the primary homelab hypervisor. Replaces Windows as the host OS. Managed entirely through the web UI from MacBook browser at https://192.168.0.X:8006

---

## Hardware
| Component | Details                    |
|-----------|----------------------------|
| CPU       | Intel Core i5              |
| RAM       | 8GB DDR4                   |
| Storage   | 931GB HDD (single disk)    |
| NICs      | nic0 (active), nic1        |
| OS        | Proxmox VE 9.1 (Debian 12) |

---

## Installation

### Boot Fix
Added nomodeset to boot parameters to resolve GPU hang
during installation. See issues-log.md for full details.

### Install Settings
- Target disk: sda (931GB HDD)
- Filesystem: ext4
- Hostname: pve.homelab.local
- IP: 192.168.0.X (static)
- Gateway: 192.168.0.1
- DNS: 192.168.0.1

---

## Storage Layout
| Volume     | Size   | Purpose              |
|------------|--------|----------------------|
| sda1       | 1007KB | BIOS boot            |
| sda2       | 1GB    | EFI / boot           |
| sda3       | 930GB  | LVM physical volume  |
| pve-swap   | 7.7GB  | Swap                 |
| pve-root   | 96GB   | Proxmox OS           |
| pve-data   | 794GB  | VM and container disks|

---

## Repository Configuration

### Problem
Proxmox VE 9 ships with four enterprise repo files
requiring a paid subscription. Causes 401 errors on
apt update without fixing.

### Fix — Files Disabled
- /etc/apt/sources.list.d/pve-enterprise.list
- /etc/apt/sources.list.d/ceph.list
- /etc/apt/sources.list.d/pve-enterprise.sources
- /etc/apt/sources.list.d/ceph.sources

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


Created reusable script at /usr/local/bin/fix-sub-nag.sh
to re-run after each system update.

---

## Virtual Networks

### vmbr0 — Services Network
- Auto-created during installation
- Bridged to nic0 (physical ethernet to router)
- All service LXC containers use this bridge
- Full internet access through router
- Subnet: 192.168.0.0/24

### vmbr1 — Isolated Lab Network
- Manually created post-installation
- No bridge ports — connects to nothing
- All vulnerable lab VMs use this bridge
- Zero internet access
- Zero access to vmbr0 services
- Subnet: 10.10.10.0/24
- Kali laptop reaches via static route only

---

## Firewall Configuration

### Levels Enabled
- Datacenter firewall: enabled
- Node (pve) firewall: enabled

### Rules (in order — order is critical)
| Priority | Direction | Action | Protocol | Source | Port | Purpose |
|----------|-----------|--------|----------|--------|------|---------|
| 0 | in | ACCEPT | TCP | 192.168.0.x/24 | 8006 | Web UI |
| 1 | in | ACCEPT | TCP | 192.168.0.x/24 | 22 | SSH |
| 2 | in | ACCEPT | UDP | any | 51820 | WireGuard |
| 3 | in | ACCEPT | ICMP | 192.168.0.x/24 | any | Home network ping
| 4 | in | DROP | any | 10.10.10.x/24 | any | Block lab vms from reaching services|
| 5 | in | DROP | any | any | any | Block all |

### Important Note
Rules are processed top to bottom. ACCEPT rules must
always come before the DROP rule. See issues-log.md
for what happens when this order is wrong.

---

## LXC Containers

| Container | IP | RAM | Purpose | Status |
|---|---|---|---|---|
| Pi-hole | 192.168.0.X | 256MB | DNS / ad blocking | OK |
| DuckDNS | 192.168.0.X | 128MB | Dynamic DNS | ⬜ |
| WireGuard | 192.168.0.X | 256MB | VPN | ⬜ |
| Nextcloud | 192.168.0.X | 512MB | File storage | OK |
| Splunk | 192.168.0.X | 512MB | SIEM | ⬜ |
| Crowdsec | 192.168.0.X | 256MB | Threat blocking | ⬜ |
| Uptime Kuma | 192.168.0.X | 100MB | Uptime monitor | ⬜ |
| Heimdall | 192.168.0.X | 64MB | Dashboard | ⬜ |
| Nginx PM | 192.168.0.X | 128MB | Reverse proxy | ⬜ |
| Grafana | 192.168.0.X | 512MB | Monitoring | ⬜ |

---

## Lab VMs


| VM | IP | RAM | Purpose | Status |
|---|---|---|---|---|
| Windows Server 2022 | 10.10.10.X | 3GB | Domain controller | ⬜ |
| Windows 10 WS01 | 10.10.10.X | 2GB | Domain workstation | ⬜ |
| Windows 10 WS02 | 10.10.10.X | 2GB | Misconfigured admin | ⬜ |
| Metasploitable 2 | 10.10.10.X | 512MB | Vulnerable Linux | ⬜ |
| DVWA | 10.10.10.X | 512MB | Vulnerable web app | ⬜ |

---

## SSH Hardening


---

## Two Factor Authentication


---

## Fail2ban


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
- [ ] LXC containers deployed
- [ ] Lab VMs deployed
- [ ] SSH hardening
- [ ] 2FA enabled
- [ ] Fail2ban installed
- [ ] WireGuard VPN configured
- [ ] Security assessment completed