# Proxmox Home Lab Build — Day 1: Server Setup & Network Configuration

## Overview

This project documents the journey of building a **production-grade home cybersecurity lab from scratch**, starting with converting an old i5 laptop into a Proxmox VE server. It's a hands-on, budget-zero approach to infrastructure that teaches real troubleshooting and network design.

**What you'll find here:** Real problems, real solutions, and the kind of knowledge that only comes from spending hours debugging a network that doesn't want to cooperate.

---

## The Goal

Transform two old laptops into a complete offensive security lab:
- **Proxmox VE Server:** Old i5-8365U laptop (477GB SSD, 16GB RAM) → Bare-metal hypervisor
- **Attacker Machine:** Acer Predator Helios Neo 16 running Kali Linux in VMware
- **Lab Network:** Isolated, segmented, and ready for vulnerable VMs and C2 frameworks

This is **Day 1** of an 8-day LinkedIn video series documenting the full build. This README captures the troubleshooting arc so you don't have to learn these lessons twice.

---

## Day 1: The Problems & Solutions

### Problem 1: Accessing Proxmox via SSH Instead of Web UI

**What happened:** Opened terminal, SSH'd into the Proxmox box, got this message:
```
Welcome to Proxmox Data Centre Manager. Please use your web browser
```

**The mistake:** Thought port 22 (SSH) would work. It doesn't.

**The fix:** Access Proxmox at `https://IP:8006` in your browser, not SSH.

---

### Problem 2: Wrong IP Subnet — Machines Couldn't See Each Other

**What happened:** 
- Proxmox was assigned `192.168.X.X:8443` during installation
- Home network was on a different subnet
- Predator and Proxmox were on different subnets → zero connectivity

**The mistake:** Not paying attention to the installer's IP assignment.

**The fix:** Manually edited `/etc/network/interfaces` and set Proxmox to match the home network subnet.

```bash
# On Proxmox, edit:
nano /etc/network/interfaces

# Changed to:
iface eno1 inet static
    address 192.168.X.X
    netmask 255.255.255.0
    gateway 192.168.X.1
    dns-nameservers 8.8.8.8
```

---

### Problem 3: WiFi Card Can't Be Bridged (The Hard Stop)

**What happened:** Proxmox was trying to bridge through `eno1` (LAN port), but there was no cable plugged in. Tried switching the bridge to `wlo1` (WiFi) → total failure.

**Why it failed:** 802.11 WiFi **cannot be bridged at the hardware level**. It's a protocol limitation, not a configuration issue. Proxmox doesn't load WiFi drivers either—it's a server OS. It expects a LAN cable.

**Command that showed the problem:**
```bash
ip a show wlo1
# Result: state DOWN NO-CARRIER
```

**The lesson:** You can't force WiFi to work as a bridge. Don't waste time trying.

---

### Problem 4: Router AP/Client Isolation — Silent Death

**What happened:** Both laptops on the same WiFi, but couldn't ping each other. 100% packet loss both ways. No error message. No hint of why.

**Root cause:** Router had **AP/Client isolation** enabled. It silently blocks devices on the same WiFi from communicating with each other.

**Why this matters:** If you can't access router admin settings, you're stuck. This is a real-world network security feature turned into your own obstacle.

**The lesson:** Check router settings first. AP isolation exists for a reason (guest networks), but it will wreck your lab if enabled.

---

### Problem 5: Wrong Proxmox Product (VE vs Backup Server)

**What happened:** After fixing everything, I successfully connected to the dashboard—but it was the *wrong* dashboard. Realized I'd installed **Proxmox Backup Server** instead of **Proxmox VE**.

**How I caught it:**
```bash
pveversion
# Command not found — dead giveaway
```

**The fix:** Downloaded the correct ISO (`proxmox-ve_9.1-1.iso` from proxmox.com), flashed a new USB with Rufus, and did a clean reinstall.

