# Snort NIDS/NIPS Homelab — Network Intrusion Detection & Prevention System

> A hands-on cybersecurity homelab project configuring Snort 3 on Ubuntu Linux and Windows 11, simulating real-world network attacks, and building toward a full defense-in-depth architecture.

---

Here is the link of the full report including screenshots on Medium:

https://medium.com/@karimsaminur123/snort-nids-nips-deployment-6ae7c99011e4

## Table of Contents

- [Overview](#overview)
- [Environment & Tools](#environment--tools)
- [Installation & Configuration](#installation--configuration)
- [Custom Rule Development](#custom-rule-development)
- [Traffic Testing & Alert Generation](#traffic-testing--alert-generation)
- [Deep Packet Inspection (DPI)](#deep-packet-inspection-dpi)
- [Multi-VM Attack Simulation](#multi-vm-attack-simulation)
- [Vulnerability Assessment — Port Scanning with Nmap](#vulnerability-assessment--port-scanning-with-nmap)
- [HTTP Traffic Detection with Apache2](#http-traffic-detection-with-apache2)
- [Community Rules Integration](#community-rules-integration)
- [Logging & Log Management](#logging--log-management)
- [Key Takeaways](#key-takeaways)
- [Next Steps](#next-steps)

---

## Overview

This project demonstrates the setup and use of **Snort 3**, the industry-standard open-source Network Intrusion Detection and Prevention System (NIDS/NIPS), across a virtual lab environment. Snort was configured to monitor live network traffic, detect malicious patterns, and generate structured alerts for a range of simulated attack types including ICMP floods, port scans, SSH/FTP access attempts, and HTTP-based attacks.

Snort operates by matching traffic against a rule set — either custom-written or sourced from the community — and taking defined actions such as alerting, logging, or blocking. When paired with SIEM platforms like **Splunk** or **Wazuh**, Snort becomes a powerful component in an enterprise security monitoring pipeline.

---

## Environment & Tools

| Component | Details |
|-----------|---------|
| **OS (Server)** | Ubuntu Linux (UTM VM) |
| **OS (Client)** | Windows 11 VM |
| **IDS/IPS** | Snort 3 |
| **Web Server** | Apache2 HTTP Server |
| **Port Scanner** | Nmap |
| **Network Mode** | Bridged Advanced (UTM) |
| **Interface** | `enp0s1` (promiscuous mode) |

---

## Installation & Configuration

### Fastest Snort 3 Installation (Zero-Error Path)

```bash
# 1. Update package lists
sudo apt update

# 2. Install Snort
sudo apt-get install snort -y

# 3. Locate the Snort 3 config file
sudo find / -name snort.lua 2>/dev/null

# 4. Create the rules directory
sudo mkdir -p /usr/local/etc/snort/rules

# 5. Create and edit your local rules file
sudo nano /usr/local/etc/snort/rules/local.rules

# 6. Verify rules aren't already linked in config
grep -n "rules" /usr/local/etc/snort/snort.lua

# 7. Add the include block for local.rules inside snort.lua
sudo nano /usr/local/etc/snort/snort.lua
```

**`snort.lua` rules block:**

```lua
ips =
{
  rules = [[
    include /usr/local/etc/snort/rules/local.rules
    variables = default_variables
  ]],
}
```

```bash
# 8. Set network interface to promiscuous mode
sudo ip link set enp0s1 promisc on

# 9. Run Snort 3
sudo snort -c /usr/local/etc/snort/snort.lua -i enp0s1 -A alert_fast
```

### Network Interface Setup

The `ip link show` command was used to identify available network adapters. The interface names in the UTM virtual environment differed from the settings panel, requiring manual discovery. The network mode was changed from **Shared Network** to **Bridged Advanced** to allow the VM to participate in the same subnet as the host and other VMs — essential for inter-VM traffic capture.

---

## Custom Rule Development

Snort rules follow a clear, readable syntax:

```
action protocol src_ip src_port -> dst_ip dst_port (options)
```

### Rule Anatomy

| Field | Description |
|-------|-------------|
| `action` | What to do — `alert`, `drop`, `log`, `pass` |
| `protocol` | Traffic type — `icmp`, `tcp`, `udp`, `ip` |
| `src/dst` | IP address and port (`any` for wildcard) |
| `msg` | Human-readable alert message |
| `sid` | Unique rule ID (local rules start at `1000001`) |
| `rev` | Revision number of the rule |

### Rules Written

```snort
# Rule 1 — Alert on any ICMPv4 request or reply
alert icmp any any -> any any (msg:"ICMP Packet Detected"; sid:1000001; rev:1;)

# Rule 2 — Alert on large ICMP packets exceeding 500 bytes (detects DoS/buffer overflow)
alert icmp any any -> any any (msg:"Large ICMP Packet Detected"; dsize:>500; sid:1000002; rev:1;)

# Rule 3 — Alert on outbound FTP connection attempts
alert tcp any any -> any 21 (msg:"FTP Connection Attempt"; sid:1000003; rev:1;)

# Rule 4 — Alert on outbound SSH/remote client access attempts
alert tcp any any -> any 22 (msg:"SSH Remote Access Attempt"; sid:1000004; rev:1;)

# Rule 5 — Alert on outbound HTTP web requests
alert tcp any any -> any 80 (msg:"HTTP Web Request Detected"; sid:1000005; rev:1;)
```

> **Note:** SID values must begin at `1000001` for local rules and increment sequentially as new rules are added.

---

## Traffic Testing & Alert Generation

### ICMP Testing

Snort was launched in quiet, logging alert mode:

```bash
sudo snort -q -l /var/log/snort -i enp0s1 -A alert_fast -c /usr/local/etc/snort/snort.lua
```

Google's public DNS (`8.8.8.8`) was pinged to generate ICMP traffic:

```bash
ping 8.8.8.8
```

After pinging for 30+ seconds, a large volume of ICMP alerts was generated and written to the log file, confirming that Rule 1 was working correctly.

### Viewing Live Alerts

```bash
sudo tail -f /var/log/snort/alert_fast.txt
```

---

## Deep Packet Inspection (DPI)

### Basic DPI (Protocol Header Inspection)

```bash
sudo snort -l /var/log/snort -i enp0s1 -A alert_full -c /usr/local/etc/snort/snort.lua 2>&1 | tee /var/log/snort/alert_full.txt
```

**Basic DPI reveals:**
- Protocol headers (TTL, TOS, ID, IpLen, DgmLen)
- ICMP Type and Code
- Source and destination IP addresses

### Full DPI (Payload Inspection)

```bash
sudo snort -d -l /var/log/snort -i enp0s1 -A alert_full -c /usr/local/etc/snort/snort.lua
```

**Full DPI reveals (`-d` flag):**
- HTTP → URLs, headers, cookies, POST data
- FTP → commands and responses
- SSH → connection handshakes
- DNS → queries and responses
- TCP/UDP payload → raw application data

---

## Multi-VM Attack Simulation

### SSH & FTP Alert Generation

Initial SSH tests from within the same Ubuntu VM did not trigger alerts as expected. The solution was to use the **Windows 11 VM** as an external client to connect to the Ubuntu server's IP address (`192.168.1.191`).

Steps taken:
1. Configured SSH on the Windows 11 host and pointed it at the Ubuntu server IP
2. Installed FTP services on the Ubuntu server
3. Windows 11 VM successfully pinged the Ubuntu server
4. Both **SSH alerts** (Rule 4) and **FTP alerts** (Rule 3) appeared in Snort

This simulates a real attacker on a separate machine attempting to gain remote access — a common threat scenario.

---

## Vulnerability Assessment — Port Scanning with Nmap

Nmap was used to perform port scanning against the Ubuntu server, replicating real-world attacker reconnaissance.

### Scans Performed

```bash
# Stealth SYN scan — identifies open/closed/filtered ports without completing TCP handshake
nmap -sS <target_ip>

# Xmas scan — sets FIN, PSH, URG flags to probe for port state
nmap -sX <target_ip>
```

### Why This Matters

| Scan Type | Technique | Threat Relevance |
|-----------|-----------|-----------------|
| Stealth SYN (`-sS`) | Half-open TCP connection | Bypasses basic logging; used to enumerate open ports quietly |
| Xmas scan (`-sX`) | Unusual flag combination | Evades some stateless firewalls; tests RFC compliance |

**Ports of interest:**
- **Port 21 (FTP)** — Vulnerable to brute-force, misconfiguration, and weak encryption
- **Port 22 (SSH)** — Common target for brute-force and weak key exploitation
- **Port 80 (HTTP)** — Entry point for web-based attacks

Snort successfully detected both scan types and generated alerts. Once detected, defenders can:
- Identify the attacker's IP address
- Apply firewall rules to drop or rate-limit packets from that source
- Enable Snort's NIPS mode to automatically block the scanning host

> **Vulnerability Assessment Note:** This Nmap scanning phase constitutes the **reconnaissance stage of a vulnerability assessment**. Identifying open ports and services is the first step toward determining which services may be exploitable. Extending this lab with OpenVAS or Nessus would allow mapping those open ports to known CVEs for a full VA report.

---

## HTTP Traffic Detection with Apache2

### Setup

Apache2 was installed on the Ubuntu server to serve as an HTTP target:

```bash
sudo apt install apache2 -y
sudo systemctl start apache2
```

The Windows 11 VM was then used to send HTTP web requests to the Apache server's IP address.

### Result

HTTP alerts (Rule 5) appeared in Snort as soon as the Windows VM made web requests. This demonstrates Snort's ability to detect layer 7 (Application Layer) attacks such as:

- Malformed HTTP requests
- SQL injection attempts via HTTP
- Cross-Site Scripting (XSS) payloads in GET/POST requests
- HTTP flood DoS attacks
- Privilege escalation through web vulnerabilities

**Defensive responses after receiving HTTP alerts:**
- Create IP blacklists for flagged sources
- Implement SSL inspection
- Whitelist trusted IPs only
- Apply rate limiting on incoming connections

---

## Community Rules Integration

Snort's community ruleset provides a broad, maintained library of signatures covering known attacks and CVEs. Integration required a single edit to `snort.lua`:

```bash
sudo nano /usr/local/etc/snort/snort.lua
```

The community rules path was added inside the `ips` block, alongside the local rules. After restarting Snort, alerts immediately appeared with detailed descriptions of potential attack types — far richer than basic local rules alone.

**Benefits of community rules:**
- Covers thousands of known attack signatures out of the box
- Updated regularly by the Snort community and Cisco Talos
- Bridges the gap before a vendor patch is available for new vulnerabilities
- Produces descriptive alert messages useful for SOC triage

---

## Logging & Log Management

### Log Directory Setup

```bash
# Create log directory
sudo mkdir -p /var/log/snort

# View fast alerts in real time
sudo tail -f /var/log/snort/alert_fast.txt

# Capture full alerts to file
sudo snort -l /var/log/snort -i enp0s1 -A alert_full -c /usr/local/etc/snort/snort.lua 2>&1 | tee /var/log/snort/alert_full.txt
```

### Alert Modes Compared

| Mode | Command Flag | Detail Level | Use Case |
|------|-------------|--------------|----------|
| Fast | `-A alert_fast` | Timestamp, rule msg, src/dst IP | Rapid triage, real-time monitoring |
| Full | `-A alert_full` | All fast data + protocol headers | Deeper analysis, incident investigation |
| Full + Payload | `-A alert_full -d` | All above + raw packet payload | Full DPI, forensic capture |

### SIEM Integration Potential

Structured log files in `/var/log/snort/` are directly ingestible by:
- **Splunk** — via Universal Forwarder or file monitor inputs
- **Wazuh** — via active response and custom decoders for Snort log format

This enables organization-wide security event correlation across multiple Snort sensors.

---

## Key Takeaways

- **Custom Snort rules are simple but powerful** — the rule syntax maps directly to real attack signatures and is easy to extend as new threats emerge
- **Promiscuous mode is required** — without it, the interface only captures traffic addressed to itself
- **Multi-VM setups surface real detection gaps** — loopback traffic behaves differently than inter-host traffic; testing from a separate VM is essential
- **Port scanning is detectable early** — stealth and Xmas scans leave fingerprints that Snort catches at the reconnaissance stage, well before exploitation
- **DPI reveals traffic that alert headers miss** — the `-d` flag exposes raw payloads, critical for detecting injection attacks and exfiltration
- **Community rules dramatically expand coverage** — thousands of maintained signatures can be added instantly with minimal configuration
- **Logging is the foundation for SIEM** — structured, timestamped log files make Snort a composable component in any SOC pipeline
- **Defense-in-depth requires layering** — Snort alone is effective, but combining it with a pfSense firewall creates overlapping detection and prevention layers

---

## Next Steps

- [ ] Integrate Snort into a **pfSense virtual router** for a full defense-in-depth architecture
- [ ] Connect Snort logs to **Splunk or Wazuh** for SIEM-level correlation and dashboarding
- [ ] Extend the vulnerability assessment with **OpenVAS or Nessus** to map open ports to CVEs
- [ ] Configure Snort in **NIPS (inline) mode** to actively drop malicious packets
- [ ] Implement **rate limiting rules** for SSH and FTP brute-force detection
- [ ] Add **IP reputation rules** to block known malicious source ranges

---

*Built on Ubuntu Linux · Snort 3 · Apache2 · Nmap · Windows 11 VM*
