
# Snort NIDS/NIPS Integration with pfSense CE

Overview
This project documents the integration of Snort 4.1.6 as a Network Intrusion Detection and Prevention System (NIDS/NIPS) into a pfSense CE 2.7.2 firewall. The lab simulates a real-world security monitoring environment where a perimeter firewall is paired with an intrusion prevention system to detect and block unauthorized network activity. All VMs are running on UTM on Apple Silicon M3.

Lab Topology
DeviceIP AddressRoleOSpfSense192.168.1.1Firewall/RouterpfSense CE 2.7.2Ubuntu Server192.168.1.10Snort 3 NIDS SensorUbuntu Server 22.04Kali Linux192.168.1.20Attack VMKali Linux 2024Metasploitable 2192.168.1.30Target VMUbuntu 8.04Host Mac192.168.64.1External AttackermacOS

Network Architecture
Internet
    |
[Host Mac - 192.168.64.1]
    |
[pfSense WAN - 192.168.64.19] ← Snort monitoring WAN
    |
[pfSense LAN - 192.168.1.1] ← Snort monitoring LAN
    |
    ├── Ubuntu Server - 192.168.1.10 (Snort 3 NIDS)
    ├── Kali Linux - 192.168.1.20 (Attack VM)
    └── Metasploitable 2 - 192.168.1.30 (Target VM)

Prerequisites

UTM installed on Apple Silicon Mac
pfSense CE 2.7.2 VM running with WAN and LAN interfaces configured
Metasploitable 2 VM running with static IP 192.168.1.30
Kali Linux VM running with static IP 192.168.1.20
Ubuntu Server VM running with static IP 192.168.1.10
All VMs on Host-Only network 192.168.1.0/24


Installation
Step 1: Install Snort on pfSense via Console Shell
The pfSense 2.7.2 package manager GUI had repository issues so Snort was installed directly through the console shell.
From the pfSense console select option 8 (Shell) and run:
bashpkg install pfSense-pkg-snort
This installs Snort 4.1.6 onto pfSense. After installation Snort is accessible through the pfSense web GUI under Services > Snort.

Step 2: Configure NAT Port Forwarding
To allow external access from the host Mac to Metasploitable 2 through pfSense NAT, a port forwarding rule was created.
Go to Firewall > NAT > Port Forward and add:

Interface: WAN
Protocol: TCP
Destination: WAN address
Destination port: 22
Redirect target IP: 192.168.1.30
Redirect target port: 22

This forwards SSH traffic arriving on pfSense's WAN IP to Metasploitable 2 on the internal LAN.
Verify the connection from host Mac:
bashssh msfadmin@192.168.64.19 -p 22 -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa
The -o HostKeyAlgorithms=+ssh-rsa flag is required because Metasploitable 2 runs an older SSH implementation that modern SSH clients don't support by default.

Step 3: Create Snort LAN Interface
Go to Services > Snort > Snort Interfaces and click Add:

Interface: LAN (em1)
Enable interface: checked
Send alerts to system log: checked

Click Save and start the interface from the Snort Status column.

Step 4: Write Custom Detection Rules
Go to Services > Snort > Interfaces, edit the LAN interface and navigate to the Custom Rules section. Add the following rules:
alert icmp any any -> $HOME_NET any (msg:"Ping Detected!"; sid:1000001; rev:1;)
alert tcp any any -> $HOME_NET 22 (msg:"SSH Detected!"; sid:1000002; rev:1;)
alert tcp any any -> $HOME_NET 21 (msg:"FTP Connection Detected!"; sid:1000003; rev:1;)
alert tcp any any -> $HOME_NET 80 (msg:"HTTP Connection Detected!"; sid:1000004; rev:1;)
Rule breakdown:

alert — action to take when rule matches (log the event)
icmp / tcp — protocol to inspect
any any — match any source IP and port
-> $HOME_NET any — match traffic destined for the home network
msg — alert message shown in the Snort alerts log
sid — unique rule identifier
rev — rule revision number


Step 5: Enable Block Offenders
Go to Services > Snort > Interfaces, edit the LAN interface and scroll to Block Settings:

Block Offenders: checked
IPS Mode: Legacy Mode
Kill States: checked
Which IP to Block: SRC

