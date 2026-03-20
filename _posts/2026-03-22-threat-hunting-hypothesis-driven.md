---
title: "Threat Hunting That Actually Works: The Hypothesis-Driven Approach"
date: 2026-03-22
author: "Andy"
categories:
- "Blue Team"
- "Threat Hunting"
tags:
- "Threat Hunting"
- "MITRE ATT&CK"
- "Detection Engineering"
- "Lateral Movement"
- "Threat Intelligence"
description: "Reactive hunting is busywork. This post covers the hypothesis-driven hunt cycle — building hunts from threat intel and ATT&CK, executing them with purpose, and converting successful hunts into durable detections."
read_time: 9
---

Most organizations that say they do threat hunting are actually doing alert review. Someone gets a tip, goes looking in the logs, finds nothing obvious, and writes "no evidence of compromise" in the ticket. That is not hunting. That is reactive log browsing.

Real threat hunting starts before an alert fires. It starts with a hypothesis.

## Why Reactive Hunting Fails

Reactive hunting — starting a hunt because something looked weird in a dashboard — has two problems. First, you are already behind. The attacker has had time to establish persistence, move laterally, or exfiltrate data before you started looking. Second, without a defined hypothesis, you do not know when to stop. You keep looking until you run out of time or convince yourself nothing is wrong.

The hypothesis-driven model flips this. You decide in advance what attacker behavior you are looking for, where the evidence should appear, and what a positive finding looks like. You hunt with a purpose and know exactly when the hunt is done.

## Building a Hypothesis

A hunt hypothesis is a falsifiable statement: "I believe threat actor group X is present in this environment and performing Y technique, which should produce evidence Z in data source W."

Hypotheses come from three sources:

**1. Threat Intelligence**
If a threat intel report says APT41 is actively targeting organizations in your sector using Cobalt Strike with named pipe communication, your hypothesis writes itself: "Cobalt Strike named pipe activity is present in endpoint telemetry." You know what to look for and where.

**2. MITRE ATT&CK**
Use ATT&CK to identify techniques that your existing detections do not cover. If you have solid coverage on initial access but nothing on T1053 (Scheduled Task/Job), that gap is a hunting target. Build a hypothesis around what scheduled task abuse looks like in your environment's logs.

**3. Environment Changes**
New VPN rollout, new SaaS tool, new M&A target onboarded to the network — these are high-risk moments. Build hypotheses around what malicious use of those new assets would look like.

A weak hypothesis: "I want to look for lateral movement."
A strong hypothesis: "NTLM authentication from workstations to servers using non-standard service accounts occurred outside business hours in the past 30 days."

The strong version tells you exactly what query to write and what a positive result looks like.

## The Hunt Cycle

### Phase 1: Hypothesis

Write it down. One sentence, specific and falsifiable. Document the ATT&CK technique ID, the relevant data sources, and the expected evidence. If you cannot write this down clearly, your hypothesis is not ready.

### Phase 2: Data Collection

Identify whether you actually have the data you need. This is where hunts die — you build the perfect hypothesis and then discover that the relevant log source is not ingested, or retention is only 7 days when you need 30.

Before executing, confirm:
- Is the data source available? (Sysmon? EDR telemetry? Firewall logs?)
- Does it have sufficient retention for the hunt timeframe?
- Is the data complete? (Missing endpoints, gaps in collection?)

If the data is not there, the hunt finding is "we have a visibility gap" — which is still a valuable output.

### Phase 3: Analysis

This is the actual hunting. Write the query, execute it, analyze the results. The goal is not to find everything — it is to find anomalies that warrant investigation.

For lateral movement using NTLM, a Sentinel query might look like:

```kql
SecurityEvent
| where TimeGenerated >= ago(30d)
| where EventID == 4624
| where LogonType == 3
| where AuthenticationPackageName == "NTLM"
| where AccountType == "User"
| where not(Account endswith "$")
| summarize
    HopCount = dcount(Computer),
    Hosts = make_set(Computer),
    Hours = make_set(hourofday(TimeGenerated))
    by Account, IpAddress
| where HopCount > 3
| extend OffHoursActivity = set_has_element(Hours, 2) or set_has_element(Hours, 3)
| sort by HopCount desc
```

