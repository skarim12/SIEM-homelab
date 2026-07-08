# Snort NIDS/NIPS Integration with pfSense

**Home Lab | Network Security | Intrusion Detection | Intrusion Prevention**

---

## Table of Contents

* [Overview](#overview)
* [Environment](#environment)
* [Lab Architecture](#lab-architecture)
* [Objectives](#objectives)
* [Implementation](#implementation)

  * [1. Installing Snort](#1-installing-snort)
  * [2. Custom Detection Rules](#2-custom-detection-rules)
  * [3. NAT Port Forwarding](#3-nat-port-forwarding)
  * [4. Enabling Intrusion Prevention](#4-enabling-intrusion-prevention)
  * [5. Passlist Configuration](#5-passlist-configuration)
* [Testing & Validation](#testing--validation)
* [Key Takeaways](#key-takeaways)
* [What's Next](#whats-next)
* [Tools & Technologies](#tools--technologies)

---

# Overview

This project builds upon my pfSense firewall lab by integrating **Snort 4.1.6** as both a **Network Intrusion Detection System (NIDS)** and **Network Intrusion Prevention System (NIPS)**. The objective was to monitor inbound network traffic, detect suspicious activity through custom Snort rules, and automatically block malicious hosts attempting to access internal services.

The lab simulates a layered security environment where pfSense provides stateful firewall protection while Snort performs signature-based intrusion detection and automated prevention.

---

# Environment

| Component        | Details             |
| ---------------- | ------------------- |
| Hypervisor       | UTM (Apple Silicon) |
| Firewall         | pfSense CE 2.7.2    |
| IDS/IPS          | Snort 4.1.6         |
| Internal Network | 192.168.1.0/24      |
| Attacker         | Host macOS          |
| Target           | Metasploitable 2    |
| Management VM    | Kali Server         |

---

# Lab Architecture

```text
                Host macOS
                     │
          NAT Port Forwarding
                     │
            pfSense CE Firewall
      Snort NIDS / NIPS Enabled
                     │
      ───────────────────────────
      │            │            │
 Ubuntu Server   Kali Linux   Metasploitable 2
                         (Target)
```

Snort monitors traffic entering the internal network through pfSense. When malicious activity matches a configured rule, the source IP is automatically added to Snort's block list, preventing any further communication.

---

# Objectives

* Install Snort directly on pfSense
* Configure Snort as both an IDS and IPS
* Create custom detection rules
* Detect ICMP reconnaissance activity
* Detect SSH connection attempts
* Configure NAT port forwarding
* Automatically block malicious hosts
* Validate intrusion prevention functionality
* Configure trusted passlists for administrative access

---

# Implementation

## 1. Installing Snort

The pfSense package manager experienced repository issues on pfSense CE 2.7.2, so Snort was installed directly from the pfSense console.

```bash
pkg install pfSense-pkg-snort
```

After installation, Snort was configured on the WAN interface to inspect inbound traffic entering the internal lab network.

---

## 2. Custom Detection Rules

Custom Snort rules were created to detect:

* ICMP Echo Requests (Ping)
* SSH connection attempts

These rules generated alerts whenever reconnaissance or remote access activity targeted the Metasploitable 2 virtual machine.

---

## 3. NAT Port Forwarding

A NAT port forwarding rule was configured on the WAN interface to allow SSH connections from the host Mac to the Metasploitable 2 VM.

Because Metasploitable uses legacy SSH algorithms, the connection was verified using the macOS SSH client with legacy key algorithm compatibility options enabled.

---

## 4. Enabling Intrusion Prevention

Snort's **Block Offenders (Legacy Mode)** was enabled to transition from passive monitoring into active intrusion prevention.

When a host triggered one of the configured Snort rules:

* An alert was generated
* The source IP was automatically added to the Snort block list
* All subsequent traffic from that IP was dropped

This provided automated protection against repeated attack attempts.

---

## 5. Passlist Configuration

A Snort passlist was configured using the IP address of the host Mac.

This ensured trusted administrative traffic was never blocked while Snort continued preventing unauthorized hosts from accessing the network.

---

# Testing & Validation

| Test                              | Result     |
| --------------------------------- | ---------- |
| Initial ICMP Ping                 | Detected   |
| Subsequent ICMP Traffic           | Blocked    |
| Packet Loss                       | ~94%       |
| SSH Connection Before Block       | Successful |
| SSH Connection After Block        | Blocked    |
| Snort Alert Generated             | Yes        |
| Source IP Added to Block List     | Yes        |
| Passlisted Host Maintained Access | Yes        |

Testing confirmed that Snort successfully detected reconnaissance activity, automatically blocked the attacking host, and prevented additional communication across multiple protocols.

---

# Key Takeaways

* Snort can operate as both a Network Intrusion Detection System and Network Intrusion Prevention System when integrated with pfSense.

* Enabling **Block Offenders** automatically isolates hosts that trigger configured detection rules.

* Custom Snort rules provide visibility into reconnaissance and unauthorized access attempts.

* NAT port forwarding alone does not secure exposed services; pairing it with intrusion prevention provides layered protection.

* Passlists are essential to prevent administrators from locking themselves out during automated blocking.

* Validating detection rules before enabling automated blocking helps reduce false positives and improves policy accuracy.

---

# What's Next

* Integrate **Splunk** as a SIEM for centralized log collection and event correlation.
* Monitor Snort alerts within Splunk dashboards.
* Build detection rules for additional attack techniques.
* Correlate firewall, authentication, and intrusion detection logs.
* Expand the lab with additional vulnerable hosts for security testing.

---

# Tools & Technologies

`pfSense CE` `Snort` `UTM` `Ubuntu Server` `Kali Linux` `Metasploitable 2` `Firewall` `NIDS` `NIPS` `Linux` `Networking` `Cybersecurity` `Home Lab` `Blue Team`

---

*Built as part of my ongoing cybersecurity home lab portfolio to develop practical skills in firewall administration, intrusion detection, intrusion prevention, and network defense.*
