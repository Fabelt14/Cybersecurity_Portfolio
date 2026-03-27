# Wireshark Network Protocol Analysis

## Overview

This lab focused on network traffic analysis using Wireshark to capture, filter, and analyze packets at the protocol level. The goal was to understand how data moves across networks, identify security vulnerabilities in unencrypted traffic, and build the skills needed for network troubleshooting and incident response.

## Objectives

- Navigate Wireshark's interface and understand packet capture workflow
- Apply display filters to isolate specific protocols (HTTP, DNS, TCP)
- Dissect individual packets to extract source/destination IPs and protocol flags
- Follow TCP streams to reconstruct application-layer conversations
- Identify unencrypted credentials and security threats in captured traffic
- Use protocol hierarchy and I/O graphs to analyze network behavior
- Document findings for forensic investigation

## Lab Environment

- **OS**: Kali Linux
- **Capture Tool**: Wireshark (GUI), Tshark (CLI)
- **Target Application**: Mutillidae (intentionally vulnerable web app)
- **Network**: Local lab environment

## Tools Used

- Wireshark - Graphical packet analyzer with real-time capture and filtering
- Tshark - Command-line version of Wireshark for scripting and automation
- tcpdump - Packet capture utility (background reference)

## Methodology

### Exercise 1: Interface Exploration

I opened Wireshark and examined the interface to locate critical analysis components.

**Interface components identified:**

**Packet List Pane (top section):**
This displays all captured packets in chronological order with columns for time, source, destination, protocol, and info. Each row represents one packet.

**Packet Details Pane (middle section):**
Shows the protocol layers of a selected packet in expandable tree format. I can drill down into Ethernet → IP → TCP → HTTP headers here.

**Packet Bytes Pane (bottom section):**
Displays the raw hexadecimal and ASCII representation of the selected packet. This is where I'd see actual payload data.

**Statistics Menu:**
Located in the top menu bar. Contains tools for protocol hierarchy, I/O graphs, conversations, and endpoints analysis.

![Wireshark interface components labeled](images/wireshark-interface.png)

**Why this layout matters:**
The three-pane design mirrors the OSI model workflow - see all packets first, drill into layers second, examine raw bytes third. This makes protocol analysis systematic rather than random.

### Exercise 2: Capture Comparison - Wireshark vs Tshark

I captured network traffic using both tools to understand when to use GUI vs CLI analysis.

**Wireshark capture:**
Started capture on eth0 interface, browsed to Mutillidae login page, stopped capture. GUI displayed packets in real-time with color-coded protocols (green for TCP, blue for DNS, black for HTTP).

![Wireshark GUI capture](images/wireshark-capture.png)

**Tshark capture:**
```bash
tshark -i eth0 -w capture.pcap
```

Output scrolled in terminal showing packet summaries in text format. No visual filtering or color coding - just raw packet data streaming to console.

![Tshark CLI capture](images/tshark-capture.png)

**Key differences:**

**Real-time analysis:**
Wireshark excels here. I can apply display filters during capture (`http.request.method == "POST"`), follow streams with right-click, and see color-coded protocol warnings. Tshark outputs raw text to stdout - I'd need to pipe it through grep/awk to filter, which interrupts the capture process.

**Post-capture review:**
Wireshark's GUI makes forensic analysis intuitive. I can:
- Click packets to see full dissection
- Right-click → Follow TCP Stream to reconstruct conversations
- Use Statistics → Protocol Hierarchy to see traffic breakdown
- Export objects (HTTP files, images) with one click

Tshark requires commands for everything:
```bash
tshark -r capture.pcap -Y "http.request.method == POST"
tshark -r capture.pcap -z follow,tcp,ascii,0
```

This is powerful for automation (bash scripts, cron jobs) but slower for interactive investigation.

