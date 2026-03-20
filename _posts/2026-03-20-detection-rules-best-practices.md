---
title: "Writing Detection Rules That Don't Cry Wolf"
date: 2026-03-20
author: "Andy"
categories:
- "Blue Team"
- "Detection Engineering"
tags:
- "detection-engineering"
- "SIEM"
- "Sigma"
- "false-positives"
- "blue-team"
description: "Why most detection rules generate noise instead of signal — and the methodology to fix them."
read_time: 8
---

# Writing Detection Rules That Don't Cry Wolf

> *A detection rule that fires on everything catches nothing. Specificity is not a weakness.*

---

## Why Most Detection Rules Fail

Here's a common scenario: a threat intel report comes out about attackers using `certutil.exe` to download payloads. Someone writes a rule: `ProcessName == "certutil.exe"`. It goes into production. The SIEM immediately starts generating hundreds of alerts a day.

Analysts start closing them without looking. Eventually someone adds a suppression so broad it catches real malicious use too. The rule is now worse than useless — it's actively creating noise that masks real threats.

The problem wasn't the idea. The problem was the implementation. `certutil.exe` is used legitimately constantly. A rule that fires on every execution of it will never produce actionable signal.

This is detection engineering done wrong, and it's extremely common.

---

## The Three Ways Rules Break

**Too broad**: Fires on behavior that's common and mostly legitimate. Every `powershell.exe` execution, every use of `net.exe`, every scheduled task creation. You end up investigating IT admins all day.

**Too narrow**: Only catches the exact IOC from one specific threat report. The attacker changes one flag, uses a different binary, or encodes their payload slightly differently — and the rule completely misses it. These rules have a shelf life measured in weeks.

**No baseline**: The rule logic is fine, but nobody checked what normal looks like in this environment first. A rule for "PowerShell downloading from the internet" in an organization where developers do this constantly will generate 500 alerts a day. In an organization where no one should ever do this, it's a high-confidence alert.

Good detection rules avoid all three failure modes.

---

## The Detection Rule Lifecycle

Rules aren't "written and deployed." They go through a lifecycle:

```
Hypothesis → Draft → Test → Baseline → Deploy → Monitor → Tune → Retire
```

Skipping any step creates problems downstream.

### Hypothesis

Every rule starts with a threat behavior you want to detect, not an IOC.

Bad hypothesis: "Detect certutil.exe"
Good hypothesis: "Detect certutil.exe being used to download files from the internet, which is a known LOLBin abuse technique (MITRE T1105)"

The hypothesis defines *what attack behavior you're trying to catch*, not *what tool is running*.

### Draft

Write the rule logic. At minimum, define:
- **Data source**: What logs does this need? (Windows Security, Sysmon, proxy, EDR?)
- **Detection logic**: What specific field values, combinations, or patterns indicate malicious behavior?
- **Filter logic**: What legitimate activity looks like this that you need to exclude?

A Sigma rule for the certutil download scenario:

```yaml
title: Certutil File Download
id: e37db05d-d1f9-49c8-9d4d-b5b7ab3e6a7c
status: test
description: Detects certutil.exe being used to download files from the internet
references:
  - https://attack.mitre.org/techniques/T1105/
logsource:
  category: process_creation
  product: windows
detection:
  selection:
    Image|endswith: '\certutil.exe'
    CommandLine|contains:
      - '-urlcache'
      - '-split'
      - 'http'
  condition: selection
falsepositives:
  - Legitimate certificate operations (review command line)
level: high
tags:
  - attack.command_and_control
  - attack.t1105
```

Notice the rule doesn't just match `certutil.exe` — it specifically looks for the `-urlcache` and `-split` flags combined with an HTTP URL, which is the download-specific abuse pattern.

### Test

Before deploying, test the rule against:
1. **Known-bad samples**: Do you have PCAP, log data, or a lab environment where you can replay the attack? Does the rule fire?
2. **Known-good data**: Run the rule against 30-90 days of production logs. How many times does it fire? On what? Is any of it legitimate?

If you skip this step, you're deploying blind.

### Baseline

Understanding what's normal in your environment is non-negotiable. Questions to answer before deploying any process-based rule:

- How many times per day does this process run across the fleet?
- Which hosts run it most? Least?
- What are the typical command line arguments?
- Which accounts typically execute it?

Outliers from this baseline are your signal. The baseline *is* the detection.

### Deploy → Monitor → Tune

After deployment, actively monitor the rule for the first 2 weeks. Track:
- **Alert volume**: Is it within expected range?
- **True positive rate**: What percentage of alerts are real?
- **False positive patterns**: What legitimate activity is getting caught?

For each false positive pattern, add a specific exclusion — not a broad suppression. "Exclude alerts where ParentProcess is `sccm.exe`" is better than "Exclude all alerts from the IT team."

### Retire

Rules have a shelf life. When a rule consistently produces no true positives for 90+ days and the underlying threat behavior is no longer relevant, retire it. Dead rules add to alert volume and maintenance overhead without adding value.

---

## Good vs. Bad Rule Examples

### Scheduled Task Creation

**Bad:**
```
EventID == 4698
```
This fires every time any scheduled task is created. Windows Update creates scheduled tasks. Every application installer creates scheduled tasks. This is useless.

**Better:**
```
EventID == 4698
AND TaskName NOT IN (known_legitimate_tasks)
AND SubjectUserName NOT IN (service_accounts)
AND TaskContent contains one of:
  - "powershell"
  - "cmd.exe"
  - "wscript"
  - "cscript"
  - "mshta"
  - "http"
```
Now you're specifically looking for scheduled tasks that execute suspicious interpreters or pull from the internet — which is the actual attack behavior.

### PowerShell Encoded Commands

**Bad:**
```
CommandLine contains "-enc"
```
`-enc` is a legitimate PowerShell parameter abbreviation used all the time.

**Better:**
```
CommandLine contains "-EncodedCommand"
OR CommandLine matches regex: "-[Ee]{1}[Nn]{1}[Cc]{1}[Oo]{0,3}[Dd]{0,3}[Ee]{0,3}[Dd]{0,6}"
AND CommandLine length > 500
AND ParentProcess NOT IN (known_legitimate_parents)
```
Long encoded commands from unexpected parent processes are the actual IOC. Short encoded commands from known management tools are probably fine.

---

## The Specificity Principle

Every condition you add to a rule narrows its scope. This feels like it reduces coverage, but it actually *increases* the signal-to-noise ratio. A rule that fires 5 times a week with a 90% true positive rate is infinitely more valuable than a rule that fires 500 times a day with a 1% true positive rate.

The analysts who respond to the 5 alerts per week will investigate every one. The analysts who face 500 will start skipping them — and the 5 real ones will get buried in the noise.

Write rules that are specific enough to be actionable, and invest in building out exclusions for known-good behavior. Your analysts will thank you.

---

## Detection Rule Quality Checklist

```
[ ] Hypothesis defined (what attack behavior, which MITRE technique)
[ ] Data source identified and available
[ ] Detection logic targets behavior, not just a binary/IOC
[ ] Exclusions written for known legitimate use cases
[ ] Tested against known-bad samples
[ ] Tested against 30+ days of production logs
[ ] Alert volume is manageable (< 10/day for high severity)
[ ] True positive rate documented
[ ] Rule tagged with MITRE ATT&CK IDs
[ ] Tuning/review schedule set
```

---

*Related: [SOC Alert Triage Playbook](#) — because great rules still generate alerts that need to be worked.*
