# Issues Log

A log of every problem encountered during the homelab
build and how it was resolved.

---

## 2026-03-15 — Installer stuck at mounting sys filesystems

**Symptom**
Both graphical and terminal Proxmox installer froze
completely at "mounting sys filesystems" and never
continued.

**Cause**
GPU driver conflict during boot causing kernel hang.

**Fix**
In the Proxmox boot menu pressed E to edit boot
parameters. Found the line ending in:
```
splash=silent proxtui
```
Added nomodeset at the end:
```
splash=silent proxtui nomodeset
```
Pressed Ctrl+X to boot. Installer completed successfully.

* key takeway: "nomodeset" is used to skip graphic drivers which often causes freezes or crashes on older GPU drivers.

---

## 2026-03-15 — apt update 401 Unauthorized errors

**Symptom**
apt update returned 401 Unauthorized errors from
enterprise.proxmox.com even after disabling .list files.

**Cause**
After investigating and researching the internet I have ocncluded that Proxmox VE 9 uses two repo file formats — legacy .list
and newer .sources. Both had enterprise repos that
needed disabling. Initial fix only disabled .list files.

**Fix**
```bash
echo "Enabled: no" >> /etc/apt/sources.list.d/ceph.sources
echo "Enabled: no" >> /etc/apt/sources.list.d/pve-enterprise.sources
apt update && apt full-upgrade -y
```

**Files Disabled**
- pve-enterprise.list
- ceph.list
- pve-enterprise.sources
- ceph.sources
