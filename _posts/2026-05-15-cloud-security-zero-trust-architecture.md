---
title: "Cloud Security: Implementing Zero Trust Architecture"
date: 2026-05-15
categories: ["Cloud"]
tags: ["Cloud Security", "Zero Trust", "IAM", "AWS", "Azure", "Security"]
read_time: 9
---

The traditional security model assumed a trusted internal network and an untrusted external one. You built a firewall, protected the perimeter, and trusted everything inside. That model is dead. It was already dying before cloud-native architecture distributed workloads across multiple providers, regions, and networks. Cloud environments need a fundamentally different mental model — one built on the assumption that no network, no host, and no user should be inherently trusted.

That model is Zero Trust. This post breaks down what Zero Trust actually means in cloud environments, how to implement its core components, and what a practical roadmap looks like.

## Zero Trust Principles

Zero Trust is often reduced to the phrase "never trust, always verify." The practical implications:

- **Authenticate and authorize every request**, regardless of where it originates — inside or outside the network perimeter
- **Assume breach**: Design systems as if attackers are already inside. Limit what they can access when they are.
- **Least privilege**: Every human, service, and process gets only the permissions needed to do its specific job — nothing more
- **Verify explicitly**: Use all available signals — identity, device health, location, behavior — not just network location
- **Microsegmentation**: Limit lateral movement by isolating workloads and enforcing access controls at fine granularity

The network perimeter is no longer meaningful. Identity is the new perimeter.

## Identity as the New Perimeter: IAM Best Practices

In cloud environments, IAM (Identity and Access Management) is your primary security control. Getting it wrong is catastrophic — overpermissioned roles and unrotated credentials are the root cause of a significant percentage of cloud breaches.

### AWS IAM

Fundamental rules for AWS IAM:

- **Never use root credentials** for anything. Create an admin user, enable MFA on root, then lock it away.
- **Use IAM roles, not users, for machine identities**. EC2 instances, Lambda functions, and ECS tasks should assume roles via instance profiles or execution roles — never have access keys baked into them.
- **Enforce MFA** for all human users, especially for privileged actions. Use IAM policy conditions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

- **Use permission boundaries** to cap the maximum permissions a role can have, even if policies would otherwise allow more
- **Enable AWS Organizations SCPs** (Service Control Policies) to enforce guardrails across your entire organization — deny specific regions, require encryption, block disabling of CloudTrail

### Azure and GCP

Azure uses Azure Active Directory (now Entra ID) with Role-Based Access Control. The same principles apply: use Managed Identities instead of service principals with secrets where possible, enforce Conditional Access policies that check device compliance and location, and use Privileged Identity Management (PIM) for just-in-time elevation of privileged roles.

GCP's IAM uses resource hierarchy (Organization → Folder → Project) with role bindings at each level. Workload Identity Federation lets GCP workloads authenticate using external identity providers without service account keys.

## Least Privilege: RBAC, ABAC, and Just-in-Time Access

### RBAC (Role-Based Access Control)

RBAC assigns permissions to roles and assigns roles to users or services. It's well-understood and audit-friendly. The risk is role accumulation over time — users collect roles, roles accumulate permissions, and gradually everyone has more access than they need.

Audit your IAM roles regularly. AWS IAM Access Analyzer and the IAM console's "last accessed" data are invaluable: if a role hasn't used a permission in 90 days, consider removing it.

### ABAC (Attribute-Based Access Control)

ABAC grants access based on attributes — resource tags, user attributes, environment context. In AWS, you can write policies that allow actions only on resources tagged with the same team tag as the user's session:

```json
{
  "Effect": "Allow",
  "Action": "ec2:*",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "ec2:ResourceTag/Team": "${aws:PrincipalTag/Team}"
    }
  }
}
```

This scales better than RBAC for large organizations because you don't need to create a new role for every team-resource combination.

### Just-in-Time Access

Standing access — permissions that exist permanently — is a security liability. Compromise a credential and you get immediate persistent access. Just-in-time (JIT) access means users request elevated access for a specific task, get it temporarily, and it expires automatically.

Tools: AWS IAM Identity Center with time-limited permission sets, Azure PIM, HashiCorp Boundary for infrastructure access, or Teleport for SSH/Kubernetes/database access. JIT access combined with mandatory approval workflows creates an audit trail for every privileged action.

## Network Segmentation: VPC Design and Private Endpoints

Even in a Zero Trust model, network segmentation provides defense in depth. Compromised credentials can't reach what they can't route to.

**VPC design principles:**

- Separate VPCs (or at least subnets) for production, staging, and development
- Never put production databases in public subnets
- Use private subnets for compute and data tiers; public subnets only for load balancers and NAT gateways
- Use VPC endpoints (AWS), Private Endpoints (Azure), or Private Service Connect (GCP) so traffic to cloud services like S3 or Secrets Manager stays within the cloud backbone, not the internet

**Security groups as a compensating control**: Treat security groups as microsegmentation. Define rules at the service level — the order service security group allows inbound on port 8080 only from the API gateway security group. No wide-open rules.

**AWS PrivateLink** is particularly valuable: it lets you expose services in one VPC to consumers in another VPC (or to on-premises) without peering or public internet exposure. Services aren't reachable from the public internet at all.

