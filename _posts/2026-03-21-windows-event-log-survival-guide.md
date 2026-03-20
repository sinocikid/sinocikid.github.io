---
title: "Windows Event Log Survival Guide: The IDs That Actually Matter"
date: 2026-03-21
author: "Andy"
categories:
- "Blue Team"
- "Windows Security"
tags:
- "Windows"
- "event-logs"
- "audit-policy"
- "SIEM"
- "blue-team"
description: "Stop collecting everything and detecting nothing. Here are the Windows Event IDs that actually matter for security detection."
read_time: 9
---

# Windows Event Log Survival Guide: The IDs That Actually Matter

> *Collecting every Windows log is not a security strategy. It's a storage bill.*

---

## The Logging Problem

Most organizations collect Windows event logs. Very few collect the *right* Windows event logs. There's a difference.

The default Windows audit policy is designed for system administration, not security detection. It logs account lockouts but misses process creation. It records successful logons but doesn't capture command line arguments. It generates millions of events per day that tell you almost nothing about whether you're being attacked.

The result: bloated SIEMs, overwhelmed analysts, and attackers operating freely because the logs that would expose them were either never collected or buried in noise.

This post covers what you actually need.

---

## Prerequisite: Fix Your Audit Policy First

Before you worry about which Event IDs to forward, you need to make sure those events are actually being generated. Windows won't log what you haven't asked it to log.

Configure via Group Policy: `Computer Configuration → Windows Settings → Security Settings → Advanced Audit Policy Configuration`

**Minimum required audit settings:**

| Category | Subcategory | Setting |
|---|---|---|
| Account Logon | Credential Validation | Success, Failure |
| Account Logon | Kerberos Authentication | Success, Failure |
| Account Management | Security Group Management | Success |
| Account Management | User Account Management | Success, Failure |
| Detailed Tracking | Process Creation | Success |
| Logon/Logoff | Logon | Success, Failure |
| Logon/Logoff | Special Logon | Success |
| Object Access | File System | Failure (selective) |
| Policy Change | Audit Policy Change | Success |
| Privilege Use | Sensitive Privilege Use | Success, Failure |
| System | Security System Extension | Success |

