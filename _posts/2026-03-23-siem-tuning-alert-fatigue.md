---
title: "SIEM Tuning: Escaping Alert Fatigue Without Going Blind"
date: 2026-03-23
author: "Andy"
categories:
- "Blue Team"
- "SOC Operations"
tags:
- "SIEM"
- "Alert Fatigue"
- "SOC"
- "Microsoft Sentinel"
- "Detection Tuning"
description: "Alert fatigue kills SOCs faster than attackers do. Here is how to tune your SIEM systematically — suppressing noise without creating blind spots — and how to measure whether your tuning is actually working."
read_time: 9
---

In 2023, the Ponemon Institute found that the average SOC receives over 11,000 alerts per day and investigates fewer than half of them. In some organizations I have worked with, the true positive rate on alerts is below 5%. That means 95 out of every 100 alerts an analyst touches is noise. At that ratio, analysts stop trying to find signal. They start rubber-stamping. And that is when the real breaches happen — quietly, in the noise.

Alert fatigue is not a people problem. It is an engineering problem. It is what happens when detections are deployed without tuning, and when the SIEM is treated as a log archive with alerts bolted on rather than a curated detection platform.

## What Causes Alert Fatigue

Three things drive the overwhelming majority of alert fatigue:

**1. Broad rules deployed without baseline**
Rules that fire on any occurrence of a behavior, without understanding what the baseline frequency of that behavior is in your specific environment. "Alert on any use of net.exe" sounds reasonable in a vacuum. In an environment where 50 sysadmins use net.exe legitimately every hour, it is a guaranteed fatigue machine.

**2. Rules that have never been tuned after initial deployment**
Every rule degrades over time. Your environment changes, new tools get deployed, user behavior shifts. A rule tuned 18 months ago for a different environment is probably generating significant noise today.

**3. Rules with no clear scope or owner**
Rules that nobody is responsible for reviewing. They accumulate in the SIEM like technical debt, firing on things nobody investigates, consuming analyst time and analyst trust.

The pattern is always the same: rules get deployed, noise accumulates, analysts learn to ignore certain rule names, attacker activity hides in the ignored queue.

## The Tuning Process: Suppress → Tune → Retire

Approach your SIEM rule library as three distinct categories:

### Suppress

Short-term suppression is for known-benign activity that is generating noise right now, while you work on a permanent fix. In Sentinel, suppression rules are called automation rules and can filter alerts by specific entity values before they create incidents.

For example: your brute force detection is firing on a monitoring system that legitimately makes 20 authentication attempts per minute. Suppress the specific source IP of that monitoring system while you build a proper exclusion into the detection query itself.

**Suppression should be temporary.** Every suppression rule needs an owner, an expiry date, and a ticket that tracks converting it into a proper query-level exclusion. If you do not set an expiry, suppressions accumulate and you lose track of what is being silently dropped.

### Tune

Tuning means modifying the detection query to reduce false positives without reducing true positive coverage. This is the main event. Most alert fatigue is solvable by tuning, not suppression.

The tuning workflow:

1. Pull the last 30 days of alerts for the rule you are tuning
2. Sample 20-30 alerts and triage each one manually — is it a true positive, false positive, or benign true positive (technically correct but not actionable)?
3. Identify the common attributes of false positives: specific accounts, IP ranges, hostnames, process parents, time windows
4. Add exclusions to the query for those attributes

Example: a PowerShell detection firing on your endpoint management tool (e.g., Tanium, BigFix) because it runs encoded PowerShell as part of its agent. The exclusion:

```kql
// Before tuning - high volume
DeviceEvents
| where ActionType == "PowerShellCommand"
| where InitiatingProcessCommandLine has "EncodedCommand"

// After tuning - excluding known-benign process parent
DeviceEvents
| where ActionType == "PowerShellCommand"
| where InitiatingProcessCommandLine has "EncodedCommand"
| where InitiatingProcessParentFileName !in ("taniumclient.exe", "bigfix.exe", "ccmexec.exe")
| where InitiatingProcessAccountName !in ("system", "network service")
    or InitiatingProcessParentFileName !startswith "system"
```

Document every exclusion. Why was it added? Who approved it? When was it last reviewed? Undocumented exclusions are the second most common reason SOCs miss breaches — the first being alert fatigue itself.

### Retire

Some rules should not be tuned. They should be deleted.

A rule is a candidate for retirement when:
- It has a true positive rate below 1% and has been tuned multiple times without improvement
- The behavior it detects is no longer relevant to your threat model
- It has been superseded by a better detection that covers the same technique
- The data source it depends on is no longer available or reliable

