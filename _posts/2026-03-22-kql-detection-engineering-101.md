---
title: "KQL Detection Engineering 101: Writing Sentinel Queries That Find Real Attacks"
date: 2026-03-22
author: "Andy"
categories:
- "Blue Team"
- "Detection Engineering"
tags:
- "KQL"
- "Microsoft Sentinel"
- "detection-engineering"
- "Azure"
- "blue-team"
description: "Practical KQL for security analysts — building real attack detections in Microsoft Sentinel from scratch."
read_time: 10
---

# KQL Detection Engineering 101: Writing Sentinel Queries That Find Real Attacks

> *KQL is the language your detections live in. Treat it like a skill, not a search box.*

---

## KQL Is Not a Search Engine

Most analysts use KQL like a Google search — type some words, hope for results. That works for ad-hoc investigation, but it won't build detections that catch sophisticated attackers.

KQL (Kusto Query Language) is a full query language with filtering, aggregation, joining, and time-series analysis capabilities. The analysts who write detections that actually matter are the ones who treat it as such.

This post isn't an exhaustive KQL tutorial. It's a focused guide on the patterns and constructs that matter most for detection engineering in Microsoft Sentinel.

---

## The Detection Query Structure

Every good detection query follows the same logical flow:

```kql
// 1. Define time window (or let Sentinel handle via scheduled alert)
// 2. Select the right table
// 3. Filter to relevant events
// 4. Enrich / join for context
// 5. Aggregate or correlate
// 6. Project only what analysts need
```

Start with this skeleton for every new detection you write:

```kql
let timeWindow = 1h;
let threshold = 5;
SecurityEvent
| where TimeGenerated > ago(timeWindow)
| where EventID == 4625                    // filter to relevant events
| summarize FailedLogins = count() by Account, IpAddress, bin(TimeGenerated, 5m)
| where FailedLogins > threshold           // apply threshold
| project TimeGenerated, Account, IpAddress, FailedLogins
| order by FailedLogins desc
```

Let's break down the key constructs.

---

## Essential KQL Operators for Detection

### `where` — Filtering

The foundation of every query. Be specific:

```kql
// Bad — too broad
SecurityEvent
| where EventID == 4688

// Better — specific command line patterns
SecurityEvent
| where EventID == 4688
| where CommandLine has_any ("certutil", "mshta", "regsvr32")
| where CommandLine has_any ("http://", "https://", "-urlcache", "-decode")
```