You are not looking for a smoking gun. You are looking for the top of a ranked list of anomalies to investigate manually.

### Phase 4: Findings

Every hunt produces one of four outcomes:

1. **Clean**: No evidence of the hypothesized technique. Document that you looked and found nothing.
2. **Visibility gap**: You cannot confirm or deny because the data is absent. Document the gap.
3. **Benign anomaly**: You found something that matched the pattern but investigated and confirmed it is legitimate. Document what you found and why it is benign — this becomes exclusion list material.
4. **True positive**: You found evidence of attacker activity. Escalate to incident response.

All four outcomes are valid. All four need to be documented. The biggest mistake hunters make is only writing down true positives.

### Phase 5: Detection

If the hunt found attacker activity, the hunt is not over. Convert the hunting query into a scheduled analytics rule so the next occurrence is detected automatically, without requiring a human-initiated hunt.

This is the compounding return on hunt investment. Every successful hunt should result in a new detection, or an improvement to an existing one.

## Practical Hunt Example: C2 Beaconing

**Hypothesis**: "A compromised host is beaconing to a C2 server at regular intervals, producing a pattern of consistent outbound connections to the same destination at fixed time intervals."

**Data sources needed**: Firewall or proxy logs with destination IP/domain, byte counts, and timestamps.

**Query approach** (using firewall logs in Sentinel):

```kql
CommonSecurityLog
| where TimeGenerated >= ago(7d)
| where DeviceVendor == "Palo Alto Networks"
| where Activity == "TRAFFIC"
| where DestinationPort in (80, 443, 8080, 8443)
| summarize
    ConnectionCount = count(),
    BytesSent = sum(SentBytes),
    Intervals = make_list(TimeGenerated, 1000)
    by SourceIP, DestinationIP
| where ConnectionCount > 20
| extend IntervalVariance = array_length(Intervals)
// Further analysis: export Intervals for statistical analysis
| sort by ConnectionCount desc
```

The interval variance calculation is the key signal. Human-generated traffic is irregular. Malware beaconing is regular. Tools like `jitter` in Cobalt Strike introduce randomness, but even jittered beaconing is statistically more regular than organic traffic.

For high-fidelity beaconing detection, pull the interval data into Python or use a notebook to calculate standard deviation. Flag anything with a coefficient of variation below 0.1.

## Documenting Hunts

Use a consistent template. Every hunt record should contain:

- **Hypothesis**: The one-sentence falsifiable statement
- **ATT&CK Technique**: ID and name (e.g., T1071.001 — Web Protocols)
- **Data Sources**: What you queried
- **Timeframe**: Date range of the hunt
- **Query**: The actual KQL (or SPL, or whatever)
- **Findings**: What you found, with specific examples
- **Outcome**: Clean / Gap / Benign / True Positive
- **Follow-up**: Detection created? Gap remediation requested?

Store these in a shared wiki or repository. After six months, you will have a library of executed hunts that tells you exactly what you have and have not looked for. This is a direct input to your detection coverage map.

## Converting Hunts to Detections

Not every hunt converts to a detection directly. Some hunts are too slow (30-day aggregation) to run as a scheduled rule. In those cases, break the hunt into a faster-running proxy:

- The 30-day NTLM lateral movement hunt becomes a 1-hour rule looking for the same account authenticating to 3+ hosts within 30 minutes.
- The 7-day beaconing hunt becomes a 24-hour rule looking for connection frequency anomalies per source IP.

The detection does not have to match the hunt exactly. It has to catch the behavior before the hunt timeframe would have caught it.

**Practical Takeaways:**
- Write hypotheses before writing queries — a hypothesis should be falsifiable
- Source hypotheses from threat intel, ATT&CK gaps, and environment changes
- All four hunt outcomes (clean, gap, benign, true positive) require documentation
- Successful hunts must produce a detection, not just a finding
- Beaconing detection requires interval regularity analysis, not just connection counts
- Store hunt records in a library — they become your coverage map
