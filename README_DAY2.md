# Proxmox Home Lab Build — Day 2: First VM Deployment & Active Attack

## Overview

**Day 2 of the 8-day build.** Yesterday, the server came online. Today, it got its first victim.

Metasploitable 2 is now live on the isolated lab network. Kali is scanning it. The attack surface is live.

This README documents the deployment process, the problems that came up, and the exact commands that took the lab from "working server" to "actively running penetration tests."

---

## Day 1 Recap (If You Missed It)

✅ Old i5 laptop wiped and converted to Proxmox VE 9.1.1  
✅ Direct LAN connection between Predator (attacker) and Proxmox server (lab)  
✅ Isolated network segment (10.0.X.0/24) with no external interference  
✅ Proxmox web UI accessible and ready for VMs  

Today builds on that foundation.

---

## Hardware & Network Baseline

**Attacker Machine:**
- Acer Predator Helios Neo 16
- i7-14700HX, RTX 4050, 16GB RAM, 1TB SSD
- Kali Linux (VMware)
- IP: 10.0.X.1

**Lab Server (Proxmox):**
- Old i5-8365U laptop, 16GB RAM, 477GB SSD
- Proxmox VE 9.1.1 (node: Aceking)
- IP: 10.0.X.2

**Connection:**
- Direct LAN cable via Predator docking station
- No WiFi, no router, no interference
- Completely isolated from home network

---

## Day 2: Metasploitable 2 Deployment

### Step 1: Download Metasploitable 2 ISO

```bash
# On Predator (Kali machine)
cd ~/Downloads
wget https://sourceforge.net/projects/metasploitable/files/Metasploitable2/metasploitable-linux-2.0.0.iso
```

Metasploitable 2 is intentionally vulnerable. It's the perfect first target for a new lab.

---

### Step 2: Upload ISO to Proxmox Storage

Accessed Proxmox web UI: `https://10.0.X.2:8006`

**Via web interface:**
1. Logged in with root credentials
2. Navigated to: `Datacenter → Aceking → local (Aceking) → Content`
3. Uploaded ISO via the "Upload" button
4. File appeared in storage as `metasploitable-linux-2.0.0.iso`

Alternatively, upload via CLI:

```bash
# SSH into Proxmox from Kali
ssh root@10.0.X.2

# Navigate to ISO storage
cd /var/lib/vz/template/iso

# Upload ISO (if not using web UI)
scp ~/Downloads/metasploitable-linux-2.0.0.iso root@10.0.X.2:/var/lib/vz/template/iso/
```

---

### Step 3: Create Metasploitable 2 VM

**Problem 1: Confused about VM ID and naming**

Proxmox assigns numeric IDs to VMs. First VM should be ID 100 (convention). Named it `metasploitable2` for clarity.

**Via Proxmox Web UI:**
1. Clicked: `Create VM`
2. General tab:
   - Name: `metasploitable2`
   - VM ID: `100`
3. OS tab:
   - ISO image: `metasploitable-linux-2.0.0.iso`
   - Type: Linux
4. System tab:
   - Memory: 2048 MB (2GB)
   - CPU: 2 cores
5. Disk tab:
   - Storage: `local-lvm`
   - Disk size: 20GB
6. Network tab:
   - Bridge: `vmbr0`
   - MAC address: auto-generated
7. Finalized and created

---

### Step 4: Boot the VM & Configure Network

**Clicked: Start**

Metasploitable 2 booted. Kernel loaded. Login prompt appeared.

**Problem 2: VM got wrong IP address via DHCP**

By default, Metasploitable 2 tries DHCP. In an isolated lab with no DHCP server, it failed to get an IP.

**Fix: Manual network configuration on Metasploitable**

Booted into the VM. Logged in:
```
Username: msfadmin
Password: msfadmin
```

Configured static IP on eth0:

```bash
# SSH'd into the VM from Kali and edited network config
ssh msfadmin@<vm-ip>

# Edit interfaces file
sudo nano /etc/network/interfaces

# Changed to:
auto eth0
iface eth0 inet static
    address 10.0.X.50
    netmask 255.255.255.0
    gateway 10.0.X.1
    dns-nameservers 8.8.8.8
```

Restarted networking:
```bash
sudo /etc/init.d/networking restart
```

Verified connectivity:
```bash
ping 10.0.X.1
# Response: 0% packet loss ✓
```

---

### Step 5: First Nmap Scan

Back on Kali, ran the first real attack of the lab:

```bash
# Ping to confirm Metasploitable is alive
ping 10.0.X.50

# Basic nmap scan
nmap -sV 10.0.X.50
```

**Output:**
```
Starting Nmap 7.x at [timestamp]
Nmap scan report for 10.0.X.50
Host is up (0.00234s latency).
Not shown: 994 closed ports
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
22/tcp  open  ssh         OpenSSH 4.7p1
23/tcp  open  telnet      Linux telnetd
25/tcp  open  smtp        Postfix smtpd
53/tcp  open  domain      ISC BIND 9.4.2
3306/tcp open mysql       MySQL 5.0.51b-3~bpo.1-log
...
```

Multiple vulnerable services. This is exactly what a lab should look like.

---

## Current Lab Architecture

