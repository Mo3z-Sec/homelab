# Firewall

## Overview

Proxmox firewall configured at both datacenter and node level.
Rules are processed top to bottom — order is critical.

---

## Levels Enabled

- Datacenter firewall: enabled
- Node (pve) firewall: enabled

---

## Rules

| Priority | Direction | Action | Protocol | Source         | Port  | Purpose                 |
|----------|-----------|--------|----------|----------------|-------|-------------------------|
| 0        | in        | ACCEPT | TCP      | 192.168.0.X/24 | 8006  | Web UI                  |
| 1        | in        | ACCEPT | TCP      | 192.168.0.X/24 | 22    | SSH                     |
| 2        | in        | ACCEPT | UDP      | any            | 51820 | WireGuard               |
| 3        | in        | ACCEPT | ICMP     | 192.168.0.X/24 | any   | Home network ping       |
| 4        | in        | DROP   | any      | 10.10.10.X/24  | any   | Block lab VMs from host |
| 5        | in        | DROP   | any      | any            | any   | Block all other inbound |

> vmbr1 has no bridge port so lab VMs have no physical path to the
> router regardless of firewall rules. Rule 4 is defence-in-depth
> in case of misconfiguration.

See [issues-log.md](issues-log.md) for what happens when rule order is wrong.