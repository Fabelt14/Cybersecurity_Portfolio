# OSINT Investigation Report
## Target: TechVision Consulting Inc.

---

> **Classification:** TLP:AMBER — For Authorised Recipients Only
> **Report Type:** Open Source Intelligence (OSINT) Assessment
> **Analyst:** Cyber Threat Intelligence Analyst — Security Operations Center
> **Report Date:** 01 October 2026
> **Overall Threat Level:** 🔴 HIGH
> **Investigation Type:** Passive Reconnaissance Only — No Active Scanning

---

## Executive Summary

An open source intelligence investigation of **TechVision Consulting Inc.** (`techvision.com`) reveals an **overall High Threat Level** — driven not by doubt about the company's legitimacy, but by a cluster of critical infrastructure misconfigurations that leave sensitive corporate assets directly exposed to the public internet, compounded by an active external brand impersonation campaign.

TechVision demonstrates verified corporate presence: ISO 27001 certification, SOC 2 Type II compliance, a nine-year domain history, identified executive leadership with verifiable careers, and consistent thought leadership at major industry conferences including RSA and Black Hat. These are markers of a legitimate, operating business.

The threat is not the company — **the threat is what attackers can reach right now without any credentials:**

- **Live PostgreSQL database servers** (`203.45.67.200` and `203.45.67.201`) are accessible from the public internet with no VPN restriction
- **Development and staging environments** (`dev.techvision.com`, `staging.techvision.com`) are publicly reachable, exposing pre-release code and internal tooling
- **Three AWS S3 buckets** (`techvision-backups`, `techvision-logs`, `techvision-documents`) allow public read access, containing system logs, database backups, and internal security playbooks
- **Employee credentials and developer handles** (`s.chen`, `m.park`, `e.watson`) have been leaked through public GitHub repositories and document metadata
- **Three impersonation domains and two spoofed social media accounts** are actively distributing fake "security patch" lures — sharing the same corporate IP subnet as legitimate TechVision assets

**Three actions require immediate execution within 24–48 hours:**

1. Restrict database servers and non-production environments behind the corporate VPC — today
2. Enable S3 Block Public Access on all three exposed buckets and audit IAM permissions
3. Initiate takedown requests for `techvision-secure.com`, `techvision-update.com`, `@TechVisionSecure`, and `@TechVisionUpdate`

---

## Investigation Scope and Methodology

### Scope

| Dimension | Detail |
|---|---|
| **Target Organisation** | TechVision Consulting Inc. |
| **Primary Domain** | `techvision.com` |
| **Investigation Type** | Passive OSINT — no active scanning, no authentication attempts, no intrusion |
| **Intelligence Sources** | WHOIS, DNS records, Certificate Transparency logs, social media, GitHub, document metadata |
| **Data Collection Period** | October 2026 |

### Methodology

All findings were produced using **passive reconnaissance only**. No port scanning, credential testing, or active probing was conducted against TechVision's infrastructure at any point. Every data point in this report is derived from publicly available open sources — the same sources available to any external threat actor conducting pre-attack reconnaissance.

Intelligence collection followed four parallel tracks:

| Track | Method |
|---|---|
| **Domain & Infrastructure Intelligence** | WHOIS lookups, DNS record parsing, Certificate Transparency log analysis, IP geolocation, ASN mapping |
| **Personnel & Social Media Intelligence (SOCMINT)** | LinkedIn profile analysis, Twitter/X account mapping, GitHub organisation review, employee activity pattern analysis |
| **Technical Analysis** | HTTP header fingerprinting, technology stack identification, vulnerability correlation against public CVE databases |
| **Metadata & Document Analysis** | ExifTool-based metadata extraction from publicly accessible documents, GitHub repository code review, S3 bucket content analysis |

**Verification method:** Findings were cross-referenced across multiple independent sources before inclusion. For example, developer usernames identified in GitHub commit logs were validated against LinkedIn employee profiles and document metadata author fields. Active IP addresses from DNS records were independently verified against WHOIS registrant data and SSL Certificate Transparency logs.

---

## Part 1 — Domain and Infrastructure Intelligence

### 1.1 — Domain Inventory

TechVision operates across seven primary domains and ten identified subdomains, with three additional privacy-protected domains flagged as suspicious.

**Primary Domains:**

| Domain | Status | Purpose |
|---|---|---|
| `techvision.com` | ✅ Legitimate | Global headquarters — primary web presence |
| `techvision.io` | ✅ Legitimate | Technology brand variant |
| `techvision-consulting.com` | ✅ Legitimate | Alternative corporate domain |
| `techvision.uk` | ✅ Legitimate | United Kingdom regional portal |
| `techvision.eu` | ✅ Legitimate | European regional portal |
| `techvision.sg` | ✅ Legitimate | Singapore regional portal |
| `techvision.au` | ✅ Legitimate | Australia regional portal |

**Identified Subdomains:**

