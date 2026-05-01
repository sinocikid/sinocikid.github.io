---
title: "CASP+ to SecurityX: What Actually Changed (And What Didn't)"
date: 2026-05-07
author: "[REDACTED]"
categories:
- "Certifications"
- "Security Engineering"
tags:
- "SecurityX"
- "CASP+"
- "CAS-005"
- "CompTIA"
- "security-architecture"
description: "CompTIA rebranded CASP+ as SecurityX and updated the exam. Here's what's actually different in CAS-005, and whether your CASP+ credential still matters."
read_time: 7
---

# CASP+ to SecurityX: What Actually Changed (And What Didn't)

> *A rebrand alone wouldn't be newsworthy. The content changes in CAS-005 are.*

---

## The Rebrand and Why It Happened

In early 2024, CompTIA officially renamed CASP+ (CompTIA Advanced Security Practitioner) to **SecurityX**. The CAS-005 exam version launched alongside the rebrand.

This wasn't purely cosmetic. The "X" naming convention aligns SecurityX with CompTIA's broader expert-tier branding — and more importantly, the CAS-005 exam content was meaningfully updated from CAS-004 to reflect how the security landscape has shifted since 2022.

If you hold a CASP+ CE (CAS-004 or earlier), here's what changed, what it means for your credential's validity, and what the new exam looks like.

---

## Is Your CASP+ Credential Still Valid?

**Yes.** If you hold CASP+ (any version), your certification remains valid for the duration of your CE cycle. CompTIA doesn't invalidate older versions when new exams launch.

However:
- When you **renew**, you renew under the SecurityX CE program
- If you choose to **retake the exam** during renewal, you'll take CAS-005
- The renewal requirements are the same: 75 CEUs per 3-year cycle

For most CASP+ holders who are actively maintaining their credential, the practical impact is minimal. Your cert still says what it says, DoD 8140 compliance is maintained, and employers recognize it.

---

## Domain Comparison: CAS-004 vs CAS-005

### CAS-004 (CASP+) Domain Structure

| Domain | Weight |
|---|---|
| Security Architecture | 29% |
| Security Operations | 30% |
| Security Engineering and Cryptography | 26% |
| Governance, Risk, and Compliance | 15% |

### CAS-005 (SecurityX) Domain Structure

| Domain | Weight |
|---|---|
| Governance, Risk, and Compliance | 20% |
| Security Architecture | 30% |
| Security Engineering | 30% |
| Security Operations | 20% |

At first glance this looks similar. But the weight shifts tell a story:

- **GRC increased from 15% to 20%** — a 33% relative increase in emphasis
- **Security Operations decreased from 30% to 20%** — reduced relative weight
- **Security Engineering increased from 26% to 30%** — engineering depth got elevated
- **Architecture remains at approximately 30%** — consistent emphasis

The most notable change is the GRC bump. CompTIA explicitly recognized that expert-level practitioners need more depth in governance, risk quantification, and compliance frameworks — not less.

---

## What's New in CAS-005 Content

### 1. Zero Trust Architecture — Elevated to Core Concept

CAS-004 mentioned Zero Trust. CAS-005 treats it as a **fundamental architectural model** that practitioners at this level should be able to design around, evaluate, and critique. Expect questions on:

- Identity-centric vs. network-centric Zero Trust models
- NIST SP 800-207 implementation patterns
- Zero Trust maturity model assessment
- Integration with existing legacy environments (the hard part nobody talks about)

### 2. Cloud Security Architecture — Substantially Expanded

CAS-004 had cloud content. CAS-005 has *much more* cloud content. This reflects reality: expert-level security practitioners in 2024+ are working in hybrid and multi-cloud environments, not just on-premises networks.

New coverage areas:
- **Cloud-native security architectures** (serverless security, container security, service mesh)
- **CNAPP (Cloud-Native Application Protection Platforms)** — evaluation and integration
- **Multi-cloud security governance** — policy consistency across AWS, Azure, GCP
- **IaC (Infrastructure as Code) security** — scanning pipelines, policy-as-code, drift detection

