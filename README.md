# Cyber Security Mini Projects Portfolio

This repository contains the comprehensive reports and technical documentation for three Project-Based Learning (PBL) mini projects in Cybersecurity and Attack Surface Management:

- **Mini Project 4:** OSINT & Digital Footprint Assessment
- **Mini Project 5:** Network Discovery & Asset Inventory
- **Mini Project 6:** AI-Assisted Attack Surface Report

---

## Project Summaries

### 1. [Mini Project 4: OSINT & Digital Footprint Assessment](Mini_Project_4_OSINT_Digital_Footprint.md)
* **Scope:** Passive external reconnaissance against authorized target infrastructure.
* **Techniques & Tools:** WHOIS, Certificate Transparency (`crt.sh`), DNSdumpster, `theHarvester`, Google Dorking, and Shodan caching.
* **Key Findings:** Uncovered direct origin IP bypassing CDN/WAF, mapped employee email patterns, and identified an exposed Tomcat administrative portal.

### 2. [Mini Project 5: Network Discovery & Asset Inventory](Mini_Project_5_Network_Discovery_Asset_Inventory.md)
* **Scope:** Active discovery and layer-2/3/4 scanning across an isolated lab subnet (`192.168.56.0/24`).
* **Techniques & Tools:** `arp-scan`, `nmap` (SYN sweep, service detection, OS fingerprinting, UDP probe), Wireshark packet analysis.
* **Key Findings:** Living asset inventory table, detection of unmanaged Shadow IT (IoT camera with live RTSP stream), unauthenticated SNMP, and critical legacy backdoors (vsftpd 2.3.4, Samba 3.0.20).

### 3. [Mini Project 6: AI-Assisted Attack Surface Report](Mini_Project_6_AI_Assisted_Attack_Surface_Report.md)
* **Scope:** Synthesis of OSINT footprinting and internal network scanning using an AI-augmented Attack Surface Management (ASM) workflow.
* **Techniques & Tools:** 4-stage sequential prompt chain (Normalization, Vulnerability Correlation, Attack Path Modeling, Human-in-the-Loop Validation against NVD).
* **Key Findings:** Executive risk scorecard (38/100 score), end-to-end exploit pivot chain, hallucination detection log, and 30/60/90-day remediation roadmap.

---

## Repository Structure

```text
.
├── README.md
├── Mini_Project_4_OSINT_Digital_Footprint.md
├── Mini_Project_5_Network_Discovery_Asset_Inventory.md
└── Mini_Project_6_AI_Assisted_Attack_Surface_Report.md
```
