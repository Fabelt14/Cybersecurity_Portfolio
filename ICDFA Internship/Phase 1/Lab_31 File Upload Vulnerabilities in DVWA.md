# Security Portfolio: File Upload Vulnerabilities in DVWA


## 1. Engagement Overview

A file upload vulnerability assessment was conducted against DVWA hosted locally
at `http://192.168.92.3/dvwa/vulnerabilities/upload/`, accessed via Firefox on Kali
Linux. The assessment tested the upload functionality across all three security
levels: Low, Medium, and High. A PHP web shell was generated using Weevely
4.0.2 and upload attempts were made at each level. Burp Suite was used to
intercept and modify HTTP requests where client or server-side restrictions blocked
direct upload. Shell access was confirmed by connecting back to the uploaded file
via the Weevely terminal client and executing commands on the server.

---

## 2. Objectives

- Identify what file types and sizes the DVWA upload form accepts and whether
  those restrictions are enforced server-side
- Upload a Weevely-generated PHP web shell at low security with no restrictions
  in place
- Bypass the medium-level MIME type check by modifying the `Content-Type`
  header in Burp Suite
- Bypass the high-level magic byte validation by prepending a GIF file signature
  to the web shell and using a double extension filename
- Confirm remote code execution by accessing the uploaded shell via Weevely
  and running commands on the server
- Assess the effectiveness of each security level's upload controls and document
  specific remediation steps

---

## 3. Scope

**In-Scope:**
- DVWA File Upload module at `http://192.168.92.3/dvwa/vulnerabilities/upload/`
- Upload directory: `../../hackable/uploads/`
- Security levels tested: Low, Medium, High
- Shell access via `http://127.0.0.1:42001/hackable/uploads/backdoor.php`

**Out-of-Scope:**
- Other DVWA vulnerability modules not related to file upload
- Any systems outside the local lab network
- Destructive post-exploitation activity beyond confirming shell access

**Authorization Statement:**
> This assessment was conducted against DVWA, an intentionally vulnerable web
> application deployed in an isolated local lab environment. No production systems,
> real user accounts, or public-facing infrastructure were accessed at any point.
> All activities were performed for educational purposes under authorized course
> instruction.

---

## 4. Methodology

### Phase 1 -- Reconnaissance
The DVWA File Upload page was accessed and the upload form was reviewed.
A harmless text file (`file1.txt`) and a shell script (`dns_resolve.sh`) were uploaded
to confirm which file types the form accepted at low security level. An OpenVPN
configuration file (`.ovpn`) was also uploaded to test whether the form applied
any file type restriction. All uploads succeeded, confirming no file type validation
was present at low level. The server response confirmed the upload directory as
`../../hackable/uploads/`.

### Phase 2 -- Web Shell Generation
Weevely 4.0.2 was used to generate an obfuscated PHP web shell:
```

weevely generate 12345 backdoor.php
```
The generated file (`backdoor.php`, 693 bytes) was reviewed in nano. The content
was confirmed as obfuscated PHP, designed to evade signature-based detection
by WAFs and security scanners.

### Phase 3 -- Low Security Upload
With security level set to Low, `backdoor.php` was uploaded directly through the
browser without modification. The server accepted the file and returned the upload
path, confirming no server-side file type validation was in place at this level.

### Phase 4 -- Medium Security Bypass via Content-Type Modification
With security level set to Medium, a direct upload of `backdoor.php` was rejected
with the message "We can only accept JPEG or PNG images." Burp Suite was
configured to intercept the upload request. The `Content-Type` header in the
multipart form data was changed from `application/x-php` to `image/jpeg`. The
modified request was forwarded and the server returned HTTP 200, confirming
the shell was uploaded. The medium-level filter checked only the `Content-Type`
header and did not inspect the actual file content.

### Phase 5 -- High Security Bypass via Magic Bytes and Double Extension
With security level set to High, both a direct upload and a Content-Type bypass
were insufficient. The high-level filter validated the file's magic bytes against
known image signatures rather than trusting the request headers. To bypass this,
the GIF magic byte signature (`GIF89a`) was prepended to the web shell content,
and the filename was changed to `backdoor.php.jpg` (double extension). The
modified request was intercepted in Burp Suite and forwarded. The server
returned HTTP 200 and confirmed the upload succeeded.

