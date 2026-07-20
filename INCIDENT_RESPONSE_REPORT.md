# Incident Response Report — INC-2026-0717-001

**Multi-Stage Attack Against Metasploitable Host**

| Field | Detail |
|---|---|
| Date Detected | July 17, 2026 |
| Analyst | Saminur Karim |
| Severity | Critical |
| Status | Contained (controlled lab exercise) |
| Attacking Host | 192.168.2.20 (Kali Linux, isolated ATTACK-NET segment) |
| Affected Host | 192.168.1.30 (Metasploitable 2) |

## Summary

The lab's detection stack identified a coordinated, multi-stage attack originating from 192.168.2.20 against the monitored target 192.168.1.30. The attacking host progressed through reconnaissance, credential brute forcing, exploitation attempts, command-and-control probing, and a denial-of-service phase, triggering six distinct Snort signatures within a short window. A Splunk correlation search flagged the source IP as associated with multiple techniques, indicating one coordinated attacker rather than unrelated background activity.

## Indicators of Compromise

| Indicator | Value |
|---|---|
| Source IP | 192.168.2.20 |
| Destination IP | 192.168.1.30 |
| Routing chokepoint | 192.168.1.1 (pfSense) |
| Ports targeted | 21, 22, 445, 4444 |

## Timeline

| Time (UTC) | Stage | MITRE ATT&CK | Detection |
|---|---|---|---|
| 06:17:55 | Reconnaissance (TCP SYN scan) | T1046 | Snort SID 1000010 |
| 06:17:55 | Denial of Service (SYN flood) | T1499 | Snort SID 1000017 |
| 06:17:56 | Credential Access (SSH brute force) | T1110 | Snort SID 1000011 |
| 06:17:56 | Credential Access (FTP brute force) | T1110 | Snort SID 1000012 |
| 06:17:56 | C2 Probe (Metasploit listener port) | T1571 | Snort SID 1000014 |
| — | Exploitation Attempt (SMB) | T1210 | Snort SID 1000013 |
| ~06:20:00 | Correlation — 6 distinct techniques from one source | Multi-stage | Splunk correlation search |

## Detection Coverage

Each stage was independently detected at the network layer by a custom Snort rule and forwarded to Splunk via syslog, where a matching scheduled alert fired. A separate Splunk correlation search (`dc(sid) by src_ip >= 2`) identified the multi-technique pattern independent of any single rule, elevating six isolated alerts into one high-confidence finding.

## Analysis

The sequence maps cleanly onto early-to-mid MITRE ATT&CK tactics: reconnaissance established target awareness, brute-force attempts sought valid credentials, the SMB probe targeted a known vulnerable service, the Metasploit port probe suggested C2 preparation, and the SYN flood represented an availability-impacting action. All six stages were detected before correlation was applied, demonstrating defense-in-depth — even without the correlation search, each stage independently generated a triage-worthy alert.

## Limitations Identified

- Snort ran in IDS (detect-only) mode during this exercise rather than IPS, so traffic was not automatically blocked.
- The correlation search operates on a 5-minute window; attacks spread across a longer duration could evade correlation unless the window is widened.
- Host-level log visibility on Metasploitable 2 was unavailable due to its legacy syslog daemon lacking support for modern remote-logging configuration.

## Recommendations

1. Block source IP 192.168.2.20 at the firewall pending investigation.
2. Enable Snort Block Offenders (IPS mode) for confirmed-malicious signatures once the ruleset is stable.
3. Patch or decommission the vulnerable SMB service on the target host.
4. Enforce account lockout policies on SSH and FTP to limit brute-force effectiveness regardless of detection.
5. Apply rate-limiting or SYN cookies to reduce SYN flood impact on production-equivalent hosts.
6. Widen the multi-stage correlation window to catch slower, more deliberate attacks.

## Conclusion

This exercise validated an end-to-end detection pipeline spanning network segmentation, IDS signature development, SIEM ingestion, and multi-source correlation, while surfacing genuine architectural findings — a Layer 2 visibility gap and a legacy host logging limitation — that mirror real constraints SOC analysts encounter in production environments.
