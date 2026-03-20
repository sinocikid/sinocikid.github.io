---
title: "Detecting Credential Abuse: From Password Spray to Pass-the-Hash"
date: 2026-03-25
author: "Andy"
categories:
- "Blue Team"
- "Identity Security"
tags:
- "credential-abuse"
- "Pass-the-Hash"
- "Kerberos"
- "Active-Directory"
- "blue-team"
description: "The credential attack spectrum explained — detection signals, event IDs, and behavioral baselines for each technique."
read_time: 10
---

# Detecting Credential Abuse: From Password Spray to Pass-the-Hash

> *Credentials are the most targeted asset in most environments. Know what their abuse looks like.*

---

## Why Credential Attacks Dominate

According to Verizon's DBIR and CrowdStrike's annual reports, credential abuse consistently ranks as the top attack vector in data breaches. The reason is simple: stolen credentials bypass most perimeter controls, look like legitimate activity, and provide persistent access without dropping malware.

The attack spectrum runs from brute-forcing to sophisticated Kerberos abuse. Each technique has distinct behavioral signals — but you need to know what to look for.

---

## The Credential Attack Spectrum

```
Low sophistication                                High sophistication
     │                                                     │
Brute Force → Password Spray → Credential Stuffing → Pass-the-Hash → Pass-the-Ticket → Golden Ticket
```

Detection difficulty increases left to right. Let's walk through each.

---

## Brute Force

**What it is**: High-volume authentication attempts against a single account.

**Detection signals**:
- Rapid sequence of Event ID 4625 (failed logon) against the same account
- Sub Status Code `0xC000006A` — wrong password (account exists)
- Account lockout events (4740) following repeated failures

**KQL Detection**:
```kql
SecurityEvent
| where TimeGenerated > ago(1h)
| where EventID == 4625
| where SubStatus == "0xc000006a"    // wrong password
| summarize FailureCount = count() by TargetAccount, IpAddress, bin(TimeGenerated, 5m)
| where FailureCount > 10
| order by FailureCount desc
```

**Tuning notes**: Adjust the threshold based on your lockout policy. If lockout triggers at 5 attempts, alert at 4. Ensure you're not alerting on locked-out accounts retrying repeatedly — filter on `0xC0000234` (already locked).

---

## Password Spray

**What it is**: Low-volume authentication attempts (1-3 per account) against many accounts using common passwords. Designed to stay under lockout thresholds.

**Why it's harder to detect**: No single account triggers enough failures to look suspicious. The signal is the *pattern* across accounts.

**Detection signals**:
- Many distinct accounts failing authentication from the same source IP in a short window
- Failures concentrated at specific times (often outside business hours to avoid immediate detection)
- Sub Status `0xC000006A` (wrong password) across many accounts

**KQL Detection**:
```kql
SecurityEvent
| where TimeGenerated > ago(1h)
| where EventID == 4625
| where SubStatus == "0xc000006a"
| summarize
    UniqueAccounts = dcount(TargetAccount),
    Attempts = count(),
    Accounts = make_set(TargetAccount, 20)
  by IpAddress
| where UniqueAccounts > 10     // spray threshold: many accounts from one source
| where Attempts > 15
| order by UniqueAccounts desc
```

**Entra ID / Azure AD signal**: Azure AD Sign-in logs show failed authentication with error code 50126 (invalid credentials). The `IPAddress` field and `UserPrincipalName` allow spray detection at the cloud layer even if on-premises logs miss it.

---

## Credential Stuffing

**What it is**: Using known username/password pairs from breached databases. Accounts reusing passwords from breached services are vulnerable.

**Why it's hardest to detect**: Successful authentications look completely legitimate — correct credentials, potentially from expected locations.

**Detection signals**:
- Successful logins from new geographic locations or ASNs
- Successful logins at unusual times for the account
- Multiple successful logins across accounts from the same IP in a short window
- Access from TOR exit nodes or known VPN/proxy ASNs

**KQL Detection**:
```kql
// Successful logins from unusual locations for each user
let BaselineLogons = SecurityEvent
    | where TimeGenerated between (ago(30d) .. ago(1d))
    | where EventID == 4624
    | where LogonType in (3, 10)
    | summarize BaselineIPs = make_set(IpAddress) by TargetAccount;

SecurityEvent
| where TimeGenerated > ago(1d)
| where EventID == 4624
| where LogonType in (3, 10)
| join kind=leftouter BaselineLogons on TargetAccount
| where not(IpAddress in (BaselineIPs))    // new source IP
| project TimeGenerated, TargetAccount, IpAddress, LogonType
```

---

## Pass-the-Hash (PtH)

**What it is**: Using an NTLM hash directly for authentication without knowing the plaintext password. The attacker dumps NTLM hashes from memory (typically via LSASS) and uses them to authenticate as that user.

**Why it works**: Windows NTLM authentication accepts the hash as proof of identity. The attacker doesn't need the password.

**Detection signals**:
- Event ID 4624 with LogonType 3, AuthenticationPackageName `NTLM`, and a blank or `-` NtlmDomainName
- The source machine's Netbios name may differ from what's expected for that account
- Pass-the-Hash typically generates 4648 (logon with explicit credentials) on the *source* machine

