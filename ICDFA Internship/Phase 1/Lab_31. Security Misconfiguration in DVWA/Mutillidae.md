# Security Misconfiguration in DVWA/Mutillidae

## 1. Engagement Overview

A security misconfiguration assessment was conducted against DVWA accessed at
`http://127.0.0.1:42001/` and `http://192.168.92.3/dvwa/` on a local Kali Linux
environment. The assessment covered five misconfiguration categories: default
credential use, directory listing exposure, verbose error messages, missing HTTP
security headers, and insecure file upload configuration. Each misconfiguration
was confirmed through direct observation or active testing, with Burp Suite used
to inspect HTTP response headers and intercept the file upload request.

---

## 2. Objectives

- Confirm that DVWA's default credentials grant authenticated admin access
  without any prior knowledge of the application
- Verify whether directory listing is enabled on accessible paths and assess
  what information is exposed as a result
- Trigger a verbose error message to determine what backend information the
  application discloses on failure
- Inspect HTTP response headers to identify missing security headers and
  assess the attack surface each absence creates
- Bypass the medium-level file upload restriction using Burp Suite to confirm
  the application trusts client-supplied MIME type metadata
- Document specific remediation steps for each confirmed misconfiguration

---

## 3. Scope

**In-Scope:**
- DVWA at `http://127.0.0.1:42001/` and `http://192.168.92.3/dvwa/`
- DVWA login page, vulnerabilities directory, SQL injection module,
  and file upload module
- HTTP response headers inspected via Burp Suite
- File upload at `/dvwa/vulnerabilities/upload/`

**Out-of-Scope:**
- Active exploitation beyond confirming upload success and error disclosure
- Any systems outside the local lab network
- Mutillidae modules not referenced in the lab exercises

**Authorization Statement:**
> This assessment was conducted against DVWA and Mutillidae, intentionally
> vulnerable applications deployed in an isolated local lab environment for
> security training. No production systems or unauthorized infrastructure were
> accessed at any point. All activities were performed for educational purposes
> under authorized course instruction.

---

## 4. Methodology

### Phase 1 -- Default Credential Testing
The DVWA login page was accessed and the default credentials `admin` /
`password` were entered without any prior reconnaissance. The application
authenticated successfully and granted admin-level access.

### Phase 2 -- Directory Listing Verification
The URL `http://192.168.92.3/dvwa/vulnerabilities/` was navigated to directly.
The Apache web server returned a directory index page listing all subdirectories
and files present in that path, confirming that the `Options Indexes` directive
was active.

### Phase 3 -- Error Handling Analysis
A single quote was injected into the `id` parameter of the SQL injection module
at `http://192.168.92.3/dvwa/vulnerabilities/sqli/?id='&Submit=Submit#`. The
application returned an unhandled MySQL exception directly in the browser
response, exposing the backend database type and query structure.

### Phase 4 -- Security Header Inspection
An HTTP response from DVWA was captured in Burp Suite. The full response
header set was reviewed to identify which standard security headers were present
and which were absent.

### Phase 5 -- File Upload Misconfiguration
A PHP web shell (`backdoor.php`) was submitted to the DVWA file upload form
at medium security level. The direct upload was rejected. Burp Suite was used
to intercept the request and the `Content-Type` header was changed from
`application/x-php` to `image/jpeg`. The modified request was forwarded and
the server confirmed a successful upload.

---

## 5. Vulnerability Summary

| ID | Vulnerability | Severity | Affected Endpoint |
|----|--------------|----------|-------------------|
| 01 | Default Credentials Accepted | High | /dvwa/login.php |
| 02 | Directory Listing Enabled | Medium | /dvwa/vulnerabilities/ |
| 03 | Verbose SQL Error Message Disclosure | High | /dvwa/vulnerabilities/sqli/ |
| 04 | Missing HTTP Security Headers | Medium | All HTTP Responses (Global) |
| 05 | MIME Type Bypass on File Upload | Critical | /dvwa/vulnerabilities/upload/ |

---

## 6. Detailed Findings

---

### Finding 01 -- Default Credentials Accepted

#### Severity
High

#### Affected Endpoint
`/dvwa/login.php` -- POST, username and password fields

#### Description
DVWA ships with the default credentials `admin` / `password`. The application
accepted these credentials without any account lockout, CAPTCHA, or
multi-factor prompt. Successful login granted full administrative access to all
DVWA modules. Default credentials are publicly documented and are the first
thing an attacker tests against any application. When they remain unchanged
in a deployed environment, they represent zero-effort unauthorized access.

