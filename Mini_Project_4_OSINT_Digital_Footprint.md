# MINI PROJECT 4: OSINT & DIGITAL FOOTPRINT ASSESSMENT

**Course:** Cyber Security / Network Security Lab  
**Academic Year:** 2026–2027  
**Assessment Type:** Passive Reconnaissance & Digital Exposure Audit  
**Target Organization (Simulated / Authorized Scope):** Novatech Financial Solutions (`novatech-fin.internal` / Public Sandbox `novatech-fin.org`)  
**Rules of Engagement:** Strictly Non-Intrusive / Passive Reconnaissance (No direct port sweeps, no active web application exploits, no unauthorized logins).

---

## 1. Executive Summary

This assessment was carried out to evaluate the external exposure and attack surface of Novatech Financial Solutions from the perspective of an external threat actor. During the reconnaissance phase, passive intelligence techniques were applied to discover publicly available information regarding the organization's domain architecture, DNS records, public email structures, employee identities, leaked credentials, and exposed cloud storage.

The investigation revealed multiple areas of high exposure:
1. **Public Subdomain Disclosures:** Unshielded Certificate Transparency logs and historical DNS dumps uncovered several internal-facing staging environments (`staging-api.novatech-fin.org`, `dev-portal.novatech-fin.org`).
2. **Identity & Employee Profiling:** Clear employee email conventions (`{first_initial}{last_name}@novatech-fin.org`) and organizational roles were mapped using professional networks and search dorks, providing viable targets for credential spraying and spear-phishing.
3. **Leaked Historical Credentials:** Historical data breach archives revealed several corporate email addresses with associated plaintext passwords and hashes.
4. **Exposed Third-Party Repository:** A public repository linked to a junior developer contained an active internal API endpoint and a staging AWS S3 bucket reference.

---

## 2. Methodology & Reconnaissance Workflow

The assessment adhered strictly to passive OSINT practices following the OSINT Framework and MITRE ATT&CK Reconnaissance tactics (`TA0043`):

```
                   +----------------------------+
                   | Passive Intelligence Scope |
                   +--------------+-------------+
                                  |
         +------------------------+------------------------+
         |                                                 |
         v                                                 v
[Domain & Infrastructure]                       [Identity & Social Footprint]
  - WHOIS Records                                 - theHarvester (Emails/Names)
  - Certificate Transparency (crt.sh)             - LinkedIn / Google Dorking
  - DNSDumpster / SecurityTrails                  - DeHashed / HaveIBeenPwned
         |                                                 |
         +------------------------+------------------------+
                                  |
                                  v
                      [Threat Exposure & Assets]
                        - Shodan / Censys (Passive)
                        - GitHub / Public Buckets
                        - Risk Matrix & Remediation
```

---

## 3. Step-by-Step Execution & Tool Evidence

### Step 3.1: Domain & WHOIS Intelligence
* **Command Executed:**
  ```bash
  whois novatech-fin.org
  ```
* **Extracted Information:**
  * **Registrar:** NameCheap, Inc.
  * **Registration Date:** 2021-04-12
  * **Name Servers:** `ns1.cloudprovider-dns.com`, `ns2.cloudprovider-dns.com`
  * **Privacy Shield:** Withheld for Privacy (WhoisGuard enabled).
  * **Registrant Country:** IN (India)

### Step 3.2: DNS & Subdomain Enumeration (Certificate Transparency)
* **Technique:** Queried public Certificate Transparency (CT) logs on `crt.sh` and parsed DNS records via DNSdumpster to map the subdomain tree without generating network traffic against the target servers.
* **Query:**
  ```bash
  curl -s "https://crt.sh/?q=%.novatech-fin.org&output=json" | jq -r '.[].name_value' | sort -u
  ```
* **Discovered Records:**

| Subdomain | Resolved IP | Service / Purpose | Risk Classification |
| :--- | :--- | :--- | :--- |
| `novatech-fin.org` | `104.21.54.12` | Main Marketing Site (Cloudflare CDN) | Low |
| `portal.novatech-fin.org` | `104.21.54.12` | Customer Login Portal | Medium |
| `staging-api.novatech-fin.org` | `198.51.100.44` | Direct AWS EC2 Instance (Bypasses CDN) | **High (Direct Origin Exposed)** |
| `vpn.novatech-fin.org` | `198.51.100.8` | Pulse Secure / Fortinet Gateway | **High (Authentication Perimeter)** |
| `mail.novatech-fin.org` | `198.51.100.15` | Microsoft 365 Exchange Hybrid | Medium |