### Phase 6 -- Shell Access Confirmation
The uploaded shell was accessed via Weevely from the terminal:
```

weevely http://127.0.0.1:42001/hackable/uploads/backdoor.php 12345
```
Commands were executed within the shell session to confirm code execution on
the server.

---

## 5. Vulnerability Summary

| ID | Vulnerability | Severity | Affected Endpoint |
|----|--------------|----------|-------------------|
| 01 | Unrestricted File Upload - No Validation (Low) | Critical | /dvwa/vulnerabilities/upload/ |
| 02 | MIME Type Bypass via Content-Type Header (Medium) | Critical | /dvwa/vulnerabilities/upload/ |
| 03 | Magic Byte Bypass via GIF Signature + Double Extension (High) | High | /dvwa/vulnerabilities/upload/ |
| 04 | Remote Code Execution via Uploaded Web Shell | Critical | /hackable/uploads/backdoor.php |
| 05 | Upload Directory Web-Accessible and Executable | Critical | /hackable/uploads/ |

---

## 6. Detailed Findings

---

### Finding 01 -- Unrestricted File Upload (Low Security)

#### Severity
Critical

#### Affected Endpoint
`/dvwa/vulnerabilities/upload/` -- File upload form, POST

#### Description
At low security level, the DVWA upload form accepts any file type regardless of
extension, MIME type, or content. A `.txt` file, a `.sh` shell script, a `.ovpn`
VPN configuration file, and a `.php` web shell were all uploaded successfully
without any restriction. The only limit confirmed was a 300KB file size cap. No
file type check existed at either the client or server layer.

#### Proof of Concept

DVWA File Upload page confirming upload form accessed at low security:



![DVWA File Upload Page - Low Security Level](image.jpg)



Text file and shell script uploaded with server confirmation:



![Upload Confirmation - file1.txt and dns_resolve.sh Successfully Uploaded](image.jpg)



OpenVPN configuration file uploaded to confirm absence of file type restriction:



![Upload Confirmation - vpnbook-ca149-tcp443.ovpn Successfully Uploaded](image.jpg)



`backdoor.php` web shell uploaded directly with no modification:



![Upload Confirmation - backdoor.php Successfully Uploaded at Low Security](image.jpg)



Server response confirming upload path:
```

../../hackable/uploads/backdoor.php succesfully uploaded!
```

#### Impact
Any authenticated user can upload a PHP web shell directly to the server at low
security level. Once the file is in a web-accessible directory with PHP execution
enabled, the attacker has full remote code execution on the web server process.
From that position, the attacker can read filesystem contents, access database
credentials stored in application configuration files, and attempt privilege
escalation from the `www-data` context to root.

#### Remediation
- Implement server-side file type validation using a strict allowlist of permitted
  MIME types and extensions (e.g., `image/jpeg`, `image/png` only)
- Validate file content using magic byte inspection in addition to extension
  and MIME type checks
- Store uploaded files outside the web root to prevent direct HTTP access
- Disable PHP execution in the upload directory via `.htaccess`:
  `php_flag engine off`

---

### Finding 02 -- MIME Type Bypass via Content-Type Header (Medium Security)

#### Severity
Critical

#### Affected Endpoint
`/dvwa/vulnerabilities/upload/` -- File upload form, POST

#### Description
At medium security level, the upload form rejected `backdoor.php` with the
message "Your image was not uploaded. We can only accept JPEG or PNG images."
The server's validation checked only the `Content-Type` header supplied by the
client in the multipart form data. This header is part of the HTTP request and
is fully controlled by the client. Changing the `Content-Type` from
`application/x-php` to `image/jpeg` in Burp Suite before forwarding the request
caused the server to accept the file without any further inspection.

#### Proof of Concept

Medium-level upload rejection without modification:



![DVWA Medium Level - Upload Rejected for PHP File](image.jpg)



Burp Suite request interception showing `Content-Type` modification from
`application/x-php` to `image/jpeg`:



![Burp Suite Intercept - Content-Type Changed to image/jpeg](image.jpg)