| Subdomain | Service |
|---|---|
| `www.techvision.com` | Web frontend |
| `mail.techvision.com` | Email server |
| `api.techvision.com` | API endpoint |
| `dev.techvision.com` | ⚠️ Development environment — publicly accessible |
| `staging.techvision.com` | ⚠️ Staging environment — publicly accessible |
| `careers.techvision.com` | Recruitment portal |
| `blog.techvision.com` | Company blog |
| `support.techvision.com` | Support portal |
| `vpn.techvision.com` | VPN gateway |
| `admin.techvision.com` | ⚠️ Administration panel |

**Suspicious / Privacy-Protected Domains:**

| Domain | Status | Risk |
|---|---|---|
| `techvision-secure.com` | 🔴 Suspicious | Privacy-protected registration — active impersonation infrastructure |
| `techvision-update.com` | 🔴 Suspicious | Privacy-protected registration — active fake "security patch" distribution |
| `techvision-support.net` | 🔴 Suspicious | Privacy-protected registration — impersonation support channel |

---

![WHOIS registration data for techvision.com showing registrant name TechVision Consulting Inc, admin email admin@techvision.com, registration date 2015-03-15, expiration 2026-03-15, registrar GoDaddy, status clientTransferProhibited](images/part1_whois.png)

*Figure 1 — WHOIS record for `techvision.com`. The nine-year domain history (registered 2015), fully transparent registrant details, and clientTransferProhibited status are positive legitimacy indicators. By contrast, the three suspicious domains (`techvision-secure.com`, `techvision-update.com`, `techvision-support.net`) use WHOIS privacy protection — obscuring their true registrant identities.*

---

### 1.2 — WHOIS Analysis

**Registrant Information (techvision.com):**

| Field | Value |
|---|---|
| **Registrant Name** | TechVision Consulting Inc |
| **Organisation** | TechVision Consulting Inc |
| **Admin Email** | admin@techvision.com |
| **Phone** | +1.415.555.0100 |
| **Address** | 1500 Tech Boulevard, San Francisco, CA 94105, USA |
| **Registration Date** | 2015-03-15 |
| **Expiration Date** | 2026-03-15 |
| **Registrar** | GoDaddy.com, Inc. |
| **Status** | clientTransferProhibited |

**Intelligence Assessment:** A nine-year domain history with transparent registrant details is a strong legitimacy indicator. Domain squatters and short-lived fraud infrastructure typically register domains weeks or months before use and rely on privacy protection to conceal registrant identity. The `clientTransferProhibited` status indicates active registrar-level lock — standard corporate domain hygiene.

**Contrast with suspicious domains:** All three suspicious domains (`techvision-secure.com`, `techvision-update.com`, `techvision-support.net`) use WHOIS privacy protection services, preventing direct attribution. Their true registrant identities are unknown and warrant formal legal process (subpoena to the respective registrars) to uncover.

---

### 1.3 — DNS Records Analysis

**Web Server A Records:**

| IP Address | Hostname | Hosting Provider | Country | Service |
|---|---|---|---|---|
| `203.45.67.89` | `www.techvision.com` | AWS CloudFront (CDN) | USA | Web Frontend |
| `203.45.67.90` | `mail.techvision.com` | AWS EC2 | USA | Email Server |
| `203.45.67.91` | `api.techvision.com` | AWS EC2 | USA | API Endpoint |
| `203.45.67.92` | `dev.techvision.com` | AWS EC2 | USA | ⚠️ Development Environment |
| `203.45.67.93` | `staging.techvision.com` | AWS EC2 | USA | ⚠️ Staging Environment |
| `203.45.67.96` | `vpn.techvision.com` | AWS EC2 | USA | VPN Gateway |
| `203.45.67.97` | `admin.techvision.com` | AWS EC2 | USA | ⚠️ Administration Panel |
| `203.45.67.100–103` | `techvision.uk/eu/sg/au` | AWS EC2 | UK/EU/SG/AU | Regional Portals |
| **`203.45.67.150`** | **`techvision-secure.com`** | **GoDaddy Hosting** | **USA** | **🔴 Impersonation Domain** |
| **`203.45.67.151`** | **`techvision-update.com`** | **NameCheap Hosting** | **USA** | **🔴 Impersonation Domain** |

**Most significant finding in the DNS data:** The suspicious impersonation domains (`203.45.67.150–152`) resolve within the same `/24` subnet (`203.45.67.x`) as TechVision's legitimate corporate infrastructure. This co-location is the most alarming signal in the entire investigation — it means the impersonation infrastructure is either hosted on TechVision's own network (indicating a rogue insider or a compromised administrative account) or the adversary has deliberately placed their infrastructure within TechVision's corporate IP range to give spoofed communications a veneer of legitimacy in IP reputation checks.

---

![DNS A record table showing all TechVision IP addresses from 203.45.67.89 to 203.45.67.152 with hostnames, hosting providers, and service descriptions — highlighting suspicious domain IPs 203.45.67.150-152 sharing the corporate subnet](images/part1_dns_records.png)

*Figure 2 — DNS A record infrastructure map. All legitimate TechVision assets and all three suspicious domains resolve within the same `203.45.67.x` subnet. The subnet co-location of impersonation infrastructure with legitimate corporate assets is the single most critical finding in this investigation.*

---

**MX Records (Mail Servers):**