Useful operators:
- `has` — case-insensitive string search (faster than `contains`)
- `has_any` — matches any value in a list
- `contains` — substring match (case-insensitive)
- `matches regex` — full regex (use sparingly, it's slow)
- `startswith` / `endswith` — for paths and file names

### `summarize` — Aggregation

Turn raw events into meaningful signals:

```kql
// Password spray detection: many accounts failed from one IP
SecurityEvent
| where TimeGenerated > ago(1h)
| where EventID == 4625
| summarize
    UniqueTargetAccounts = dcount(TargetAccount),
    FailureCount = count(),
    Accounts = make_set(TargetAccount, 20)
  by IpAddress
| where UniqueTargetAccounts > 10        // spray threshold
| where FailureCount > 20
```

Key aggregation functions:
- `count()` — number of records
- `dcount(field)` — distinct count (approximate, fast)
- `make_set(field, limit)` — collect unique values into an array
- `make_list(field, limit)` — collect all values (preserves duplicates)
- `min()` / `max()` / `avg()` — statistical aggregations

### `let` — Variables and Subqueries

`let` is your best friend for readable, maintainable queries:

```kql
// Define reusable values
let watchedBinaries = dynamic(["certutil.exe", "mshta.exe", "regsvr32.exe", "rundll32.exe", "bitsadmin.exe"]);
let excludedHosts = dynamic(["DC01", "SCCM01", "WSUS01"]);
let timeWindow = 1d;

SecurityEvent
| where TimeGenerated > ago(timeWindow)
| where EventID == 4688
| where Process has_any (watchedBinaries)
| where Computer !in (excludedHosts)
| project TimeGenerated, Computer, Account, Process, CommandLine, ParentProcessName
```

You can also use `let` for subqueries:

```kql
// Find accounts that failed login then succeeded (potential brute force)
let failedLogins = SecurityEvent
    | where TimeGenerated > ago(1h)
    | where EventID == 4625
    | project FailTime = TimeGenerated, Account, IpAddress;

let successLogins = SecurityEvent
    | where TimeGenerated > ago(1h)
    | where EventID == 4624
    | project SuccessTime = TimeGenerated, Account, IpAddress;

failedLogins
| join kind=inner successLogins on Account, IpAddress
| where SuccessTime > FailTime
| where SuccessTime - FailTime < 10m    // success within 10 min of failures
| project Account, IpAddress, FailTime, SuccessTime
```

### `join` — Correlating Tables

Joining is how you build multi-stage detections:

```kql
// Find processes that also made network connections (LOLBin download detection)
let suspiciousProcesses = SecurityEvent
    | where TimeGenerated > ago(1h)
    | where EventID == 4688
    | where Process has_any ("certutil.exe", "mshta.exe", "bitsadmin.exe")
    | project TimeGenerated, Computer, ProcessId, Process, CommandLine;

let networkConnections = DeviceNetworkEvents
    | where TimeGenerated > ago(1h)
    | where RemotePort in (80, 443, 8080, 8443)
    | project TimeGenerated, DeviceName, InitiatingProcessId, RemoteIP, RemoteUrl;

suspiciousProcesses
| join kind=inner networkConnections
    on $left.Computer == $right.DeviceName
    and $left.ProcessId == $right.InitiatingProcessId
| project TimeGenerated, Computer, Process, CommandLine, RemoteIP, RemoteUrl
```

Join types matter:
- `inner` — only records with matches on both sides
- `leftouter` — all left records, null on right if no match
- `leftanti` — left records with NO match on right (useful for "X happened without Y")

### `bin` — Time Bucketing

Essential for time-series detections:

```kql
// Detect beaconing: regular intervals of outbound connections
DeviceNetworkEvents
| where TimeGenerated > ago(6h)
| where RemoteIP == "suspicious.ip"
| summarize ConnectionCount = count() by bin(TimeGenerated, 5m)
| where ConnectionCount > 0
| render timechart
```

`bin(TimeGenerated, 5m)` groups events into 5-minute buckets. For beaconing detection, consistent counts in every bucket is the signal.

---

## Real Detection Examples

### Pass-the-Hash Detection

```kql
// PtH typically shows up as NTLM logon (Type 3) with Null NTLM Domain
SecurityEvent
| where TimeGenerated > ago(1h)
| where EventID == 4624
| where LogonType == 3
| where AuthenticationPackageName == "NTLM"
| where WorkstationName != ""
| where TargetDomainName == "-" or TargetDomainName == ""
// Exclude legitimate NTLM (workgroup machines, specific services)
| where Computer !in (knownNTLMSources)
| project TimeGenerated, Computer, TargetAccount, IpAddress, WorkstationName
```

### Suspicious Scheduled Task

```kql
SecurityEvent
| where TimeGenerated > ago(1d)
| where EventID == 4698
| extend TaskName = tostring(parse_xml(EventData).DataItem.TaskName)
| extend TaskContent = tostring(parse_xml(EventData).DataItem.TaskContent)
| where TaskContent has_any (
    "powershell", "cmd.exe", "wscript", "cscript",
    "mshta", "rundll32", "http://", "https://"
  )
| where SubjectUserName !in ("SYSTEM", "LOCAL SERVICE", "NETWORK SERVICE")
| project TimeGenerated, Computer, SubjectUserName, TaskName, TaskContent
```

### Admin Tool on Non-Admin Workstation

```kql
let adminTools = dynamic(["net.exe", "net1.exe", "nltest.exe", "dsquery.exe", "adfind.exe"]);
let adminHosts = dynamic(["ADMWKS01", "ADMWKS02", "DC01", "DC02"]);

SecurityEvent
| where TimeGenerated > ago(1h)
| where EventID == 4688
| where Process has_any (adminTools)
| where Computer !in (adminHosts)
| where Account !has "admin"
| project TimeGenerated, Computer, Account, Process, CommandLine
```

### PowerShell AMSI Bypass Attempt

```kql
Event
| where TimeGenerated > ago(1h)
| where Source == "Microsoft-Windows-PowerShell"
| where EventID == 4104
| where EventData has_any (
    "AmsiUtils",
    "amsiInitFailed",
    "AmsiContext",
    "System.Runtime.InteropServices.Marshal"
  )
| extend ScriptBlock = extract(@"<Data Name='ScriptBlockText'>(.*?)</Data>", 1, EventData)
| project TimeGenerated, Computer, ScriptBlock
```

---

## Tuning Your Queries

The first version of a detection query is almost never production-ready. Tuning is the work:

**Step 1: Run against historical data**
```kql
// Check last 30 days to understand baseline volume
SecurityEvent
| where TimeGenerated > ago(30d)
| where EventID == 4698
| where ... // your detection logic
| summarize count() by bin(TimeGenerated, 1d)
| render barchart
```

**Step 2: Analyze false positives**
```kql
// Who/what is generating most alerts?
| summarize count() by SubjectUserName, Computer
| order by count_ desc
```

**Step 3: Add exclusions for known-good patterns**
```kql
| where SubjectUserName !in (legitimateServiceAccounts)
| where Computer !in (scheduledTaskServers)
| where TaskName !startswith ("\\Microsoft\\")    // exclude Microsoft built-in tasks
```

**Step 4: Set the right alert threshold**
Tune the `summarize` threshold until the alert fires on real anomalies, not daily routine.

---

## Scheduling Analytics Rules in Sentinel

Once your query is tuned, convert it to an Analytics Rule:

- **Query frequency**: How often to run (5min, 1h, 1d)
- **Lookup period**: How far back to look (must be ≥ frequency)
- **Alert threshold**: Alert when results count is > 0 (for high-confidence rules) or > N (for volume-based rules)
- **Event grouping**: Group all results into single alert vs. one alert per row

For high-confidence detections (AMSI bypass, log clear, Pass-the-Hash): alert on every single result, group as individual alerts.

For volume-based detections (brute force, spray): alert when count exceeds threshold, single grouped alert.

---

## KQL Resources Worth Bookmarking

- **Sentinel GitHub** — Microsoft's official detection rules repository
- **KQL Search** (kqlsearch.com) — community KQL query library
- **MSTIC GitHub** — Microsoft Threat Intelligence Center's Sentinel rules
- **Reprise99's Sentinel Queries** — battle-tested community detections

KQL fluency is a career-defining skill for any Sentinel-focused blue teamer. The investment pays dividends every time you need to hunt or build a new detection quickly.

---

*Next: [Threat Hunting with a Hypothesis-Driven Approach](#) — using your KQL skills proactively before the alert fires.*
