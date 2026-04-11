# IoT & Enterprise Network Forensic Investigation Using Wireshark

## 1. Engagement Overview

A forensic investigation was conducted on a provided PCAP capture file related to a
confirmed security incident affecting BookWorldStore.com, an e-commerce web
application hosted at IP address `73.124.22.98`. The investigation covered network
traffic recorded on **March 15, 2024, between 12:07 and 12:26 UTC**. The testing
approach was black-box forensic analysis, working solely from captured network
traffic without prior knowledge of the application internals. Tools used included
Wireshark for packet analysis and DNS statistics review.

---

## 2. Objectives

- Identify the origin, nature, and sequence of the attack captured in the PCAP file
- Detect and document all suspicious IP addresses, User-Agent strings, and endpoints
involved in the incident
- Determine which vulnerabilities were exploited and assess their exploitability
- Evaluate the risk to sensitive customer data stored within the backend database
- Identify all Indicators of Compromise (IOCs) for use in incident response
- Provide actionable remediation recommendations to prevent recurrence

---

## 3. Scope

**In-Scope:**
- Network traffic captured in the provided PCAP file
- Web server at `73.124.22.98` (bookworldstore.com)
- All HTTP endpoints observed communicating with the server
- External IP addresses: `111.224.250.131` and `170.40.150.126`

**Out-of-Scope:**
- Live system access or active exploitation
- Internal network infrastructure beyond what is visible in the PCAP
- Physical systems or social engineering vectors

**Authorization Statement:**
> This forensic investigation was conducted on a pre-captured PCAP file provided
> for educational and training purposes. No live systems were accessed or exploited.
> All analysis was performed in a controlled academic environment under authorized
> course instruction.

---

## 4. Methodology

### Phase 1: Reconnaissance (Traffic Behavior Analysis)
Wireshark was opened with the provided PCAP file. A high-level review of all
captured packets was performed to identify traffic patterns, unique IP addresses, and
protocol distribution. Endpoint statistics and DNS statistics were reviewed to
establish baseline behavior and identify anomalies.

### Phase 2: Mapping / Spidering (Attack Surface Identification)
HTTP traffic was filtered and analyzed to enumerate all endpoints accessed during
the capture window. Suspicious GET and POST requests were identified by examining
request URIs, HTTP methods, and User-Agent strings. The site map of accessed
endpoints was reconstructed manually from packet data.

### Phase 3: Vulnerability Identification
HTTP POST requests were inspected for brute force patterns against
`/admin/login.php`. GET requests to `/search.php` were analyzed for SQL injection
signatures including `UNION ALL SELECT NULL` patterns and `sqlmap` User-Agent
headers. File upload activity at `/admin/index.php` was identified and the uploaded
filename extracted from packet data.

### Phase 4: Exploitation Analysis
The attack chain was reconstructed step by step using packet timestamps and
content. Authentication bypass via brute force, database enumeration via SQLi, and
web shell installation via unrestricted file upload were confirmed. The reverse shell
trigger was identified via a GET request to `/admin/uploads/NVri2vhp.php`.

### Phase 5: Validation
DNS statistics were reviewed to confirm the absence of DNS-based C2 activity.
Data exfiltration channels were examined, no interactive session or outbound data
transfer was observed in the PCAP beyond the shell trigger.

### Phase 6: Documentation
All findings including IOCs, attacker IPs, payloads, endpoints, and timestamps were
recorded and compiled into this report.

---

## 5. Vulnerability Summary

| ID | Vulnerability | Severity | Affected Endpoint |
|----|--------------|----------|-------------------|
| 01 | Brute Force Authentication | High | /admin/login.php |
| 02 | SQL Injection | High | /search.php |
| 03 | Unrestricted File Upload | Critical | /admin/index.php |
| 04 | Web Shell Execution | Critical | /admin/uploads/NVri2vhp.php |
| 05 | Information Disclosure (Server Headers) | Medium | All HTTP Responses |

---

## 6. Detailed Findings

---

### Finding 01: Brute Force Authentication Attack

#### Severity
High

#### Affected Endpoint
`/admin/login.php` (HTTP POST)

#### Description
The attacker (IP `111.224.250.131`) submitted repeated POST requests to the admin
login endpoint with varying username and password combinations. The application
lacked rate limiting, account lockout, or CAPTCHA mechanisms, allowing the attacker
to cycle through credentials until a valid combination was accepted. The final
successful credential was `admin:admin123!`, which returned an HTTP 302 redirect
to the admin dashboard.

