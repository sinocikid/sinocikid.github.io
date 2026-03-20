---
title: "Memory Forensics for Blue Teamers: What Lives in RAM That Won't Show on Disk"
date: 2026-03-25
author: "Andy"
categories:
- "Blue Team"
- "Forensics"
tags:
- "memory-forensics"
- "Volatility"
- "fileless-malware"
- "LSASS"
- "blue-team"
description: "Fileless malware, injected shellcode, and credential artifacts exist only in RAM. Here's how to find them."
read_time: 10
---

# Memory Forensics for Blue Teamers: What Lives in RAM That Won't Show on Disk

> *The most important evidence in your investigation might never touch the disk. If you don't capture memory, it's gone at reboot.*

---

## Why Memory Matters

Traditional forensics focuses on disk artifacts: files, registry, event logs. But modern attackers — and even commodity malware — increasingly avoid disk writes entirely.

Things that exist in memory and nowhere else:

- **Fileless malware** — PowerShell payloads, reflective DLL injection, shellcode executed via LOLBins
- **Credentials** — NTLM hashes and Kerberos tickets in LSASS; cleartext passwords cached in browser processes
- **Injected code** — Malicious code injected into legitimate processes (svchost.exe, explorer.exe)
- **Decrypted configuration** — Malware that stores its C2 config encrypted on disk, decrypted only in memory
- **Command history** — Commands executed via PowerShell or cmd.exe that weren't logged
- **Active network connections** — Connections that aren't in firewall logs if they're local or filtered

A system that looks clean on disk may be fully compromised in memory. Reboot it without capturing memory and you've destroyed evidence.

---

## Memory Acquisition

Before analysis, you need a memory image. Two approaches:

### Live Acquisition (Preferred When Possible)

Use an acquisition tool on the running system before isolation or reboot:

**WinPmem** (free, open source):
```powershell
# Dump physical memory to file
winpmem_mini_x64_rc2.exe -o memory.dmp

# Or acquire directly to network share
winpmem_mini_x64_rc2.exe \\server\share\memory.dmp
```

**DumpIt** (Magnet Forensics — free version available):
```powershell
# Simple full memory capture
DumpIt.exe /output C:\evidence\memory.raw
```

**Considerations for live acquisition**:
- Memory changes as you acquire it — some processes may appear inconsistent. This is normal.
- Acquisition takes time proportional to RAM size (~1-2 minutes per 8GB)
- The acquisition tool itself will appear in memory — note this during analysis
- Store the image on external media or network share, not the target system

### Crash Dumps (When Live Acquisition Isn't Possible)

Windows creates several types of memory dumps automatically on crash (BSOD) or on demand. These are smaller than full physical memory images but often sufficient:

- **Complete memory dump**: Full RAM — best for analysis
- **Kernel memory dump**: Kernel space only — faster, misses user-space malware in some cases
- **Minidump**: Minimal context — insufficient for most forensics

Force a dump from a running system (requires admin):
```powershell
# Enable complete memory dump and trigger immediately
reg add "HKLM\SYSTEM\CurrentControlSet\Control\CrashControl" /v CrashDumpEnabled /t REG_DWORD /d 1 /f
notmyfault64.exe /crash    # SysInternals NotMyFault tool
```

---

## Volatility 3: The Analysis Framework

Volatility 3 is the standard for memory forensics. It's Python-based, open source, and has a comprehensive plugin ecosystem for Windows, Linux, and macOS analysis.

```bash
# Install
pip3 install volatility3

# Basic syntax
python3 vol.py -f memory.dmp <plugin>

# List available plugins
python3 vol.py --help | grep windows
```

### Essential Plugins for Triage

**Process enumeration:**

```bash
# Standard process list (from PsActiveProcessHead)
python3 vol.py -f memory.dmp windows.pslist

# Process tree (shows parent-child relationships)
python3 vol.py -f memory.dmp windows.pstree

# Process scan (carves processes from memory — finds hidden/unlinked processes)
python3 vol.py -f memory.dmp windows.psscan
```

Compare `pslist` and `psscan` output. Processes in `psscan` but not `pslist` may be rootkit-hidden. Processes in `pslist` with anomalous properties (no parent, unusual path) need investigation.

**What to look for in process output:**
- Legitimate process names with unusual parent processes (`svchost.exe` with parent `explorer.exe` is wrong — should be `services.exe`)
- Misspelled process names (`svch0st.exe`, `lsasss.exe`)
- Multiple instances of processes that should be singular (`lsass.exe`, `csrss.exe`)
- Processes running from unusual paths (`C:\Users\Public\svchost.exe`)

**Command line arguments:**

```bash
# View command lines for all processes
python3 vol.py -f memory.dmp windows.cmdline
```

Malware often has no command line, or a command line that doesn't match the process name. PowerShell with encoded commands (`-EncodedCommand`) also shows here in decoded form after Volatility parses it.

**Network connections:**

```bash
# Active and recently closed network connections
python3 vol.py -f memory.dmp windows.netstat
```

Look for:
- Connections from `lsass.exe`, `svchost.exe`, or other system processes to external IPs
- Connections on unusual ports from trusted processes
- `CLOSE_WAIT` or `TIME_WAIT` connections that indicate recent activity

**DLL listing:**

```bash
# DLLs loaded by a specific process (use PID from pslist)
python3 vol.py -f memory.dmp windows.dlllist --pid 1234
```