**When to use each:**
- **Wireshark**: Initial triage, hunting for unknown threats, teaching/demonstrating attacks
- **Tshark**: Automated monitoring, parsing massive captures (100GB+), headless servers with no GUI

### Exercise 3: Filter Application

I applied display filters to count specific protocol traffic and identify communicating hosts.

**HTTP packet count:**

Applied filter: `http`

Result: 20 packets (10 requests + 10 responses)

![HTTP filter showing 20 packets](images/filter-http.png)

**Why 10 pairs:**
Each HTTP transaction involves:
1. Client → Server: GET /index.php
2. Server → Client: HTTP/1.1 200 OK + HTML content

The capture included multiple page loads (login page, submit form, redirect, error page), accounting for the 10 request/response cycles.

**DNS packet count:**

Applied filter: `dns`

Result: 173 packets

![DNS filter showing 173 packets](images/filter-dns.png)

**Why so many DNS packets:**
Web browsers perform DNS lookups for every hostname encountered:
- Main domain (mutillidae.local)
- External resources (CDNs, analytics, ads)
- Failed queries (NXDOMAIN responses) that retry
- Background services (OS updates, time sync)

173 packets suggests heavy web browsing or DNS reconnaissance scanning in the background.

**IP address identification:**

Applied filter: `ip.addr == 10.0.2.15 || ip.addr == 146.75.89.91`

**10.0.2.15**: This is my Kali VM's internal IP (VirtualBox NAT default range). This is the client making HTTP requests.

**146.75.89.91**: This is the Mutillidae web server (or external server hosting resources). This is the target responding to requests.

![IP address filter](images/filter-ip.png)

**Security observation:**
The IP addresses are visible in cleartext because I'm analyzing unencrypted HTTP. With HTTPS, the IP headers would still be visible (routing requirement), but the payload would be encrypted.

### Exercise 4: Packet Dissection

I selected a TCP packet with the SYN flag to examine connection establishment.

**Packet selected:**
Frame 23 in packet list - first packet in TCP three-way handshake.

**Details extracted from Packet Details Pane:**

**Source IP**: 10.0.2.15 (my Kali VM initiating connection)

**Destination IP**: 146.75.89.91 (Mutillidae server)

**Protocol**: TCP (6) - this is the IP protocol number for TCP in the IP header

**TCP Flags**: SYN set (0x002)

![TCP packet dissection](images/tcp-dissection.png)

**Expanding the layers:**

**Ethernet II:**
- Source MAC: (client NIC)
- Destination MAC: (gateway/router MAC)

**Internet Protocol Version 4:**
- Header checksum: 0xa9e7 (validated)
- TTL: 64 (decrements at each router hop)
- Protocol: 6 (TCP)

**Transmission Control Protocol:**
- Source Port: (ephemeral port 40000+)
- Destination Port: 80 (HTTP standard port)
- Sequence Number: 0 (relative sequence, SYN starts at random number)
- Acknowledgment Number: 0 (not set in SYN packet)
- **Flags: 0x002 (SYN)**
  - Window size: 64240 bytes (receive buffer size)
  - Checksum: Unverified (depends on NIC offload settings)

![TCP flags detail](images/tcp-flags.png)