**Additionally, enable via registry:**

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit
ProcessCreationIncludeCmdLine_Enabled = 1
```

Without this, Event ID 4688 (process creation) captures the process name but not the command line arguments — making it nearly useless for detecting LOLBin abuse and malicious scripting.

---

## The Event IDs That Matter

### Authentication & Logon

**4624 — Successful Logon**

The volume here is enormous, so raw collection is only useful with filtering. What matters:

- **Logon Type** is the key field:
  - Type 2: Interactive (physical keyboard)
  - Type 3: Network (SMB, file shares)
  - Type 7: Unlock
  - Type 10: RemoteInteractive (RDP)
  - Type 11: CachedInteractive (cached credentials — offline logon)

- **Alert on**: Type 3 logons using domain admin accounts; Type 10 logons outside business hours; logons from new source IPs; service accounts logging on interactively (Type 2 or 10)

**4625 — Failed Logon**

Failed logons are your brute force signal. Alone, one 4625 means nothing. But:
- 10+ failures in 60 seconds from one source = brute force
- Failures across multiple accounts from one IP = password spray
- Failures on non-existent accounts = account enumeration

The **Sub Status Code** tells you *why* the logon failed:
- `0xC000006A` — wrong password (account exists)
- `0xC0000064` — account doesn't exist (enumeration signal)
- `0xC000006F` — outside logon hours
- `0xC0000234` — account locked out

**4648 — Logon with Explicit Credentials**

This fires when an account uses credentials other than its current context — think `runas`, `net use`, or Pass-the-Hash/Pass-the-Ticket. Lateral movement often generates 4648 events on the source machine. Alert on this from non-admin workstations and service accounts.

**4771 — Kerberos Pre-Authentication Failed**

The Kerberos equivalent of 4625. Critical for detecting AS-REP roasting (accounts without pre-auth required) and Kerberos brute force. Alert on repeated failures, especially against privileged accounts.

---

### Process Activity

**4688 — Process Created**

This is the most important detection event on Windows — but only when command line logging is enabled (see audit policy above). Without command lines, you just see process names, which is nearly useless.

With command lines enabled, you can detect:
- LOLBin abuse (certutil, mshta, regsvr32 with suspicious args)
- PowerShell encoded commands and download cradles
- Suspicious parent-child relationships (Word spawning PowerShell)
- Reconnaissance tools (net.exe, whoami, ipconfig, nltest)

The `ParentProcessName` field is gold. Build detections around unexpected parent-child combinations:

```
winword.exe → powershell.exe  ✗ (macro-based attack)
excel.exe → cmd.exe           ✗ (macro-based attack)
svchost.exe → cmd.exe         ✗ (service-based persistence or exploit)
explorer.exe → powershell.exe ✓ (user opened PowerShell — context-dependent)
```

---

### Persistence Mechanisms

**4698 — Scheduled Task Created**
**4702 — Scheduled Task Updated**

Scheduled tasks are one of the most common persistence mechanisms. Every 4698 deserves at least a quick look. Immediately alert on tasks that:
- Execute PowerShell, cmd.exe, wscript, cscript, mshta, or rundll32
- Reference files in temp directories (%TEMP%, %APPDATA%, C:\Windows\Temp)
- Were created by unexpected accounts (users creating tasks, service accounts)

**7045 — New Service Installed**

New services being installed is significant — especially outside of change windows. Malware often installs itself as a service for persistence and auto-start capability. Alert on services installed by non-SYSTEM accounts, services with unusual executable paths, and services with names that look like they're trying to blend in (single random characters, slight misspellings of legitimate service names).

---

### Account & Privilege Changes

**4720 — User Account Created**
**4722 — User Account Enabled**
**4723 / 4724 — Password Changed / Reset**
**4732 — Member Added to Security Group**
**4728 — Member Added to Global Group**

Account management events are your identity attack surface. Alert on:
- New accounts created outside your provisioning process
- Accounts being added to Domain Admins, Enterprise Admins, or any privileged group
- Password resets on privileged accounts outside business hours

**4672 — Special Privileges Assigned to New Logon**

This fires when an account with admin-equivalent privileges logs on. High volume on DCs, but valuable for tracking privilege escalation on workstations and non-DC servers.

---

### Defense Evasion

**1102 — Audit Log Cleared**

An attacker clearing the Security event log to cover tracks. This should be a P1 alert. Legitimate log clearing is extremely rare and should go through documented change management. Any 1102 that isn't on your approved list is an incident until proven otherwise.

**4719 — Audit Policy Changed**

If someone changes the audit policy — especially to disable logging — that's a major red flag. Legitimate changes should only come from your GPO infrastructure, not individual hosts.

---

### PowerShell Logging

These require explicit configuration beyond standard audit policy:

**4103 — Module Logging**
Captures PowerShell module activity. Enable via GPO: `Computer Configuration → Administrative Templates → Windows Components → Windows PowerShell → Turn on Module Logging`

**4104 — Script Block Logging**
The most valuable PowerShell log. Captures the actual script content *after* deobfuscation — so even if an attacker uses encoded commands or string concatenation, 4104 shows you what actually ran. Enable via: `Turn on PowerShell Script Block Logging`

Alert on 4104 events containing:
- `IEX` / `Invoke-Expression`
- `DownloadString` / `DownloadFile`
- `AmsiUtils` (AMSI bypass attempts)
- `Net.WebClient`
- `Reflection.Assembly`
- Base64 encoded blocks within scripts

---

## Log Forwarding Architecture

Once you've got the right events generating, you need to forward them efficiently.

**Windows Event Forwarding (WEF)** is the native, free option. Configure a Windows Event Collector, push subscriptions via GPO, and forward to your SIEM. Works well at scale when configured properly.

**Key decisions:**
- Forward all events to SIEM, or pre-filter at the collector? Pre-filtering reduces SIEM ingestion costs but risks missing context.
- Which hosts are most critical? Domain controllers, servers, and admin workstations should be forwarded in full. Standard workstations can be more selective.
- How much retention? Security events should be retained for at least 90 days locally, 1 year in SIEM if possible.

**Minimum forwarding list for a lean but effective SIEM:**

```
Security.evtx: 1102, 4624 (Types 3,10), 4625, 4648, 4672, 4688, 4698, 4702, 4720, 4732, 4728, 4719, 4771, 7045
System.evtx: 7045
Microsoft-Windows-PowerShell/Operational: 4103, 4104
Microsoft-Windows-Sysmon/Operational: 1, 3, 7, 11, 12, 13 (if Sysmon deployed)
```

---

## The Logging Maturity Path

If you're starting from zero, don't try to implement everything at once. Prioritize in this order:

1. **Week 1**: Enable process creation logging with command lines (4688). This single change dramatically improves detection capability.
2. **Week 2**: Enable PowerShell script block logging (4104). Catches fileless attacks.
3. **Week 3**: Forward authentication events (4624, 4625, 4648) with proper logon type filtering.
4. **Month 2**: Deploy Sysmon with a community config (SwiftOnSecurity or Olaf Hartong's modular config). This fills gaps that native logging misses.
5. **Month 3**: Build SIEM alerts on the high-value event IDs above.

You don't need to collect everything. You need to collect the right things consistently.

---

*The logs are only valuable if you're running detections against them. Next up: [KQL Detection Engineering 101](#) — writing queries that find real attacks.*
