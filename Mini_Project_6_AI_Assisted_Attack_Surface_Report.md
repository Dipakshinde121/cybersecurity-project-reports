# MINI PROJECT 6: AI-ASSISTED ATTACK SURFACE REPORT

**Course:** Cyber Security / Network Security Lab  
**Academic Year:** 2026–2027  
**Assessment Type:** AI-Augmented Attack Surface Management (ASM) & Threat Synthesis  
**Input Data Sources:** OSINT Assessment Findings (Mini Project 4) + Active Network Discovery Logs (Mini Project 5)  
**AI Framework Used:** Multi-Stage LLM Prompt Engineering Pipeline (Gemini / Claude / Local Llama-3) with Human-in-the-Loop (HITL) Verification

---

## 1. Executive Summary & Posture Overview

Modern enterprise infrastructure changes too rapidly for manual spreadsheet analysis to remain effective. In this project, an AI-augmented Attack Surface Management (ASM) methodology was developed to ingest raw reconnaissance data from external OSINT (Mini Project 4) and internal subnet scanning (Mini Project 5). 

The AI pipeline processed 8 distinct network endpoints, 14 unique services, and 5 external domain assets. By cross-referencing software Common Platform Enumeration (CPE) signatures against vulnerability feeds (NVD / CISA KEV), the pipeline automatically synthesized risk priorities, identified likely lateral movement attack paths, and generated both non-technical executive briefings and technical engineering remediation plans.

### Overall Risk Scorecard

```
+-----------------------------------------------------------------------+
|  Overall Security Posture Score: 38 / 100  (CRITICAL EXPOSURE)        |
+-----------------------------------------------------------------------+
|  Total Assets Assessed:        9 (5 External Domains, 4 Internal VMs) |
|  Critical Severity Findings:   2 (Immediate RCE / Backdoor risk)      |
|  High Severity Findings:       3 (Origin IP leak, SMBv1, Tomcat Mgr)  |
|  Medium Severity Findings:     4 (Exposed RTSP, Cleartext Telnet/FTP) |
|  Low / Informational:          5 (DNS records, OS banners)           |
+-----------------------------------------------------------------------+
```

---

## 2. AI Ingestion & Analysis Architecture

Rather than executing a single generic prompt, a **4-stage sequential prompt chain** was implemented to minimize hallucinations and maximize technical depth:

```
+--------------------------+     +--------------------------+
|  OSINT Logs (Project 4)  |     | Network Scans (Project 5)|
+------------+-------------+     +------------+-------------+
             |                                |
             +---------------+----------------+
                             |
                             v
           +-----------------------------------+
           | Step 1: Ingestion & Normalization | (Convert XML/Text to clean JSON)
           +-----------------+-----------------+
                             |
                             v
           +-----------------------------------+
           | Step 2: Vulnerability Correlation | (Map CPEs -> CVEs & EPSS scores)
           +-----------------+-----------------+
                             |
                             v
           +-----------------------------------+
           | Step 3: Attack Path Modeling      | (Graph external-to-internal pivot)
           +-----------------+-----------------+
                             |
                             v
           +-----------------------------------+
           | Step 4: Human-in-the-Loop Review  | (Validate against NVD / CISA KEV)
           +-----------------+-----------------+
                             |
                             v
           +-----------------------------------+
           | Final Executive & Tech ASM Report |
           +-----------------------------------+
```

---

## 3. The AI Prompt Engineering Pipeline

### Stage 1: Data Normalization Prompt
```text
System: You are an automated SecOps log parser.
Input: Raw Nmap output and OSINT text tables.
Task: Extract each distinct asset into a unified JSON object containing:
      IP, Hostname, Network Zone (External/Internal), Open Port, Protocol, 
      Software Product Name, and Exact Version String.
```

### Stage 2: Vulnerability & Threat Mapping Prompt
```text
System: You are a Senior Threat Intelligence Analyst.
Task: For each software version in the parsed JSON:
      1. Identify known high-impact CVEs (CVSS >= 7.0).
      2. Provide the EPSS (Exploit Prediction Scoring System) likelihood percentage.
      3. State whether the flaw has known public exploit code (Metasploit / PoC).
      4. Explicitly cite the CVE identifier.
```

### Stage 3: Attack Path Generation Prompt
```text
System: You are a Red Team Lead.
Task: Connect the external OSINT discoveries (Mini Project 4) to internal network assets (Mini Project 5).
      Construct the most probable end-to-end exploit chain an adversary would follow.
```

---

## 4. Correlated Vulnerability & Attack Surface Matrix

The table below synthesizes findings across both projects, correlated through the AI pipeline and verified manually against the National Vulnerability Database (NVD):