```
┌─────────────────────────────────────────────────────┐
│           Attacker Machine (Predator)               │
│     Kali Linux (VMware) — 10.0.X.1                  │
│                                                     │
│  - nmap, metasploit, burp, custom scripts           │
│  - Connected via Docking Station Ethernet           │
└─────────────────┬───────────────────────────────────┘
                  │
                  │ Direct LAN Cable
                  │ (Isolated Network: 10.0.X.0/24)
                  │
┌─────────────────┴───────────────────────────────────┐
│         Lab Server (Proxmox VE 9.1.1)               │
│              Node: Aceking                          │
│          10.0.X.2 — hypervisor                      │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │  VM 100: Metasploitable 2 — 10.0.X.50         │  │
│  │  (Vulnerable Linux target for testing)         │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  (Storage: local + local-lvm ready for more VMs)    │
└─────────────────────────────────────────────────────┘
```

---

## Attack Timeline (First 2 Hours)

| Time | Action | Result |
|------|--------|--------|
| 14:00 | Metasploitable 2 ISO uploaded to Proxmox | ✓ 611MB in storage |
| 14:15 | VM created (ID 100, 2GB RAM, 20GB disk) | ✓ VM started |
| 14:30 | Booted into Metasploitable, logged in | ✓ msfadmin/msfadmin |
| 14:45 | Configured static IP (10.0.X.50) | ✓ Network up |
| 15:00 | First ping from Kali → Metasploitable | ✓ 0% loss |
| 15:05 | Ran nmap -sV against target | ✓ 6 vulnerable services found |

---

## Problems Encountered & Fixes

### Problem 1: DHCP Failed on First Boot

**Symptom:** Metasploitable got no IP. Couldn't reach it from Kali.

**Root cause:** Isolated lab has no DHCP server.

**Fix:** Static IP assignment (10.0.X.50) on eth0. Simple, permanent.

---

### Problem 2: SSH Connectivity Flaky on First Attempt

**Symptom:** SSH timeout when trying to connect to Metasploitable from Kali.

**Root cause:** VM boot still in progress. Didn't wait long enough.

**Fix:** Waited 30 seconds after login prompt appeared, then SSH worked.

---

### Problem 3: Forgot Metasploitable Default Credentials

**Symptom:** Tried random logins. Got locked out of TTY.

**Root cause:** Brain fart.

**Fix:** Default credentials are `msfadmin` / `msfadmin`. All vulnerability in one place.

---

## Current Lab State — Day 2 Complete

✅ **Proxmox VE 9.1.1** running — node Aceking stable  
✅ **Metasploitable 2 VM 100** deployed, booted, networked  
✅ **Isolated network segment** (10.0.X.0/24) with Predator + Proxmox + Metasploitable  
✅ **First nmap scan** completed — 6 vulnerable services identified  
✅ **Attack surface live** — ready for exploitation  

**VM Stats:**
- 2 GB RAM allocated, ~400MB in use
- 20GB disk allocated, ~2GB in use
- Network: stable, 0% packet loss
- Services: FTP, SSH, Telnet, SMTP, DNS, MySQL (all vulnerable)

---

## What's Next (Day 3–8)

**Day 3:** Deploy Windows Server VM + Active Directory  
**Day 4:** Domain join Metasploitable, add users  
**Day 5:** Set up C2 framework (Sliver / Empire)  
**Day 6:** Configure vulnerable web app (DVWA / WebGoat)  
**Day 7:** Integrate everything into one attack scenario  
**Day 8:** Full end-to-end penetration test walkthrough  

---

## Key Lessons from Day 2

1. **Metasploitable 2 is perfect for a first VM.** It boots fast, asks nothing, and bleeds vulnerabilities.
2. **DHCP doesn't exist in an isolated lab.** Plan for static IPs from the start.
3. **Document your VM IDs and IP scheme.** You'll have 10+ VMs eventually. Make them findable.
4. **Proxmox web UI is good enough.** You don't need CLI ninja skills to manage basic VMs.
5. **First successful nmap scan hits different.** This is where the lab stops being a project and starts being real.

---

## Tech Stack (Current)

- **Proxmox VE 9.1.1** — Hypervisor
- **Metasploitable 2** — Vulnerable Linux target (FTP, SSH, Telnet, SMTP, DNS, MySQL)
- **Kali Linux** — Attacker machine (nmap, metasploit, burp, custom tools)
- **Direct LAN connection** — Isolated from home network
- **Static IPs** — No DHCP, no surprises

---

## Files in This Repo

- `README.md` — This document (Day 2 progress)
- `Day1-ServerSetup.md` — Previous day's work (Proxmox installation & troubleshooting)
- `network-diagram.txt` — ASCII network architecture
- `attack-timeline.log` — Timestamped attack progression

---

## Community

If you're building a home lab:
- Hit problems? Check the troubleshooting sections.
- Found a better way? Open a PR or issue.
- Deploying your first VM? This README is for you.

This lab proves you don't need enterprise hardware to learn real skills.

---

**Day 2 Status:** ✅ Complete  
**Lab State:** 🟢 Online & attacking  
**Budget spent:** $0  
**Time invested:** 2 hours (Day 2), 2 hours (Day 1) = 4 hours total  
**Vulnerabilities found:** 6  
**Exploits run:** 0 (coming Day 3)  

---

*Day 2 of 8. 6 days left to build a complete penetration testing lab from scratch.*
