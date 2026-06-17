# 🔥 pfSense Firewall & Internal LAN Lab

**Home Lab | Network Security | Firewall Configuration | IDS Prep**

📝 **Full write-up on Medium:** [pfSense Configuration](https://medium.com/@karimsaminur123/pfsense-configuration-fdbaabd73084?postPublishedType=initial)

---

## Table of Contents

- [Overview](#overview)
- [Environment](#environment)
- [Virtual Machines](#virtual-machines)
- [Architecture Diagram](#architecture-diagram)
- [Lab Setup](#lab-setup)
  - [1. Metasploitable VM Preparation](#1-metasploitable-vm-preparation)
  - [2. pfSense Installation](#2-pfsense-installation)
  - [3. Kali Linux Installation](#3-kali-linux-installation)
  - [4. Accessing the pfSense Web UI](#4-accessing-the-pfsense-web-ui)
  - [5. Network Configuration](#5-network-configuration)
  - [6. Firewall Rule — Block All Outbound Traffic](#6-firewall-rule--block-all-outbound-traffic)
- [Testing & Validation](#testing--validation)
- [Key Takeaways](#key-takeaways)
- [What's Next](#whats-next)
- [Tools & Technologies](#tools--technologies)

---

## Overview

This project documents the end-to-end setup of a virtualized network security environment using **pfSense CE** as the core firewall/router, running on **UTM** (Apple Silicon / ARM). The lab includes five virtual machines connected over an internal LAN, with custom firewall rules enforced to block external internet access while preserving full internal communication — simulating a real-world isolated network segment.

This lab was built as part of an ongoing portfolio effort to develop hands-on skills in network defense, firewall policy, and security infrastructure management.

---

## Environment

| Component | Details |
|-----------|---------|
| Hypervisor | UTM (Apple Silicon / aarch64) |
| Firewall/Router | pfSense CE 2.7.2 (x86_64, emulated via UTM) |
| Network Mode | Host Only (internal LAN) |
| Subnetting | 192.168.1.0/24 |
| Default Gateway | 192.168.1.1 (pfSense LAN interface) |

---

## Virtual Machines

| VM | Role | Notes |
|----|------|-------|
| pfSense CE 2.7.2 | Firewall / Router | WAN + LAN interfaces, DHCP server |
| Ubuntu Server | Admin workstation | Static IP, pfSense web UI access |
| Kali Linux | Attacker / Pen Test | Static IP, ARM build via UTM gallery |
| Metasploitable 2 | Vulnerable target | Converted `.vmdk` → `.qcow2` via QEMU |
| Metasploitable 3 | Vulnerable target | Same conversion workflow as MS2 |

---

## Architecture Diagram

```
                    [ WAN Interface ]
                          |
                    [ pfSense CE ]
                    192.168.1.1
                          |
              +-----------+-----------+
              |           |           |
         [Ubuntu]      [Kali]    [MS2 / MS3]
       .1.10/24      .1.20/24    .1.30-40/24
```

All VMs are connected via an internal **Host Only** network. No traffic leaves the LAN when the block rule is active.

---

## Lab Setup

### 1. Metasploitable VM Preparation

Metasploitable 2 and 3 are distributed as `.vmdk` disk images, which UTM does not natively support. The conversion workflow:

**Download:** [Metasploitable — SourceForge](https://sourceforge.net/projects/metasploitable/)

```bash
# Install QEMU on macOS (Homebrew)
brew install qemu

# Convert VMDK to QCOW2
qemu-img convert -f vmdk Metasploitable.vmdk -O qcow2 Metasploitable.qcow2
```

UTM settings for each Metasploitable VM:
- Architecture: x86_64 (emulated)
- RAM: 512 MB | CPU: 1 core
- UEFI Boot: **disabled**
- Network: Host Only
- Drive: imported `.qcow2` file

Metasploitable 3 was installed using the same exact settings as Metasploitable 2 above.

---

### 2. pfSense Installation

pfSense does not offer a native ARM build, so it runs under **emulation** in UTM.

**Download:** [pfSense CE — pfsense.org/download](https://www.pfsense.org/download/) → use `pfSense-CE-2.7.2-RELEASE-amd64.iso.gz` (direct: [atxfiles.netgate.com](https://atxfiles.netgate.com/mirror/downloads/pfSense-CE-2.7.2-RELEASE-amd64.iso.gz))

> **Note:** The latest pfSense ISO caused fatal errors under UTM emulation. Falling back to CE 2.7.2 resolved the issue.

```bash
# Decompress ISO on macOS
cd ~/Downloads
gunzip pfSense-CE-2.7.2-RELEASE-amd64.iso.gz
```

**UTM VM Settings:**

| Setting | Value |
|---------|-------|
| Mode | Emulate |
| Architecture | x86_64 |
| Chipset | Intel ICH9 |
| RAM | 2 GB |
| CPU | 2 cores |
| Storage | 16 GB |
| Display | VGA |
| Network 1 (WAN) | Shared Network |
| Network 2 (LAN) | Host Only |

**Installation steps:**
1. Boot from ISO → pfSense installer launches automatically
2. Select **Guided UFS** disk setup
3. Partition entire disk → complete installation
4. **Before reboot:** shut down VM and remove ISO from CD/DVD drive
5. Reboot → pfSense console loads with WAN/LAN interfaces displayed

---

### 3. Kali Linux Installation

Kali was installed using UTM's pre-built image gallery rather than a manual ISO install.

**Download:** [Kali Linux ARM64 UTM Image — archive.org](https://archive.org/download/kali-linux-2023-arm64-utm)

Import the downloaded `.utm` bundle directly into UTM. No manual configuration needed — Kali boots natively on Apple Silicon via this image.

---

### 4. Accessing the pfSense Web UI

Ubuntu was used as the admin workstation to access the pfSense web UI at `http://192.168.1.1`.

**Troubleshooting — IP not assigned via DHCP:**
- Changed Ubuntu's network adapter from **Shared Network** → **Host Only**
- Manually configured a static IP within the `192.168.1.0/24` subnet
- Verified with `ip a` and pinged the gateway at `192.168.1.1`

**Default credentials (changed immediately):**

| Field | Default |
|-------|---------|
| Username | admin |
| Password | pfsense |

> **Security note:** Default credentials should always be changed on first login. Leaving them as-is is a critical vulnerability in any firewall deployment.

---

### 5. Network Configuration

All five VMs were placed on the same **Host Only** network and assigned static IPs within `192.168.1.0/24`.

**Additional pfSense configuration required:**
- Unblocked **private networks** and **loopback addresses** on the WAN interface to allow internal routing

Each VM was tested with cross-VM pings to confirm LAN connectivity before any firewall rules were applied. All pings were successful across Ubuntu, Kali, Metasploitable 2, and Metasploitable 3.

---

### 6. Firewall Rule — Block All Outbound Traffic

**Path:** Firewall → Rules → LAN → Add

| Field | Value |
|-------|-------|
| Action | Block |
| Interface | LAN |
| Protocol | Any |
| Source | LAN subnets |
| Destination | Any |

Applied rule → changes saved.

---

## Testing & Validation

### With Firewall Rule Enabled (Block)

| Test | Result |
|------|--------|
| Ping `8.8.8.8` (Google DNS) from Ubuntu | ❌ All packets lost |
| Ping `8.8.8.8` from Kali | ❌ All packets lost |
| Ping `8.8.8.8` from Metasploitable 2 | ❌ All packets lost |
| Ping `8.8.8.8` from Metasploitable 3 | ❌ All packets lost |
| Ping internal LAN IPs (all VMs) | ✅ All successful |

### With Firewall Rule Disabled

| Test | Result |
|------|--------|
| Ping `facebook.com` | ✅ Successful |
| Ping `google.com` | ✅ Successful |
| Ping `yahoo.com` | ✅ Successful |

This confirmed the firewall rule was functioning correctly — internal traffic was unaffected while outbound traffic was fully blocked.

---

## Key Takeaways

- **Stateful packet inspection** via pfSense's LAN rules controls traffic at the subnet level, not just per-host
- **VMDK → QCOW2 conversion** is required for running legacy VM images in UTM/QEMU environments
- pfSense running under emulation (x86 on ARM) is viable for lab use, though performance is limited vs. native
- Changing default credentials and auditing WAN-side settings are critical first steps in any firewall deployment
- The isolated LAN setup mirrors real-world DMZ and air-gap network architectures used in enterprise security

---

## What's Next

This lab is the foundation for further network security projects:

- [ ] Deploy **Snort 3** or **Suricata** as an IDS on the Ubuntu VM to inspect internal LAN traffic
- [ ] Integrate a **SIEM** (e.g., Wazuh or Security Onion) for centralized log collection
- [ ] Launch attacks from Kali against Metasploitable targets and analyze traffic in pfSense logs
- [ ] Configure **pfSense VPN** (WireGuard or OpenVPN) for secure remote access simulation
- [ ] Add **Azure AD hybrid** identity configuration to the lab environment

---

## Tools & Technologies

`pfSense CE` `UTM` `QEMU` `VirtualMachine` `Ubuntu` `Kali Linux` `Metasploitable` `Networking` `Firewall` `FreeBSD` `HomeLab` `CyberSecurity`

---

*Built as part of an ongoing home lab portfolio — Junior IT/Cybersecurity student actively seeking internship opportunities.*
