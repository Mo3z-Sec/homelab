# Windows Server 2022 — Domain Controller

## Overview

Active Directory domain controller for the isolated lab environment
on vmbr1. Acts as the central authentication and DNS server for the
`homelab.local` domain. All Windows workstations are joined to this
domain. Primary target for AD-based attack simulation.

---

## VM Details

| Field    | Value               |
|----------|---------------------|
| VM ID    | 202                 |
| IP       | 10.10.10.10         |
| RAM      | 3GB                 |
| Storage  | 60GB                |
| CPU      | 2 cores             |
| Bridge   | vmbr1               |
| OS       | Windows Server 2022 |
| Hostname | DC01                |
| Domain   | homelab.local       |
| Status   | ✅ Running           |

---

## Installation

### VM Creation
- Machine: q35
- BIOS: OVMF (UEFI)
- SCSI Controller: VirtIO SCSI
- VirtIO drivers ISO attached as second CD drive during installation
  for hard disk detection

### VirtIO Driver Installation
During Windows setup — no drives visible by default:
1. Click **Load driver** → **Browse**
2. Navigate to VirtIO CD  `vioscsi`  `2k22`
3. Select driver  **Next**
4. Disk appears — select and continue installation

### Post Install
- Installed `virtio-win-guest-tools.exe` from VirtIO CD
- Enabled QEMU Guest Agent in Proxmox VM options
- Set static IP `10.10.10.10/24`
- Renamed computer to `DC01`

---

## Active Directory Setup

### Installation
- Added **Active Directory Domain Services** role via Server Manager
- Promoted server to domain controller
- Created new forest: `homelab.local`
- Forest/Domain functional level: Windows Server 2016
- DNS server enabled during promotion

### Domain Users

| Username   | Full Name    | Role                        |
|------------|-------------|------------------------------|
| jsmith     | John Smith  | Regular domain user          | 
| madmin     | Mike Admin  | Domain Admin                 |      |
| sqlservice | SQL Service | Service account (Kerberoast) 

### Service Principal Name
Added SPN to sqlservice for Kerberoasting practice:

```cmd
setspn -a MSSQLSvc/dc01.homelab.local:1433 homelab\sqlservice
```

### Organisational Units
| OU | Purpose |
|----|---------|
| Workstations | Domain joined machines |
| Users | Domain user accounts |

---

## Network

| Setting | Value |
|---------|-------|
| IP | 10.10.10.10 |
| Subnet | 255.255.255.0 (/24) |
| Gateway | 10.10.10.1 |
| DNS | 10.10.10.10 (itself) |

DC is its own DNS server — required for AD to function. All domain
joined machines use `10.10.10.10` as their DNS server.

---

## Planned Attacks

| Attack | Description |
|--------|-------------|
| Kerberoasting | Request TGS for sqlservice, crack offline |
| AS-REP Roasting | Target accounts with pre-auth disabled |
| Pass the Hash | Steal NTLM hash, authenticate without password |
| BloodHound | Enumerate AD, find privilege escalation paths |
| DCSync | Simulate domain replication to dump hashes |

---

## Splunk Integration — Planned

Windows Event logs will be forwarded to Splunk via Universal Forwarder
on port 9997 once the forwarder is installed and configured.
Key event IDs to monitor:

| Event ID | Description |
|----------|-------------|
| 4624 | Successful logon |
| 4625 | Failed logon |
| 4768 | Kerberos TGT request |
| 4769 | Kerberos service ticket request (Kerberoasting) |
| 4771 | Kerberos pre-auth failed |
| 4776 | NTLM authentication |

---

## Notes

- Passwords are intentionally weak for attack simulation
- Never expose this VM to vmbr0 or the internet
- VM has no internet access by design — vmbr1 is fully isolated
- All tools and software transferred via Proxmox before isolation