| Priority | Hostname | Purpose |
|---|---|---|
| 10 | `mail.techvision.com` | Primary corporate mail server |
| 10 | `mail.techvision.io` | Secondary brand mail |
| 10 | `mail.techvision-consulting.com` | Alternative corporate mail |
| 10 | `mail.techvision.uk/eu/sg/au` | Regional mail servers |
| 10 | **`mail.techvision-secure.com`** | ⚠️ **Impersonation mail server** |
| 10 | **`mail.techvision-update.com`** | ⚠️ **Impersonation mail server** |
| 10 | **`mail.techvision-support.net`** | ⚠️ **Impersonation mail server** |
| 20 | `mail2.techvision.com` | Backup mail server |

**Critical observation:** The impersonation domains maintain active mail servers at MX priority 10 — the same priority as TechVision's legitimate regional mail infrastructure. This means the impersonation domains are fully configured to send and receive email, not merely host static phishing pages. Spearphishing emails sent from `@techvision-secure.com` would appear to originate from a TechVision-affiliated address and would pass many basic email filtering checks.

---

**Email Authentication Records:**

| Record Type | Value | Assessment |
|---|---|---|
| **SPF** | `v=spf1 include:_spf.google.com ~all` | ✅ Google Workspace authorised as sender |
| **DMARC** | `v=DMARC1; p=quarantine; rua=...` | ✅ Quarantine policy — failed emails held, not delivered |
| **DKIM** | RSA public key in `default._domainkey` | ✅ Cryptographic email signing active |

**Intelligence Assessment:** The core `techvision.com` domain implements all three email authentication standards (SPF, DKIM, DMARC) with a quarantine enforcement policy. This is above-average corporate email hygiene. However, SPF/DKIM/DMARC on the primary domain does **not** protect against emails sent from the impersonation domains (`techvision-secure.com`, `techvision-update.com`) — those domains have their own mail infrastructure and can send emails that pass their own authentication checks.

---

### 1.4 — Infrastructure Mapping

**IP Address Summary:**

| IP Address | Hostname | Hosting Provider | Service |
|---|---|---|---|
| `203.45.67.89` | `www.techvision.com` | AWS CloudFront | Web Frontend (CDN) |
| `203.45.67.90` | `mail.techvision.com` | AWS EC2 | Email Server |
| `203.45.67.91` | `api.techvision.com` | AWS EC2 | API Endpoint |
| `203.45.67.92` | `dev.techvision.com` | AWS EC2 | ⚠️ Dev Environment |
| `203.45.67.93` | `staging.techvision.com` | AWS EC2 | ⚠️ Staging Environment |
| `203.45.67.94` | `blog.techvision.com` | Medium.com | Company Blog |
| `203.45.67.95` | `support.techvision.com` | Zendesk | Support Portal |
| `203.45.67.96` | `vpn.techvision.com` | AWS EC2 | VPN Gateway |
| `203.45.67.97` | `admin.techvision.com` | AWS EC2 | ⚠️ Admin Panel |
| `203.45.67.100–103` | Regional domains | AWS EC2 | UK/EU/SG/AU Portals |
| **`203.45.67.200`** | DB Server 1 | AWS RDS | 🔴 **Live Database — Public** |
| **`203.45.67.201`** | DB Server 2 | AWS RDS | 🔴 **Live Database — Public** |
| `203.45.67.150–152` | Impersonation domains | GoDaddy / NameCheap | 🔴 Threat Actor Infrastructure |

**Hosting Provider Breakdown:**
- **Primary:** Amazon Web Services — EC2, RDS, CloudFront, Route53 (us-west-2 / Oregon region, ASN 16509)
- **Email SaaS:** Google Workspace
- **Support:** Zendesk
- **Blog:** Medium.com
- **Impersonation:** GoDaddy Hosting, NameCheap Hosting

---

![Infrastructure IP mapping table showing all TechVision assets from 203.45.67.89 to 203.45.67.201 with hosting providers — highlighting database servers 203.45.67.200 and 203.45.67.201 as publicly accessible AWS RDS instances with no VPN restriction](images/part1_infrastructure_map.png)

*Figure 3 — Full infrastructure IP mapping. The two database servers at `203.45.67.200` and `203.45.67.201` (bottom of the table) are AWS RDS instances reachable from the public internet — not isolated within a private VPC. These represent the most immediately exploitable finding in the investigation.*

---

**Vulnerabilities Identified Through Infrastructure Analysis:**

| Vulnerability | Severity | Description |
|---|---|---|
| Internet-exposed database servers (`203.45.67.200`, `203.45.67.201`) | 🔴 **Critical** | Live PostgreSQL databases reachable from the public internet without VPN or firewall restriction |
| Publicly accessible dev and staging environments | 🔴 **Critical** | Pre-release source code, test datasets, and internal tooling accessible without authentication |
| Misconfigured public S3 buckets (`techvision-backups`, `techvision-logs`, `techvision-documents`) | 🔴 **Critical** | Public read access permits immediate download of system backups, logs, and internal playbooks |
| Internal DNS naming leakage | 🟡 **Medium** | Public DNS exposes internal naming conventions (`db-primary.internal`, `cache.internal`, `queue.internal`) revealing network topology to external actors |