Before retiring a rule, document what technique it was covering and confirm that coverage exists elsewhere. Retiring a noisy rule that happens to be your only detection for T1059 is not tuning — it is creating a blind spot.

## Building Exclusion Lists Properly

Exclusion lists that live only in query logic are brittle. When you add the same exclusion to 15 different rules, you now have 15 places to update when that exclusion changes. Build shared watchlists instead.

In Sentinel, watchlists are lookup tables you can reference across multiple analytics rules. Create a watchlist called `TrustedProcesses` or `TrustedAdminHosts` and reference it from your detections:

```kql
let TrustedHosts = (
    _GetWatchlist('TrustedAdminHosts')
    | project SearchKey
);
SecurityEvent
| where EventID == 4624
| where LogonType == 3
| where not(Computer in (TrustedHosts))
```

When you onboard a new admin jump host, you update the watchlist once. All 15 rules that reference it update automatically. This is the difference between maintainable exclusions and exclusion debt.

## Measuring SIEM Health

You cannot tune what you do not measure. Track these metrics per detection rule, and roll them up weekly:

**True Positive Rate (TPR)**
`(Confirmed True Positives / Total Alerts) * 100`

Target varies by rule criticality. High-severity rules should be above 30% TPR. If a high-severity rule is below 10% and has been tuned multiple times, it is a retirement candidate.

**Mean Time to Triage (MTTT)**
How long from alert creation to analyst disposition (true positive / false positive / benign). MTTT above 30 minutes on most alerts is a workload problem. MTTT above 2 hours across the board suggests volume is overwhelming capacity.

**Alert Volume Trend**
Week-over-week alert volume per rule. Sudden spikes indicate either an active incident or a misconfigured rule. Sustained upward trends indicate your environment changed and the rule has not kept up.

**Suppression Rate**
What percentage of rule firings are being suppressed before reaching analysts? A suppression rate above 50% on a rule means the rule itself needs to be fixed, not the suppressions.

Review these metrics in a weekly SOC health check. Use them to prioritize tuning work. The rule with the highest volume, lowest TPR, and highest MTTT is your top tuning priority every week until it is fixed or retired.

## Tracking Tuning as Changes

Every tuning action is a change to a production detection. It should be tracked like any other change:

- What rule was modified?
- What was changed? (Show the before/after query diff)
- Why was the change made? (Link to the alert volume data or specific false positive examples)
- Who approved it?
- When does this exclusion get reviewed again?

Store this in your ticketing system or a detection engineering repository. If you do not, you end up in a situation six months later where a rule stops firing on real attacks and nobody knows why — because some exclusion was added silently and nobody documented it.

Use a Git repository for your KQL queries. Diff-based review is the right model for detection logic. Pull request for any rule change. Peer review before merge. This is not overhead — it is the difference between a detection library and a detection pile.

## When to Retire vs. Fix

The decision tree is simple:

**Retire when:**
- TPR has been below 5% for 90+ days despite multiple tuning attempts
- The underlying behavior being detected is no longer part of your threat model
- A better detection covering the same technique exists and is running

**Fix when:**
- The technique is still relevant but the rule is poorly scoped
- The exclusions exist but have not been implemented
- The data source quality is the problem, not the rule logic

One important note: never retire a rule without first confirming coverage elsewhere. Map the ATT&CK technique the rule covers. Confirm another rule or hunt covers the same technique. If you cannot confirm alternative coverage, the rule needs to be fixed, not retired — regardless of how much noise it generates.

## The Goal: Signal, Not Silence

The objective of SIEM tuning is not to reduce alert volume. It is to increase the signal-to-noise ratio. A SOC that receives 100 alerts per day with a 40% TPR is in better shape than a SOC that receives 2,000 alerts with a 2% TPR, even though the second SOC's analysts are "doing more."

The analysts who can investigate every alert, follow every lead, and trust that the queue contains real threats are the analysts who catch attackers. You get there through systematic, documented, metric-driven tuning. Not by adding suppressions until the queue is quiet.

**Practical Takeaways:**
- Alert fatigue is an engineering problem, not a people problem — fix the rules
- Suppress short-term, tune as the permanent fix, retire when the rule cannot be fixed
- Every suppression and exclusion needs an owner, an expiry, and documentation
- Build shared watchlists for exclusions that apply to multiple rules
- Measure TPR, MTTT, and volume trend per rule weekly
- Track all tuning actions as changes in version control or a ticket system
- Never retire a rule without confirming alternative coverage for the technique