#### Proof of Concept

Wireshark capture showing repeated POST requests from `111.224.250.131` to
`/admin/login.php` with form-encoded credentials:

![Repeated POST Requests to /admin/login.php](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/07_01_Repeated%20POST%20Requests%20to%20admin-login.jpg)

Successful authentication response captured in packet data:
```
username=admin&password=admin123%21
HTTP/1.1 302 Found
Date: Fri, 15 Mar 2024 12:17:34 GMT
Server: Apache/2.4.52 (Ubuntu)
```

![Authentication Response](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/07_06_Authentication%20Response.jpg)

#### Impact
Authentication bypass granted the attacker full administrative access to the
BookWorldStore dashboard. This directly enabled the subsequent file upload attack.
Weak default credentials combined with no brute force protection made this trivially
exploitable.

#### Remediation
- Implement account lockout after 5 failed login attempts
- Enforce strong password policies and prohibit default or common credentials
- Add CAPTCHA or multi-factor authentication to the admin login
- Implement IP-based rate limiting on authentication endpoints
- Alert on repeated failed login attempts via security monitoring

---

### Finding 02: SQL Injection

#### Severity
High

#### Affected Endpoint
`/search.php` (HTTP GET), `search` parameter

#### Description
The search functionality on BookWorldStore concatenates user-supplied input
directly into SQL queries without parameterization. The attacker leveraged this
using `sqlmap v1.8.3` to enumerate the backend database structure. The presence
of `UNION ALL SELECT NULL` patterns in captured GET requests confirms active
database structure mapping. The backend database was identified as MySQL. While
500 Internal Server Errors were observed during enumeration attempts, the database
remained accessible to further probing.

#### Proof of Concept

Wireshark capture showing sqlmap User-Agent and UNION SELECT NULL payload
in the GET request to `/search.php`:

![SQL Injection GET Request with sqlmap User-Agent](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/07_03_SQL%20Injection%20GET%20Request%20with%20sqlmap%20User-Agent.jpg)

Payload observed in packet:
```
GET /search.php?search=book%27%20UNION%20ALL%20$SELECT%20NULL%
2CCONCAT%280x7178766271%2CJSON_ARRAY...
User-Agent: sqlmap/1.8.3#stable (https://sqlmap.org)
Host: bookworldstore.com
```

#### Impact
SQL injection on an e-commerce platform with direct database access places all
stored customer data at immediate risk, including names, email addresses, physical
addresses, payment details, and order history. Successful enumeration of table and
column structure is the precursor to full data extraction. Even without confirmed
exfiltration in this PCAP, the database structure was actively being mapped.

#### Remediation
- Replace all raw query construction with prepared parameterized statements:
```php
$stmt = $pdo->prepare("SELECT * FROM books WHERE title = ?");
$stmt->execute([$search]);
```
- Validate and whitelist all search input and reject or sanitize special SQL characters
- Implement a Web Application Firewall (e.g., ModSecurity) to block SQLi patterns
- Apply least-privilege database accounts, the application user should have no
  schema-level permissions
- Suppress verbose database error messages in production; log them server-side only

---

### Finding 03: Unrestricted File Upload

#### Severity
Critical

#### Affected Endpoint
`/admin/index.php` (Authenticated file upload form)

#### Description
After gaining authenticated access via brute force, the attacker used the admin
dashboard file upload functionality to upload a malicious PHP web shell named
`NVri2vhp.php`. The application performed no file type validation, extension
filtering, or content inspection on uploaded files. The file was stored directly in
`/admin/uploads/` and was immediately accessible via HTTP GET request.

#### Proof of Concept

Wireshark capture confirming successful upload via POST to `/admin/index.php`
and server confirmation response:

![Malicious File Upload POST Request and Confirmation](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/07_02_Malicious%20File%20Upload%20POST%20Request%20and%20Confirmation.jpg)

Server response confirming upload success:
```
The file NVri2vhp.php has been uploaded.
```

Attacker subsequently triggered the shell via:
```
GET /admin/uploads/NVri2vhp.php HTTP/1.1
```