---

## Part 2 — Personnel and Social Media Intelligence

### 2.1 — Executive and Security Leadership

**Executive Leadership:**

| Name | Title | Department | LinkedIn |
|---|---|---|---|
| James Mitchell | Chief Executive Officer | Executive | `linkedin.com/in/james-mitchell-ceo` |
| Sarah Chen | Chief Technology Officer | Engineering | Cloud Architecture, Zero-Trust |
| David Rodriguez | VP Sales | Sales | Business Development |

**Security Team:**

| Name | Title | Department | Expertise |
|---|---|---|---|
| Emily Watson | VP Security | Cybersecurity | Threat Intelligence |
| Michael Park | Senior Security Engineer | Security Engineering | Incident Response, Security Tooling |

**Organisational Structure:**

```
James Mitchell (CEO)
    │
    ├── Sarah Chen (CTO)
    │       ├── Senior Software Engineers
    │       └── Network & DevOps Engineers
    │
    ├── Emily Watson (VP Security)
    │       ├── Michael Park (Senior Security Engineer)
    │       └── Security Analysts / IR Team
    │
    └── David Rodriguez (VP Sales)
            ├── Sales Team
            └── Business Development Team
```

**Global Office Presence:**
San Francisco HQ · London · Berlin · Paris · Tokyo · Singapore · Sydney

---

![LinkedIn profile and organisational chart for TechVision Consulting showing executive hierarchy from CEO James Mitchell through CTO Sarah Chen and VP Security Emily Watson with department breakdowns and global office locations](images/part2_org_chart.png)

*Figure 4 — TechVision organisational mapping derived from LinkedIn profiles, GitHub contributor histories, and whitepaper author metadata. The executive team is fully identifiable with verifiable career backgrounds — a positive legitimacy signal, but also a spearphishing attack surface: targeted email lures impersonating James Mitchell or Emily Watson would carry high credibility with employees.*

---

**Intelligence Assessment — Personnel Exposure:**

The complete executive team is publicly identifiable through LinkedIn, with verifiable career histories. This creates two competing intelligence values:

**Positive signal:** Genuine executives with traceable professional backgrounds are a legitimacy indicator — front organisations typically cannot construct this level of verifiable executive presence.

**Threat signal:** A fully mapped executive hierarchy enables highly targeted spearphishing. An attacker can construct a Business Email Compromise (BEC) lure impersonating CEO James Mitchell's communication style and email formatting, directed at the finance team. The email pattern (`firstname.lastname@techvision.com`) for 20+ identified employees was publicly recoverable — providing a ready-made targeted phishing list.

---

### 2.2 — Social Media Intelligence

**Official Accounts:**

| Platform | Handle / URL | Followers | Verification |
|---|---|---|---|
| LinkedIn | `/company/techvision-consulting` | 12,500+ | ✅ Official company page |
| Twitter (X) | `@TechVisionCorp` | 8,200+ | ✅ Verified |
| Facebook | `/techvisionconsulting` | 3,400+ | ✅ Official page |
| Instagram | `@techvisioncorp` | 2,100+ | ✅ Official account |
| GitHub | `github.com/techvision-consulting` | 15+ repositories | ✅ Official organisation |

**Suspicious / Impersonation Accounts:**

| Account | Platform | Created | Activity | Risk |
|---|---|---|---|---|
| `@TechVisionSecure` | Twitter / X | June 2024 | Minimal — uses official TechVision branding | 🔴 Brand impersonation |
| `@TechVisionUpdate` | Twitter / X | July 2024 | Posts fake "security updates" and "patch" notices | 🔴 Active social engineering / credential phishing |
| TechVision Consulting (unverified) | Facebook | August 2024 | Posts "urgent security updates" — mimics corporate communications | 🔴 Active phishing / scam operation |

---

![Twitter/X side-by-side comparison showing official verified @TechVisionCorp account versus suspicious unverified @TechVisionSecure and @TechVisionUpdate accounts with similar branding but no verification badge and recent creation dates](images/part2_social_media.png)

*Figure 5 — Comparison of official TechVision social media accounts versus identified impersonation accounts. The impersonation accounts use TechVision's official brand imagery without verification, were created 4–6 months before this investigation, and are actively distributing fake "security patch" lures. These accounts share IP infrastructure with TechVision's corporate subnet — suggesting the threat actor has direct knowledge of TechVision's network addressing.*

---

**Employee Activity Pattern Analysis:**

| Dimension | Finding |
|---|---|
| **Most active hours** | Weekdays 9:00 AM – 5:00 PM PT (consistent with San Francisco HQ business hours) |
| **Peak days** | Tuesday through Thursday |
| **Primary platforms** | LinkedIn (executive and general staff), Twitter/X (leadership and security commentary), GitHub (engineering and security team) |
| **Common topics** | Cloud Security (25%), Threat Intelligence (20%), Incident Response (15%), Zero-Trust Architecture (12%) |

**Social Engineering Exposure Assessment:** The combination of publicly mapped executive profiles, identifiable email patterns, consistent posting schedules, and active impersonation infrastructure creates a high-risk social engineering environment. Employees and clients who receive emails or social media messages from `@TechVisionSecure` or `techvision-update.com` have no immediately visible technical signal that these sources are fraudulent.

