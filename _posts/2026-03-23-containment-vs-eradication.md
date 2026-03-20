---
title: "Containment vs. Eradication: Getting the Sequence Right in Incident Response"
date: 2026-03-23
author: "Andy"
categories:
- "Blue Team"
- "Incident Response"
tags:
- "Incident Response"
- "Containment"
- "Eradication"
- "EDR"
- "Forensics"
description: "Sequence matters in incident response. Acting too fast tips off the attacker. Acting too slow gives them time to dig in deeper. Here is how to think through containment and eradication decisions under pressure."
read_time: 9
---

I have seen incident response go wrong in two directions. The first is the team that isolates the endpoint the moment they get an alert, before anyone understands what they are dealing with. The attacker notices their beacon died, pivots to a secondary implant on a different host, and now the team is starting over with less visibility. The second is the team that spends so long observing and gathering evidence that the attacker finishes exfiltrating the data while they are still in the analysis phase.

Both failures come from the same root cause: not understanding that containment and eradication are distinct phases with different objectives, and that sequence is not optional — it is the job.

## Why Sequence Matters

Containment and eradication are not synonyms. They are sequential phases with different goals:

- **Containment** stops the bleeding. It limits the attacker's ability to move, persist, or exfiltrate while you still have visibility into what they are doing.
- **Eradication** removes the attacker from the environment. It cleans up implants, closes access vectors, and returns systems to a known-good state.

You cannot eradicate effectively without first containing. If you start deleting malware files while the attacker still has an active session, they will reinstall it, pivot, or take destructive action. And you cannot safely contain without first understanding what you are containing. If you do not know all the attacker's footholds, you will contain one and leave three others active.

The gap between "we know the attacker is here" and "we start containment" is where intelligence gathering happens. That window has a cost — the attacker may be doing damage during it — but acting without that intelligence has a higher cost.

## What Containment Actually Means

Containment is a collection of actions, not a single button. Depending on the incident, you might use any combination of:

**Network Isolation**
Isolating a host from the network via EDR is the fastest and cleanest option when available. In CrowdStrike Falcon, this is `containment` from the console or via API. In Microsoft Defender for Endpoint, it is "Isolate device." The host can still talk to the EDR cloud (so you maintain visibility and control) but all other network traffic is blocked. This is the containment action of choice for confirmed malicious hosts.

**Account Disable**
If the attacker is using a compromised credential, disabling or rotating that account cuts the authentication path. In Active Directory: `Disable-ADAccount -Identity <username>`. In Azure AD / Entra ID, disable the account in the portal or via `Set-AzureADUser -ObjectId <id> -AccountEnabled $false`. Do this for all accounts the attacker is known to have used, not just the first one you found.

**Firewall Rules / Network ACLs**
Blocking specific source IPs, destination IPs, or C2 domains at the network layer. Less surgical than EDR isolation but useful when the attacker is operating from multiple hosts you cannot individually isolate quickly. Block at the perimeter firewall, internal segmentation firewall, or DNS layer depending on what the C2 traffic looks like.

**Credential Rotation**
For service accounts, API keys, or admin credentials the attacker may have accessed: rotate on a scheduled basis once you know the scope, not immediately when you first suspect compromise. Rotating a credential prematurely signals to the attacker that they have been detected.

## The Danger of Tipping Off the Attacker

Advanced threat actors monitor for signs of detection. Sudden loss of a C2 beacon, account password resets, unexpected network blocks — these are detection signals to the attacker. If they detect your detection, they have options:

- Activate secondary implants on hosts you have not identified yet
- Accelerate exfiltration to beat your containment timeline
- Take destructive action (ransomware deployment, log deletion) as a last resort
- Go quiet and wait for your containment to be lifted

This is why "contain everything immediately" is a bad default. It feels decisive. It sometimes is the right call. But it should be a deliberate choice based on the attacker's sophistication and your current intelligence, not a reflex.

## When to Contain Immediately vs. When to Observe

