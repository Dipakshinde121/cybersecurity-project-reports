# MINI PROJECT 5: NETWORK DISCOVERY & ASSET INVENTORY

**Course:** Cyber Security / Network Security Lab  
**Academic Year:** 2026–2027  
**Assessment Type:** Active Network Scanning, Service Fingerprinting & Asset Inventory  
**Target Environment:** Isolated Virtual Laboratory Subnet (`192.168.56.0/24`)  
**Scanning Host:** Kali Linux 2024.1 (`192.168.56.5/24`)

---

## 1. Executive Summary

An unmapped network is an undefended network. This project focused on performing full layer-2, layer-3, and layer-4 network discovery, operating system fingerprinting, and service enumeration across an isolated enterprise lab segment (`192.168.56.0/24`).

The primary objective was to discover all active physical and virtual nodes, eliminate blind spots caused by unrecorded "Shadow IT" systems, and build an automated asset inventory matrix. Through comprehensive TCP/UDP port audits and network traffic inspection via Wireshark, four distinct live endpoints were identified. Several critical vulnerabilities and misconfigurations were detected, including unauthenticated SMB shares, cleartext administrative services (Telnet/FTP), and an unpatched, legacy web application stack.

---

## 2. Lab Topology & Environment Setup

The scanning environment was established on an isolated host-only virtual network to prevent packet leakage into production networks:

```
                      +-----------------------------+
                      |   Scanning Station (Kali)   |
                      |       192.168.56.5/24       |
                      +--------------+--------------+
                                     |
               ======================+======================
               |           Virtual Switch (vboxnet0)       |
               ======================+======================
                  |                  |                  |                  |
                  v                  v                  v                  v
         [192.168.56.10]    [192.168.56.20]    [192.168.56.35]    [192.168.56.50]
           Web / App          Legacy DB         Windows Server     Simulated IoT
          (Ubuntu 20)      (Metasploitable)       (Win 2016)       (BusyBox/Light)
```

---

## 3. Step-by-Step Discovery Walkthrough & Execution Evidence

### Step 3.1: Layer-2 Local Discovery (ARP Scanning)
* **Objective:** Discover all live hosts on the broadcast domain regardless of personal firewall or ICMP blocking rules.
* **Command:**
  ```bash
  sudo arp-scan --interface=eth1 --localnet
  ```
* **Terminal Output:**
  ```text
  Interface: eth1, type: EN10MB, MAC: 08:00:27:12:ef:aa, IPv4: 192.168.56.5
  Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
  192.168.56.10   08:00:27:3b:41:9a       Cadence Design Systems / VirtualBox
  192.168.56.20   08:00:27:a4:90:bc       Cadence Design Systems / VirtualBox
  192.168.56.35   08:00:27:c8:71:02       Cadence Design Systems / VirtualBox
  192.168.56.50   08:00:27:ff:18:43       Cadence Design Systems / VirtualBox

  4 packets received by filter, 0 packets dropped by kernel
  Ending arp-scan 1.10.0: 256 hosts scanned in 1.482 seconds (172.74 hosts/sec). 4 responded
  ```

### Step 3.2: Comprehensive TCP Port Scanning & Service Fingerprinting
* **Objective:** Identify all listening TCP ports, extract service banners, and deduce operating system versions.
* **Command:**
  ```bash
  sudo nmap -sS -sV -O -p- --open -T4 192.168.56.10,20,35,50 -oA /home/kali/Desktop/lab_discovery_tcp
  ```
* **Detailed Host Breakdown:**

#### Host 1: `192.168.56.10` (Web & API Server)
* **OS Guess:** Linux 5.4.0 (Ubuntu 20.04 LTS)
* **Open Ports & Services:**
  * `22/tcp` - OpenSSH 8.2p1 Ubuntu-4ubuntu0.5
  * `80/tcp` - nginx 1.18.0 (Serving primary web frontend)
  * `3000/tcp` - Node.js Express API framework (REST API)

#### Host 2: `192.168.56.20` (Legacy Storage & Database Server)
* **OS Guess:** Linux 2.6.x (Metasploitable 2 VM)
* **Open Ports & Services:**
  * `21/tcp` - vsftpd 2.3.4 (*Critical: Known backdoor package signature*)
  * `23/tcp` - Linux Telnet-d (Cleartext remote login)
  * `80/tcp` - Apache httpd 2.2.8 ((Ubuntu) DAV/2)
  * `139/tcp` & `445/tcp` - Samba smbd 3.0.20-Debian
  * `3306/tcp` - MySQL 5.0.51a-3ubuntu5

#### Host 3: `192.168.56.35` (Domain Controller / Internal Workstation)
* **OS Guess:** Microsoft Windows Server 2016 Standard Build 14393
* **Open Ports & Services:**
  * `88/tcp` - Microsoft Windows Kerberos (Domain controller role)
  * `135/tcp` - Microsoft Windows RPC
  * `445/tcp` - Windows Server SMB (Dialects SMB 2/3 supported, SMBv1 enabled)
  * `3389/tcp` - Microsoft Terminal Services (RDP with NLA enabled)