Server response after forwarding the modified request:



![Upload Confirmation - backdoor.php Uploaded After Content-Type Bypass](image.jpg)



HTTP response confirming success:
```

HTTP/1.1 200 OK
../../hackable/uploads/backdoor.php succesfully uploaded!
```

The file extension remained `.php` throughout. No extension change or double
extension was required to bypass the medium-level filter.

#### Impact
The `Content-Type` header is metadata the client sends to describe the file. It
is not derived from the file itself and can be set to any value in any HTTP client
or proxy. Trusting it as a security control is equivalent to trusting user input.
This bypass demonstrates CWE-20 (Improper Input Validation) -- the server
validated the metadata rather than the actual file content.

#### Remediation
- Replace `Content-Type` header checking with server-side magic byte
  inspection that reads the first bytes of the uploaded file to determine its
  actual type independent of what the client claims
- In PHP, use `finfo_file()` rather than `$_FILES['file']['type']` for MIME
  type determination:
```php
$finfo = finfo_open(FILEINFO_MIME_TYPE);
$mime = finfo_file($finfo, $_FILES['uploaded']['tmp_name']);
```

- Maintain an allowlist of permitted MIME types derived from actual file
  content, not from client-supplied headers

---

### Finding 03 -- Magic Byte Bypass via GIF Signature and Double Extension (High Security)

#### Severity
High

#### Affected Endpoint
`/dvwa/vulnerabilities/upload/` -- File upload form, POST

#### Description
At high security level, the server validated the uploaded file's content using
magic byte inspection rather than relying on the `Content-Type` header. A
direct upload of `backdoor.php` failed. A Content-Type header change alone
also failed. Bypassing this level required three simultaneous modifications:
prepending the GIF magic byte signature (`GIF89a`) to the beginning of the web
shell content, renaming the file to `backdoor.php.jpg` (double extension), and
changing the `Content-Type` header to `image/jpeg`. All three changes were
applied in the Burp Suite intercept before forwarding the request. The server
accepted the file and returned HTTP 200.

#### Proof of Concept

Web shell content modified in nano to include GIF magic bytes prepended
before the PHP code:



![nano Editor - GIF89a Magic Bytes Prepended to backdoor.php.jpg](image.jpg)



Burp Suite request showing the filename set to `backdoor.php.jpg` and
`Content-Type` set to `image/jpeg`:



![Burp Suite Intercept - Double Extension and image/jpeg Content-Type for High Level](image.jpg)



Server response confirming the file was accepted:



![Upload Confirmation - backdoor.php.jpg Successfully Uploaded at High Security](image.jpg)



HTTP response:
```
HTTP/1.1 200 OK
../../hackable/uploads/backdoor.php.jpg succesfully uploaded!
```

#### Impact
Although the high-level filter is significantly stronger than the lower levels,
the combination of magic byte spoofing and double extension bypassed it. The
effectiveness of the bypass depends on how the web server handles files with
double extensions. If the server is configured to execute `.php.jpg` files as PHP
(which some misconfigured Apache setups do), the uploaded shell is fully
executable. This demonstrates that magic byte validation alone is not sufficient
when the web server's execution configuration is permissive.

#### Remediation
- Rename all uploaded files to a randomly generated string with a forced safe
  extension (e.g., a UUID with `.jpg`) so the original filename and extension
  have no influence on server behavior:
```php
$filename = bin2hex(random_bytes(16)) . '.jpg';
```
- Configure the web server to treat the upload directory as serving static files
  only, with PHP execution explicitly disabled regardless of file extension
- Combine magic byte validation with the above controls so that a bypassed
  content check does not translate to execution even if the file reaches the
  server

---

### Finding 04 -- Remote Code Execution via Uploaded Web Shell

#### Severity
Critical

#### Affected Endpoint
`http://127.0.0.1:42001/hackable/uploads/backdoor.php`

#### Description
After successful upload at low security level, the web shell was accessed via
Weevely using the password set during generation (`12345`). The connection
was established immediately, providing an interactive terminal session running
on the server. Commands were run to confirm the execution context and
filesystem access.

#### Proof of Concept