If your security architecture knowledge skews heavily toward traditional network perimeter models, this is the most important new content area to address.

### 3. AI and Machine Learning Security Considerations

CAS-004 didn't address AI/ML security meaningfully. CAS-005 includes it — because expert practitioners in 2024+ need to understand:

- Security risks introduced by ML pipelines (training data poisoning, model extraction, adversarial inputs)
- AI-assisted security tooling evaluation criteria
- Governance considerations for AI-generated decisions in security systems (false positives, accountability)

This isn't a deep ML exam topic, but it's present and reflects CompTIA's effort to keep the content current.

### 4. Supply Chain Security

CAS-005 elevated supply chain security from a niche topic to a first-class concern. Coverage includes:

- Software supply chain attack surface (SolarWinds, Log4Shell pattern)
- SBOM (Software Bill of Materials) — creation, consumption, and risk assessment
- Third-party and vendor risk management at the architecture level
- Secure development lifecycle integration with supply chain assurance

### 5. Risk Quantification Methods

The GRC domain upgrade includes more depth on **quantitative risk analysis**, not just qualitative frameworks:

- FAIR (Factor Analysis of Information Risk) concepts
- Risk appetite vs. risk tolerance vs. risk threshold distinctions
- Loss magnitude estimation in security architecture decisions

This is content that CASP+ touched lightly. SecurityX goes deeper because architect-level practitioners should be able to quantify security risk in business terms.

---

## What Stayed the Same

### Cryptography Depth

The cryptography expectations in SecurityX remain substantive and unchanged in scope:

- PKI design and certificate lifecycle management
- Algorithm selection for specific use cases (symmetric, asymmetric, hybrid)
- Key management and escrow
- Cryptographic failures and their architectural implications
- TLS configuration and cipher suite selection

Don't skip cryptography prep assuming it was softened in the rebrand. It wasn't.

### Incident Response Integration

The Security Operations domain still covers IR at an expert level:

- Multi-vector incident coordination
- Forensic methodology and evidence preservation at scale
- Threat intelligence integration into IR workflows
- Post-incident review and architectural remediation

### Security Architecture Fundamentals

Network segmentation, microsegmentation, identity federation, IAM design, and hybrid architecture patterns remain core content. The fundamentals didn't change — they just got augmented with cloud-native extensions.

---

## Study Material Implications

If you're preparing for CAS-005, **verify that your study materials are updated for CAS-005, not CAS-004**. The differences aren't trivial — cloud architecture, Zero Trust depth, and supply chain coverage are meaningfully expanded.

Signs your materials are outdated:
- No coverage of CNAPP or cloud-native application security
- Zero Trust is mentioned but not treated as a primary architectural model
- No AI/ML security considerations
- Limited supply chain security content
- SBOM is absent

Material published before mid-2024 is likely based on CAS-004. Use it as a supplement, not a primary source.

---

## For CASP+ Holders Approaching Renewal

Your options at renewal time:

**Option 1: CEU-based renewal (most common)**
Accumulate 75 CEUs through relevant activities before your expiration date. No exam required. Your SecurityX CE status renews.

**Option 2: Retake the exam**
Pass CAS-005 before your expiration date. Resets your cycle.

**Option 3: Let it expire, then recertify**
Not recommended — you lose the CE status and must earn the credential from scratch.

Most practitioners should pursue Option 1. If you're actively working in security, you're accumulating CEUs through professional development, related certifications, and technical work anyway. The renewal cycle rewards active practitioners.

---

## Bottom Line

The SecurityX rebrand is more than cosmetic. CAS-005 meaningfully updated the content to reflect current reality: Zero Trust is architecture, not aspiration; cloud security is non-optional; supply chain is a first-class threat surface; and AI introduces new risks that expert practitioners need to understand.

If you hold CASP+, your credential is valid and respected. If you're studying for the exam now, study for CAS-005 with current materials. If you're renewing, build your CEU plan and maintain the credential you've earned.

The name changed. The seriousness didn't.

---

*Last updated: May 2026*