**Contain immediately when:**
- Ransomware deployment is in progress or imminent
- Active data exfiltration is confirmed and ongoing
- The attacker has shown willingness to take destructive action
- You have full scope awareness — you know all the footholds
- The incident is a commodity threat (script kiddie, automated malware) that does not adapt to detection

**Observe before containing when:**
- The attacker is a sophisticated actor with likely secondary access
- You do not have full scope — you have found one foothold but suspect more
- The business can tolerate the observation window (i.e., no active exfiltration)
- The attacker is dormant and you have time to map the full compromise

The practical answer to "how long do we observe" is: as long as it takes to achieve scope confidence, up to a maximum that the business can tolerate. In most organizations with a competent IR team, that window is 24-72 hours. After that, the risk of continued attacker access exceeds the intelligence benefit.

## The Eradication Checklist

Eradication only begins when you have scope confidence — when you have reasonable certainty you have identified all attacker footholds, not just some of them. A partial eradication is often worse than no eradication: the attacker sees the partial cleanup, activates the assets you missed, and now operates more cautiously.

Work through this checklist before declaring eradication complete:

- [ ] All identified malware binaries, scripts, and artifacts removed or quarantined
- [ ] All identified persistence mechanisms removed: scheduled tasks, registry run keys, WMI subscriptions, services, startup folders, modified DLLs
- [ ] All compromised accounts disabled and credentials rotated (user accounts, service accounts, API keys, certificates)
- [ ] All attacker-created accounts removed from AD/Entra ID
- [ ] All identified C2 domains and IPs blocked at network layer
- [ ] EDR policy confirmed active and reporting on all affected hosts
- [ ] No active network connections to known C2 infrastructure from any host
- [ ] Patch or remediate the initial access vulnerability that was exploited

Tools that help here: your EDR's threat hunting interface (timeline view showing all attacker activity on each host), Velociraptor for forensic artifact collection at scale, BloodHound to identify what AD paths the attacker may have abused, and network flow data to map all C2 destinations.

## Clean vs. Recovered: Know the Difference

A system is **recovered** when it is back in service. A system is **clean** when you have confirmed it no longer contains attacker artifacts. These are not the same thing.

Recovering a system quickly by restoring from backup is fine — as long as that backup predates the initial compromise. Restoring from a snapshot taken after the attacker established persistence means you have restored their foothold along with the system.

Always confirm the backup date against the estimated initial access timestamp before restoring from it. If you are unsure when initial access occurred, assume the earliest evidence date and work backward. When in doubt, rebuild from a known-clean gold image.

## Post-Eradication Validation

After eradication, you need to confirm the attacker is actually gone. This is not the time to trust your own checklist. Run independent validation:

**EDR sweep**: Run a full hunt across the environment using the IOCs and TTPs from the incident. Look for any remaining instances of the malware signatures, persistence mechanisms, or C2 connections you identified.

**Network validation**: Pull 24-48 hours of post-eradication firewall/proxy logs and confirm zero traffic to known C2 infrastructure. If you see it, you missed something.

**Authentication audit**: Review all privileged authentications for 24-48 hours post-eradication. Look for the compromised accounts being used (they should be disabled), or any unusual authentication patterns from previously clean hosts.

**Vulnerability confirmation**: Confirm the initial access vector is actually closed. If it was a phishing link that delivered a dropper, that is one thing. If it was an unpatched internet-facing system, confirm the patch was applied and the service is no longer vulnerable.

Declare eradication complete only after validation passes. Not after the checklist is done.

**Practical Takeaways:**
- Containment and eradication are sequential, not simultaneous — respect the order
- Contain immediately for ransomware and active exfiltration; observe for sophisticated actors with unknown scope
- Do not rotate credentials or block C2 until you are ready to move to containment — it tips off the attacker
- Eradication requires scope confidence first; partial eradication is often worse than none
- Know the difference between clean and recovered before you restore systems
- Post-eradication validation is mandatory, not optional