#### Impact
Successful upload and execution of a PHP web shell grants the attacker persistent
remote code execution on the web server. The reverse shell payload embedded in
`NVri2vhp.php` was configured to establish a raw TCP connection on port 443,
creating a persistent backdoor that survives server restarts. Even without confirmed
active interaction in this PCAP, the shell remained accessible for future exploitation.

#### Remediation
- Restrict uploaded file types to a strict whitelist (e.g., images only: `.jpg`, `.png`,
  `.gif`) reject `.php`, `.phtml`, `.phar`, and all executable extensions
- Validate file content using MIME-type inspection, not just extension checking
- Store uploaded files outside the web root to prevent direct HTTP access
- Rename uploaded files server-side using randomly generated non-guessable names
- Disable PHP execution in upload directories via `.htaccess`:
  `php_flag engine off`
- Implement file size limits and malware scanning on uploads

---

### Finding 04: Web Shell Execution & Reverse Shell

#### Severity
Critical

#### Affected Endpoint
`/admin/uploads/NVri2vhp.php` (HTTP GET)

#### Description
Following the upload of the malicious PHP file, the attacker triggered its execution
by sending a direct GET request to its publicly accessible path. The PHP file
contained a reverse shell payload configured to connect back to the attacker on TCP
port 443. While no interactive session or command output was captured in the PCAP,
the execution of the shell was confirmed by the server returning an HTTP 500 error
during the trigger, indicating PHP code executed but the reverse connection could
not be established within the capture window.

#### Proof of Concept

Wireshark capture showing the GET request to the uploaded shell and the 500
Internal Server Error response during trigger:

![Web Shell GET Request and 500 Error Response](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/07_04_Web%20Shell%20GET%20Request%20and%20500%20Error%20Response.jpg)

Packets captured:
```
GET /admin/uploads/NVri2vhp.php HTTP/1.1
→ HTTP/1.1 500 Internal Server Error
```

#### Impact
A successfully triggered reverse shell provides the attacker with an interactive
command-line session on the web server, equivalent to direct SSH access. From this
position, the attacker can read and exfiltrate all database credentials, customer data,
and application source code; pivot to internal network systems; and maintain
persistent access indefinitely.

#### Remediation
- Remove `NVri2vhp.php` from the server immediately
- Audit all files in `/admin/uploads/` and all other upload directories for malicious
  content
- Implement file integrity monitoring (e.g., Tripwire) to detect unauthorized file
  creation
- Block outbound TCP connections from the web server to unauthorized external IPs
  via firewall rules
- Conduct a full system forensic review to confirm no secondary persistence
  mechanisms were installed

---

### Finding 05: Information Disclosure via HTTP Response Headers

#### Severity
Medium

#### Affected Endpoint
All HTTP Responses (Global)

#### Description
Every HTTP response from the web server included detailed technology version
information in response headers. This allowed the attacker to immediately identify
the exact server stack without active fingerprinting effort.

#### Proof of Concept

Response headers captured in Wireshark:
```
Server: Apache/2.4.52 (Ubuntu)
```

![Server Display](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/07_05_Response%20headers%20captured%20in%20Wireshark.jpg)

#### Impact
Exposure of Apache version information allows an attacker to query public CVE
databases for version-specific exploits, reducing the effort required to achieve
further compromise. Combined with the PHP version inferred from uploaded shell
behavior, the attacker has a complete picture of the server technology stack.

#### Remediation
- Suppress the `Server` header or genericize it in Apache configuration:
  `ServerTokens Prod`
  `ServerSignature Off`
- Remove or sanitize all technology-identifying headers across the application

---

## 7. Attack Chain

```
[12:07 UTC] Reconnaissance
IP 111.224.250.131 browses BookWorldStore using Firefox
→ Identifies /search.php and /admin/login.php
        ↓
[12:09 UTC] SQL Injection & Database Enumeration
sqlmap v1.8.3 targets /search.php
→ UNION ALL SELECT NULL patterns sent to map database structure
→ 500 errors returned but database probed successfully
        ↓
[12:17 UTC] Brute Force Authentication
Repeated POST to /admin/login.php with credential guessing
→ Successful login: admin:admin123!
→ HTTP 302 redirect to /admin/index.php (admin dashboard)
        ↓
[12:XX UTC] Unrestricted File Upload
Malicious PHP web shell (NVri2vhp.php) uploaded via admin dashboard
→ File stored at /admin/uploads/NVri2vhp.php
        ↓
[12:26 UTC] Web Shell Trigger
GET /admin/uploads/NVri2vhp.php
→ Reverse shell payload executes
→ TCP port 443 reverse connection attempted
→ No interactive session captured in PCAP
```