**KQL Detection**:
```kql
SecurityEvent
| where TimeGenerated > ago(1h)
| where EventID == 4624
| where LogonType == 3
| where AuthenticationPackageName == "NTLM"
| where TargetDomainName == "-" or TargetDomainName == ""
| where TargetUserName != "ANONYMOUS LOGON"
| where not(WorkstationName == Computer)    // source differs from target
| project TimeGenerated, Computer, TargetAccount = TargetUserName, IpAddress, WorkstationName
```

**Mitigation that also aids detection**: Enabling Protected Users group membership for sensitive accounts forces Kerberos-only authentication. NTLM logons from Protected Users members become an immediate anomaly signal.

---

## Pass-the-Ticket (PtT)

**What it is**: Stealing a valid Kerberos ticket (TGT or service ticket) and using it on a different machine. The ticket proves identity to Kerberos services without needing the password.

**Common sources of stolen tickets**: Mimikatz `kerberos::list /export`, Rubeus `harvest /interval:10`

**Detection signals**:
- Event ID 4769 (Kerberos Service Ticket Request) with encryption type 0x17 (RC4-HMAC) when your environment should be using AES
- Service ticket requests from machines that don't match the account's normal workstation
- TGT usage from a machine that never performed an initial TGT request (no corresponding 4768)

**KQL Detection**:
```kql
// Kerberos tickets using RC4 (weaker, often used in ticket attacks)
SecurityEvent
| where TimeGenerated > ago(1h)
| where EventID == 4769
| where TicketEncryptionType == "0x17"    // RC4-HMAC — should be AES in modern environments
| where ServiceName != "krbtgt"
| where not(AccountName endswith "$")    // exclude machine accounts
| summarize Count = count() by AccountName, IpAddress, ServiceName
| order by Count desc
```

---

## Golden Ticket

**What it is**: Forging a Kerberos TGT using the NTLM hash of the krbtgt account. Since the TGT is signed with the krbtgt key, any forged ticket that passes validation is accepted as legitimate. Provides indefinite, unchecked access as any user.

**Why it's catastrophic**: The forged ticket doesn't touch the domain controller for authentication — it goes directly to service providers. The attacker can impersonate any user, including non-existent ones. Tickets can be backdated. Without specific Golden Ticket detection, you will not see this in standard logs.

**Detection signals** (all indirect):
- Event ID 4769 with Ticket Options that don't match what a real TGT would produce
- Ticket lifetime values that exceed domain policy maximums
- Service ticket requests for accounts that don't exist in AD
- 4672 (Special Logon) on a DC for accounts that shouldn't be authenticating there
- Accounts appearing to be members of groups they're not actually in

**Most reliable detection**: Microsoft Defender for Identity (formerly Azure ATP) has specific Golden Ticket detection using behavioral baselining. If you're on-prem with a DC, MDI is worth the license cost for this capability alone.

**Post-compromise mitigation**: If you suspect a Golden Ticket attack, you must rotate the krbtgt account password **twice** with a 10-hour wait between rotations to invalidate all forged tickets (see the containment post for procedure).

---

## Honeypot Accounts as Tripwires

One of the highest-signal, lowest-noise detection mechanisms available: create accounts that should never be used.

**Implementation**:
1. Create a user account with a name that sounds legitimate (e.g., "svc-backup-old", "admin-deprecated", "svcMonitor2022")
2. Set a complex, random password
3. Do NOT add it to any groups or give it any access
4. Alert on ANY authentication attempt — successful or failed — involving this account

Any authentication attempt against a honeypot account is either:
- A password sprayer enumerating accounts
- An attacker who has already compromised something and is moving laterally
- A misconfigured application (investigate regardless)

```kql
// Alert on any activity involving honeypot accounts
let honeypotAccounts = dynamic(["svc-backup-old", "admin-deprecated", "svcMonitor2022"]);

SecurityEvent
| where TimeGenerated > ago(1h)
| where EventID in (4624, 4625, 4648, 4768, 4769)
| where TargetAccount in (honeypotAccounts)
    or SubjectUserName in (honeypotAccounts)
| project TimeGenerated, EventID, Computer, TargetAccount, IpAddress
```

This generates zero false positives. Every hit is worth investigating.

---

## Microsoft Entra ID Detection Capabilities

If your environment uses Entra ID (Azure AD) for authentication, leverage the built-in detection capabilities:

- **Identity Protection Risk Detections**: Impossible travel, leaked credentials, anonymous IP, unfamiliar sign-in properties
- **Risky Sign-ins**: Aggregated risk score for each authentication event
- **Risky Users**: Accounts with sustained risk signals

These detections use Microsoft's massive telemetry and ML models — they'll catch credential stuffing and spray patterns across your tenant that you'd never detect manually. They're not a replacement for log-based detections, but they're a powerful complement.

---

*Your credentials are your identity. If attackers have them, they are you. Next: [Memory Forensics for Blue Teamers](#) — finding credential theft artifacts in RAM.*
