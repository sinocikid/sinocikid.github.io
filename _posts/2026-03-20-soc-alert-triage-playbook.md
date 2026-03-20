---
title: "SOC Alert Triage Playbook: Finding Real Threats in the Noise"
date: 2026-03-20
author: "Andy"
categories:
- "Blue Team"
- "SOC Operations"
tags:
- "SOC"
- "triage"
- "alert-fatigue"
- "incident-response"
- "blue-team"
description: "A practical 5-step triage framework for SOC analysts to cut through alert noise and focus on real threats."
read_time: 9
---

# SOC Alert Triage Playbook: Finding Real Threats in the Noise

> *Alert volume is not your enemy. Undisciplined triage is.*

---

## The Real Problem Isn't Too Many Alerts

Every SOC analyst knows the feeling: you start your shift, open the queue, and there are 300 alerts sitting there from the last 8 hours. Some are high severity. Most are noise. A few are real. Your job is to figure out which is which — fast.

The mistake most analysts make is treating every alert as a standalone event. They look at it in isolation, check a couple of indicators, shrug, and close it. Rinse and repeat 299 more times. This approach burns analysts out and misses actual incidents.

Effective triage is a *process*, not a gut feeling. Here's the one that works.

---

## The 5-Step Triage Framework

### Step 1: Context First

Before you touch any indicator, answer three questions:

- **Who is the affected asset?** Is it a workstation, a server, a domain controller, a developer's laptop? A suspicious process on a DC is a very different situation than the same process on a standard user machine.
- **Who is the affected user?** Is this account a service account, a privileged admin, an intern who started last week? Pull their login history. Do they normally work at this hour? From this location?
- **What is this alert rule supposed to catch?** Go back to the rule definition. Know what behavior it was designed to detect before you decide if this instance is meaningful.

Skipping context is how analysts waste 20 minutes investigating a legitimate IT admin running scripts during a maintenance window.

### Step 2: Enrichment

Now you pull external context. Minimum enrichment for any alert:

- **IP addresses**: VirusTotal, Shodan, AbuseIPDB. Is this a known bad actor? A cloud provider IP? A Tor exit node?
- **File hashes**: VirusTotal, MalwareBazaar. Has this binary been seen before? What does the detection ratio look like?
- **Domains/URLs**: URLhaus, URLScan.io, Cisco Talos. Is this a freshly registered domain? Does it have a suspicious pattern (DGA-like)?
- **User agent strings**: Is the HTTP request from a browser that matches what you'd expect for this user, or is it `python-requests/2.28.0`?

The goal isn't a definitive verdict yet — it's building the picture. Keep notes as you go; your case documentation starts now.

### Step 3: Correlation

This is where triage separates the good analysts from the great ones. Single alerts are context-free. *Correlated* alerts tell a story.

Ask yourself:
- Are there other alerts on the same host in the last 24 hours?
- Has this user triggered alerts on other machines?
- Is there a pattern? (Failed logins followed by a successful one followed by lateral movement is a story. Three failed logins is probably a user who forgot their password.)

In your SIEM, pivot on:
- `SourceIP` → what else has this IP done?
- `AccountName` → what has this account done across all hosts?
- `ProcessName` + `ParentProcessName` → is this process chain normal?

**Event IDs that help with correlation:**

| Event ID | What it tells you |
|---|---|
| 4624 | Successful logon — type matters (Type 3 = network, Type 10 = remote interactive) |
| 4625 | Failed logon — pattern of failures = brute force signal |
| 4648 | Logon using explicit credentials — often used in lateral movement |
| 4688 | Process creation — requires command line auditing to be useful |
| 4698 | Scheduled task creation — persistence mechanism |

### Step 4: Severity Assessment

You've got context, enrichment, and correlation. Now make a call:

- **True Positive (TP)**: Malicious activity confirmed. Escalate immediately.
- **True Positive — Low Risk**: Malicious activity, but contained or low impact. Document and track.
- **False Positive (FP)**: Legitimate behavior that matched the rule. Close with a note on why, and flag the rule for tuning.
- **Benign True Positive**: The rule worked, but the activity is authorized (e.g., a pen tester). Verify with context, close with documentation.
- **Undetermined**: You need more data or more eyes. Don't close — escalate to Tier 2 with your findings so far.

**Never close an alert as FP just because you couldn't find evidence of malice.** Absence of evidence is not evidence of absence. If something feels off but you can't prove it, escalate.

### Step 5: Document and Escalate

Your notes are the handoff. If you escalate to Tier 2 or Incident Response without documentation, you've just wasted your own work and added friction to the response.

Minimum documentation:
- Asset and user affected
- Alert trigger details (what fired, when)
- Enrichment findings (what you checked, what you found)
- Correlation findings (what else is happening around this alert)
- Your assessment and reasoning
- Recommended next step

This takes 3-5 minutes to write and saves 30 minutes of repeated investigation upstream.

---

## Common False Positive Patterns (Know These Cold)

These are the time-wasters that will eat your shift if you don't recognize them fast:

**1. IT Admin Activity After Hours**
Scheduled maintenance often runs at night. Before escalating an "admin tool running at 2AM" alert, check the change management calendar and verify the account.

**2. Antivirus/EDR Scanning Its Own Files**
EDR solutions often enumerate system files and registry keys as part of their own protection. This can trigger process and file access rules. Know what your EDR binary looks like in process trees.

**3. Software Updates**
`msiexec.exe`, `wusa.exe`, `WindowsUpdateBox.exe` spawning child processes during patch windows is expected. Correlate with patch schedules.

**4. Security Scanning Tools**
Your own vulnerability scanner or SIEM connector pulling logs can look like network reconnaissance. Whitelist your internal security tooling properly.

**5. Developers Running Builds**
`cmd.exe` → `powershell.exe` → `git.exe` → `node.exe` is a build pipeline, not an attack chain. Know your developer population and their machines.

---

## The Triage Mindset

Triage is not about being right 100% of the time. It's about being *right fast* on the cases that matter and not getting bogged down on the ones that don't.

The analysts who burn out are the ones who treat every alert like it might be the breach that ends the company. The analysts who last — and who catch the real stuff — are the ones who have a system, follow it consistently, and let the process carry the cognitive load.

Build a playbook, follow it, and iterate on it when it fails. That's the job.

---

## Quick Reference Checklist

```
[ ] Asset identified and classified
[ ] User account identified — normal behavior baseline checked
[ ] Alert rule purpose understood
[ ] IPs/hashes/domains enriched
[ ] Correlated with other alerts on same host/user
[ ] Severity assessed: TP / FP / Benign TP / Undetermined
[ ] Documentation written
[ ] Escalated or closed with rationale
```

---

*Next up: [Writing Detection Rules That Don't Cry Wolf](#) — because the alert you're triaging is only as good as the rule that generated it.*
