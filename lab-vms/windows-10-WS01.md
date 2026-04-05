# Windows 10 WS01

## Overview

Domain-joined workstation simulating a regular employee machine in the
`homelab.local` Active Directory environment. Primary machine for
practicing domain user attacks, credential theft, and post-exploitation
on a standard domain workstation.

---

## VM Details

| Field    | Value          |
|----------|----------------|
| VM ID    | 203            |
| IP       | 10.10.10.11    |
| RAM      | 2GB            |
| Storage  | 40GB           |
| CPU      | 2 cores        |
| Bridge   | vmbr1          |
| OS       | Windows 10 Pro |
| Hostname | WS01           |
| Domain   | homelab.local  |
| Status   | ✅ Running      |

---

## Installation

- Machine: q35
- BIOS: OVMF (UEFI)
- SCSI Controller: VirtIO SCSI
- VirtIO drivers ISO attached as second CD drive during installation
- Installed `virtio-win-guest-tools.exe` post install
- QEMU Guest Agent enabled in Proxmox

---

## Configuration

### Static IP
| Field   | Value         |
|---------|---------------|
| IP      | 10.10.10.11   |
| Subnet  | 255.255.255.0 |
| Gateway | 10.10.10.1    |
| DNS     | 10.10.10.10   |

DNS points to DC01 — required for domain join and AD functionality.

### Domain Join
- Joined to `homelab.local`
- Credentials used: `Administrator` from DC01

### Primary User
- Logged in as `homelab\jsmith`
- Regular domain user — no elevated privileges

---

## Purpose

Standard domain workstation with no intentional misconfigurations.
Used for:
- Practicing attacks against a normal domain user account
- Kerberos ticket requests and enumeration
- BloodHound data collection
- Splunk log generation from normal and malicious activity

---

## Notes

- No misconfigurations — clean machine by design
- WS02 is the misconfigured machine for privilege escalation practice
- Shut down WS01 before starting WS02 due to RAM constraints
- Always start DC01 first before booting WS01

---

## Splunk Integration — Planned

Windows Event logs will be forwarded to Splunk via Universal Forwarder
on port 9997 once configured.