Click Save and restart the Snort interface.
Why Legacy Mode:
Inline Mode was tested but is incompatible with UTM's emulated NIC drivers (em1). Legacy Mode inspects copies of packets as they traverse the interface — the first packet gets through before Snort can block, but all subsequent traffic from the offending IP is dropped. This is acceptable behavior for this lab environment.

Step 6: Configure Pass List
To maintain trusted administrative access while Block Offenders is active, a pass list was configured for the host Mac's IP address.
Go to Services > Snort > Pass Lists and click Add:

Name: trusted_hosts
Add IP: 192.168.64.1 (host Mac WAN side IP)
Add IP: 192.168.1.194 (host Mac LAN side IP)

Then go to Services > Snort > Interfaces, edit the LAN interface and assign the pass list. Save and restart Snort.

Testing
Test 1: ICMP Ping Detection
From host Mac:
bashping 192.168.64.19
Expected result: First ping gets through, subsequent pings dropped. Source IP added to Snort blocked list. 94% packet loss confirmed.
Verify in pfSense: Go to Services > Snort > Alerts — Ping Detected! alert should appear with source IP.

Test 2: SSH Connection Blocking
From host Mac:
bashssh msfadmin@192.168.64.19 -p 22 -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa
Expected result: SSH session fails to establish. Source IP blocked across all protocols.
Verify in pfSense: Go to Services > Snort > Alerts — SSH Detected! alert should appear. Go to Services > Snort > Blocked — source IP should be listed.

Test 3: Pass List Verification
From host Mac (after adding to pass list):
bashping 192.168.64.19
Expected result: All pings succeed with 0% packet loss. Host Mac bypasses Snort blocking entirely.

Test 4: External Host Blocking
Ping google.com's IP from the pfSense network — Snort detects the returning ICMP traffic and blocks google.com's IP from communicating with the network.
Expected result: google.com's IP appears in Services > Snort > Blocked and all subsequent traffic from that IP is dropped.

Manually Managing the Block List
To clear all blocked IPs from the pfSense console shell:
bashpfctl -t snort2c -T flush
To manually add an IP to the block list:
bashpfctl -t snort2c -T add 192.168.64.1
To view all currently blocked IPs:
bashpfctl -t snort2c -T show

Key Findings
Block Offenders scope — when Snort blocks an IP via Block Offenders it loses access to the entire network, not just the service that triggered the alert. A single ICMP rule violation results in full network isolation for that host.
Legacy Mode behavior — the first packet always gets through in Legacy Mode before Snort can react. This is a known limitation of PCAP-based inspection. Inline Mode eliminates this but requires compatible NIC drivers not available in UTM's emulated environment.
Pass list is critical — without a pass list for trusted administrative IPs, enabling Block Offenders can lock out the administrator from the pfSense web GUI entirely. Always configure the pass list before enabling blocking.
NAT and NIPS interaction — NAT port forwarding rules alone provide no security. Without Snort monitoring the forwarded traffic, any external host could freely access internal services through the open port. Snort adds the detection and prevention layer on top of NAT.
Detection before prevention — running Snort in alert-only mode before enabling Block Offenders allowed traffic patterns to be baselined and rules to be validated before automated blocking was activated.

Troubleshooting
IssueCauseFixPackage manager can't loadpfSense 2.7.2 repo issuesInstall via console shell pkg install pfSense-pkg-snortInline Mode errorsUTM emulated NIC incompatible with NetmapSwitch to Legacy ModeAdmin locked out of GUIBlock Offenders blocked admin IPRun pfctl -t snort2c -T flush from consoleSnort not seeing trafficWrong interface selectedConfirm interface is em1 for LANSSH needs legacy flagsMetasploitable 2 old SSH versionUse -o HostKeyAlgorithms=+ssh-rsa flag

Next Steps

Integrate Splunk as a SIEM for centralized log analysis and correlation across all lab devices
Forward Snort alerts and pfSense firewall logs to Splunk for real-time dashboard monitoring
Build Splunk detection rules correlating Snort alerts with pfSense firewall events


References

pfSense Documentation
Snort Documentation
UTM Documentation