**The lesson:** These are completely different products. One is a hypervisor, one is a backup appliance. Read the download page carefully.

---

## The Solution That Worked: Direct LAN via Docking Station

After hours of troubleshooting, the simplest fix was the winner:

**Hardware setup:**
- Used the Acer Predator's docking station (includes Realtek USB Ethernet adapter)
- Connected a LAN cable directly: Predator docking station → Old laptop's LAN port
- This created a **completely isolated network segment** between the two machines

**Network configuration:**

On the **Predator (Attacker Machine):**
```
Ethernet Adapter:
  IP Address: 10.0.X.1
  Subnet Mask: 255.255.255.0
  DNS: 8.8.8.8
```

On the **Proxmox Server** (via `/etc/network/interfaces`):
```
nic0:
  IP: 10.0.X.2/24
  Gateway: 10.0.X.1
  DNS: 8.8.8.8
```

**Verification:**
```bash
ping 10.0.X.2
# PING 10.0.X.2 (10.0.X.2) 56(84) bytes of data.
# 64 bytes from 10.0.X.2: icmp_seq=1 ttl=64 time=0.234 ms
# 0% packet loss ✓
```

**Access Proxmox:**
```
https://192.168.X.X:8006
```

Dashboard loads. **Node: Aceking**. Storage ready. All green.

---

## Final State — Day 1 Complete

✅ **Proxmox VE 9.1.1** running on old i5 laptop  
✅ **Direct isolated LAN** between Predator and lab server (10.0.X.0/24)  
✅ **Storage pools** (local + local-lvm) ready  
✅ **Zero budget, two old machines, one working lab server**  

**Tomorrow:** First VM deployment (Metasploitable 2).

---

## Key Lessons (So You Don't Learn Them The Hard Way)

1. **Proxmox is a server OS.** It doesn't load WiFi drivers. It expects a LAN cable. Period.
2. **WiFi bridging doesn't work.** 802.11 has hardware limitations. Don't fight it.
3. **Router AP isolation is invisible.** Check your router settings before blaming your config.
4. **Proxmox VE ≠ Proxmox Backup Server.** Two completely different products. Download carefully.
5. **A docking station LAN cable solved what hours of config couldn't.** Sometimes the answer is hardware, not software.
6. **Direct LAN connections are your friend.** No WiFi interference, no router isolation, no surprises.

---

## What This Means for the Community

This README exists because troubleshooting is a skill, and documentation of failures is as valuable as documentation of wins. If you're building a home lab:

- You'll hit these problems. When you do, this guide will save you hours.
- You'll find your own creative solutions. When you do, share them. That's how we all get better.
- You don't need expensive hardware or enterprise infrastructure. You need patience, curiosity, and willingness to learn from mistakes.

**This is a budget-zero lab built from scratch.** It proves that cybersecurity infrastructure doesn't require a big budget—it requires real problem-solving.

---

## Next Steps

- **Day 2:** Deploy first VM (Metasploitable 2)
- **Day 3–8:** Build out AD environment, C2 framework, vulnerable apps
- **Entire series:** Posted on LinkedIn as an 8-day video journey

---

## Tech Stack

- **Proxmox VE 9.1.1** — Bare-metal hypervisor
- **Kali Linux** — Attacker machine (VMware on Predator)
- **Old i5-8365U laptop** — Proxmox server (16GB RAM, 477GB SSD)
- **Acer Predator Helios Neo 16** — Primary attacking platform
- **Direct LAN cable** — The MVP of this entire setup

---

## Contributions Welcome

If you've built a home lab, hit these same walls, or found a better way—open an issue or PR. Let's build this together and make the next person's journey easier.

---

**Start date:** Day 1 of 8-day series  
**Status:** ✅ Proxmox server online  
**Budget:** $0 (laptops repurposed, docking station already owned)  
**Patience required:** High  
**Worth it:** Absolutely
