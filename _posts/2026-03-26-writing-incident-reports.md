---
title: "Writing Incident Reports That Actually Help: A Blue Team Template"
date: 2026-03-26
author: "Andy"
categories:
- "Blue Team"
- "Incident Response"
tags:
- "incident-response"
- "documentation"
- "reporting"
- "IR-template"
- "blue-team"
description: "Most IR reports are either too technical for executives or too vague for engineers. Here's the structure and template that serves both."
read_time: 9
---

# Writing Incident Reports That Actually Help: A Blue Team Template

> *The incident is over. Now write a report that actually prevents the next one.*

---

## Why Most IR Reports Fail

After a difficult incident — sleepless nights, escalations, frantic containment — the last thing anyone wants to do is write a report. The report gets written in a hurry, reflects the exhaustion of the team, and ends up being one of these:

**The Timeline Dump**: A chronological list of every alert, every action, every note from the ticket system. Technically accurate. Practically unreadable. Nobody learns anything.

**The Blame Report**: Focuses on what went wrong and who made what mistake. Politically damaging, drives defensive behavior in future incidents, doesn't improve the system.

**The Technical Deep Dive That Only Engineers Read**: Eight pages of KQL queries and Volatility output. The executive team ignores it. Leadership never allocates resources for the improvements it recommends.

**The Marketing Report**: "We responded swiftly and effectively." Nothing specific, nothing actionable, nothing learned.

Good incident reports do three things:
1. Create a shared, accurate record of what happened
2. Identify specific, implementable improvements
3. Communicate findings appropriately to different audiences

---

## The Three Audiences

Before you write a single word, decide who will read this report.

**Executive / Leadership**
What they care about: business impact, financial exposure, regulatory implications, what changed to prevent recurrence. They do not want technical details. They want confidence that the situation was handled professionally and that concrete steps are being taken.

**Technical / Engineering**
What they care about: root cause, detection gaps, what worked and what didn't, specific improvements to tools and processes. They want enough detail to implement the recommendations.

**Legal / Compliance**
What they care about: timeline and scope of data exposure, notification obligations, evidence preservation, documentation of the response process. They want accuracy and completeness, not analysis.

One report should serve all three audiences — but each section should be written with its primary audience in mind.

---

## The Structure That Works

```
1. Executive Summary          ← Leadership reads this
2. Incident Timeline          ← Everyone needs this
3. Technical Analysis         ← Engineers read this
4. Root Cause Analysis        ← Everyone needs this
5. Remediation Actions        ← Engineers implement, leadership approves
6. Lessons Learned            ← Future-facing — most valuable section
7. Appendix                   ← Supporting evidence, IOCs, queries
```

---

## Section by Section

### 1. Executive Summary (1 page maximum)

Write this last, even though it goes first.

Answer these questions in plain language:
- **What happened**: One paragraph. No jargon. "An attacker gained access to our corporate network using compromised employee credentials and accessed the HR file server for approximately 6 hours."
- **When it happened**: Date range, duration of attacker presence.
- **What was affected**: Systems, data, accounts. Be specific about what was confirmed vs. suspected.
- **Business impact**: Operational disruption, data exposure (volume and type), financial impact if quantifiable, regulatory implications.
- **What we did**: Brief summary of response actions.
- **Current status**: Is the incident resolved? Are there ongoing risks?
- **Key recommendations**: 2-3 bullet points, high level.

Executive summary example paragraph:

> *On March 15, 2026, a threat actor used valid credentials belonging to a compromised employee account to gain unauthorized access to our corporate network. The attacker was present for approximately 14 hours before detection. During this time, they accessed the HR file server and accessed files containing personnel data for approximately 240 employees. No evidence of data exfiltration was found; however, this cannot be confirmed with certainty. The attacker's access was terminated on March 16 at 03:42 UTC following EDR detection of lateral movement activity. All affected accounts have been disabled and passwords reset across the domain.*

### 2. Incident Timeline

The timeline is the core of the report. It establishes facts and serves as a reference for all other sections.

**Format:**

| Date/Time (UTC) | Event | Source |
|---|---|---|
| 2026-03-14 22:15 | Attacker authenticates to VPN using jsmith credentials from IP 45.67.89.10 | Azure AD sign-in logs |
| 2026-03-14 22:18 | First successful authentication to internal network | Firewall logs |
| 2026-03-14 22:31 | Lateral movement to HR-FILE-01 via SMB (Event ID 4624 Type 3) | Windows Security logs |
| 2026-03-15 07:43 | EDR alert: suspicious process execution on HR-FILE-01 | CrowdStrike |
| 2026-03-15 07:51 | SOC analyst begins investigation | Ticket system |
| 2026-03-15 08:20 | Incident escalated to Tier 2 | Ticket system |
| 2026-03-16 03:42 | Affected accounts disabled, hosts isolated | Active Directory / EDR |
| 2026-03-16 04:15 | Incident declared contained | IR call recording |

**Timeline rules:**
- Use UTC consistently. Mix of timezones creates confusion and legal problems.
- Include the source of each data point — this is how you defend the timeline in a legal context.
- Include attacker actions AND defender actions in the same timeline.
- Be precise about what you know vs. what you inferred.

### 3. Technical Analysis

This section is for the engineers who will implement improvements.

Include:
- **Initial access vector**: How did the attacker get in? Specific technique, CVE if applicable, evidence.
- **Persistence mechanisms**: What did they establish? Event IDs, file paths, registry keys.
- **Lateral movement**: Which hosts were accessed, in what order, via what technique.
- **Actions on objectives**: What did the attacker actually do? File access, data access, tool deployment.
- **Detection path**: What triggered the alert? Why did it take as long as it did?
- **Indicators of Compromise (IOCs)**: IPs, domains, file hashes, Sigma rules. (Full list in Appendix)