---

## Part 3 — Technical Analysis and Vulnerability Assessment

### 3.1 — Technology Stack Fingerprinting

| Layer | Technology | Risk |
|---|---|---|
| **Web Server** | Nginx (version disclosed in default response headers) | Server version exposed aids targeted CVE exploitation |
| **Frontend** | React.js with npm dependency stack | Outdated dependencies detected with known CVEs |
| **Backend** | Node.js / Python REST API | Custom framework — configuration errors possible |
| **Database** | PostgreSQL on AWS RDS | 🔴 Publicly accessible — critical exposure |
| **CDN** | AWS CloudFront | No identified risk |
| **Email** | Google Workspace | Standard risk — SaaS supply chain exposure |
| **Support** | Zendesk | SaaS supply chain exposure |
| **Analytics** | Google Analytics | Data sharing with third party |

---

### 3.2 — Vulnerability Assessment

**Critical Severity:**

**1 — Internet-Exposed PostgreSQL Database Servers**
`203.45.67.200` and `203.45.67.201` are AWS RDS instances reachable from the public internet. These are not development databases — they are production PostgreSQL servers hosting live corporate data. Any threat actor with valid credentials (recoverable from the leaked developer handles and exposed GitHub repositories) has a direct, authenticated path to the full database. Even without credentials, public exposure enables brute-force authentication attempts and exploitation of PostgreSQL-specific vulnerabilities.

> **Immediate action:** Restrict all inbound connections to `203.45.67.200` and `203.45.67.201` to the private VPC security group. No production database should have a public IP assignment.

**2 — Publicly Accessible Development and Staging Environments**
`dev.techvision.com` and `staging.techvision.com` are publicly reachable without authentication or VPN restriction. Development environments routinely contain hardcoded credentials, debug endpoints, disabled security controls, and pre-release code that has not undergone security review. Access to the staging environment provides an attacker with a near-complete replica of the production application — including its vulnerabilities.

> **Immediate action:** Place both subdomains behind the corporate VPN (`vpn.techvision.com`). Remove public DNS resolution for `dev` and `staging` subdomains.

**3 — Publicly Misconfigured AWS S3 Buckets**
Three S3 buckets (`techvision-backups`, `techvision-logs`, `techvision-documents`) permit unauthenticated public read access. Confirmed contents include:

- System transaction logs and database backup structures (`techvision-backups`)
- Application and infrastructure event logs (`techvision-logs`)
- Internal security playbooks and cloud architecture documentation (`techvision-documents`)

Exposure of incident response playbooks and internal architecture documentation provides an attacker with a complete understanding of TechVision's defensive capabilities, escalation procedures, and detection thresholds before conducting any active intrusion.

> **Immediate action:** Enable S3 Block Public Access on all three buckets. Audit IAM bucket policies for overly permissive `s3:GetObject` grants.

**4 — Exposed Employee Credentials in Public Repositories**
Developer handles (`s.chen`, `m.park`, `e.watson`) and API key fragments were identified in public GitHub repositories (`techvision-scanner`, `incident-response-playbooks`). Combined with the exposed email naming pattern (`firstname.lastname@techvision.com`), this provides both the username and a credential starting point for targeted account compromise attempts.