#### Host 4: `192.168.56.50` (Shadow IoT Gateway / Surveillance Camera)
* **OS Guess:** Embedded Linux (BusyBox v1.20)
* **Open Ports & Services:**
  * `80/tcp` - lighttpd 1.4.35 (Administrative web panel)
  * `554/tcp` - Real Time Streaming Protocol (RTSP video feed, anonymous stream enabled)

### Step 3.3: UDP Port Probing (Top Critical Services)
* **Command:**
  ```bash
  sudo nmap -sU --top-ports 25 -T4 192.168.56.10,20,35,50
  ```
* **Significant Findings:**
  * Host `192.168.56.20` exposed `161/udp` (SNMPv1/v2c). An unauthenticated SNMP probe revealed public community string `public` was active, exposing system uptime, process tables, and interface IP lists.

### Step 3.4: Wireshark Packet Inspection & Verification
* **Technique:** Captured traffic on `eth1` during TCP SYN (`-sS`) vs. TCP Connect (`-sT`) scans.
* **Analysis Observations:**
  * During the SYN scan, the scanner sent a single TCP packet with `SYN` flag set. Upon receiving `SYN, ACK` from port 80 of `192.168.56.10`, the scanner immediately answered with a `RST` (Reset) packet instead of completing the 3-way handshake with an `ACK`. This confirms the stealth nature of SYN scanning by preventing full socket allocation in application-level logs.
  * For filtered ports, no response was returned, causing Nmap to retransmit probes until the timeout threshold was reached.

---

## 4. Comprehensive Network Asset Inventory Matrix

| IP Address | MAC Address | Hostname / Role | Operating System | Open Ports / Protocols | Primary Services & Versions | Risk Level |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `192.168.56.10` | `08:00:27:3b:41:9a` | `web-prod-01` | Ubuntu Linux 20.04 | 22, 80, 3000 | OpenSSH 8.2p1, Nginx 1.18.0, Node.js Express | Low |
| `192.168.56.20` | `08:00:27:a4:90:bc` | `corp-storage-old` | Linux 2.6.24 (Debian) | 21, 23, 80, 139, 445, 3306, 161/udp | vsftpd 2.3.4, Telnet, Apache 2.2.8, Samba 3.0.20, MySQL 5.0 | **Critical** |
| `192.168.56.35` | `08:00:27:c8:71:02` | `DC-WIN2016` | Windows Server 2016 | 88, 135, 445, 3389 | Kerberos, MS-RPC, SMB 2/3, RDP | High |
| `192.168.56.50` | `08:00:27:ff:18:43` | `cam-lobby-04` | BusyBox Linux | 80, 554 | lighttpd 1.4.35, RTSP Video Feed | Medium |

---

## 5. Security Flaws & Anomalies Discovered

1. **Shadow IT / Unmanaged IoT Camera (`192.168.56.50`):**
   * This device was not listed on the network management records. It runs a decade-old `lighttpd` server and broadcasts live RTSP video over port 554 without authentication.
2. **Obsolete and Insecure Protocols on Storage Node (`192.168.56.20`):**
   * Telnet (port 23) and FTP (port 21) transmit administrative credentials in unencrypted plaintext across the wire.
   * `vsftpd 2.3.4` contains a historic smile-face backdoor trigger (`:)`) that grants an interactive root shell on port 6200.
3. **Insecure SNMP Community Strings:**
   * SNMP configured with default string `public` allows unauthorized network mapping and enumeration of internal process memory.

---

## 6. Network Hardening & Remediation Roadmap

1. **Decommission & Isolate Legacy Nodes:**
   * Immediately disconnect or firewall `192.168.56.20` from the broadcast domain.
   * Replace Telnet and plain FTP with SSH (`sftp`/`scp`).
   * Disable default and backdoored packages (`vsftpd 2.3.4`, Samba 3.0.20).

2. **Network Segmentation & VLAN Isolation:**
   * Isolate the IoT and surveillance devices (`192.168.56.50`) into a segregated IoT VLAN (e.g., VLAN 40) with no access to the management or server subnet.
   * Block egress internet access for surveillance cameras.

3. **Active Directory & Windows Hardening:**
   * Disable SMBv1 explicitly on `192.168.56.35` via PowerShell:
     ```powershell
     Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
     ```
   * Enforce SMB Signing and restrict RDP access to designated jump-boxes via firewall whitelist.

---

## 7. Student Reflection & Conclusion

This lab provided practical experience in moving from theoretical networking concepts to real-world security enumeration. Analyzing the differences between ARP and TCP SYN scans in Wireshark showed why network defenders must inspect raw packets rather than relying purely on high-level tool dashboards. The discovery of an unmanaged IoT camera highlighted the constant challenge security teams face with Shadow IT in modern networks.
