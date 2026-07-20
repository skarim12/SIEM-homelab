
# Home SOC Lab: Splunk SIEM + pfSense/Snort Detection Pipeline

A self-built Security Operations Center (SOC) lab simulating a segmented enterprise network, red team attack simulation, and blue team detection/monitoring using Splunk Enterprise as the SIEM.

## Overview

This project resembles a real-world SOC dashboard: Splunk Enterprise ingests firewall and IDS logs from pfSense/Snort, correlates attack activity across multiple techniques, and displays it through a live dashboard showing attacker IPs, attack types, and detection volume. The lab is split into red team (Kali Linux) and blue team (pfSense, Snort, Splunk on Ubuntu Server) roles, connected across a segmented virtual network built in UTM.

## Architecture

| Component | Role | IP / Network |
|---|---|---|
| pfSense CE | Firewall / router / Snort IDS host | LAN: 192.168.1.1, ATTACK-NET: 192.168.2.1 |
| Ubuntu Server | Splunk Enterprise (Docker) — blue team / SIEM | 192.168.1.10 |
| Kali Linux | Attacker platform — red team | 192.168.2.20 (isolated ATTACK-NET) |
| Metasploitable 2 | Intentionally vulnerable target | 192.168.1.30 |

**Why two subnets:** Kali and Metasploitable were originally on the same flat subnet. Attack traffic between them was switching directly at Layer 2 and never reaching pfSense — meaning neither the firewall nor Snort ever saw it. Creating a dedicated, isolated `ATTACK-NET` (192.168.2.0/24) for Kali and routing all traffic to the target LAN through pfSense fixed this and mirrors a standard real-world network segmentation control (isolating an untrusted/attacker-facing zone from protected assets).

## Tools Used

- **pfSense CE** — firewall, router, remote syslog forwarding
- **Snort 3** (pfSense package) — network intrusion detection, custom rule authoring
- **Splunk Enterprise** (via Docker, x86_64 emulation on Apple Silicon) — SIEM, correlation search, alerting, dashboarding
- **Kali Linux** — attack simulation (nmap, ssh/ftp brute force, SMB probes, DoS floods)
- **Metasploitable 2** — intentionally vulnerable attack target
- **UTM** — VM hypervisor / network segmentation

## Splunk Setup Highlights

Splunk Enterprise has no native ARM64 Linux build, so it was deployed via Docker with x86_64 emulation:

```bash
sudo apt install -y qemu-user-binfmt binfmt-support
docker run --privileged --rm tonistiigi/binfmt --install all
```

```bash
sudo docker run -d --platform linux/amd64 \
  --privileged \
  -p 8000:8000 -p 8089:8089 -p 9997:9997 -p 1514:1514/udp \
  -e SPLUNK_START_ARGS="--accept-license" \
  -e SPLUNK_GENERAL_TERMS=--accept-sgt-current-at-splunk-com \
  -e SPLUNK_PASSWORD="<redacted>" \
  -e SPLUNK_SKIP_PREINSTALL_CPU_CHECKS_CORRUPTING_DATA_IF_UNSUPPORTED=1 \
  -v splunk-etc:/opt/splunk/etc \
  -v splunk-var:/opt/splunk/var \
  --name splunk \
  splunk/splunk:latest
```

pfSense forwards System, Firewall, and Snort logs to Splunk's UDP input on port **1514** (moved off the default 514, which was already bound by the host's local `rsyslogd`).

## Detection Rules

Seven custom Snort rules were written and mapped to MITRE ATT&CK, covering reconnaissance through denial-of-service:

| Snort SID | Detection | MITRE ATT&CK |
|---|---|---|
| 1000010 | TCP SYN Scan | T1046 – Network Service Scanning |
| 1000011 | SSH Brute Force | T1110 – Brute Force |
| 1000012 | FTP Brute Force | T1110 – Brute Force |
| 1000013 | SMB Exploit Attempt | T1210 – Exploitation of Remote Services |
| 1000014 | Metasploit Default Listener Port | T1571 – Non-Standard Port |
| 1000017 | SYN Flood (DDoS) | T1499 – Endpoint Denial of Service |
| 1000018 | ICMP Flood | T1499 – Endpoint Denial of Service |

A separate Splunk correlation search flags any source IP triggering **two or more distinct techniques**, surfacing coordinated multi-stage attacks rather than isolated alerts.

## Dashboard

The Splunk dashboard includes:
- **Total Detections** — running count of all custom-rule alerts
- **Alert Volume Over Time** — timechart of activity by alert type
- **Top Source IPs** — attacker IPs ranked by alert count
- **MITRE ATT&CK Breakdown** — technique distribution
- **Recent Detection Events** — detailed table (timestamp, source/destination, alert, MITRE mapping)
- **Multi-Stage Attack Sources** — correlation view of IPs triggering multiple techniques

*(Add dashboard screenshot here)*

## Attack Simulation

All seven detections were validated against real attacks launched from Kali:
- `nmap -sS -p- -T4` — TCP SYN scan
- Repeated failed SSH/FTP logins — brute force
- `nmap --script smb-vuln*` — SMB exploitation probe
- Direct connection to port 4444 — Metasploit listener probe
- `hping3` SYN/ICMP flood — denial-of-service

## Lessons Learned

- **Layer 2 visibility gap:** same-subnet traffic bypasses firewall/IDS inspection entirely — a real blind spot worth checking for in any network you're asked to monitor.
- **Legacy host logging limitations:** Metasploitable 2's ancient `syslogd` could not be configured to forward logs to a non-default port, illustrating the visibility gaps legacy systems create in real environments.
- **Rule syntax evolution matters:** an older `threshold` Snort keyword silently failed under this environment's Snort version; `detection_filter` was required instead — a reminder to always verify a rule actually fires rather than assuming it did.
- **IDS vs. IPS tradeoffs:** running Snort in Block Offenders / IPS mode during active testing caused a real (if minor) service disruption when legitimate testing traffic got blocked — a useful, low-stakes lesson in the operational tradeoffs of automated blocking.

## Related Documents

- [Incident Response Report](./INCIDENT_RESPONSE_REPORT.md) — full attack chain writeup with timeline, IOCs, and remediation recommendations