#### Proof of Concept

DVWA accessed and login attempted with `admin` / `password`:



![DVWA Login - Default Credentials admin/password Accepted](image.jpg)



Successful login confirmed, admin dashboard accessible.

#### Impact
Admin access to DVWA exposes all vulnerability modules and application
settings. In a real deployment, default credentials on an admin account give
an attacker immediate access to the highest-privilege functions in the application
without needing to exploit any technical vulnerability. The consequences include
unauthorized data access, configuration changes, and a platform for further
exploitation of backend systems.

#### Remediation
- Change all default credentials immediately after deployment, before the
  application is accessible on any network
- Implement an account lockout policy after a defined number of failed login
  attempts (e.g., lock after 5 failures, require manual unlock or time-delay)
- Enforce a minimum password policy that prevents the use of the application
  name, common words, or any credential found in public default credential lists
- Add multi-factor authentication to administrative accounts

---

### Finding 02 -- Directory Listing Enabled

#### Severity
Medium

#### Affected Endpoint
`http://192.168.92.3/dvwa/vulnerabilities/` -- Apache directory index

#### Description
Navigating directly to the `/dvwa/vulnerabilities/` path returned an Apache
directory index page rather than a 403 Forbidden response or a redirect.
The listing displayed all subdirectory names and PHP files present in that
path including `brute/`, `captcha/`, `csrf/`, `exec/`, `fi/`, `sqli/`,
`sqli_blind/`, `upload/`, `view_help.php`, `view_source.php`,
`view_source_all.php`, `xss_r/`, and `xss_s/`. Each directory name
corresponds to a vulnerability module and each PHP file was accessible by
clicking its link.

#### Proof of Concept

Browser navigating to `/dvwa/vulnerabilities/` displaying the Apache
directory index:



![Apache Directory Index - /dvwa/vulnerabilities/ Listing All Subdirectories](image.jpg)



Directories and files visible in the listing:
```

brute/         captcha/       csrf/          exec/
fi/            sqli/          sqli_blind/    upload/
view_help.php  view_source.php  view_source_all.php
xss_r/         xss_s/
```

#### Impact
Directory listing hands an attacker a complete map of the application's
vulnerability modules and source files without needing to brute-force or
enumerate paths. The presence of `view_source.php` and `view_source_all.php`
is particularly notable, as these files may expose application source code
directly. The `upload/` directory listing confirms the existence and path of the
upload functionality, which directly supports the attack in Finding 05.

#### Remediation
- Disable directory listing in the Apache configuration by removing
  `Options Indexes` or explicitly setting `Options -Indexes` in the relevant
  `<Directory>` block or `.htaccess` file
- Return a 403 or redirect to the application root for any directory request
  that does not have a default index file
- Review all directories under the web root for the presence of files that
  should not be directly accessible (source viewers, configuration files,
  backup files) and remove or relocate them

---

### Finding 03 -- Verbose SQL Error Message Disclosure

#### Severity
High

#### Affected Endpoint
`/dvwa/vulnerabilities/sqli/?id='&Submit=Submit#` -- GET, `id` parameter

#### Description
Injecting a single quote into the `id` parameter caused the application to throw
an unhandled MySQL exception that was returned in the browser response without
any error handling wrapper. The error message confirmed the backend database
type as MySQL, exposed the structure of the failing query, and referenced the
internal file path of the MySQL handler. This is the same category of information
disclosure observed in the SQL injection lab, but here it arises purely from
inadequate error handling configuration rather than a deliberate injection attack.

#### Proof of Concept

URL with single quote injected into the id parameter:
```

http://192.168.92.3/dvwa/vulnerabilities/sqli/?id='&Submit=Submit#
```

Application response visible in the browser:



![Verbose MySQL Error Message in Browser Response](image.jpg)



Error message content:
```
You have an error in your SQL syntax; check the manual that corresponds to
your MySQL server version for the right syntax to use near ''''' at line 1
```

#### Impact
A verbose error message that confirms the database type reduces the attacker's
reconnaissance effort significantly. Knowing the target runs MySQL allows an
attacker to use MySQL-specific payloads immediately rather than testing syntax
variations for multiple database types. The partial query structure visible in the
error also reveals how the application constructs its SQL, which informs more
precise injection payloads.

#### Remediation
- Set `display_errors = Off` in `php.ini` to prevent any PHP or database
  error from being rendered in the browser response
- Replace all unhandled exceptions with a generic user-facing message such
  as "An error occurred. Please try again."
