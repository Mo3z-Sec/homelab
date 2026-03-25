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

---

## 2026-03-22 — Power outage caused Proxmox to restart and all containers shut down

**Symptom**
Power went out in the house causing the Proxmox server
to shut down unexpectedly. When power was restored and
the PC restarted, all LXC containers remained offline
and did not start automatically. Pi-hole and Nextcloud
were both unreachable after the power cut.

**Cause**
Proxmox containers do not auto start after a reboot
by default. The Start at boot option is disabled on
all containers unless explicitly enabled. Without this
setting every container requires manual starting after
any power loss or reboot.

**Fix — Enable Start at Boot for All Containers**
For each container in Proxmox web UI:
- Click container in left panel
- Click Options
- Double click Start at boot
- Set to Yes
- Click OK

Applied to:
- 100 (pihole)    | Start at boot: Yes
- 101 (nextcloud)  | Start at boot: Yes

Will apply to every future container during creation.

**Fix — Set Startup Order**
Configured startup order so Pi-hole starts before
Nextcloud ensuring DNS is ready before other services:

Container 100 (pihole):
- Order: 1
- Up delay: 30 seconds

Container 101 (nextcloud):
- Order: 2
- Up delay: 30 seconds

**Fix — BIOS Power Recovery**
Entered BIOS on Proxmox PC and set AC Power Recovery
to Power On so the PC automatically powers on when
electricity is restored after a power cut.

**Auto Recovery Chain After This Fix**
1. Power restored
2. PC powers on automatically via BIOS setting
3. Proxmox boots up
4. Pi-hole starts first (order 1)
5. 30 second delay
6. Nextcloud starts (order 2)
7. All services running without any manual action

**Lesson Learned**
Always enable Start at boot on every container
immediately after creating it. Always configure
startup order so dependency services like Pi-hole
start before services that depend on DNS.
Always configure BIOS power recovery on any machine
running as a server so it survives power cuts
automatically. A proper server should require zero
manual intervention after a power outage.

---

## 2026-03-23 — Nextcloud container disk filled to 100%

**Symptom**
Nextcloud showing Internal Server Error 507 storage
quota and 412 precondition failed. Apache error log
showing No space left on device. Nextcloud completely
inaccessible from browser.

**Cause**
Phone auto upload was writing photos directly to the
Nextcloud data directory inside the container disk.
The container disk was only 8GB and filled up completely
with 4.9GB of photos leaving only 1.8MB free.

Container disk usage at time of failure:
- Total: 7.8GB
- Used: 7.4GB
- Available: 1.8MB
- Use: 100%

**Immediate Fix — Free Emergency Space**
```bash
apt clean
truncate -s 0 /var/log/apache2/access.log
truncate -s 0 /var/log/apache2/error.log
truncate -s 0 /var/log/apache2/other_vhosts_access.log
```

**Fix 1 — Expand Container Disk**
In Proxmox web UI:
- Click 101 (nextcloud) → Resources
- Double clicked Hard Disk
- Expanded from 8GB to 20GB
- Ran resize2fs to expand filesystem:
```bash
resize2fs /dev/mapper/pve-vm--101--disk--0
```

**Fix 2 — Move Data Directory**
Moved Nextcloud data directory outside the container
disk to prevent this from happening again:
```bash
systemctl stop apache2
mkdir -p /mnt/ncdata
chown -R www-data:www-data /mnt/ncdata
mv /var/www/html/nextcloud/data /mnt/ncdata/
chown -R www-data:www-data /mnt/ncdata/data
```

Updated config.php:
```php
'datadirectory' => '/mnt/ncdata/data',
```

Started Apache again:
```bash
systemctl start apache2
```

**Result**
Nextcloud fully restored. Data directory now sits
outside the container disk. Future uploads go to
the larger storage pool and will not fill the
container disk.

**Lesson Learned**
Always move Nextcloud data directory outside the
container disk before enabling auto upload on any
device. Container disk should only hold the
application files. User data should always go to
a dedicated external mount on the larger storage pool.
Monitor container disk usage regularly in Proxmox
Summary tab and never let it exceed 80%.

---

## 2026-03-24 — WireGuard failed to start due to iptables not installed
 
**Symptom**
After fixing the config formatting, `systemctl start wg-quick@wg0`
failed again with a different error:
```
/usr/bin/wg-quick: line 295: iptables: command not found
```
 
**Cause**
The PostUp rules in `wg0.conf` call iptables to set up
NAT and forwarding when the tunnel starts. iptables was
not installed on the fresh Debian 12 LXC container —
it is not included by default.
 
**Fix**
```bash
apt install iptables -y
systemctl start wg-quick@wg0
```
 
WireGuard started successfully after installation.
 
**Lesson Learned**
Always install iptables explicitly on fresh Debian 12
containers before using WireGuard with PostUp/PostDown
NAT rules. Do not assume it is pre-installed.

---

## 2026-03-25 — ISP blocking inbound ports 80 and 443

**Symptom**
Nginx Proxy Manager was correctly configured and running. Port forwards
for ports 80 and 443 were set up on the router. However all attempts
to reach services externally via mo3z-homelab.duckdns.org failed with
no response. SSL certificate requests via HTTP challenge also failed
with an internal error.

**Cause**
Saudi Arabia ISP blocks inbound ports 80 and 443 on residential connections
at the ISP level. This is common practice — ISPs reserve web hosting
ports for business lines only. The block happens upstream before
traffic even reaches the home router, so port forwarding rules have
no effect.

**Confirmed via**
Tested port 80 and 443 using NMAP from an external mobile
device — both showed closed. WireGuard on port 51820 UDP
confirmed open and working on the same test, proving the router and
port forwarding are correctly configured.

**Planned Fix**
Cloudflare Tunnel — establishes an outbound connection from the
homelab to Cloudflare's edge servers. Since the connection is
outbound the ISP block is completely bypassed with no port forwarding
required.

**Lesson Learned**
Always check if your ISP blocks common ports before building
infrastructure that depends on them. Residential ISPs frequently
block ports 80, 443, and 25. Test early using an external port
checker before investing time in configuration.