Weevely connection command:
```
weevely http://127.0.0.1:42001/hackable/uploads/backdoor.php 12345
```

Weevely terminal output confirming shell access and command execution:



![Weevely Shell - Connected to backdoor.php, Commands Executed](image.jpg)



Commands run within the shell session:
```
ls
backdoor.php
backdoor.php.jpg
dvwa_email.png

uname -a
Linux kali 6.17.10+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.17.10-1kali1
(2025-12-08) x86_64 GNU/Linux

pwd
_dvwa@kali:/var/lib/dvwa/uploads
```

The session confirmed the working directory as `/var/lib/dvwa/uploads`,
the kernel version, and the presence of all previously uploaded files including
both the low-level and high-level shells.

#### Impact
An interactive shell session gives the attacker persistent access to the web
server without needing to re-exploit the upload vulnerability. From the confirmed
working directory and user context, the attacker can read database configuration
files containing credentials, browse the application source code, attempt local
privilege escalation to root using kernel exploits, and scan the internal network
for further pivot targets.

#### Remediation
- The shell access was only possible because the upload directory is
  web-accessible and PHP execution is enabled within it. Storing uploaded
  files outside the web root removes the ability to trigger PHP execution via
  a direct HTTP request to the uploaded file, regardless of what the file
  contains.
- Monitor the upload directory for file creation events using a host-based
  intrusion detection tool (e.g., Tripwire or auditd) and alert on any new
  `.php` files appearing in upload paths.

---

### Finding 05 -- Upload Directory Web-Accessible with PHP Execution Enabled

#### Severity
Critical

#### Affected Endpoint
`/hackable/uploads/` -- Web-accessible directory

#### Description
The directory where DVWA stores uploaded files (`/hackable/uploads/`) is
directly accessible via HTTP and the web server executes PHP files requested
from within it. This is the condition that converts a file upload vulnerability into
remote code execution. Without this condition, an attacker could upload a PHP
file but could not trigger its execution. The `ls` command output from the Weevely
session confirmed that both `backdoor.php` and `backdoor.php.jpg` were stored
in this directory and were reachable at a predictable URL.

#### Proof of Concept

The upload path disclosed in every successful upload response:
```
../../hackable/uploads/backdoor.php succesfully uploaded!
```

Shell access confirmed at the full URL:
```
http://127.0.0.1:42001/hackable/uploads/backdoor.php
```

#### Impact
A web-accessible upload directory with PHP execution enabled means that
uploading any PHP file is equivalent to planting a backdoor. Even well-crafted
file type validation can be bypassed given enough effort, as demonstrated across
all three security levels. Making uploaded files non-executable by design removes
the execution risk regardless of what filters are in place.

#### Remediation
- Move the upload directory outside the web root so uploaded files cannot
  be accessed via HTTP directly
- If files must remain inside the web root (e.g., for user-facing download), add
  an `.htaccess` file in the uploads directory to disable script execution:

```
<FilesMatch "\.(php|php5|phtml|cgi|pl|py)$">
    Deny from all
</FilesMatch>
Options -ExecCGI
```
- Serve user-uploaded files through a PHP script that reads and streams the
  file, rather than allowing direct URL access to the stored path

---

## 7. Attack Chain

```
[Step 1] Reconnaissance
Upload form tested with .txt, .sh, and .ovpn files at Low security
All files accepted - no file type restriction present
Upload directory confirmed: ../../hackable/uploads/
        |
        v
[Step 2] Web Shell Generation
weevely generate 12345 backdoor.php
693-byte obfuscated PHP shell generated
File content confirmed as obfuscated to evade signature scanners
        |
        v
[Step 3] Low Level - Direct Upload
backdoor.php uploaded via browser with no modification
Server accepts file, returns upload path
No server-side validation present at low level
        |
        v
[Step 4] Medium Level - Content-Type Bypass
Direct upload rejected: "We can only accept JPEG or PNG images"
Burp Suite intercept: Content-Type changed from application/x-php to image/jpeg
Modified request forwarded, HTTP 200 returned
backdoor.php uploaded with .php extension intact
        |
        v
[Step 5] High Level - Magic Byte + Double Extension Bypass
Direct upload blocked, Content-Type change alone insufficient
GIF89a magic bytes prepended to shell content in nano
Filename changed to backdoor.php.jpg, Content-Type set to image/jpeg
All three modifications applied simultaneously in Burp Suite
HTTP 200 returned, backdoor.php.jpg confirmed uploaded
        |
        v
[Step 6] Shell Access Confirmed
weevely http://127.0.0.1:42001/hackable/uploads/backdoor.php 12345
Interactive terminal session established on server
uname -a, ls, pwd executed - full filesystem access confirmed
Working directory: /var/lib/dvwa/uploads
```

