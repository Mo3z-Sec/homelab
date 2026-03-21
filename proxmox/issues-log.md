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

---

## 2026-03-17 — Firewall locked out all access including web UI

**Symptom**
After enabling the Proxmox firewall and adding rules,
the web UI became completely inaccessible from all devices on the network.

**Cause**
The DROP rule was placed above the ACCEPT rules in the
firewall rule list. Proxmox processes rules top to bottom
and the DROP rule matched all traffic before the ACCEPT
rules ever got checked. This blocked everything including
legitimate access from the home network.

**Incorrect rule order:**
```
1. DROP    any   any             any         
2. ACCEPT  tcp   192.168.x.x/24  port 8006
3. ACCEPT  tcp   192.168.x.x/24  port 22
4. ACCEPT  udp   any             port 51820
```

**Correct rule order:**
```
1. ACCEPT  tcp   192.168.x.x/24  port 8006  
2. ACCEPT  tcp   192.168.x.x/24  port 22
3. ACCEPT  udp   any             port 51820
4. DROP    any   any             any         
```

**First attempted fix — SSH from Kali laptop**
Attempted to SSH into Proxmox from Kali Linux laptop
to fix the firewall rules remotely:
```bash
ssh root@192.168.x.x
```
This did not work because the DROP rule was also blocking
port 22 SSH connections before the ACCEPT rule for SSH
could be reached.

**Actual fix — physical access**
Plugged a monitor and keyboard directly into the Proxmox
PC and logged in as root at the terminal. Edited the
firewall config file directly:
```bash
nano /etc/pve/firewall/cluster.fw
```
Changed enable: 1 to enable: 0 to disable the firewall.
Then restarted the firewall service:
```bash
systemctl restart pve-firewall
```
Web UI became accessible again. Fixed rule order in the
web UI, moved DROP rule to the bottom, then re-enabled
the firewall.

**Lesson learned**
Always add all ACCEPT rules before the DROP rule.
Firewall rules are processed top to bottom — order is
critical. When setting up a firewall remotely always
have a physical access plan in case you lock yourself out.
A monitor and keyboard connected to the server is your
emergency access method.