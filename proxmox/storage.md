# Storage Configuration

## Overview
Proxmox installed on a single 931GB HDD. Proxmox
automatically partitioned the disk and created an LVM
storage pool for VM and container disks. No separate
SSD — OS and data share the same physical disk.

---

## Physical Disk
| Disk | Size    | Type |
|------|---------|------|
| sda  | 931.5GB | HDD  |

---

## Partition Layout
| Partition | Size    | Purpose             |
|-----------|---------|---------------------|
| sda1      | 1007KB  | BIOS boot partition |
| sda2      | 1GB     | EFI / boot          |
| sda3      | 930.5GB | LVM physical volume |

---

## LVM Volumes
| Volume   | Size  | Purpose                 |
|----------|-------|-------------------------|
| pve-swap | 7.7GB | Swap space              |
| pve-root | 96GB  | Proxmox OS              |
| pve-data | 794GB | VM disks and containers |

---

## Proxmox Storage Pools
| Name      | Type     | Purpose                     |
|-----------|----------|-----------------------------|
| local     | dir      | ISOs, backups, CT templates |
| local-lvm | lvm-thin | VM disks, container disks   |

---

## Container Disk Allocation
| Container | CT ID | Disk Size | Purpose       |
|-----------|-------|-----------|---------------|
| pihole    | 100   | 4GB       | Pi-hole app   |
| nextcloud | 101   | 20GB      | Nextcloud app |

---

## Nextcloud Data Directory
Nextcloud uploaded files and photos are stored at:
```
/mnt/ncdata/data
```
This keeps user data separate from the application
disk so the container disk never fills up from uploads.

Config.php setting:
```php
'datadirectory' => '/mnt/ncdata/data',
```

---

## Planned Storage Allocation
| Purpose              | Planned Size |
|----------------------|--------------|
| VM disks + snapshots | ~200GB       |
| LXC containers       | ~50GB        |
| Nextcloud data       | ~300GB       |
| Backups              | ~150GB       |
| Free buffer          | ~94GB        |

---

## Important Notes
- Never fill container disk above 80%
- Monitor disk usage in Proxmox Summary tab
- All containers have Start at boot enabled
- Nextcloud data directory points to external mount
  not inside the container disk