### Step 3.3: Email Harvesting & Identity Profiling
* **Tool Used:** `theHarvester`
* **Command Executed:**
  ```bash
  theHarvester -d novatech-fin.org -b google,duckduckgo,linkedin -l 100
  ```
* **Observed Findings:**
  * **Email Syntax Pattern:** Identified consistent pattern: `first.last@novatech-fin.org`.
  * **Discovered Addresses:**
    * `r.kulkarni@novatech-fin.org` (Infrastructure & DevOps Lead)
    * `s.deshmukh@novatech-fin.org` (Senior Financial Analyst)
    * `admin.portal@novatech-fin.org` (Shared Service Account)
    * `billing@novatech-fin.org` (Finance Support)
* **Credential Breach Exposure Check (HaveIBeenPwned & Intelligence Datasets):**
  * `r.kulkarni@novatech-fin.org` appeared in two previous third-party breaches (Dropbox 2016, LinkedIn 2021).
  * The existence of old plain-text password dumps poses a credential-reuse risk across internal SSO services.

### Step 3.4: Google Dorking for Information Leakage
* **Queries Executed:**
  1. `site:novatech-fin.org filetype:pdf OR filetype:xlsx "confidential"`
     * *Result:* Found a publicly indexed PDF titled `Q2_Internal_Compliance_Checklist.pdf` containing internal system hostnames and software versions.
  2. `site:github.com "novatech-fin.org"`
     * *Result:* Located a personal repository of a former contractor containing an unencrypted `.env.example` referencing internal hostnames and an Amazon S3 bucket name: `novatech-prod-backups-mumbai`.

### Step 3.5: Passive Infrastructure Profiling via Shodan
* **Technique:** Queried Shodan database cache using the discovered direct IP `198.51.100.44` without initiating active packets.
* **Cached Results:**
  * **Port 22/tcp:** OpenSSH 7.4 (Outdated release; deprecated cipher suites enabled).
  * **Port 8080/tcp:** Apache Tomcat/8.5.39 (Manager interface exposed to the public internet).
  * **SSL Certificate:** Self-signed certificate issued to `staging-internal.novatech-fin.local`.

---

## 4. Risk Assessment & Threat Modeling

| Finding Reference | Vulnerability / Exposure | Threat Actor Vector | CVSS Base Score |
| :--- | :--- | :--- | :--- |
| **OBS-01** | Origin IP leakage via staging subdomain (`198.51.100.44`) bypassing CDN/WAF. | Direct DDoS or unshielded vulnerability exploitation. | 7.5 (High) |
| **OBS-02** | Exposure of employee email naming convention and corporate accounts in breach dumps. | Targeted spear-phishing and password spraying against Microsoft 365. | 6.5 (Medium) |
| **OBS-03** | Publicly accessible compliance PDF exposing internal hostnames. | Low-friction intelligence gathering for network lateral movement planning. | 4.3 (Low) |
| **OBS-04** | Exposed Tomcat Manager interface on non-standard port 8080. | Brute-force attacks against default Tomcat credentials to achieve RCE. | 8.8 (High) |

---

## 5. Defensive Recommendations & Hardening Plan

1. **Origin IP Cloaking & DNS Hygiene:**
   * Remove legacy and development DNS records (`staging-api`, `dev-portal`) from public DNS zones. Route all non-production traffic through an authenticated VPN or internal zero-trust proxy (e.g., Cloudflare Access or Tailscale).
   * Restrict access to `198.51.100.44` using AWS Security Group ingress rules allowing traffic only from Cloudflare IP ranges.

2. **Credential & Phishing Defense:**
   * Enforce phishing-resistant Multi-Factor Authentication (MFA via FIDO2/WebAuthn or Authenticator apps) for all corporate Microsoft 365 and VPN logins.
   * Mandate an automated password reset for all email addresses flagged in historical breach datasets.

3. **Public Exposure Monitoring:**
   * Implement automated alerts on public source repositories (GitHub secret scanning) and continuous Certificate Transparency log monitoring to detect orphaned subdomains.
   * Add a `robots.txt` disallow directive and secure HTTP headers (`X-Robots-Tag: noindex, noarchive`) on non-public sensitive document directories.

---

## 6. Student Reflection & Conclusion

Executing this passive footprinting exercise demonstrated how much operational intelligence can be gathered about a company without sending a single intrusive probe. Attackers do not need to exploit a complex zero-day when outdated staging servers, default administrative interfaces, and employee email structures sit unprotected in public indices. Modern enterprise defense requires ongoing OSINT monitoring of the organization's own external perimeter.
