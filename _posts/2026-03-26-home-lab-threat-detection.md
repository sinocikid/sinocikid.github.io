---
title: "Building a Threat Detection Home Lab: From Zero to Hunting in a Weekend"
date: 2026-03-26
author: "Andy"
categories:
- "Blue Team"
- "Lab Setup"
tags:
- "home-lab"
- "detection-engineering"
- "Active-Directory"
- "Atomic-Red-Team"
- "blue-team"
description: "A realistic guide to building a threat detection home lab — hardware, software stack, and how to generate meaningful attack telemetry."
read_time: 11
---

# Building a Threat Detection Home Lab: From Zero to Hunting in a Weekend

> *You don't get good at detecting attacks by reading about attacks. You get good by watching attacks happen in a lab.*

---

## Why a Lab Changes Everything

Reading about Pass-the-Hash is one thing. Watching the exact event IDs appear in your SIEM as Mimikatz runs is another. A home lab bridges the gap between conceptual understanding and muscle memory.

Security engineers who can build a detection are worth more than those who can only tune a detection someone else wrote. Labs are where you learn to build.

This post is a realistic guide — not the fantasy "buy a $10k rack server" version. You can build a functional threat detection lab for $0 in additional hardware if you have a reasonably modern machine, or for a few hundred dollars if you want dedicated hardware.

---

## Hardware Reality Check

**What you actually need:**

A modern laptop or desktop with:
- **16GB RAM minimum** (32GB makes it much more comfortable)
- **SSD with 200-500GB free** for VM storage
- A CPU that supports virtualization (VT-x/AMD-V) — enabled in BIOS

**What works for dedicated hardware:**

- Repurposed workstation (Dell OptiPlex, HP EliteDesk) — available used for $100-200
- An old gaming PC you're not using
- A mini PC (Intel NUC, Beelink, similar) with 32GB RAM — runs ~$300-400 new

**Cloud alternative:**

AWS, Azure, and GCP all offer free tiers and trial credits. You can build an entire lab in the cloud for free or near-free for the first few months. Downside: network egress costs and the lab goes away if you stop paying. Upside: access from anywhere, easy snapshots, no hardware to manage.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                   Host Machine                       │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────┐ │
│  │  DC01    │  │  WIN10   │  │   SIEM / Logging  │ │
│  │  (AD DC) │  │ (victim) │  │   (Security Onion │ │
│  │ Server   │  │ workstat.│  │    or Wazuh+ELK)  │ │
│  └────┬─────┘  └────┬─────┘  └─────────┬─────────┘ │
│       │              │                  │            │
│  ┌────┴──────────────┴──────────────────┴────────┐  │
│  │          Internal Network (NAT/Host-only)     │  │
│  └──────────────────────────────────────────────┘  │
│                                                      │
│  ┌──────────┐                                       │
│  │  Kali /  │  ← Attack VM (isolated or gated)     │
│  │ Parrot   │                                       │
│  └──────────┘                                       │
└─────────────────────────────────────────────────────┘
```

**Minimum viable lab:**
- 1x Windows Server VM (Domain Controller)
- 1x Windows 10/11 VM (victim workstation, domain-joined)
- 1x SIEM/logging VM
- 1x Attack VM (Kali Linux)

**Expanded lab:**
- Additional Windows Server (member server — for lateral movement practice)
- Linux victim VM (web server)
- Firewall VM (pfSense) for network-level detection

---

## Software Stack Options

### Option A: Security Onion (Recommended for Beginners)

Security Onion is a pre-integrated platform that bundles:
- Zeek (network traffic analysis)
- Suricata (IDS/IPS)
- Elastic Stack (log ingestion, SIEM)
- Kibana (visualization)
- TheHive (case management, optional)

**Why to choose it**: Everything is pre-configured. You download an ISO, install it, and it works. No integration headaches. The documentation is excellent.

**Resource requirements**: 8-16GB RAM, 200GB disk for the Security Onion VM. Beefy, but it eliminates hours of configuration work.

```bash
# After installation, enroll endpoints via Elastic Agent or Winlogbeat
# Security Onion has a built-in enrollment guide at https://securityonion:443
```

### Option B: Wazuh + Elastic SIEM (More Flexible)

Wazuh is an open-source SIEM/XDR that integrates with the Elastic Stack. More configuration required than Security Onion but more customizable.

Components:
- **Wazuh Manager**: Correlation engine, rule processing
- **Wazuh Agents**: Installed on endpoints (Windows, Linux)
- **Elastic Stack**: Indexing and visualization
- **Kibana**: Dashboards and alerting

This setup gives you more control over detection rules and is closer to what enterprise deployments look like with commercial SIEMs.

### Option C: Elastic SIEM Only (Lightweight)

If resources are constrained, run just Elastic Stack with Winlogbeat/Filebeat forwarding logs from Windows VMs. No network sensor — endpoint telemetry only.

**Minimum footprint**: 4GB RAM for Elastic + Kibana VM. Runs on 8GB host total with one Windows victim VM.

---

## Building the Active Directory Lab

A Windows Active Directory environment is essential for realistic attack/detection scenarios. Most enterprise attacks target AD.

**Step 1: Install Windows Server**
- Download Windows Server evaluation (180-day trial, renewable) from Microsoft
- Create a VM with 2-4GB RAM, 50GB disk
- Install as Server Core or Desktop Experience

**Step 2: Promote to Domain Controller**
```powershell
# Install AD Domain Services
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# Create the forest
Install-ADDSForest `
    -DomainName "lab.local" `
    -DomainNetBiosName "LAB" `
    -InstallDns `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "Lab@123456" -AsPlainText -Force)