- Log full error details including query structure and stack trace to a
  server-side log file accessible only to administrators
- Use a centralized error handler in the application code to catch all
  exceptions before they reach the output layer

---

### Finding 04 -- Missing HTTP Security Headers

#### Severity
Medium

#### Affected Endpoint
All HTTP responses (global)

#### Description
HTTP response headers captured in Burp Suite showed that the DVWA application
does not include any standard browser security headers. The full header set
returned was limited to server identification, date, content type, connection,
cache control, expiry, and content length. None of the headers that instruct
browsers to enforce security policies were present.

#### Proof of Concept

HTTP response headers captured in Burp Suite:



![Burp Suite Response - No Security Headers Present](image.jpg)



Headers present in the response:
```

HTTP/1.1 200 OK
Server: nginx/1.28.0
Date: Tue, 21 Apr 2026 09:40:57 GMT
Content-Type: text/html; charset=utf-8
Connection: keep-alive
Pragma: no-cache
Cache-Control: no-cache, must-revalidate
Expires: Tue, 23 Jun 2009 12:00:00 GMT
Content-Length: 6093
```

Headers absent from every response:

| Missing Header | Attack Mitigated |
|---|---|
| Content-Security-Policy | Cross-Site Scripting (XSS) |
| X-Frame-Options | Clickjacking |
| X-Content-Type-Options | MIME type sniffing |
| Strict-Transport-Security | SSL stripping, protocol downgrade |
| Referrer-Policy | Referrer information leakage |

#### Impact
The absence of these headers leaves browser-level security controls
unactivated. Without `Content-Security-Policy`, inline scripts injected via XSS
execute freely. Without `X-Frame-Options`, the application can be embedded
in an attacker-controlled iframe for clickjacking attacks. Without
`X-Content-Type-Options`, browsers may execute uploaded files based on
content sniffing rather than the declared MIME type, which compounds the
risk from Finding 05.

#### Remediation
Add the following headers to every HTTP response in the web server
configuration:
```

Content-Security-Policy: default-src 'self'; script-src 'self'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Strict-Transport-Security: max-age=31536000; includeSubDomains
Referrer-Policy: no-referrer
```
In nginx, these can be set globally in the server block. In Apache, they can
be set in `httpd.conf` or `.htaccess` using the `Header always set` directive.

---

### Finding 05 -- MIME Type Bypass on File Upload (Medium Security)

#### Severity
Critical

#### Affected Endpoint
`/dvwa/vulnerabilities/upload/` -- File upload form, POST

#### Description
At medium security level, the DVWA file upload form restricts uploads to JPEG
and PNG image files. A direct upload of `backdoor.php` was rejected with the
message "Your image was not uploaded. We can only accept JPEG or PNG images."
The restriction was implemented by checking the `Content-Type` header in the
multipart form data. This header is part of the HTTP request and is fully
controlled by the submitting client. Using Burp Suite to intercept the upload
request and change `Content-Type` from `application/x-php` to `image/jpeg`
caused the server to accept the PHP file and store it in the upload directory.
The file extension remained `.php` throughout and the server confirmed the
upload was successful.

#### Proof of Concept

Burp Suite intercept showing `Content-Type` changed from `application/x-php`
to `image/jpeg` while the filename remained `backdoor.php`:



![Burp Suite Intercept - Content-Type Modified to image/jpeg for backdoor.php](image.jpg)



Server confirmation after forwarding the modified request:



![Upload Confirmation - backdoor.php Successfully Uploaded via MIME Bypass](image.jpg)



Server response:
```

../../hackable/uploads/backdoor.php succesfully uploaded!
```

#### Impact
Trusting the `Content-Type` header as a file type validation mechanism is
equivalent to trusting user input. Any attacker with a proxy or HTTP client
can set this header to any value regardless of the actual file content. A
PHP shell uploaded via this bypass is stored under a `.php` extension in a
web-accessible directory, making it directly executable via HTTP GET request.
This converts a misconfiguration into remote code execution on the web server.

#### Remediation
- Replace `Content-Type` header checking with server-side magic byte
  inspection using PHP's `finfo_file()` function, which reads the actual
  file content rather than client-supplied metadata:
```php
$finfo = finfo_open(FILEINFO_MIME_TYPE);
$mime = finfo_file($finfo, $_FILES['uploaded']['tmp_name']);
if (!in_array($mime, ['image/jpeg', 'image/png'])) {
    die("Invalid file type.");
}
```