**Why the SYN flag matters:**
This packet initiates the TCP three-way handshake:
1. Client → Server: SYN (I want to connect)
2. Server → Client: SYN-ACK (OK, I'm ready)
3. Client → Server: ACK (Connection established)

Without SYN, no TCP connection can start. Port scanners (nmap) send SYN packets to detect open ports - if the server responds with SYN-ACK, the port is open.

### Exercise 5: Stream Analysis

I followed a TCP stream to reconstruct an HTTP login session and extract credentials.

**Steps:**
1. Located an HTTP POST packet in packet list (packet with "POST /index.php" in Info column)
2. Right-click → Follow → TCP Stream
3. Wireshark displayed the full conversation in a new window

**Stream contents:**

**Client → Server (HTTP Request):**
```
POST /mutillidae/index.php?page=login.php HTTP/1.1
Host: 192.168.0.25
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: text/html,application/xhtml+xml
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 95
Origin: http://192.168.0.25
Connection: keep-alive
Referer: http://192.168.0.25/mutillidae/index.php?page=login.php

username=user&password=user&login-php-submit-button=Login
```

![TCP stream - HTTP POST request](images/stream-request.png)

**Server → Client (HTTP Response):**
```
HTTP/1.1 302 Found
Date: Sun, 20 Dec 2025 10:21:32 GMT
Server: Apache/2.4.18 (Ubuntu)
X-Powered-By: PHP/5.3-1ubuntu4.30
```

![TCP stream - HTTP response](images/stream-response.png)

**Credentials extracted:**
- **Username**: user
- **Password**: user

**How the credentials were exposed:**

The POST request uses `application/x-www-form-urlencoded` content type. This means form data is sent in the URL-encoded format:
```
username=user&password=user
```

Because this is HTTP (not HTTPS), the entire request travels across the network in plaintext. Anyone with Wireshark on the same network segment can see these credentials.

**Attack scenario:**
If an attacker sat on the same WiFi network as this user:
1. Run Wireshark with filter `http.request.method == "POST"`
2. Wait for login attempts
3. Follow TCP streams to extract usernames/passwords
4. Use credentials to compromise accounts

**User-Agent string revealed:**
```
Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
```

This tells the attacker:
- OS: Linux (X11)
- Architecture: 64-bit (x86_64)
- Browser: Firefox 68.0 (outdated - released 2019, vulnerable to known exploits)

In a forensic investigation, this identifies the attacker's system.

### Exercise 6: Protocol Analysis

I used Statistics → Protocol Hierarchy to identify the most common protocol in my capture.

**Steps:**
1. Statistics → Protocol Hierarchy
2. Examined percentage breakdown of all protocols

![Protocol hierarchy statistics](images/protocol-hierarchy.png)

**Results:**

**Frame (100%)**
- Ethernet (100%)
  - IP (100%)
    - **HTTP (31.1%)** ← Most prevalent
      - HTML Form URL Encoded (13.3%)
      - Line-based text data (12.2%)
      - Compressive GIF (2.2%)
    - TCP (45 packets, 3.8%)
    - UDP (45 packets, 3.9%)
    - ICMP (924 packets, 112 bytes)

**What this indicates:**

**HTTP dominance:**
31.1% of packets are HTTP, meaning this capture was primarily a web browsing session. The user was interacting with a web application (Mutillidae) rather than doing file transfers, video streaming, or other network activities.

**HTML Form URL Encoded (13.3%):**
This is the smoking gun. Form URL Encoded data means the user submitted a login form, uploaded data, or performed a search. Combined with the TCP stream analysis from Exercise 5, this confirms a login event with credentials transmitted in cleartext.

**Security implication:**
If this were a corporate network and 30%+ of traffic is HTTP instead of HTTPS, it means:
- Legacy web applications still deployed
- Man-in-the-middle attacks possible
- Compliance failures (PCI-DSS requires HTTPS for payment data)

A security analyst would flag this for immediate remediation.

### Exercise 7: Traffic Visualization

I created an I/O graph to visualize TCP traffic patterns over time.

**Steps:**
1. Statistics → I/O Graph
2. Y Axis: Packets (count)
3. X Axis: Time (seconds)
4. Filter: `tcp` (show only TCP traffic)
5. Observed the graph for spikes and anomalies

![TCP I/O graph](images/io-graph.png)

**Pattern observed:**

**Baseline traffic (0-40 seconds):**
Graph shows minimal activity - 0 to 50 packets per second. This represents normal HTTP browsing: user loads page, reads content, clicks next link. There are gaps where no packets flow (user is reading, not transmitting).

**Traffic spike (40-55 seconds):**
Sudden jump to **550 packets per second** sustained for 15 seconds. This is abnormal for human web browsing.

**Anomaly analysis:**

**What causes 550 packets in 15 seconds:**
This pattern matches ICMP ping flood or network reconnaissance:
- Tool: `ping -f 192.168.0.25` (flood ping, sends packets as fast as possible)
- Or: `nmap -sn 192.168.0.0/24` (ping sweep to discover live hosts)

**Why this isn't web browsing:**
- Web browsing is bursty (load page, pause, next page)
- 550 packets/sec sustained = automated tool, not human clicking
- ICMP Echo Request/Reply pairs are 2 packets per ping × 275 pings = 550 packets

**Security implication:**
If I'm investigating a breach and see this pattern:
1. Attacker gained initial access (stolen credentials from Exercise 5)
2. Performed host discovery (this ping sweep)
3. Mapped the network to find additional targets

This is reconnaissance phase of the cyber kill chain. The next step would be lateral movement to other systems.

### Exercise 8: Documentation and Reporting

I saved the capture file for future forensic analysis.

**Save process:**
```bash
File → Save As → test.pcap
```

**File saved:** test.pcap (packet capture file in libpcap format)

![Saved capture file](images/save-pcap.png)

**Forensic scenario:**

**Investigation trigger:**
Unauthorized access detected on 192.168.0.25 (Mutillidae server). Attacker modified database records and exfiltrated user data. I need to determine:
- How did the attacker gain access?
- What IP address did they connect from?
- What tools did they use?
- What data was stolen?

**Evidence extraction from saved PCAP:**

**1. Attacker credentials (from Exercise 5 stream):**
- Username: user
- Password: user

This proves the attacker used stolen credentials, not a system exploit. The compromise started with credential theft (phishing, keylogger, or shoulder surfing).

**2. Attacker source IP:**
Filter: `ip.src == 10.0.2.15 && http.request`

This identifies the attacking machine. Cross-reference with DHCP logs to determine which physical device was assigned this IP at the time of attack.

**3. User-Agent string:**
```
Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
```

Indicates:
- Attacker used Linux
- Outdated Firefox 68.0 (released July 2019)
- If corporate environment standardizes on Windows + Chrome, this is an external attacker

**4. Reconnaissance activity:**
The 550-packet spike from Exercise 7 proves the attacker performed network mapping after initial access. This indicates:
- Not an opportunistic attack (they planned to move laterally)
- Likely used nmap or similar scanning tool
- Timeline: Logged in at 40 seconds, started scanning at 42 seconds

**5. Timeline reconstruction:**
```
00:00 - User browsing Mutillidae normally
00:38 - POST request to login.php with credentials
00:40 - Successful login (HTTP 302 redirect)
00:42 - ICMP ping flood begins (reconnaissance)
00:57 - Ping flood ends
01:05 - Additional HTTP POST requests (possible SQL injection or data exfil)
```

**Evidence preservation:**
- PCAP file hashed with SHA-256 to prove integrity
- Stored on write-once media (WORM drive or blockchain timestamping)
- Chain of custody documented

This PCAP becomes Exhibit A in incident response report or legal proceedings.

### Exercise 9: Troubleshooting Scenario

**Scenario: Slow File Transfer**

An employee reports that downloading a 10MB report from the internal file server (`\\fileserver\reports\Q4-2025.pdf`) takes 5 minutes instead of the expected 10 seconds. I need to determine if the problem is network congestion, server performance, or client-side issues.

**Troubleshooting approach with Wireshark:**

**Step 1: Capture during file transfer**
- Start Wireshark on client machine
- Initiate download of Q4-2025.pdf
- Let it run until download completes (or times out)
- Stop capture

**Step 2: Analyze I/O Graph**
Statistics → I/O Graph, filter: `tcp`

**Expected pattern (healthy transfer):**
```
Packets/sec: Steady ~1000 packets/sec for duration of transfer
Duration: 10 seconds for 10MB = 1MB/sec throughput
```

![Expected I/O graph for fast transfer](images/troubleshoot-fast.png)

**Actual pattern (problem scenarios):**

**Scenario A: Network congestion**
Graph shows:
- Frequent drops to 0 packets/sec
- Spikes to 200-300 packets/sec
- Retransmissions visible (filter: `tcp.analysis.retransmission`)

Diagnosis: Network path saturated. Competing traffic (video streams, backups) consuming bandwidth.

![I/O graph showing congestion](images/troubleshoot-congestion.png)

**Scenario B: Server performance issue**
Graph shows:
- Initial burst of packets (SYN, data frames)
- Long pauses (5-10 seconds) with zero activity
- Another burst, another pause

Diagnosis: Server processing delay. File server's disk is slow, or CPU is maxed serving other requests. Network is idle during these gaps.

**Scenario C: TCP window scaling problem**
Check packet details for:
- Window Size: Should scale to 64KB+ for modern networks
- If stuck at 8KB or 16KB, old router or firewall is blocking TCP window scaling options

**Step 3: Check for specific indicators**

**DNS issues:**
Filter: `dns.flags.rcode != 0`

If server name resolution takes 30+ seconds, DNS timeout is adding delay before transfer even starts.

**Packet loss:**
Filter: `tcp.analysis.retransmission || tcp.analysis.duplicate_ack`

High retransmission count (>5% of packets) indicates lossy network path - WiFi interference, bad cable, failing switch port.

**TCP Expert Info:**
Analyze → Expert Information

Wireshark highlights:
- Window Full warnings (receiver can't keep up)
- ACKed unseen segment (missing packets in capture)
- Out-of-order packets (routing issues)

**Step 4: Determine root cause**

Compare:
- Client → Server packets: Do ACKs arrive promptly?
- Server → Client packets: Are data segments sent continuously or in bursts with delays?

If delays appear in server→client direction with long gaps between data segments, the server is the bottleneck. If ACKs from client are delayed, client's CPU or disk is struggling to process incoming data.

**Resolution:**
- Network congestion → QoS policy to prioritize file transfer traffic
- Server issue → Upgrade server RAM/disk, balance load across multiple servers
- Client issue → Check local antivirus scanning downloaded files in real-time

This systematic approach isolates the failure point without guessing.

### Exercise 10: Threat Identification

I analyzed the captured traffic to identify security threats and suspicious activities.

**Threat 1: Unencrypted Login Credentials**

**Evidence:**
From Exercise 5 TCP stream:
```
POST /mutillidae/index.php?page=login.php
Content-Type: application/x-www-form-urlencoded

username=user&password=user
```

**Packet indicators:**
- Protocol: HTTP (port 80)
- No TLS/SSL handshake observed
- POST request with form data in plaintext
- No encryption headers (e.g., `Strict-Transport-Security`)

![Unencrypted credentials in stream](images/threat-credentials.png)

**Why this is a threat:**
Any user on the same network segment can:
1. Capture this traffic with Wireshark
2. Extract credentials
3. Log in as the victim
4. Steal data, modify records, escalate privileges

**Mitigation:**
- Enforce HTTPS with TLS 1.3
- Implement HSTS (HTTP Strict Transport Security) header
- Redirect all HTTP requests to HTTPS
- Use certificate pinning to prevent MITM attacks

**Threat 2: Network Reconnaissance (ICMP Flood)**

**Evidence:**
From Exercise 7 I/O Graph: 550 packets in 15 seconds burst starting at 40-second mark.

**Packet indicators:**
Applied filter: `icmp`

Observed:
- ICMP Echo Request packets from 10.0.2.15
- ICMP Echo Reply packets from multiple IPs in 192.168.0.0/24 range
- Packet rate: 35+ packets/second sustained
- Pattern: Sequential destination IPs (.1, .2, .3, .4...)

![ICMP flood pattern](images/threat-icmp.png)

**Why this is suspicious:**

**Normal user behavior:**
- Pings are manual (one at a time)
- Used to test connectivity ("Can I reach google.com?")
- Typical rate: 1 packet/second for 4-5 seconds

**Attacker behavior:**
- Automated ping sweep with tools (nmap, angry IP scanner)
- Goal: Map live hosts on network
- Precursor to lateral movement or exploit targeting

**Attack timeline:**
1. 00:40 - Attacker logged in with stolen credentials (Threat 1)
2. 00:42 - Began ping sweep to discover other targets
3. 00:57 - Completed network map
4. Next phase: Port scanning (nmap -sS), vulnerability scanning, or exploit attempts

**Mitigation:**
- Block ICMP Echo Request at firewall (or rate-limit to 10/second)
- Deploy IDS/IPS to alert on ping sweeps
- Network segmentation (prevent lateral movement even if attacker maps network)
- Monitor for nmap signatures (TCP SYN scans, UDP scans)

**Additional indicators:**

**Excessive DNS queries (173 packets):**
While not inherently malicious, 173 DNS queries in a short capture could indicate:
- DNS tunneling (exfiltrating data via DNS queries)
- Command and control (C2) beaconing to resolve malicious domains
- Normal if user browsed many external sites, but warrants investigation

Filter: `dns.qry.name contains "suspicious-domain.com"`

**Port scanning attempts:**
Filter: `tcp.flags.syn == 1 && tcp.flags.ack == 0`

If I see SYN packets to sequential ports (22, 23, 80, 443, 3389, 8080) from same source IP, that's a port scan. Wireshark would show many packets with no corresponding SYN-ACK replies for closed ports.

## Findings

**Wireshark vs Tshark:**
Wireshark's GUI provides faster triage for unknown threats through point-and-click filtering, color coding, and stream following. Tshark excels in automation (scripted monitoring, log parsing) and works on headless servers but requires command-line expertise for real-time analysis.

**Filter effectiveness:**
Display filters isolated specific traffic instantly. `http` returned 20 packets from thousands captured. `dns` found 173 queries. Combining filters (`ip.addr == X && http`) narrowed results to specific conversations. This is how analysts sift through 100GB captures in minutes instead of hours.

**TCP stream reconstruction:**
Following TCP Stream reassembled fragmented application-layer data into readable conversations. A login attempt split across 10 TCP segments became one cohesive HTTP POST request with visible credentials. This technique works for any TCP protocol - FTP, SMTP, HTTP, SSH (before encryption).

**Protocol hierarchy reveals user behavior:**
31.1% HTTP traffic confirmed web browsing session. HTML Form URL Encoded (13.3%) pinpointed data submission. If hierarchy showed 80% BitTorrent or 60% SMB, that would indicate file sharing instead. This statistical view identifies network usage patterns without examining individual packets.

**I/O graphs detect anomalies:**
Normal web traffic is bursty (load, pause, load). The 550 packets/sec spike was a clear outlier. In enterprise monitoring, setting baseline thresholds (alert if packets/sec > 100) flags reconnaissance, DDoS, or data exfiltration automatically.

**Unencrypted HTTP exposes everything:**
Credentials, session tokens, search queries, uploaded files - all visible in cleartext. Even metadata (User-Agent, Referer headers) leaks system info. HTTPS would encrypt payload but IP addresses, DNS queries, and TLS SNI extension still leak some information.

**ICMP reconnaissance precedes attacks:**
Attackers don't randomly scan ports. They map live hosts first (ICMP sweep), then scan open ports (TCP SYN scan), then attempt exploits. Seeing the ping flood at 00:42 means the attacker is in early stages - there's time to isolate the system before lateral movement.

**PCAP files are legal evidence:**
Saved captures with timestamps, IP addresses, and payload data prove what happened on the network. Chain of custody, cryptographic hashing (SHA-256), and immutable storage make PCAPs admissible in court. This is why network taps are deployed at ISP boundaries and corporate gateways.

## Challenges Faced

**Wireshark display filter syntax:**
Initially tried `http and ip.addr == 10.0.2.15` which failed. Wireshark uses `&&` for AND, not `and`. The error message was vague ("syntax error"). Learning to use the Expression builder (right-click filter bar → Expression) helped build correct filters faster.

**Stream following confusion:**
Right-clicked on an ICMP packet and "Follow TCP Stream" was grayed out. Realized ICMP has no streams - only TCP/UDP support this feature. Needed to filter for TCP first (`tcp.stream eq 0`), then follow.

**I/O graph scale:**
Initial graph showed flat line at bottom because Y-axis auto-scaled to maximum (550 packets), making baseline traffic invisible. Solution: Manually set Y-axis range to 0-100 for first analysis, then 0-600 for spike investigation.

**Protocol hierarchy interpretation:**
Saw "Hypertext Transfer Protocol" at 31.1% but also "HTML Form URL Encoded" at 13.3%. Confused whether these overlapped. Realized hierarchy is nested - Form data is a subset of HTTP, shown separately for detail. Percentages don't sum to 100% when parent/child relationships exist.

**Export objects failure:**
Tried File → Export Objects → HTTP to save embedded images. Got "No objects found" error. The captured traffic was HTTP POST (form submission), not GET requests for images/CSS. Export Objects only works when files are actively transferred in capture.

## Key Takeaways

- **Display filters are investigation tools, not packet blockers:** Filters hide packets from view but don't delete them from capture. Removing filter reveals all traffic again. This lets analysts focus on specific protocols without losing context.

- **TCP streams reconstruct application behavior:** A web login split across 50 TCP segments becomes one readable POST request. Stream following is how analysts read emails, FTP transfers, or Telnet sessions from packet captures.

- **Unencrypted protocols leak everything:** HTTP, FTP, Telnet, SMTP (without STARTTLS) transmit credentials and data in plaintext. Wireshark proves this by displaying passwords directly in Packet Details pane. HTTPS/SFTP/SSH/TLS make packet captures show encrypted gibberish instead.

- **Baseline behavior detection catches attacks:** Normal web browsing: 10-50 packets/sec. Ping flood: 550 packets/sec. Port scan: Hundreds of SYN packets to sequential ports. Establishing baselines (what's normal) makes anomalies (attacks) obvious in I/O graphs.

- **Packet timestamps enable timeline reconstruction:** Every packet has microsecond-precision timestamp. In investigations, this creates exact attack timelines: "Attacker logged in at 00:40:12.385, began reconnaissance at 00:42:03.891, exfiltrated data at 00:55:47.102."

- **Protocol hierarchy shows network usage patterns:** 80% HTTP = web browsing. 60% SMB = file sharing. 40% DNS + no HTTP = possible DNS tunneling. Statistics menu provides this breakdown without manual packet counting.

- **Capture files preserve evidence:** PCAPs store everything - packets, timestamps, checksums. Hashing with SHA-256 and storing on WORM media makes them legally admissible. This is why organizations keep 90-day PCAP archives of gateway traffic.

- **User-Agent strings identify systems:** Every HTTP request includes browser/OS details. "Windows NT 10.0" vs "X11; Linux" reveals attacker's platform. Outdated browsers (Firefox 68 in 2025) suggest compromised legacy system.

## Disclaimer

This lab was performed in a controlled Kali Linux environment for educational purposes as part of the ICDFA Kali Linux Tools and System Security course. All traffic captures analyzed the intentionally vulnerable Mutillidae web application in an isolated lab network. No unauthorized network monitoring or real-world traffic interception was conducted. Wireshark was used to demonstrate network analysis techniques and security vulnerabilities, not to facilitate attacks.