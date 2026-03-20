---
title: "Network Threat Detection with Zeek and Suricata: A Practical Setup Guide"
date: 2026-03-24
author: "Andy"
categories:
- "Blue Team"
- "Network Security"
tags:
- "Zeek"
- "Suricata"
- "network-detection"
- "IDS"
- "blue-team"
description: "Why network visibility still matters even in EDR-heavy environments, and how to set up Zeek and Suricata to actually find threats."
read_time: 10
---

# Network Threat Detection with Zeek and Suricata: A Practical Setup Guide

> *EDR sees what happens on endpoints. The network sees what endpoints talk to. You need both.*

---

## Why Network Detection Still Matters

In a world where EDR is on every endpoint and cloud security logs everything, it's tempting to deprioritize network visibility. Don't.

Network detection catches things that endpoint tools miss:

- **Unmanaged devices** — IoT, OT equipment, BYOD, guest networks, rogue systems. Your EDR isn't on these.
- **Encrypted but suspicious traffic** — Even with TLS, network tools can fingerprint C2 frameworks by JA3/JA3S hashes, certificate characteristics, and connection patterns.
- **Lateral movement** — SMB, RPC, and Kerberos traffic between endpoints often goes unmonitored by individual host EDR agents.
- **Data exfiltration** — High-volume outbound transfers, DNS tunneling, and FTP exfil are visible at the network layer regardless of what the endpoint reports.
- **EDR blind spots** — EDR has bugs, gaps, and exclusions. Defenders who only rely on EDR are one exclusion away from being blind.

Zeek and Suricata are the two tools that fill this gap. They're complementary — not competing.

---

## Zeek vs. Suricata: Different Jobs, Not Alternatives

This is the most common misconception: people treat these as competing products and pick one. They should both be running.

| | Zeek | Suricata |
|---|---|---|
| **Primary model** | Protocol analysis → structured logs | Signature/rule matching → alerts |
| **Output** | Rich log files (conn.log, dns.log, etc.) | Alerts with metadata (eve.json) |
| **Strength** | Contextual data for hunting and SIEM | Real-time detection of known patterns |
| **Weakness** | Doesn't alert by default — requires analysis | Alert-heavy without tuning, less context |
| **Use case** | Threat hunting, SIEM enrichment, baselining | IDS/IPS, known-bad detection |

Run both on a mirror/tap port. Zeek feeds your SIEM and hunting platform. Suricata provides real-time signature-based alerting.

---

## Zeek: Understanding the Logs

Zeek's value is in its log files. Each protocol gets its own log, structured as TSV or JSON. These integrate directly into Elastic, Splunk, Microsoft Sentinel, or any log management platform.

### The Logs You Care About

**conn.log** — Every network connection: source/dest IP, port, protocol, bytes, duration, connection state.

```
ts          uid         id.orig_h    id.orig_p  id.resp_h    id.resp_p  proto  duration  orig_bytes  resp_bytes
1711234567  CXXXabc123  10.1.1.50    54231      1.2.3.4      443        tcp    0.23      1240        85400
```

This is your baseline. Unusual connections, unexpected destinations, large data transfers — all visible here.

**dns.log** — DNS queries and responses: query name, query type, answer, response code, TTL.

```
# Suspicious: very long subdomain (possible DNS tunneling)
query: aGVsbG8gd29ybGQ.attacker-domain.com  type: A
```

Alert on: queries to newly registered domains, unusually long subdomains (> 60 chars), high query rates to a single domain, NX response rates (failed lookups = possible DGA).

**http.log** — HTTP requests: method, URI, host, user agent, referrer, response code, body size.

Look for: non-browser user agents making regular requests, large POST bodies (exfiltration), requests to IP addresses without hostnames, suspicious URIs.

**ssl.log** — TLS connections: server name (SNI), JA3/JA3S hashes, certificate validity, cipher suites.

The JA3 hash fingerprints the TLS client implementation. Known malware families have characteristic JA3 hashes. Cross-reference against JA3er.com or Salesforce's JA3 signature database.

**files.log** — Files transferred over HTTP, FTP, SMTP, and other protocols. Includes MD5/SHA1/SHA256 of transferred files.

Critical for identifying malware downloads and data exfiltration. Submit hashes to VirusTotal automatically using Zeek's VirusTotal script.

**smtp.log / ftp.log / rdp.log** — Protocol-specific logs. Valuable for detecting credential brute force, suspicious file transfers, and unusual RDP activity.

### Zeek Configuration Essentials

```bash
# Install Zeek (Ubuntu/Debian)
sudo apt-get install zeek

# Configure the network interface to monitor
# Edit /etc/zeek/node.cfg
[zeek]
type=standalone
host=localhost
interface=eth0   # your span/tap interface

# Enable JSON logging (better for SIEM integration)
# In /etc/zeek/local.zeek
@load policy/tuning/json-logs

# Enable JA3 fingerprinting
@load policy/protocols/ssl/ja3
```

### Useful Zeek Scripts for Security

```bash
# Load community ID for correlation with other tools
@load packages/zeek-community-id

# Detect long connections (possible C2 keepalive)
@load policy/misc/detect-traceroute

# Track software versions (useful for vulnerability management)
@load policy/protocols/conn/known-services
```

---

## Suricata: Writing Rules That Matter

Suricata rules follow the Snort rule format. The free Emerging Threats Ruleset covers most common attack patterns and is actively maintained.