- Rename all uploaded files server-side to a randomly generated string with
  a forced safe extension, removing the original filename and extension from
  the stored path entirely
- Disable PHP execution in the upload directory via `.htaccess`:
```
php_flag engine off
Options -ExecCGI
```
- Store uploaded files outside the web root where possible to prevent
  direct HTTP access regardless of execution configuration

---

## 7. Attack Chain

```
[Step 1] Default Credentials
admin / password accepted on first attempt
Full admin access granted with no rate limiting or MFA
        |
        v
[Step 2] Directory Listing
/dvwa/vulnerabilities/ returns Apache index
Upload path, source files, and all module directories disclosed
        |
        v
[Step 3] Error Disclosure
Single quote in id parameter triggers unhandled MySQL exception
Database type (MySQL) and query structure returned in browser
Informs SQL injection payload selection
        |
        v
[Step 4] Missing Security Headers
No CSP, X-Frame-Options, X-Content-Type-Options, or HSTS
Browser security controls inactive across all pages
XSS payloads execute without restriction (confirmed in XSS lab)
        |
        v
[Step 5] File Upload Bypass
backdoor.php rejected at medium security level
Burp Suite intercept: Content-Type changed to image/jpeg
Server accepts PHP file, stores at ../../hackable/uploads/backdoor.php
Remote code execution possible via direct HTTP access to uploaded file
```

Each misconfiguration in the chain feeds the next. Directory listing confirms
the upload path. Error disclosure confirms the database type for injection
attacks. Missing security headers allow XSS to execute freely. The file upload
bypass completes the chain by providing persistent remote code execution.

---

## 8. Tools Used

- Kali Linux (lab environment)
- Firefox (DVWA access and directory listing observation)
- Burp Suite (HTTP response header inspection and upload request interception)
- DVWA (intentionally vulnerable target application)

---

## 9. Challenges Encountered

- **Directory listing was observed on the DVWA vulnerabilities path specifically:**
  The listing was not present on all DVWA paths. The Apache `Options Indexes`
  directive appeared to apply to the `/dvwa/vulnerabilities/` directory but the
  root DVWA path returned the application home page normally. This is consistent
  with a partial or directory-specific configuration rather than a server-wide
  setting.
- **Error message required a specific injection point:** Accessing a genuinely
  non-existent page returned a standard 404. The verbose MySQL error only
  appeared when injecting into an existing SQL-driven parameter. The single quote
  in the `id` parameter was the most direct way to trigger an unhandled database
  exception without crafting a full injection payload.

---

## 10. Key Takeaways

- **Misconfigurations compound each other:** None of the five findings in this
  lab required a sophisticated technique. Default credentials, enabled directory
  listing, verbose errors, absent headers, and trusted client headers are all
  configuration choices. Each one individually reduces the work an attacker
  needs to do. Together they create a path from zero knowledge to remote code
  execution without exploiting a single code-level vulnerability.
- **Client-controlled data cannot be a security boundary:** The MIME type
  bypass is the clearest example of this. The `Content-Type` header is submitted
  by the client and can be set to any string. Basing a security decision on it
  is no different from asking the attacker whether they are uploading a safe file
  and trusting their answer.
- **Security headers cost nothing to add and remove several attack categories:**
  Adding `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`,
  and `Strict-Transport-Security` to a server configuration takes minutes. Their
  absence in DVWA's response headers means the browser offers no resistance
  to XSS execution, clickjacking, or MIME sniffing attacks that those headers
  would otherwise block at no application code cost.
- **Directory listing gives attackers a free site map:** An attacker who cannot
  enumerate paths because they lack a wordlist match can still map the full
  application structure if directory listing is on. Disabling it is a one-line
  configuration change (`Options -Indexes`) with no functional impact on the
  application.
- **Verbose errors are reconnaissance tools:** The MySQL error returned by
  the single quote injection told the attacker the database type, the syntax of
  the failing query, and an internal file path. That same information takes
  significantly more effort to obtain through blind injection techniques. Turning
  off error display removes that shortcut entirely.

---

## 11. Disclaimer

> This assessment was conducted exclusively against DVWA and Mutillidae,
> intentionally vulnerable web applications deployed in an isolated local lab
> environment at `http://127.0.0.1:42001/` and `http://192.168.92.3/dvwa/`.
> No production systems, live databases, or unauthorized infrastructure were
> accessed at any point. All techniques described in this report were performed
> for educational purposes under authorized course instruction. These techniques
> must only be applied against systems for which explicit written authorization
> has been obtained.