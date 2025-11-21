# Admin Jump Box Guide

This will guide through the set-up of an Admin VM used to access the MGMT network.

---

# 1. Overview



This document describes how to deploy an Ubuntu Server jump box on **VLAN 20** and use it to securely reach resources on **VLAN 10**, when working from **VLAN 30**.

This is required because:
- VLAN 10 is a secure management VLAN (ESXi host, iDRAC, switches, routers).
- VLAN 30 cannot directly access VLAN 10.
- Some devices in VLAN 10 are legacy and only support old SSH ciphers, which modern OpenSSH clients refuse.


This guide covers:
- Jump box installation and configuration
- SSH tunneling (SOCKS proxy + port forwarding)
- Browser access to ESXi/iDRAC via SOCKS proxy
- PuTTY access to old SSH devices using tunnels
- Network flow diagrams

---

# 2. Traffic Flow

![TrafficFlow](/assets/AdminTrafficFlow.png)


---

# Jump Box Deployment (Ubuntu Server)

### Installation Steps
Use **Ubuntu Server ISO** (not minimized).

During installer:

#### Networking:
- Subnet: `10.100.20.0/24`
- Jump Box IP: `10.100.20.10`
- Gateway: `10.100.20.1`
- DNS: `8.8.8.8,1.1.1.1`

#### Software:
- Enable **OpenSSH Server**
- Do **not** install optional snaps
- No GUI required

### Initial Verification After Install

Verify IP:

ip a
ip r


Test VLAN 20 gateway:

ping 10.100.20.1


Test VLAN 10 reachability:

ping 10.100.10.1
ping 10.100.10.10


Update system:

sudo apt update && sudo apt upgrade -y


Confirm SSH:

systemctl status ssh


---

# SSH Tunneling for Accessing VLAN 10 Web Interfaces

workstation is in VLAN 30 and cannot reach ESXi/iDRAC directly.  
Solution: create a **SOCKS5 proxy** via SSH.

### Start the SOCKS tunnel

On your **Linux workstation** (VLAN 30):

ssh -D 1080 user@10.100.20.10


Leave this SSH session open.

### Configure Firefox

Firefox > Settings > Network Settings > **Manual Proxy Configuration**

Set:
- SOCKS Host: `127.0.0.1`
- Port: `1080`
- SOCKS v5
- Enable **Proxy DNS when using SOCKS v5**

### Access VLAN 10 GUI Services

Now visit:
- ESXi → `http://10.100.10.10`
- iDRAC → `http://10.100.10.5`

### Notes
- The jump box simply forwards packets; it does not proxy web traffic.
- Upload/download speeds are nearly identical to direct access.
- SSH encryption overhead is minimal.

---

# PuTTY Access to Legacy SSH Devices (Router & Switch)

Devices:
- Router: `10.100.10.1`
- Switch: `10.100.10.2`

These devices only support old SSH versions/ciphers.  
OpenSSH blocks the connection, so PuTTY must be used.

### Create PuTTY tunnel session (to Jump Box)

1. Open PuTTY  
2. Hostname:

10.100.20.10

3. In left pane →  
**Connection → SSH → Tunnels**

Add:

#### Router tunnel:
- Source port: `2221`
- Destination: `10.100.10.1:22`
- Type: Local
- Click **Add**

#### Switch tunnel:
- Source port: `2222`
- Destination: `10.100.10.2:22`
- Type: Local
- Click **Add**

4. Click **Open** and log into the jump box.  
**Leave this PuTTY window open** (it *is* the tunnel).

---

### Open PuTTY sessions to router and switch

#### Router:
Open another PuTTY instance:

Host: 127.0.0.1
Port: 2221


#### Switch:
Open another PuTTY instance:

Host: 127.0.0.1
Port: 2222


These sessions now route:
**VLAN 30 > SSH Tunnel > Jump Box > VLAN 10 > Device**

---

# Troubleshooting

### SSH: “REMOTE HOST IDENTIFICATION HAS CHANGED”
Caused by reinstalling the jump box.  
Fix on workstation:

ssh-keygen -R 10.100.20.10


### Firefox cannot reach VLAN 10
- SOCKS proxy not set  
- DNS proxy not enabled  
- SSH tunnel not running  
- Using Chromium/Brave instead of Firefox

### PuTTY connection fails
- Did not leave the original PuTTY tunnel window open  
- Wrong port (2221/2222)  
- Wrong destination IP  
- Router/switch SSH disabled or requires weak cipher

### Jump box cannot reach VLAN 10
Run:

ping 10.100.10.x
curl http://10.100.10.x

If unreachable: switch/router firewall / VLAN routing issue.

---
