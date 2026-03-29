# Homelab

A self-hosted homelab built on Proxmox VE for learning
penetration testing and SIEM, also running privacy-focused personal
services. Built from scratch on a budget Dell desktop PC.

---

## Hardware

| Machine              | Role               | OS              |
|----------------------|--------------------|-----------------|
| Dell i5-6500 PC   | Proxmox server     | Proxmox VE 9.1  |
| Kali Linux laptop    | Attack machine     | Kali Linux      |
| MacBook              | Management and ops | macOS           |


---

## Infrastructure

- **Hypervisor:** Proxmox VE 9.1
- **Storage:** 931GB HDD (794GB allocated to VM/container storage via `pve-data` LV)
- **Network:** Two virtual bridges — `vmbr0` for services, `vmbr1` for isolated lab

---

## Network Diagram

![Network Diagram](assets/homelab-network.png)

---

## Services

| Service         | Purpose                     |
|-----------------|-----------------------------|
| Pi-hole         | Network-wide DNS ad blocker |
| Nextcloud       | Self-hosted cloud storage   |
| WireGuard       | VPN for remote access       |
| CrowdSec        | Active threat blocking      |
| Uptime Kuma     | Service uptime monitoring   |
| Heimdall        | Services dashboard          |
| Nginx Proxy Mgr | Reverse proxy and SSL       |
| Splunk          | SIEM                        |
| Grafana         | Visual dashboards           |
| DuckDNS         | Dynamic DNS / domain        |
| Cowrie          | SSH Honeypot                |
| OPNsense        | Firewall, IDS/IPS           |

---

## Lab Environment

| Machine             | Purpose                          |
|---------------------|----------------------------------|
| Windows Server 2022 | Active Directory domain controller |
| Windows 10 x2       | Domain-joined workstations       |
| Metasploitable 2    | Vulnerable Linux target          |
| DVWA                | Vulnerable web application       |

---

## Skills Demonstrated

- Hypervisor administration (Proxmox KVM and LXC)
- Linux system administration and hardening
- Network design and segmentation
- Self-hosted service deployment
- Security monitoring and firewall configuration
- Technical documentation

## Planned Skills (In Progress)

- Active Directory deployment and attack simulation
- VPN configuration and remote access
- SIEM log ingestion and detection engineering
- Penetration testing and security assessment

---

## Progress

- [x] Proxmox VE 9.1 installed and configured
- [x] Repository hardening completed
- [x] System updated
- [x] Network isolation configured
- [x] Firewall configured
- [x] LXC services deployed
- [ ] Lab VMs deployed
- [x] WireGuard VPN configured
- [ ] SSH hardening
- [ ] 2FA enabled
- [ ] Fail2ban installed
- [ ] Security assessment completed
- [ ] Cowrie honeypot deployed
- [ ] Splunk ingesting honeypot logs

---

## Documentation

- [proxmox/setup.md](proxmox/setup.md) — installation and configuration
- [proxmox/issues-log.md](proxmox/issues-log.md) — problems encountered and fixes
- [proxmox/storage.md](proxmox/storage.md) — storage layout
- [proxmox/firewall.md](proxmox/firewall.md) — firewall rules
- [proxmox/hardening.md](proxmox/hardening.md) — security hardening steps
- [network/network-map.md](network/network-map.md) — full network diagram
- [services/](services/) — individual service documentation
- [lab-vms/](lab-vms/) — lab environment documentation
- [security-assessment/](security-assessment/) — penetration test findings
    