Injected DLLs often have no path, no name, or load from unusual locations. Legitimate DLLs should load from `C:\Windows\System32\` or the application directory.

---

## Detecting Process Injection

Process injection is a core evasion technique — run malicious code inside a legitimate process so EDR and AV blame `svchost.exe` for the network connection.

### Malfind: The Injection Scanner

```bash
python3 vol.py -f memory.dmp windows.malfind
```

Malfind identifies memory regions that:
- Are executable (PAGE_EXECUTE_*)
- Are not backed by a file on disk (no mapped file)
- Contain PE headers (shellcode loaders, injected DLLs)

Output includes the memory address, protection flags, and a hex dump + disassembly of the region. False positives exist (JIT-compiled code, packed legitimate software), but each hit deserves investigation.

```
PID    Process    Start      End        Tag  Protection
1234   svchost.exe 0x400000  0x500000   VadS PAGE_EXECUTE_READWRITE
0000: 4d 5a 90 00 03 00 00 00   MZ......   ← PE header = injected DLL
```

A `MZ` header (4D 5A) in an unlinked memory region is a strong injection indicator.

### Hollowing Detection

Process hollowing: attacker creates a legitimate process (e.g., `svchost.exe`), unmaps its code from memory, and replaces it with malicious code.

Detection via Volatility:
```bash
# Compare mapped image path vs in-memory PE header
python3 vol.py -f memory.dmp windows.pslist --pid 1234

# Check VAD entries for suspicious memory mappings
python3 vol.py -f memory.dmp windows.vadinfo --pid 1234
```

Signs of hollowing:
- Process path points to a legitimate binary
- In-memory PE headers don't match the file on disk (use `windows.dlllist` and compare with file hashes)
- Memory sections are writable+executable when they shouldn't be

---

## LSASS: The Credential Treasury

LSASS (Local Security Authority Subsystem Service) holds:
- NTLM hashes for recently authenticated users
- Kerberos tickets
- Cleartext passwords in some configurations (older systems, WDigest enabled)

Attackers dump LSASS to steal credentials. Your job is to detect both the dumping *attempt* and find artifacts after the fact.

### Live Detection: LSASS Access

```bash
# Find processes that opened handles to lsass.exe with suspicious access rights
# (Requires Sysmon Event 10 in your event logs)

# In memory: which processes have handles to lsass?
python3 vol.py -f memory.dmp windows.handles --pid <lsass_pid>
```

Suspicious access rights to LSASS:
- `0x1010` (PROCESS_VM_READ | PROCESS_QUERY_INFORMATION)
- `0x1410` (PROCESS_VM_READ | PROCESS_QUERY_INFORMATION | PROCESS_DUP_HANDLE)
- `0x1fffff` (PROCESS_ALL_ACCESS)

Legitimate processes that access LSASS: `MsMpEng.exe` (Defender), `WerFault.exe` (error reporting), EDR agents. Everything else needs justification.

### Credential Extraction from Memory (Understanding What Attackers Do)

Mimikatz extracts credentials by:
1. Opening a handle to `lsass.exe`
2. Reading process memory
3. Parsing the credential structures

The artifact is the LSASS handle. If you find an unexpected process with an open handle to LSASS in your memory image, you've found the dumping tool — even if it's since been deleted from disk.

### Detecting WDigest (Cleartext Password Storage)

WDigest is disabled by default in Windows 8.1+ but can be enabled by attackers:
```
HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest
UseLogonCredential = 1
```

Check for this registry key in your memory image:
```bash
python3 vol.py -f memory.dmp windows.registry.printkey --key "SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest"
```

If `UseLogonCredential` is set to 1 and you didn't set it — someone else did.

---

## Practical Triage Workflow

When you have a memory image and need answers fast:

```bash
# 1. Get process list and look for anomalies
python3 vol.py -f memory.dmp windows.pstree 2>/dev/null | tee pstree.txt

# 2. Get all command lines — look for encoded PS, unusual args
python3 vol.py -f memory.dmp windows.cmdline 2>/dev/null | tee cmdlines.txt

# 3. Network connections — who is talking to whom?
python3 vol.py -f memory.dmp windows.netstat 2>/dev/null | tee netstat.txt

# 4. Run malfind — flag injected code regions
python3 vol.py -f memory.dmp windows.malfind 2>/dev/null | tee malfind.txt

# 5. For each suspicious PID, dump the DLL list
python3 vol.py -f memory.dmp windows.dlllist --pid <suspicious_pid> 2>/dev/null

# 6. Extract suspicious processes for further analysis
python3 vol.py -f memory.dmp windows.procdump --pid <suspicious_pid> --dump-dir ./dumps/

# 7. Submit extracted samples to sandbox
```

This workflow takes 10-15 minutes and answers most immediate triage questions: Is this system compromised? What's malicious? What did it connect to?

---

## Memory Forensics + EDR: Better Together

Modern EDR platforms perform continuous memory scanning — similar to what Volatility does, but in real time. If your EDR detected process injection, use memory forensics to understand *what* was injected and what it did after injection. EDR gives you the alert; memory forensics gives you the full picture.

Don't skip memory acquisition just because you have EDR. EDR has blind spots, and memory forensics is often the only way to definitively confirm what an attacker was doing on a compromised system.

---

*Next: [Building a Threat Detection Home Lab](#) — practice these skills in a safe environment before you need them in production.*