The attack demonstrates a complete multi-stage compromise: passive reconnaissance
leading to active SQLi enumeration, authentication bypass via brute force, and
persistent access established through an unrestricted file upload. Customer data was
at highest risk during the SQL injection phase, and full server control was achieved
upon shell execution.

---

## 8. Tools Used

- Wireshark (packet capture analysis and DNS statistics)
- Firefox (used by attacker, was observed in User-Agent headers)
- sqlmap v1.8.3 (used by attacker and was identified via User-Agent header)
- Gobuster (used by attacker, was identified via User-Agent header)
- Kali Linux (investigation environment)

---

## 9. Challenges Encountered

- **No DNS traffic present:** Wireshark DNS statistics returned zero packets across
  all categories, ruling out DNS-based C2 analysis entirely. The investigation was
  limited to HTTP-layer evidence only.
- **Incomplete exfiltration capture:** The PCAP captured the shell upload and
  trigger but did not include any subsequent interactive session. It was not possible
  to confirm whether data exfiltration occurred outside the capture window.
- **Obfuscated SQL payloads:** URL-encoded SQLi payloads required manual
  decoding to reconstruct the full injection strings and understand the database
  enumeration strategy.
- **Attacker identity masking:** The use of multiple User-Agent strings by
  `111.224.250.131` required cross-referencing packet content rather than relying
  on a single traffic signature for attribution.

---

## 10. Key Takeaways

- **User-Agent strings are reliable IOCs:** The presence of `sqlmap/1.8.3` in HTTP
  headers is an unambiguous indicator of automated injection tooling and should be
  blocked at the WAF layer before any payload reaches the application.
- **Brute force success reveals weak credential policy:** The attacker succeeded with
  `admin:admin123!`, a trivially guessable credential on a production e-commerce
  admin panel. Enforcing strong passwords and MFA would have broken this attack
  chain entirely at the second stage.
- **File upload is a critical trust boundary:** Allowing authenticated users to upload
  arbitrary files without content validation is effectively granting remote code
  execution. File type enforcement and web-root isolation are non-negotiable controls
  for any upload functionality.
- **Multi-stage attacks require layered defenses:** No single control would have
  prevented this compromise. The attacker pivoted across three distinct vulnerability
  classes: SQLi, brute force, and file upload, meaning each layer must be
  independently hardened.
- **PCAP analysis has temporal limits:** Forensic conclusions are bounded by the
  capture window. The absence of confirmed exfiltration in this PCAP does not rule
  out data theft outside the recorded period, a critical consideration when scoping
  incident response.

---

## 11. Indicators of Compromise (IOCs)

| Type | Value | Description |
|------|-------|-------------|
| Attacker IP | `111.224.250.131` | Primary attacker: brute force, SQLi, file upload |
| Scanner IP | `170.40.150.126` | Preliminary scanner: TCP RST packets observed |
| Suspicious User-Agent | `sqlmap/1.8.3#stable` | Automated SQL injection tool |
| Suspicious User-Agent | `gobuster/3.0` | Directory brute-forcing tool |
| Malicious File | `NVri2vhp.php` | PHP reverse shell uploaded to `/admin/uploads/` |
| Vulnerable Endpoint | `/admin/login.php` | Susceptible to brute force, no rate limiting |
| Vulnerable Endpoint | `/search.php` | Susceptible to SQL injection |
| Vulnerable Endpoint | `/admin/index.php` | Susceptible to unrestricted file upload |
| Compromised Directory | `/admin/uploads/` | Web-accessible upload storage, shell hosted here |

---

## 12. Disclaimer

> This forensic investigation was conducted exclusively on a pre-captured PCAP file
> (`WebInvestigation.pcap`) provided as part of an academic lab exercise for the
> course **Kali Linux Tools and System Security** at ICDFA. No live systems were
> accessed, attacked, or disrupted at any point during this investigation. All findings
> are presented for **educational purposes only**. Any techniques or indicators
> described in this report must only be applied in environments where explicit written
> authorization has been obtained.

---