### Rule Anatomy

```
action  protocol  src_ip  src_port  direction  dst_ip  dst_port  (options)

alert tcp $HOME_NET any -> $EXTERNAL_NET any (
    msg:"Possible Cobalt Strike Beacon";
    flow:established,to_server;
    content:"|00 00 00 00|";
    offset:0;
    depth:4;
    content:"AAAA";
    distance:0;
    within:4;
    threshold:type both, track by_src, count 1, seconds 60;
    sid:9000001;
    rev:1;
)
```

**Actions**: `alert` (log + alert), `drop` (IPS mode), `pass` (whitelist), `reject` (active block)
**Flow**: `established` (existing connection), `to_server`/`to_client` (direction)
**Content**: byte pattern matching — the core detection logic

### Rule Categories to Enable (Emerging Threats)

For a new deployment, start with these categories and tune from there:

```bash
# In /etc/suricata/suricata.yaml, enable:
- emerging-botnet.rules          # Known botnet C2 traffic
- emerging-exploit.rules         # Exploit kit traffic
- emerging-malware.rules         # Known malware communication patterns
- emerging-trojan.rules          # Trojan/RAT C2 patterns
- emerging-dns.rules             # Malicious DNS activity
- emerging-phishing.rules        # Phishing infrastructure
```

Disable categories that generate too much noise without value in your environment:
```bash
# Often too noisy to start:
# emerging-games.rules
# emerging-policy.rules
# emerging-p2p.rules
```

### Tuning Suricata Rules

New Suricata deployments generate enormous alert volumes. Tune immediately:

**Suppress alerts from known-good sources:**
```yaml
# In threshold.conf
suppress gen_id 1, sig_id 2008446, track by_src, ip 192.168.1.0/24
```

**Create threshold for high-frequency alerts:**
```yaml
# Only alert once per 5 minutes per source IP
threshold gen_id 1, sig_id 2013028, type threshold, track by_src, count 5, seconds 300
```

**Write environment-specific rules:**
```
# Detect PowerShell download cradle over HTTP
alert http $HOME_NET any -> $EXTERNAL_NET any (
    msg:"PowerShell Download Cradle - DownloadString";
    flow:established,to_server;
    http.method;
    content:"GET";
    http.user_agent;
    content:"PowerShell";
    sid:9000002;
    rev:1;
)
```

---

## Detection Examples

### C2 Beaconing (Zeek + KQL)

```kql
// In your SIEM: find hosts with regular connection intervals (Zeek conn.log)
ZeekConn
| where TimeGenerated > ago(4h)
| where dest_port in (80, 443, 8080, 8443)
| where dest_ip_is_private == false
| summarize
    ConnectionCount = count(),
    TotalBytes = sum(orig_bytes),
    TimeBuckets = dcount(bin(TimeGenerated, 5m))
  by src_ip, dest_ip
| where ConnectionCount > 30
| where TimeBuckets > 20     // consistent across time = beaconing
| order by TimeBuckets desc
```

### DNS Tunneling (Zeek dns.log)

```kql
ZeekDns
| where TimeGenerated > ago(1h)
| extend QueryLength = strlen(query)
| where QueryLength > 50         // long subdomain
| where query_type_name == "A" or query_type_name == "TXT"
| summarize
    QueryCount = count(),
    AvgLength = avg(QueryLength),
    UniqueSubdomains = dcount(query)
  by src_ip, tld = extract(@"([^.]+\.[^.]+)$", 1, query)
| where UniqueSubdomains > 20    // many unique subdomains to same TLD
| order by UniqueSubdomains desc
```

### JA3 Fingerprint Matching (Zeek ssl.log)

```kql
// Cross-reference JA3 hashes against known malware signatures
ZeekSsl
| where TimeGenerated > ago(1h)
| where ja3 in (
    "51c64c77e60f3980eea90869b68c58a8",   // Cobalt Strike default
    "6734f37431670b3ab4292b8f60f29984",   // Metasploit
    "b386946a5a3dca9b5b0bf1cad3a5f7a9"    // Emotet
)
| project TimeGenerated, src_ip, dest_ip, dest_port, server_name, ja3, ja3s
```

---

## Architecture and Deployment

**Sensor placement:**
- At the internet boundary (north-south traffic — essential)
- Between network segments (east-west/lateral movement — high value)
- At data center ingress (server-to-server traffic)

**For virtual environments:**
Most hypervisors support port mirroring to a dedicated monitoring VM. For cloud (AWS/Azure/GCP), use Traffic Mirroring (AWS) or Network Watcher packet capture (Azure).

**Resource requirements:**
- Zeek: ~1 CPU core per 500Mbps of monitored traffic; 4GB RAM for a 1Gbps sensor
- Suricata: ~2 CPU cores per 1Gbps in IDS mode; more with IPS enabled
- Storage: Plan for 50-100GB/day for Zeek logs at 1Gbps sustained — adjust retention accordingly

**SIEM integration:**
Both tools output JSON. Filebeat or the Elastic Agent works well for Elastic/Sentinel. For Splunk, use the Zeek Technology Add-on. For a quick start, Security Onion bundles Zeek, Suricata, and Elasticsearch in a pre-integrated platform.

---

*Logs and alerts are only useful when you respond to them correctly. Next: [Detecting Credential Abuse](#) — from password spray to Golden Ticket.*