> **Immediate action:** Run automated secret scanning across all public repositories (GitHub's built-in secret scanning or `truffleHog`). Immediately revoke any identified credentials. Force MFA enrollment for all identified developer accounts.

---

![Vulnerability assessment table showing four critical vulnerabilities — internet-exposed databases, public dev/staging environments, misconfigured S3 buckets, and leaked credentials — with severity ratings, impact descriptions, and remediation recommendations](images/part3_vulnerability_table.png)

*Figure 6 — Critical vulnerability summary from infrastructure analysis. All four critical vulnerabilities are exploitable today by an external attacker using only publicly available information — no zero-day exploits or insider access required.*

---

**High Severity:**

| Vulnerability | Impact | Recommendation |
|---|---|---|
| Vulnerable API endpoints (`api.techvision.com`) | Authentication bypass, rate-limit evasion, data leakage | API security review, rate limiting, strict authentication enforcement |
| Outdated frontend dependencies (React.js, Node.js packages) | Known CVEs in third-party libraries — potential remote code execution | Automated dependency scanning (Dependabot / Snyk), regular patching schedule |
| Social media impersonation accounts | Spearphishing and credential harvesting against employees and clients | Platform takedown requests, continuous brand monitoring |
| Spearphishing and social engineering exposure | Email pattern + executive profile exposure enables targeted attacks | Email filtering, mandatory MFA, security awareness training |

**Medium Severity:**

| Vulnerability | Impact |
|---|---|
| Internal DNS naming conventions exposed (`db-primary.internal`, `cache.internal`) | Network topology revealed to external actors — aids lateral movement planning |
| GitHub repositories exposing code comments and dependencies | Internal architecture patterns, third-party dependency vulnerabilities visible to attackers |
| Third-party SaaS integration risk (Zendesk, Medium, Google Workspace) | Supply chain attack surface across three external platforms |
| Unencrypted legacy microservice communications | Sensitive data potentially transmitting over unencrypted protocols |

---

## Part 4 — Metadata and Document Analysis

### 4.1 — Exposed Documents Identified

| Document | Source | Information Revealed |
|---|---|---|
| Incident Response Playbooks (`incident-response-playbooks`) | Public GitHub repository | Internal IR workflows, escalation contacts, automation scripts, internal network naming conventions |
| Cloud Security Framework Whitepapers | Main website / S3 (`techvision-documents`) | Internal cloud architecture diagrams, system requirements, security configurations |
| System Backup and Audit Logs | Public S3 (`techvision-backups`, `techvision-logs`) | Server transaction logs, configuration settings, database backup structures |

### 4.2 — Metadata Findings

**Document metadata extracted via ExifTool analysis revealed:**

| Metadata Field | Exposed Information | Risk |
|---|---|---|
| **Author Names** | Individual employee usernames: `s.chen`, `m.park`, `e.watson` | Confirms executive-to-handle mapping — enables targeted spearphishing |
| **Creation Dates** | File timestamps spanning early 2024 through November 2025 | Reveals active document update cycles and organisational publishing cadence |
| **Software Versions** | Microsoft Office 2021, Adobe Acrobat Pro DC, specific PDF generators | Enables targeted client-side exploit selection based on confirmed installed software versions |
| **Internal File Paths** | `C:\Users\m.park\Documents\TechVision\Security\Playbooks\` and `/home/schen/projects/cloud-sec/` | Exposes local file system structure and operating system (Windows for m.park, Linux for s.chen) |

---

![ExifTool metadata extraction output showing document author usernames s.chen, m.park, e.watson, creation dates spanning 2024-2025, software versions Microsoft Office 2021 and Adobe Acrobat Pro DC, and internal file paths from Windows and Linux workstations](images/part4_metadata.png)

*Figure 7 — Metadata extracted from publicly accessible TechVision documents. The combination of employee usernames, internal file system paths, and installed software versions provides an attacker with reconnaissance data that would normally require active intrusion to obtain. All of this was accessible passively from public documents.*

---

**Intelligence Value of Metadata Exposure:**

The internal file path `C:\Users\m.park\Documents\TechVision\Security\Playbooks\` reveals three intelligence facts about Senior Security Engineer Michael Park simultaneously:
1. He uses a Windows workstation (not Linux or macOS)
2. His Windows username is `m.park` — matching the organisational email pattern `m.park@techvision.com`
3. Security playbooks are stored locally in unencrypted documents on his workstation, not solely in a secured repository

A threat actor targeting Michael Park specifically could use this metadata to construct a highly credible spearphishing lure referencing these specific file paths — dramatically increasing the likelihood of a successful social engineering attack.

---

## Part 5 — Threat Assessment

### 5.1 — Positive Threat Indicators (Legitimacy Signals)

| Indicator | Evidence |
|---|---|
| ✅ Verified brand presence | Active, verified social media accounts across LinkedIn, Twitter/X with consistent engagement |
| ✅ Industry recognition and compliance | ISO 27001 certified, SOC 2 Type II compliant, Gartner Magic Quadrant featured |
| ✅ Public thought leadership | Executives present at RSA and Black Hat; whitepapers published on cloud security and zero-trust |
| ✅ Standard email authentication | SPF, DKIM, and DMARC with quarantine policy active on `techvision.com` |
| ✅ Established domain history | Nine-year domain registration (2015–2026), transparent WHOIS registrant |
| ✅ Identifiable leadership | Verifiable executive team with traceable LinkedIn career histories |

### 5.2 — Negative Threat Indicators (Risk Signals)

| Indicator | Evidence |
|---|---|
| 🔴 Critical infrastructure exposure | Databases `203.45.67.200/201` and dev/staging environments directly internet-accessible |
| 🔴 Exposed cloud storage | Three S3 buckets with public read access containing backups, logs, and internal documents |
| 🔴 Subnet co-location with impersonation infrastructure | Privacy-protected domains (`techvision-secure.com`, `techvision-update.com`) share the `203.45.67.x` corporate subnet |
| 🔴 Active brand impersonation campaign | `@TechVisionSecure`, `@TechVisionUpdate`, and unverified Facebook page actively distributing fake security patch lures |
| 🔴 Credential and metadata leakage | Developer handles, internal file paths, software versions exposed in public GitHub and document metadata |

### 5.3 — Overall Threat Assessment

**Threat Level: 🔴 HIGH**

TechVision Consulting is a legitimate, operating business. The High threat level does not reflect doubt about corporate legitimacy — it reflects the **severity of the security posture gaps** identified through passive reconnaissance, and the **immediate, active nature of the external impersonation campaign** currently targeting the company's employees and clients.

An external threat actor conducting the same passive reconnaissance as this investigation would exit with: live database server IPs, confirmed public S3 bucket names and contents, employee email patterns for 20+ staff, executive handles and workstation metadata, and fully configured impersonation email infrastructure sharing TechVision's own subnet. This is a complete attack package assembled entirely from open sources in a single investigation session.

---

## Part 6 — OSINT Methodology Reflection

### Most Valuable Intelligence Source

**DNS records and infrastructure data** produced the highest-value intelligence across the entire investigation. A single DNS query against `techvision.com` and its subdomains:
- Mapped the full 14-host IP inventory across the `203.45.67.x` subnet
- Revealed the public exposure of database servers `203.45.67.200/201`
- Identified the staging and development environment subdomains
- Exposed the impersonation domain co-location on the corporate subnet

DNS data is passive, non-intrusive, and freely available — yet it produced findings that most organisations would classify as internal security intelligence.

### Hardest Information to Find

**True registrant ownership of the suspicious domains** could not be determined through passive means. `techvision-secure.com`, `techvision-update.com`, and `techvision-support.net` all use WHOIS privacy protection services, preventing direct registrant attribution. Uncovering the true registrant would require formal legal process — a subpoena to the respective registrars (GoDaddy and NameCheap) via law enforcement or legal counsel.

**Internal API architecture** beyond the leaked GitHub repository content also remained opaque. The `api.techvision.com` endpoint was identified and confirmed public, but specific API route structure, authentication implementation, and backend logic were not recoverable from passive OSINT.

### Cross-Source Verification Method

No finding was included in this report based on a single source. The verification approach used **multi-source triangulation**:

- Developer usernames from GitHub commit logs (`s.chen`, `m.park`) were validated against LinkedIn profile handle formats and document metadata author fields — all three sources produced consistent results
- Active IP addresses from DNS A records were independently verified against WHOIS registrant data and SSL Certificate Transparency logs — confirming the same hosting provider and ASN across all sources
- Impersonation domain identification was confirmed by comparing registration dates, WHOIS privacy status, and DNS co-location with corporate infrastructure against the naming pattern of official TechVision domains

### Ethical Boundaries Maintained

All investigation activity was **strictly passive**. The following actions were explicitly not taken:

- No port scanning against any TechVision infrastructure
- No authentication attempts against exposed endpoints or database servers
- No access or download of content from the exposed S3 buckets (existence and public accessibility were confirmed; content was not accessed)
- No contact with TechVision employees or systems
- No credential testing using leaked developer handles

Handling of exposed personnel data (phone numbers, email addresses, usernames, internal file paths) followed responsible disclosure principles — documented for threat assessment purposes only, not redistributed or used for any purpose beyond this report.

### How This Investigation Supports Broader Threat Intelligence Operations

This OSINT assessment feeds directly into three defensive intelligence functions:

**Attack surface reduction:** The four critical vulnerabilities and four medium-severity findings provide the security team with a prioritised remediation backlog derived from an attacker's-eye view of the organisation — the most operationally accurate way to understand which gaps matter most.

**Proactive brand protection:** The identified impersonation domains and social media accounts represent active threat infrastructure. Blocking `techvision-secure.com` and `techvision-update.com` in corporate email gateways and endpoint DNS resolvers, and initiating platform takedowns for the impersonation social media accounts, directly neutralises an ongoing threat campaign.

**Threat hunting and kill chain defence:** The exposed IR playbooks provide the threat intelligence team with intelligence about their own detection and response procedures — which is exactly what an attacker would use to plan a campaign designed to avoid triggering known detection thresholds. Knowing this exposure has occurred should drive an immediate playbook review and update cycle.

---

## Recommendations

### Immediate Actions — 24 to 48 Hours

| Action | Target | Method |
|---|---|---|
| **Restrict database servers to private VPC** | `203.45.67.200`, `203.45.67.201` | Modify AWS RDS security group to remove public accessibility — restrict inbound to private VPC subnet only |
| **Block public access to dev and staging** | `dev.techvision.com`, `staging.techvision.com` | Remove public DNS resolution; place subdomains behind `vpn.techvision.com` |
| **Enable S3 Block Public Access** | `techvision-backups`, `techvision-logs`, `techvision-documents` | AWS Console → S3 → Block Public Access settings; audit bucket IAM policies |
| **Revoke exposed credentials** | Developer accounts (`s.chen`, `m.park`, others) | Rotate all credentials identified in GitHub repositories; enforce mandatory MFA |
| **Initiate impersonation domain takedowns** | `techvision-secure.com`, `techvision-update.com`, `@TechVisionSecure`, `@TechVisionUpdate` | File abuse reports with GoDaddy, NameCheap, Twitter/X trust and safety teams; engage legal counsel for formal registrar subpoena |

### Short-Term Actions — 1 to 4 Weeks

| Action | Method |
|---|---|
| **Automated secret scanning on GitHub** | Enable GitHub secret scanning on all public repositories; run `truffleHog` across commit history; sanitise internal paths and code comments from `techvision-scanner` and `incident-response-playbooks` |
| **Metadata sanitisation pipeline** | Deploy ExifTool or MAT2 as a pre-publication pipeline for all whitepapers and public documents — strip author names, software versions, and internal file paths before release |
| **Configure split-horizon DNS** | Prevent internal naming conventions (`db-primary.internal`, `cache.internal`, `queue.internal`) from resolving via public DNS — restrict internal DNS to VPN-connected clients only |
| **Social engineering awareness training** | Targeted training for executive and technical staff on spearphishing lures, fake "security patch" social media posts, and BEC scenarios impersonating leadership |

### Long-Term Actions — 1 to 3 Months

| Action | Method |
|---|---|
| **Network microsegmentation** | Formally separate production, non-production, database, and administrative network segments — each with independent firewall policies and no default inter-segment routing |
| **Continuous Brand and Threat Exposure Management (CTEM)** | Deploy automated brand protection tooling to monitor for new domain registrations matching TechVision naming patterns and detect spoofed social media handles within hours of creation |
| **SBOM and automated dependency management** | Integrate Software Bill of Materials tracking and automated CVE scanning (Dependabot, Snyk) into the CI/CD pipeline for React.js, Node.js, and Python package dependencies |
| **SaaS and supply chain risk governance** | Establish security review standards and access governance for Zendesk, Medium, and Google Workspace integrations — review what data each platform can access and what breach exposure they create |

### Further Investigation Required

| Area | Action |
|---|---|
| **S3 access log forensic review** | Analyse AWS S3 server access logs for `techvision-backups`, `techvision-logs`, and `techvision-documents` to determine whether unauthorized read access occurred while buckets were publicly exposed |
| **Subnet attribution analysis** | Determine why impersonation domains (`techvision-secure.com`, `techvision-update.com`) resolve to the corporate `203.45.67.x` subnet — investigate whether this represents a compromised DNS record, a rogue internal admin action, or deliberate threat actor infrastructure placement |
| **API penetration testing** | Commission comprehensive authenticated and unauthenticated penetration testing against `api.techvision.com` to identify rate limiting failures, authentication bypasses, and logic flaws not discoverable through passive OSINT |

---

## Key Findings Summary

### Most Significant Finding

**Public database server exposure combined with impersonation infrastructure co-location on the corporate subnet.**

The public internet accessibility of live PostgreSQL servers (`203.45.67.200/201`) represents the most immediately exploitable vulnerability. An attacker does not need to bypass a firewall, phish an employee, or exploit an application — they can attempt direct database authentication using credentials already recoverable from public GitHub repositories. Combined with the physical co-location of impersonation infrastructure on TechVision's own `203.45.67.x` subnet, this creates both an immediate exfiltration pathway and an active social engineering campaign targeting the same organisation simultaneously.

### Infrastructure Summary

- Monolithic subnet: all corporate assets, regional domains, and impersonation domains share `203.45.67.x`
- Primary cloud: AWS EC2/RDS/CloudFront (us-west-2, ASN 16509)
- SaaS footprint: Google Workspace (email), Zendesk (support), Medium (blog)
- Two live database servers publicly accessible, three S3 buckets with public read access

### Personnel Summary

- Executive team fully identifiable through LinkedIn with verifiable career backgrounds
- 20+ employee email addresses following a recoverable naming pattern
- Developer handles leaked through GitHub commits and document metadata
- Internal file system paths and installed software versions exposed through document metadata

### Threat Summary

- Three suspicious privacy-protected domains with active mail infrastructure
- Two active Twitter/X impersonation accounts distributing fake security lures
- One unverified Facebook page conducting social engineering against employees and clients
- All impersonation infrastructure sharing the corporate `203.45.67.x` subnet

---

## Source Documentation

| Source Type | Files / References | Information Type |
|---|---|---|
| Domain Intelligence | `techvision_domains.txt` | Domain registrations, WHOIS data, subdomains |
| DNS Records | `techvision_dns_records.csv` | A, MX, NS, TXT, CNAME records |
| Infrastructure Inventory | `techvision_infrastructure.csv` | IP geolocation, hosting providers, ASN, SSL certificates |
| Personnel Directory | `techvision_personnel.csv` | Employee profiles, job titles, departments |
| Social Media Intelligence | `techvision_social_media.txt` | Official and impersonation accounts, activity patterns |
| Vulnerability Assessment | `techvision_vulnerabilities.txt` | Infrastructure exposures, S3 buckets, credential leakage |

**Collection Methods:**
- Passive domain and DNS enumeration (WHOIS, CT logs, DNS record parsing)
- Social Media Intelligence (SOCMINT) — LinkedIn, Twitter/X, Facebook, GitHub
- Metadata and document analysis (ExifTool-based extraction from public documents)
- Infrastructure mapping and vulnerability correlation against public CVE databases

**Chain of Custody:**
All raw datasets were acquired via non-intrusive passive intelligence collection. Raw files (`.csv`, `.txt`) were maintained in read-only format. Extracted intelligence was processed into structured findings without modifying source files or executing active scans. Findings were cross-referenced across multiple dataset sources to ensure full reproducibility.

---

> **Disclaimer:** This investigation was conducted as part of a structured Open Source Intelligence training exercise. TechVision Consulting is a fictional target organisation created for educational analysis. All domain names, IP addresses, employee names, and vulnerabilities described in this report are simulated. All reconnaissance was strictly passive. No real organisations, live infrastructure, or individuals were targeted. This document is published as a portfolio artifact demonstrating applied OSINT methodology.