```

**Step 3: Create a realistic user environment**
```powershell
# Create OUs
New-ADOrganizationalUnit -Name "Workstations" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Servers" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Users" -Path "DC=lab,DC=local"

# Create users
New-ADUser -Name "John Smith" -SamAccountName jsmith -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) -Enabled $true -Path "OU=Users,DC=lab,DC=local"
New-ADUser -Name "svc-backup" -SamAccountName svc-backup -AccountPassword (ConvertTo-SecureString "Backup@Service1" -AsPlainText -Force) -Enabled $true -Path "OU=Users,DC=lab,DC=local"

# Add a privileged user for attack scenarios
Add-ADGroupMember -Identity "Domain Admins" -Members jsmith
```

**Step 4: Join Windows 10/11 VM to domain**
```powershell
# On the workstation VM:
Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart
```

---

## Logging Configuration

Before you can detect anything, configure logging properly. Apply these to all Windows VMs via GPO:

**Audit Policy:**
```powershell
# Set minimum useful audit policy via auditpol
auditpol /set /subcategory:"Process Creation" /success:enable
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Account Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Security Group Management" /success:enable

# Enable command line logging
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1 /f
```

**Deploy Sysmon:**
```powershell
# Download Sysmon and use SwiftOnSecurity config
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile sysmon.zip
Expand-Archive sysmon.zip -DestinationPath sysmon/
# Use SwiftOnSecurity sysmonconfig-export.xml from GitHub
.\sysmon\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

**PowerShell Script Block Logging:**
```powershell
# Enable via registry
$registryPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
New-Item -Path $registryPath -Force
Set-ItemProperty -Path $registryPath -Name "EnableScriptBlockLogging" -Value 1
```

---

## Generating Attack Telemetry: Atomic Red Team

Atomic Red Team (by Red Canary) is a library of small, focused tests mapped to MITRE ATT&CK. Each "atomic test" executes a specific technique and generates the expected log artifacts.

```powershell
# Install Invoke-AtomicRedTeam
Install-Module -Name invoke-atomicredteam,powershell-yaml -Scope CurrentUser -Force

# Import
Import-Module invoke-atomicredteam

# List available tests for a technique
Invoke-AtomicTest T1003.001 -ShowDetails    # Credential dumping: LSASS

# Execute a test
Invoke-AtomicTest T1003.001 -TestNumbers 1  # Run test #1

# Clean up after test
Invoke-AtomicTest T1003.001 -TestNumbers 1 -Cleanup
```

**Useful techniques to test first:**

| ATT&CK ID | Technique | What to detect |
|---|---|---|
| T1059.001 | PowerShell | Script block logs, encoded commands |
| T1003.001 | LSASS dumping | Process access to lsass.exe (Sysmon 10) |
| T1053.005 | Scheduled Task | Event ID 4698, 4702 |
| T1105 | Ingress Tool Transfer | certutil/bitsadmin download |
| T1021.002 | SMB Lateral Movement | 4624 Type 3, 7045 service install |
| T1136.001 | Create Local Account | Event ID 4720 |

Run the test, then verify you see the expected events in your SIEM. If you don't — that's a detection gap, and now you know about it.

---

## Common Beginner Mistakes

**1. Not enabling logging before running attack tests.**
You run Atomic Red Team and then realize your SIEM received nothing because Sysmon wasn't installed. Enable logging first, verify it's working (search for a test event), then run attack simulations.

**2. Running attacks from the SIEM VM.**
Keep your attack VM on a separate network segment with limited access to your SIEM. Otherwise you can't detect lateral movement properly.

**3. Trying to build everything at once.**
Get one VM logging to your SIEM and verify end-to-end first. Then add complexity. Troubleshooting a 5-VM setup when nothing is working is miserable.

**4. Using domain admin for everything.**
Create regular user accounts. Practice detecting privilege escalation. If everything runs as Domain Admin, you'll miss half the detection scenarios.

**5. Not taking snapshots.**
Take VM snapshots before attack simulations. Some tests make significant changes. You want to be able to revert quickly.

---

## Where to Go Next

Once you have the lab working and you can see attack telemetry in your SIEM:

1. Work through Atomic Red Team systematically by ATT&CK tactic
2. Write a detection rule for each technique you verify fires correctly
3. Practice the tools covered in this blog series — Volatility, Chainsaw, Wireshark — against your own simulated incidents
4. Try DFIR Report blog posts — they document real incidents in detail, and you can simulate the TTPs in your lab

The lab is a forcing function. Every gap you find in your detections is a gap you can close before it matters in production.

---

*Final post in this series: [Writing Incident Reports That Actually Help](#) — documenting the incidents your lab practice prepares you to handle.*