| Asset Identifier | Zone | Port / Service | Software & Version | Matched CVE | CVSS v3 | Threat Impact |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `staging-api.novatech-fin.org` (`198.51.100.44`) | External | `8080/tcp` (Tomcat) | Apache Tomcat 8.5.39 | CVE-2020-1938 / Weak Auth | **8.8 High** | Tomcat Manager exposure allows WAR deployment and remote shell. |
| `staging-api.novatech-fin.org` (`198.51.100.44`) | External | `22/tcp` (SSH) | OpenSSH 7.4 | CVE-2018-15473 | **5.3 Med** | User enumeration flaw allows automated username brute-forcing. |
| `192.168.56.20` (`corp-storage-old`) | Internal | `21/tcp` (FTP) | vsftpd 2.3.4 | CVE-2011-2523 | **9.8 Crit** | Hardcoded smiley-face backdoor triggers root shell on port 6200. |
| `192.168.56.20` (`corp-storage-old`) | Internal | `139/445` (Samba) | Samba 3.0.20 | CVE-2007-2447 | **9.8 Crit** | `username map script` vulnerability enables unauthenticated root RCE. |
| `192.168.56.35` (`DC-WIN2016`) | Internal | `445/tcp` (SMB) | Windows SMBv1 active | CVE-2017-0144 | **8.1 High** | SMBv1 protocol active; vulnerable to EternalBlue / lateral worming. |
| `192.168.56.50` (`cam-lobby-04`) | Internal | `554/tcp` (RTSP) | Live Stream Daemon | CWE-306 | **6.5 Med** | Unauthenticated video stream allows physical perimeter snooping. |

---

## 5. End-to-End Attack Path Synthesis

The AI model constructed the following realistic attack sequence demonstrating how disparate low/medium findings chain into complete network compromise:

```
[Phase 1: External Recon (Project 4)]
  Attacker searches crt.sh -> Discovers 'staging-api.novatech-fin.org'
  Origin IP 198.51.100.44 is exposed directly, completely bypassing Cloudflare WAF.
                     │
                     ▼
[Phase 2: Initial Compromise]
  Attacker accesses Tomcat Manager on port 8080.
  Credentials harvested from public GitHub leak (.env.example) yield successful login.
  Attacker deploys a malicious WAR payload -> Gains reverse shell on staging machine.
                     │
                     ▼
[Phase 3: Internal Pivot (Project 5)]
  Staging machine has secondary network interface into lab subnet (192.168.56.0/24).
  Attacker runs internal ping sweep -> Locates legacy storage host 192.168.56.20.
                     │
                     ▼
[Phase 4: Root Privilege Escalation & Lateral Spread]
  Attacker executes Samba 'usermap script' exploit (CVE-2007-2447) against 192.168.56.20.
  Instant root access acquired.
                     │
                     ▼
[Phase 5: Domain Dominance]
  From the compromised storage server, attacker launches credential extraction against
  Windows Server (192.168.56.35) exploiting SMBv1 / Kerberoasting -> Total Domain Control.
```

---

## 6. Human-in-the-Loop (HITL) Validation & Hallucination Check

A critical aspect of using AI in security operations is human validation. During our review, the AI's raw output was compared against the official NVD database:

* **Finding 1 (vsftpd 2.3.4):** AI attributed `CVE-2011-2523` with a CVSS score of 9.8. **Verified Correct.** The backdoor mechanism is well-documented in NVD.
* **Finding 2 (OpenSSH 7.4):** AI initially claimed that OpenSSH 7.4 suffered from a remote pre-authentication code execution vulnerability (referencing a non-existent CVE). **Flagged & Corrected.** Upon manual verification, OpenSSH 7.4 on that build only suffers from user enumeration (`CVE-2018-15473`, Medium severity). The score was adjusted from 9.8 down to 5.3.
* **Takeaway:** While AI accelerates correlation by 90%, all high and critical vulnerability tags must be verified against official CVE dictionaries and vendor patch notes before presenting to executive leadership.

---

## 7. Prioritized Remediation Roadmap (30 / 60 / 90 Days)

### Immediate Remediation (Within 30 Days) — Eliminate Critical Exposure
1. **Firewall Staging Origin Server:**
   * Restrict access to `198.51.100.44` so port 8080 is accessible only via internal VPN IP addresses.
2. **Decommission Vulnerable Legacy Services:**
   * Shut down `vsftpd` and old Samba on `192.168.56.20`. Migrate necessary data to modern encrypted network shares.
3. **Disable SMBv1 Across the Domain:**
   * Run the following PowerShell command across all Windows endpoints:
     ```powershell
     Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol -NoRestart
     ```

### Mid-Term Remediation (Within 60 Days) — Identity & Segmentation Hardening
1. **Enforce Multi-Factor Authentication (MFA):**
   * Mandate MFA on all remote access gateways (`vpn.novatech-fin.org` and O365).
2. **VLAN Segmentation:**
   * Move the IoT camera (`192.168.56.50`) into an isolated IoT VLAN with firewall rules prohibiting outbound communication to internal servers.

### Long-Term Remediation (Within 90 Days) — Continuous Attack Surface Management
1. **Automated Secret Scanning:**
   * Deploy GitHub secret scanning (TruffleHog / GitGuardian) in developer CI/CD pipelines to prevent internal hostnames and API keys from leaking into public repos.
2. **Continuous External Attack Surface Monitoring:**
   * Schedule automated weekly certificate transparency queries and Shodan alerts to detect new unmapped subdomains as soon as they appear.

---

## 8. Student Reflection & Conclusion

This project illustrated the power and caveats of utilizing AI in offensive and defensive security operations. Processing hundreds of lines of scan output manually is tedious and error-prone; the AI pipeline organized the data, highlighted key correlations, and generated cohesive attack narratives in minutes. However, the human validation phase confirmed that security analysts cannot blindly accept AI outputs without cross-checking against official CVE records. The combination of automated AI parsing and human analytical judgment represents the future of effective cyber defense.
