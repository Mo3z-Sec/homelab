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

---

## 2026-03-25 — Uptime Kuma showing WireGuard as down

**Symptom**
Added WireGuard to Uptime Kuma and it immediately showed down.
Checked the container — it was running fine and the tunnel was
active on my MacBook.

**Cause**
I set the monitor to TCP port 51820 without thinking. WireGuard
runs on UDP, not TCP. Uptime Kuma was never actually reaching
anything — a TCP probe on a UDP port always fails.

**Fix**
Switched to monitoring SSH port 22 on 192.168.0.53 instead.
The container runs SSH so port 22 being open is a reliable enough
sign that WireGuard is up.

**Lesson Learned**
Know your protocols before setting up monitors. UDP services
cannot be checked with TCP monitors — always check what protocol
a service runs on first.

---

## 2026-04-01 — Insufficient RAM to deploy WS02

**Symptom**
Only ~1.6GB free after running Windows Server, WS01, Splunk,
and all LXC containers. WS02 needs 2GB — not enough headroom.

**Cause**
8GB RAM is not enough for the full lab stack simultaneously.
Splunk (4GB) + Windows Server (3GB) + WS01 (2GB) = 9GB before
containers.

**Workaround**
WS02 postponed. Running VMs one at a time depending on what
I'm practicing.

**Planned fix**
Second 8GB DDR4 stick — brings total to 16GB and solves the
problem permanently.

**Lesson Learned**
16GB should be the minimum RAM target for a homelab running
a SIEM, multiple Windows VMs, and service containers together.

---
## 2026-04-04 — Metasploitable 2 got no IP on vmbr1

**Symptom**
After booting Metasploitable 2 it had no IP address and was
unreachable from anywhere.

**Cause**
vmbr1 has no DHCP server. Metasploitable tried DHCP on boot,
got no response, and ended up with nothing.

**Fix**
Struggled with this for a while then found the fix on Reddit —
someone had the exact same issue with an isolated lab network.
Manually set a static IP in /etc/network/interfaces:
```
auto eth0
iface eth0 inet static
address 10.10.10.20
netmask 255.255.255.0
gateway 10.10.10.1
```
```bash
sudo /etc/init.d/networking restart
```

**Lesson Learned**
Every VM on vmbr1 needs a static IP set manually on first boot.
There is no DHCP — nothing will assign an address automatically.

---

## 2026-04-05 — Kali could not reach vmbr1 lab network

**Symptom**
Kali had no way to reach anything on vmbr1. Metasploitable,
Windows Server, and WS01 were all unreachable. Nmap and ping
returned nothing.

**Cause**
vmbr1 is isolated with no physical port and no IP on the
Proxmox host. No network path existed between Kali on
192.168.0.x and the lab VMs on 10.10.10.x.

**What I tried first — SSH**
Thought about using SSH as a jump host into vmbr1. The problem
is SSH gives a shell, not actual network access. Tools like
Metasploit and nmap need to send traffic directly to targets —
SSH tunneling doesn't support that cleanly.

**Fix**
Gave Proxmox an IP on vmbr1 so it acts as a router:
```bash
ip addr add 10.10.10.1/24 dev vmbr1
ip link set vmbr1 up
```

Made permanent in /etc/network/interfaces.

Enabled IP forwarding on Proxmox:
```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

Disabled ICMP redirects — without this Proxmox was telling
Kali to use the home router which killed the packets:
```bash
sysctl -w net.ipv4.conf.all.send_redirects=0
sysctl -w net.ipv4.conf.default.send_redirects=0
sysctl -w net.ipv4.conf.vmbr0.send_redirects=0
```

Added static route on Kali made permanent in
/etc/network/interfaces:
```bash
sudo ip route add 10.10.10.0/24 via 192.168.0.169
```

**Why static route over SSH tunneling**
Static routing gives Kali real layer 3 access to vmbr1 — every
tool works natively. SSH tunneling requires wrapping each
tool individually which doesn't work well with Metasploit.
Static routing is also how real enterprise networks handle
inter-VLAN access — through a proper router, not workarounds.

**How OPNsense will improve this**
Proxmox is currently acting as a basic unmanaged router with
no visibility into cross-network traffic. OPNsense will take
over this routing properly — with firewall rules, logging, and
full control over what Kali can reach on vmbr1.

**Lesson Learned**
Isolated networks need a layer 3 path to be reachable. Always
assign an IP to the bridge interface on the hypervisor if you
need to route traffic to isolated VMs.

---