---

## 8. Tools Used

- Kali Linux (lab environment)
- Firefox (DVWA access)
- Weevely 4.0.2 (PHP web shell generation and terminal access)
- Burp Suite (HTTP request interception and Content-Type modification)
- nano (web shell content editing for magic byte prepending)
- DVWA (intentionally vulnerable target application)

---

## 9. Challenges Encountered

- **ModuleNotFoundError for telnetlib when connecting via Weevely:** When
  the shell access was first attempted, Weevely threw a `ModuleNotFoundError`
  for `telnetlib`. This occurred because Python's standard library removed
  `telnetlib` in Python 3.13 as it was deprecated. The fix was to reinstall
  Weevely using the UV installation method documented in the project's
  `README.md`, which provided a clean environment with compatible
  dependencies. After reinstallation, Weevely connected to the shell without
  errors.
- **High-level bypass required multiple simultaneous changes:** Applying
  only the GIF magic bytes or only the double extension individually was
  insufficient to pass the high-level filter. The bypass only succeeded when
  the file signature, filename, and Content-Type were all modified together in
  a single intercepted request. Testing each change in isolation first helped
  identify exactly which combination the filter required.
- **Double extension execution depends on server configuration:** The
  `backdoor.php.jpg` file was uploaded successfully, but whether PHP executes
  it depends on the Apache configuration. In this lab environment it executed
  because PHP was not restricted in the upload directory. On a more hardened
  server, the double extension bypass would upload the file but not achieve
  code execution.

---

## 10. Key Takeaways

- **Content-Type is client-controlled data, not a security control:** The entire
  medium-level bypass required changing one header value in Burp Suite. Any
  validation that relies solely on `$_FILES['file']['type']` in PHP is checking
  data the client provided, which the client can set to anything. Server-side
  magic byte inspection is the minimum viable check for file type validation.
- **Magic byte validation is necessary but not sufficient on its own:** The
  high-level filter validated file content rather than headers, which is the right
  approach. It still fell to a bypass because execution was possible in the upload
  directory. Defense in depth requires both content validation and execution
  prevention in the storage location.
- **The upload directory's executability is the real root cause:** Across all three
  levels, the condition that converted file upload into remote code execution was
  the same: the upload directory was web-accessible and PHP execution was
  enabled. Fixing only the upload filter without addressing the directory's
  execution permissions leaves the risk in place. An attacker who finds a new
  bypass technique still gets code execution. Removing execution from the
  directory entirely removes the risk at the infrastructure level, independent of
  how good the filter is.
- **Obfuscated shells evade signature-based detection:** The Weevely-generated
  shell was deliberately obfuscated. A simple grep for `<?php` or `system(` in
  upload directories would not flag it. File integrity monitoring using hashes
  rather than content pattern matching is needed to detect new files in sensitive
  directories.
- **Weevely's persistence is its real threat:** After the initial upload, the
  attacker reconnects to the shell at any time without exploiting the upload
  vulnerability again. The foothold persists until the file is removed and the
  database reset. This makes detection and cleanup time-sensitive once an
  upload vulnerability is exploited.

---

## 11. Disclaimer

> This assessment was conducted exclusively against DVWA, an intentionally
> vulnerable web application deployed in an isolated local lab environment at
> `http://192.168.92.3`. No production systems, live servers, or unauthorized
> infrastructure were accessed at any point. The web shell generated and uploaded
> during this assessment existed only within the local lab environment and was
> used solely to demonstrate remote code execution for educational purposes. All
> techniques described in this report must only be applied against systems for
> which explicit written authorization has been obtained prior to testing.