## Secrets Management

Hardcoded secrets in source code, environment variables in CI systems, credentials in container images — these are all security anti-patterns that regularly lead to breaches.

**HashiCorp Vault**: The most flexible secrets management platform. Vault issues dynamic secrets (time-limited, generated on demand), manages PKI certificates, encrypts data with a KMS-like API, and integrates with cloud IAM for authentication. The dynamic secrets feature is powerful: instead of a long-lived database password, your application gets a unique, time-limited credential generated at startup.

```bash
# Vault issues a dynamic PostgreSQL credential
vault read database/creds/my-role
# Key                Value
# ---                -----
# lease_duration     1h
# username           v-app-xyz-AbC123
# password           A1b2-C3d4-E5f6
```

**AWS Secrets Manager**: Native to AWS, with automatic rotation for RDS credentials and integration with IAM for access control. Simpler than Vault for AWS-only environments.

**Azure Key Vault**: Stores secrets, certificates, and keys with HSM backing available. Managed Identities make secret retrieval seamless for Azure workloads.

**What to avoid**: `.env` files in version control, secrets in environment variables set at the system level (visible in `/proc`), secrets in container image layers, and anywhere that secrets appear in logs.

## Workload Identity

Workload identity is about giving your applications and services cryptographically verifiable identities — so they can authenticate to other services without credentials stored anywhere.

**Kubernetes Service Accounts with IRSA (AWS)**: IRSA (IAM Roles for Service Accounts) lets pods assume IAM roles using OIDC federation. The pod proves its identity using a projected service account token; AWS STS validates it against the cluster's OIDC provider and issues short-lived credentials. No secrets stored anywhere.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/order-service-role
```

**GCP Workload Identity**: Similar mechanism — Kubernetes service accounts are bound to GCP service accounts via annotations. Pods get GCP credentials automatically.

**Azure Workload Identity**: AKS pods authenticate to Azure AD using federated credentials, without any client secrets.

The common thread: short-lived, automatically rotated credentials tied to workload identity, not to a person or a long-lived secret.

## Audit Logging and SIEM Integration

You can't detect what you don't log. In cloud environments, the audit log is your source of truth for who did what, when.

- **AWS CloudTrail**: Logs all API calls. Enable it in all regions, send logs to a dedicated S3 bucket in a separate security account, and enable CloudTrail Insights for anomaly detection.
- **AWS GuardDuty**: Threat detection that analyzes CloudTrail, VPC Flow Logs, and DNS logs for suspicious patterns.
- **Azure Monitor + Microsoft Sentinel**: Centralizes logs across Azure resources with built-in threat intelligence.
- **GCP Cloud Audit Logs + Security Command Center**: Equivalent for GCP environments.

For cross-cloud visibility, aggregate logs into a SIEM (Splunk, Elastic Security, or AWS Security Lake). Define detection rules for:

- IAM role assumption from unusual locations or at unusual times
- S3 bucket policy changes (especially public access grants)
- Security group rules opening broad internet access
- Disabling of logging or security services
- Use of root credentials

Alert on anomalies, not just known-bad patterns. Establish a baseline of normal behavior for each service account and role, then alert on deviation.

## Practical Zero Trust Checklist

Use this as a starting point for assessing your current posture:

**Identity and Access**
- [ ] MFA enforced for all human users
- [ ] Root/admin credentials protected, MFA enforced, alerts on use
- [ ] No long-lived access keys for machine identities (use roles/workload identity)
- [ ] IAM Access Analyzer enabled; unused permissions reviewed quarterly
- [ ] JIT access implemented for privileged operations

**Network**
- [ ] Production workloads in private subnets
- [ ] VPC endpoints for cloud services (no public internet for S3, Secrets Manager, etc.)
- [ ] NetworkPolicy (or equivalent) enforcing service-to-service communication rules
- [ ] No 0.0.0.0/0 inbound rules except for load balancers on 443

**Secrets**
- [ ] No hardcoded secrets in code, CI environment variables, or container images
- [ ] Secrets stored in a managed secrets service with IAM-controlled access
- [ ] Automatic rotation configured for database credentials

**Workloads**
- [ ] Workload identity (IRSA, Workload Identity, Managed Identity) configured for all cloud-accessing services
- [ ] Container images scanned for vulnerabilities in CI pipeline
- [ ] SBOM (Software Bill of Materials) generated for production images

**Detection**
- [ ] CloudTrail/Azure Monitor/Cloud Audit Logs enabled in all regions
- [ ] Logs shipped to immutable storage in a separate account
- [ ] SIEM with alerting rules for high-risk events
- [ ] GuardDuty or equivalent threat detection enabled

## Conclusion

Zero Trust is not a product you buy — it's an architectural posture you build incrementally. The checklist above won't be fully satisfied overnight. Start with identity: enforce MFA, eliminate long-lived machine credentials, and audit your IAM roles for excessive permissions. Then layer in network controls, secrets management, and monitoring.

The payoff is significant. Organizations with mature Zero Trust implementations have dramatically smaller blast radius when credentials are compromised, far better visibility into what's happening in their environment, and a much clearer answer to the auditor's question: "who had access to what, and what did they do with it?"

Build the controls, build the logs, and build the habit of reviewing both.