Write this with enough detail that an engineer who wasn't on the response team can understand exactly what happened.

### 4. Root Cause Analysis

Not "how did the attacker get in" — that's the technical analysis. Root cause is "why were we vulnerable to this attack and unable to detect it sooner."

**Use the 5 Whys:**

*The attacker accessed the HR file server for 14 hours before detection.*

Why? → Our EDR alert for suspicious file access was suppressed for HR-FILE-01 because it was flagged as a "trusted server" in a blanket exclusion.

Why? → The exclusion was added 8 months ago by a SOC analyst to stop a specific false positive, without proper peer review.

Why? → Our change management process for EDR exclusions didn't require approval for individual suppression rules.

Why? → The EDR exclusion management process was never formally defined when we deployed the tool.

Why? → When we deployed EDR, we prioritized endpoint coverage over process documentation.

**Root cause**: Absence of a formal EDR exclusion management process allowed security controls to be weakened without oversight.

This is the root cause. The fix is a process, not a technical tweak.

### 5. Remediation Actions

Two categories:

**Immediate (completed during incident):**
- ✅ Disabled compromised account (jsmith) — completed 2026-03-16 03:42 UTC
- ✅ Isolated affected hosts via EDR — completed 2026-03-16 04:00 UTC
- ✅ Reset domain administrator passwords — completed 2026-03-16 05:15 UTC

**Ongoing / Planned:**
- [ ] Implement EDR exclusion management process with peer review requirement — Owner: SOC Manager, Due: 2026-04-15
- [ ] Review and audit all existing EDR exclusions for HR-FILE-01 and high-risk servers — Owner: Detection Team, Due: 2026-04-01
- [ ] Enable MFA for VPN authentication — Owner: IT Operations, Due: 2026-03-31
- [ ] Deploy honeypot accounts for early lateral movement detection — Owner: Detection Team, Due: 2026-04-30

Each planned action needs an owner and a due date. Unowned action items don't happen.

### 6. Lessons Learned

The most valuable and most neglected section.

Three perspectives:

**What worked well:**
- EDR lateral movement detection worked exactly as designed once the suppression was removed
- The Tier 2 escalation process was fast and effective
- Communication with legal team was established within 2 hours of escalation

**What didn't work:**
- Suppression management process allowed critical detection to be silently disabled
- Initial triage took 29 minutes — too long for an active intrusion alert
- HR-FILE-01 was not included in our high-value asset inventory, so it didn't receive enhanced monitoring

**What we'll do differently:**
- Monthly audit of all SIEM and EDR exclusions with documented business justification
- Reduce target triage time for active intrusion alerts to < 10 minutes
- Add all file servers containing HR/finance data to high-value asset group for enhanced monitoring

### 7. Appendix

Supporting material that doesn't belong in the main body:

- Full IOC list (IPs, domains, hashes, file paths)
- Detection queries used during investigation (Sentinel/KQL/Sigma)
- Evidence artifacts and file hash inventory
- Network diagrams of attacker movement path
- Log excerpts supporting timeline entries
- Communication log (who was notified and when)

---

## Writing Style Rules

**Be specific**: "The attacker accessed 47 files" is better than "some files were accessed."

**Distinguish confirmed from probable**: "The attacker exfiltrated data" vs "The attacker may have exfiltrated data — no evidence of exfiltration was found, but cannot be ruled out."

**Use past tense consistently**: The incident is over. Write it that way.

**Avoid jargon in the executive summary**: "Lateral movement" → "moved from one computer to another within our network."

**Don't assign personal blame**: "The exclusion was added without peer review" not "John added the exclusion without asking anyone."

**Include what you don't know**: Gaps in visibility are findings. "We cannot confirm whether data was exfiltrated between 22:15 and 03:42 UTC due to insufficient proxy log retention" is critical information for leadership and legal.

---

## A Reusable Template

```markdown
# Incident Report: [Incident Name / ID]
**Classification**: [Confidential / Restricted / Internal]
**Report Date**: 
**Incident Date**: 
**Severity**: [Critical / High / Medium / Low]
**Status**: [Closed / Monitoring]

---

## 1. Executive Summary
[Plain language summary — max 1 page]

## 2. Incident Timeline
| Time (UTC) | Event | Source |
|---|---|---|

## 3. Technical Analysis
### Initial Access
### Persistence
### Lateral Movement
### Actions on Objectives
### Detection

## 4. Root Cause Analysis
[5 Whys or equivalent]

## 5. Remediation Actions
### Completed
- 

### Planned
| Action | Owner | Due Date | Status |
|---|---|---|---|

## 6. Lessons Learned
### What Worked
### What Didn't Work
### What We'll Do Differently

## 7. Appendix
### IOCs
### Detection Queries
### Evidence Inventory
```

---

## The Report Is Part of the Incident

The incident isn't over when you contain the attacker. It's over when you've documented what happened, identified what to fix, and assigned ownership of those fixes. The report is the mechanism for that.

Write it while the details are fresh. A report written three weeks later is missing half the context and most of the lessons.

And the best time to start writing? The moment the incident is declared contained. The timeline is already in your ticket system. Copying it into a structured template takes 30 minutes. The rest follows.

---

*This post concludes the Blue Team Defensive Strategy series. If you found it useful, the [full series index](#) covers SOC triage, detection engineering, LOLBin detection, event logging, KQL, threat hunting, containment/eradication, SIEM tuning, tools, network detection, credential abuse, memory forensics, and lab setup.*
