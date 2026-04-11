# Packet Analysis with Wireshark: Capture, Filter, and Analyze Network Traffic

## 1. Engagement Overview

This lab analyzed network traffic between a Kali Linux attack machine and an OWASP Broken Web Applications VM to understand TCP/IP protocol behavior at each layer of the network stack. Traffic was captured using Wireshark during various network operations (web browsing, SSH connections, ping tests, port scans) and dissected to examine protocol headers, connection establishment sequences, error handling mechanisms, and layer interactions. The goal was to build practical knowledge of how protocols function in real network communication and develop troubleshooting skills by deliberately introducing network misconfigurations.

## 2. Objectives

- Understand the relationship between TCP/IP and OSI network models
- Analyze TCP connection establishment and teardown (three-way handshake, FIN/RST)
- Examine IP packet headers and routing behavior on local subnets
- Compare application layer protocols (HTTP vs SSH) for security implications
- Study TCP reliability mechanisms (sequence numbers, acknowledgments, retransmissions)
- Understand ICMP's role in network diagnostics (ping, traceroute)
- Compare TCP and UDP protocol structures and use cases
- Perform OS fingerprinting through packet analysis
- Analyze ARP address resolution at the link layer
- Troubleshoot network connectivity issues using packet inspection

## 3. Scope

**In-scope protocols and layers:**
- Application Layer: HTTP, SSH, DNS
- Transport Layer: TCP, UDP
- Network Layer: IP, ICMP
- Link Layer: Ethernet, ARP

**In-scope systems:**
- Attacker Machine: Kali Linux (192.168.92.4)
- Target Machine: OWASP-BWA VM (192.168.92.3)
- Network Segment: 192.168.92.0/24 (local subnet, no external routing)

**Out-of-scope:**
- Encrypted traffic decryption (SSH payload remains encrypted)
- Wireless protocols (802.11)
- IPv6 traffic analysis
- Network devices beyond the two test VMs

## 4. Methodology

### Exercise 1: TCP/IP vs OSI Model Mapping

I started by understanding the conceptual difference between the TCP/IP model (practical implementation) and the OSI model (theoretical framework). The OSI model defines seven distinct layers with specific responsibilities at each level, while TCP/IP consolidates these into four broader layers for efficiency.

**Layer mapping:**

| TCP/IP Layer | Corresponding OSI Layers |
|--------------|--------------------------|
| Application | Application, Presentation, Session |
| Transport | Transport |
| Internet | Network |
| Link | Data Link, Physical |

**Why this matters:** When analyzing packets, Wireshark displays information using OSI terminology (Layer 2, Layer 3, Layer 4), but real network stacks implement TCP/IP. Understanding the mapping helps translate between theory and practice. For example, HTTP headers appear at the Application layer in both models, but TCP/IP treats all application-level protocols (HTTP, SSH, DNS) as a single layer, whereas OSI separates presentation (data formatting) from session management (connection state).

### Exercise 2: TCP Three-Way Handshake Analysis

I captured traffic while browsing to the OWASP-BWA VM to observe the TCP connection establishment sequence.

![Wireshark capture showing SYN, SYN-ACK, ACK packets](screenshots/tcp_handshake.png)



**Handshake sequence:**

1. **SYN (Synchronize):** My Kali machine (192.168.92.4) sends a TCP packet with the SYN flag set to port 80 on the OWASP VM (192.168.92.3), requesting a connection. This packet includes an initial sequence number that will be used to track data bytes.

2. **SYN-ACK (Synchronize-Acknowledgment):** The OWASP VM responds with both SYN and ACK flags set. The SYN portion provides the server's own initial sequence number, while the ACK portion confirms receipt of the client's SYN by acknowledging the client's sequence number + 1.

3. **ACK (Acknowledgment):** My Kali machine sends a final ACK to confirm receipt of the server's SYN-ACK. At this point, the connection is fully established and data transfer can begin.

![tcpdump output showing the three-way handshake with sequence numbers](screenshots/tcpdump_handshake_sequence.png)



**TCP flags observed:**
- **SYN (0x02):** Connection initiation
- **ACK (0x10):** Acknowledgment of received data
- **FIN (0x01):** Graceful connection termination
- **RST (0x04):** Abrupt connection reset (seen when connecting to closed ports)

**Reliability mechanisms:** TCP uses sequence numbers to label every byte of data. If my machine sends bytes 1000-1500 and doesn't receive an ACK within the retransmission timeout, it assumes those bytes were lost and resends them automatically. This acknowledgment system ensures that corrupted or dropped packets are detected and recovered without application-level intervention.

### Exercise 3: IP Packet Header Examination

I examined IP packet headers to understand how routing decisions are made at the network layer.

![IP packet header showing Source IP, Destination IP, TTL](screenshots/ip_packet_header.png)



**Key IP header fields:**
- **Source IP:** 192.168.92.4 (my Kali machine)
- **Destination IP:** 192.168.92.3 (OWASP VM)
- **TTL (Time to Live):** 64

**TTL significance:** TTL is a hop counter that decrements by 1 at each router. If TTL reaches 0, the packet is discarded and an ICMP "Time Exceeded" message is sent back to the source. This prevents packets from looping infinitely if routing tables are misconfigured. Linux systems typically start with TTL=64, Windows with TTL=128. This difference is used for OS fingerprinting.

**Routing analysis:** Both machines are on the same subnet (192.168.92.0/24), so routing is direct with zero hops. My system performs an ARP lookup to find the VM's MAC address, then delivers the packet directly over the Ethernet link without involving any routers. If the destination were on a different subnet, my system would send the packet to the default gateway instead.

### Exercise 4: Application Layer Protocol Comparison

#### HTTP Traffic Analysis

I browsed to the WebGoat application on the OWASP VM and captured the HTTP request/response exchange.



![HTTP request showing GET method, Host, User-Agent, Cookies](screenshots/http_request_headers.png)



**HTTP request details:**
- **Method:** GET /WebGoat/attack
- **Host:** 192.168.92.3
- **User-Agent:** Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
- **Cookies:** JSESSIONID, PHPSESSID (session identifiers)



![HTTP response showing 401 Unauthorized status](screenshots/http_response_401.png)



**HTTP response details:**
- **Status Code:** 401 Unauthorized
- **Server:** Apache-Coyote (reveals server software)
- **WWW-Authenticate:** Basic realm="WebGoat Application" (requires credentials)
- **Content-Type:** text/html; charset=UTF-8
- **Content-Encoding:** gzip (compressed response)

**Security implications:** HTTP transmits everything in plaintext. The User-Agent string reveals my operating system and browser version. Session cookies are visible to anyone sniffing the network. The 401 status code means authentication is required, but credentials will also be sent in plaintext (Base64 encoding is not encryption). Any network eavesdropper can capture the Authorization header and extract the username:password pair.

#### SSH Traffic Analysis

I established an SSH connection to the OWASP VM and examined the encrypted traffic.



![SSH Key Exchange Init showing cipher negotiation](screenshots/ssh_key_exchange.png)



**SSH handshake observations:**
- **Protocol Version:** SSH-2
- **Key Exchange Method:** diffie-hellman-group-exchange-sha256
- **Packet Length:** 780 bytes
- **Message Code:** 20 (Key Exchange Init)

After the initial key exchange negotiation, all subsequent traffic appears as encrypted payload in Wireshark. The packet details show random hex data with no discernible structure.



![SSH encrypted payload showing unreadable hex data](screenshots/ssh_encrypted_payload.png)



**Why SSH payload is unreadable:** SSH uses the negotiated cipher to encrypt everything after the key exchange completes. Wireshark can dissect the handshake because those packets follow a known protocol structure, but once encryption activates, the payload becomes opaque. Even if an attacker captures the entire session, they cannot extract login credentials or command data without breaking the encryption. This is the critical difference between HTTP and SSH - HTTP leaks sensitive data to passive eavesdroppers, SSH protects it.

**Application layer's role:** The Application Layer translates user actions (clicking a link, typing a command) into protocol-specific messages. When I browse via HTTP, this layer formats GET requests with proper headers and cookies. When I connect via SSH, it handles authentication challenges and terminal emulation. Because the OWASP VM is intentionally vulnerable, application-layer flaws (broken authentication, missing input validation) are visible in HTTP traffic but protected in SSH traffic.

### Exercise 5: TCP Error Handling and Retransmission

To observe TCP's reliability mechanisms, I simulated packet loss by toggling the network interface during an active connection.

**Expected behavior:** When packets are dropped or delayed, the sender waits for an ACK that never arrives. After the retransmission timeout expires, the sender assumes the packet was lost and resends it automatically.

**How retransmissions work:** TCP uses Positive Acknowledgment with Retransmission (PAR). Every outgoing packet starts a timer. If the receiver's ACK doesn't arrive before the timer expires, the packet is retransmitted. This continues with exponentially increasing delays until the connection times out entirely.

**Sequence number ordering:** Each byte is assigned a unique sequence number. If packets arrive out of order (packet 1000-1500, then 2000-2500, then 1500-2000), the receiver uses sequence numbers to reassemble them correctly. Gaps in the sequence trigger selective acknowledgments (SACKs) that specifically request the missing ranges.



![Wireshark showing retransmitted packets with duplicate sequence numbers](screenshots/tcp_retransmission.png)



**Observation:** When I disabled the network interface, pending packets failed to receive ACKs. Wireshark flagged these as "[TCP Retransmission]" because the same sequence numbers appeared multiple times. The connection eventually timed out after several retry attempts, demonstrating that TCP refuses to move forward until all data is confirmed as received.

### Exercise 6: ICMP and Ping Analysis

I used ping to send ICMP echo requests to the OWASP VM and analyzed the response structure.



![ICMP packet showing Type 8 (Echo Request), Code 0, Checksum](screenshots/icmp_echo_request.png)



**ICMP packet fields:**
- **Type:** 8 (Echo Request)
- **Code:** 0
- **Checksum:** 0x56cf [correct]
- **Identifier:** 0x0001
- **Sequence Number:** 0x0100

**ICMP's diagnostic role:** Ping uses ICMP to verify basic connectivity. If the target responds with Type 0 (Echo Reply), the path is clear. If no response arrives, the issue could be: target is down, firewall is blocking ICMP, or a router along the path is dropping packets. Tools like traceroute exploit ICMP to reveal the exact hop where failures occur.

**TTL in ICMP context:** Traceroute sends packets with incrementing TTL values (TTL=1, TTL=2, TTL=3...). Each router decrements TTL and, when it hits zero, sends back an ICMP "Time Exceeded" message that reveals the router's IP. By deliberately causing TTL expiration at each hop, traceroute maps the entire path from source to destination.

![Wireshark showing ICMP Echo Request and Echo Reply pairs](screenshots/icmp_echo_reply.png)



### Exercise 7: UDP vs TCP Comparison

I captured both TCP and UDP traffic to compare protocol structure and behavior.



![Wireshark showing UDP packet with minimal header](screenshots/udp_packet_structure.png)



**UDP packet structure:**
- Source Port: (varies)
- Destination Port: (varies)
- Length: (payload size)
- Checksum: (error detection only)

**Key differences from TCP:**

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Requires handshake | Connectionless |
| Reliability | Acknowledgments, retransmissions | None - fire and forget |
| Ordering | Sequence numbers guarantee order | Packets may arrive out of order |
| Header Size | 20-60 bytes (larger) | 8 bytes (minimal) |
| Speed | Slower due to overhead | Faster, lower latency |

**When UDP is preferred:** Live video streaming tolerates occasional dropped frames better than buffering delays. Online games need low latency more than perfect accuracy (missing one position update is acceptable). DNS queries are single request/response pairs where retransmission logic can be handled at the application level if needed.

**UDP's management strategy:** UDP simply hands packets to IP and moves on. It has no state tracking, no connection establishment, no feedback loop. If reliability is required, the application must implement it. For example, TFTP (Trivial File Transfer Protocol) runs over UDP but adds its own acknowledgment system at the application layer.

### Exercise 8: OS Detection via Nmap

I ran an Nmap scan against the OWASP VM and examined the packet characteristics that reveal operating system identity.

![Nmap scan results showing OS detection](screenshots/nmap_os_detection.png)



**OS fingerprinting indicators:**

**TTL value:** The OWASP VM responds with TTL=64, which is the default for Linux systems. Windows typically uses TTL=128. This single field immediately narrows the OS family.

**TCP Window Size:** The initial SYN-ACK includes window size values (Win=1024, Win=5840 observed in capture). Different operating systems use different default window sizes and scaling factors. Linux 2.6.x kernels have a distinctive window size progression that differs from Windows 7/10 or BSD systems.



![Wireshark showing TCP window size in SYN-ACK packet](screenshots/tcp_window_size.png)



**TCP Options:** The order and content of TCP options (MSS, SACK permitted, timestamps, window scale) form a unique fingerprint. Linux arranges these options differently than Windows. Nmap maintains a database of known option patterns for hundreds of OS versions.

**Response to malformed packets:** Nmap sends unusual packets (invalid flag combinations, odd TCP options) and observes how the target responds. Some OS stacks silently drop malformed packets, others send RST, others send ICMP errors. These responses reveal the TCP/IP stack implementation.

**Why OS detection matters:** Knowing the target OS allows attackers to search for OS-specific exploits (kernel vulnerabilities, default configurations). It also helps administrators validate that systems are running expected software versions and detect unauthorized devices on the network.

### Exercise 9: ARP Address Resolution

I examined ARP traffic to understand how IP addresses are mapped to MAC addresses on the local network.



![ARP request broadcast asking "Who has 192.168.92.3?"](screenshots/arp_request.png)

**ARP packet details:**
- **Sender MAC:** c8:f7:33:67:d6:17 (my Kali machine)
- **Sender IP:** 192.168.92.4
- **Target MAC:** 00:00:00:00:00:00 (unknown - this is what we're asking for)
- **Target IP:** 192.168.92.3 (OWASP VM)

**ARP operation:** My system broadcasts an ARP request to the entire local segment (destination MAC: ff:ff:ff:ff:ff:ff) asking "Who has 192.168.92.3? Tell 192.168.92.4." Every device on the segment receives this broadcast, but only the machine with IP 192.168.92.3 responds.



![ARP reply providing MAC address](screenshots/arp_reply.png)



**ARP reply:** The OWASP VM sends a unicast ARP reply directly to my MAC address (c8:f7:33:67:d6:17) saying "192.168.92.3 is at [target MAC address]." My system stores this mapping in its ARP cache for future use.

**ARP's role in communication:** Without ARP, my system would know the VM's IP address (Layer 3) but couldn't construct the Ethernet frame (Layer 2) needed to actually deliver the packet. ARP bridges the gap between logical addressing (IP) and physical addressing (MAC). Once resolved, the Ethernet frame has the correct destination MAC, allowing the switch to forward it to the right physical port.

**ARP cache:** After resolution, the mapping is cached (typically 60-300 seconds) to avoid re-broadcasting for every packet. I can view the cache with `arp -n` to see which IP addresses are currently resolved.

### Exercise 10: Network Troubleshooting Simulation

I deliberately misconfigured the Kali machine's subnet mask to simulate a common networking error, then used packet analysis to diagnose and resolve the issue.

**Simulated issue:** Changed subnet mask from /24 (255.255.255.0) to /30 (255.255.255.252).

**Impact:** With a /30 mask, only 4 IP addresses fit in the subnet (.0, .1, .2, .3). My Kali machine at 192.168.92.4 now believes the OWASP VM at 192.168.92.3 is on a different network, even though they're physically connected to the same switch. The routing table reflects this - 192.168.92.3 is no longer in the "directly connected" route.

**Diagnosis process:**

**Layer 3 (Network Layer):** Ran `route -n` to check routing table. The destination 192.168.92.3 no longer matched the local network range, so the system tried to route through a gateway that doesn't exist.

```
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.92.0    0.0.0.0         255.255.255.252 U     0      0        0 eth0
```

**Layer 2 (Link Layer):** Ran `arp -n` to check ARP cache. No MAC address was resolved for 192.168.92.3 because the system never attempted local ARP resolution - it thought the VM was remote.

```
Address          HWtype  HWaddress           Flags Mask            Iface
192.168.92.1     ether   (gateway MAC)       C                     eth0
```

**Wireshark observation:** No ARP requests were broadcast for 192.168.92.3. Instead, the system sent packets to the default gateway (which didn't exist), resulting in "Destination Host Unreachable" ICMP errors.

![Wireshark showing failed ARP resolution due to subnet mask error](screenshots/subnet_mask_troubleshooting.png)



**Resolution:** Corrected the subnet mask back to /24 (255.255.255.0). Immediately after applying the change:
- Routing table updated to show 192.168.92.0/24 as directly connected
- ARP request broadcast for 192.168.92.3
- ARP reply received with VM's MAC address
- Connectivity restored

**Lesson:** Subnet masks define what "local" means. Even if two machines are on the same physical wire, incorrect subnet configuration makes them believe they're on different networks. The routing layer makes decisions before the link layer ever attempts communication.

## 5. Protocol Analysis Summary

| Protocol | Layer | Key Observation | Security Implication |
|----------|-------|-----------------|---------------------|
| HTTP | Application | Plaintext transmission, credentials visible | Session hijacking, credential theft via sniffing |
| SSH | Application | Encrypted payload after key exchange | Confidentiality protected from passive eavesdropping |
| TCP | Transport | Three-way handshake, sequence numbers, retransmissions | Reliable delivery, but SYN flood DoS possible |
| UDP | Transport | No connection state, no reliability guarantees | Faster but application must handle packet loss |
| IP | Network | TTL prevents loops, routing based on destination IP | OS fingerprinting via TTL values |
| ICMP | Network | Echo request/reply for connectivity testing | Network mapping, traceroute reveals topology |
| ARP | Link | Broadcasts requests, caches responses | ARP spoofing enables MITM attacks |

## 6. Detailed Findings

### Finding: HTTP Transmits Sensitive Data in Plaintext

**Severity:** High (in production environments)

**Affected Protocol:** HTTP (Application Layer)

**Description**

HTTP sends all request and response data without encryption. During analysis of traffic to the WebGoat application, the following sensitive information was visible in Wireshark:

- User-Agent string revealing OS and browser version
- Session cookies (JSESSIONID, PHPSESSID) that maintain authenticated sessions
- HTTP 401 Unauthorized response indicating authentication is required
- Server software version (Apache-Coyote)



![HTTP request showing plaintext cookies and headers](screenshots/http_plaintext_exposure.png)



**Proof of Observation**

I captured HTTP traffic while browsing to http://192.168.92.3/WebGoat/attack. The request included:

```
Cookie: JSESSIONID=...; PHPSESSID=...
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
```

The response header revealed:
```
Server: Apache-Coyote
WWW-Authenticate: Basic realm="WebGoat Application"
```

All of this information is immediately visible to anyone with network access.

**Impact**

An attacker on the same network segment can:
- Steal session cookies and impersonate the authenticated user (session hijacking)
- Capture credentials if Basic Authentication is used (Base64 is easily decoded)
- Map the server software to known vulnerabilities (Apache-Coyote version-specific exploits)
- Profile the client's browser and OS for targeted client-side attacks

**Remediation**

Use HTTPS instead of HTTP. TLS encryption protects:
- Request/response headers from inspection
- Cookies from theft
- Credentials from capture
- Data integrity (prevents tampering)

For internal applications, implement TLS with self-signed certificates or internal CA. For public applications, use certificates from trusted CAs.

### Finding: SSH Protects Payload After Key Exchange

**Severity:** Informational (demonstrates proper encryption)

**Affected Protocol:** SSH (Application Layer)

**Description**

SSH successfully negotiates encryption and protects all post-handshake traffic. The initial key exchange is visible in plaintext (necessary for cipher negotiation), but subsequent traffic appears as encrypted payload.



![SSH Key Exchange showing cipher selection](screenshots/ssh_key_exchange_detail.png)



**Observation**

Wireshark dissects the Key Exchange Init packet (Message Code 20) showing:
- Selected cipher: diffie-hellman-group-exchange-sha256
- Protocol version: SSH-2
- Packet length: 780 bytes

After this handshake completes, all further packets show only:
- SSH Protocol header
- Encrypted payload (random hex data)
- No command data, no login credentials, no file contents



![SSH encrypted payload appearing as random data](screenshots/ssh_encrypted_payload_detail.png)



**Impact**

Positive: Passive network monitoring cannot extract:
- Login credentials
- Commands executed during the session
- File contents transferred via SCP/SFTP
- Any application data

This demonstrates proper encryption implementation. Even with complete packet capture, the session remains confidential.

**Best Practice Validation**

SSH correctly implements:
- Strong cipher negotiation (no fallback to weak algorithms)
- Encrypted payload after key exchange
- Protection against passive eavesdropping

This is the security model all network protocols should follow.

### Finding: TCP Reliability Mechanisms Prevent Data Loss

**Severity:** Informational (demonstrates protocol behavior)

**Affected Protocol:** TCP (Transport Layer)

**Description**

TCP's acknowledgment and retransmission system successfully recovers from packet loss. When I simulated network disruption by toggling the interface, TCP automatically retransmitted unacknowledged packets.

**Observation**

During the network disruption test:
1. Packets were sent but no ACK received (interface down)
2. Retransmission timer expired
3. Same sequence numbers were retransmitted (flagged as "[TCP Retransmission]" in Wireshark)
4. Connection eventually timed out after multiple retries



![Wireshark showing retransmitted packets](screenshots/tcp_retransmission_detail.png)



**How Reliability Works**

- **Sequence Numbers:** Every byte is assigned a unique number. If bytes 1000-1500 are sent, the ACK confirms receipt by acknowledging sequence 1501 (next expected byte).
- **Retransmission Timer:** Each packet starts a timer. If ACK doesn't arrive before timeout, the packet is assumed lost and resent.
- **Ordered Delivery:** If packets arrive as 1000-1500, 2000-2500, 1500-2000, TCP uses sequence numbers to reorder them correctly before delivering to the application.

**Impact**

This reliability ensures:
- File downloads don't become corrupted
- Database transactions maintain data integrity
- Web pages load completely even on unreliable networks

However, retransmissions add latency. Applications requiring low latency over perfect accuracy (VoIP, gaming, live video) use UDP instead.

### Finding: OS Detection Possible Through Packet Analysis

**Severity:** Informational (reconnaissance technique)

**Affected Protocols:** TCP, IP, ICMP (multiple layers)

**Description**

Operating systems can be identified through passive and active packet analysis. Different OS implementations have unique TCP/IP stack behaviors that create fingerprints.



![Nmap OS detection results](screenshots/nmap_os_detection_results.png)



**Detection Indicators Observed:**

**TTL Value:** OWASP VM consistently responds with TTL=64 (Linux default). Windows systems use TTL=128. This single field immediately identifies the OS family.

**TCP Window Size:** Initial SYN-ACK packets included window sizes of 1024 and 5840. These values, combined with window scaling options, are characteristic of Linux 2.6.x kernels.

**TCP Options Order:** The sequence of options (MSS, SACK, Timestamps, Window Scale) is arranged differently in Linux vs Windows vs BSD. Nmap compares observed patterns against a database of known fingerprints.

**Response to Probes:** When Nmap sent packets to closed ports, the VM responded with [RST, ACK] flags. The exact timing and flag combinations reveal the TCP/IP stack implementation.

**Impact**

OS detection enables:
- Attackers to search for OS-specific vulnerabilities
- Network administrators to validate expected systems
- Inventory management (detecting unauthorized devices)
- Targeted exploit selection based on confirmed OS version

**Reconnaissance Value**

After fingerprinting the OWASP VM as Linux 2.6.x, an attacker can:
- Search exploit databases for kernel vulnerabilities
- Assume default file paths (/etc/passwd, /var/www)
- Tailor payloads to Linux architecture
- Skip Windows-specific attack vectors

This is why OS detection is Step 2 in most penetration test methodologies (after network discovery, before vulnerability scanning).

### Finding: ARP Has No Authentication - Spoofing Possible

**Severity:** Medium (enables MITM attacks)

**Affected Protocol:** ARP (Link Layer)

**Description**

ARP accepts responses without verification. Any device can claim ownership of an IP address, and the requesting system will believe it. This enables ARP spoofing attacks where an attacker intercepts traffic intended for another machine.



![ARP request and reply exchange](screenshots/arp_request_reply_detail.png)



**How ARP Spoofing Works:**

1. Legitimate flow: My machine broadcasts "Who has 192.168.92.3?" and the VM replies "I am 192.168.92.3 at [legitimate MAC]"
2. Attack scenario: Attacker sends unsolicited ARP reply "192.168.92.3 is at [attacker's MAC]"
3. My machine updates ARP cache with poisoned entry
4. All packets intended for 192.168.92.3 are now sent to attacker's MAC
5. Attacker can intercept, read, modify, then forward to legitimate destination

**Impact**

ARP spoofing enables:
- Man-in-the-middle attacks (attacker relays traffic between victim and target)
- Session hijacking (steal cookies from HTTP traffic)
- Credential capture (intercept login attempts)
- Traffic analysis (monitor all communication)

**Why This Works**

ARP has no authentication mechanism. The protocol was designed for trusted local networks and assumes all participants are honest. Modern switched networks limit broadcast domains, but ARP spoofing still works within a VLAN.

**Mitigation**

- Use static ARP entries for critical systems (manual management overhead)
- Enable Dynamic ARP Inspection (DAI) on switches (validates ARP packets against DHCP bindings)
- Implement 802.1X port authentication (prevents unauthorized devices from joining network)
- Use HTTPS/SSH to protect data even if ARP is compromised

This finding demonstrates why encryption at higher layers (TLS, SSH) is necessary - link-layer security cannot be guaranteed on shared networks.

## 7. Tools Used

- Wireshark (GUI packet analyzer)
- tcpdump (command-line packet capture)
- Nmap (OS detection and port scanning)
- ping (ICMP connectivity testing)
- traceroute (path discovery)
- netstat (connection status)
- route (routing table inspection)
- arp (ARP cache management)

## 8. Challenges Encountered

**Challenge: Wireshark display filter syntax confusion**

During initial packet capture, I attempted to filter HTTP traffic using `protocol == HTTP`, which returned no results. Wireshark uses `http` (lowercase, no comparison operator) as the display filter. This is different from capture filters (BPF syntax) which use `port 80`. I resolved this by consulting Wireshark's filter reference and learning that display filters use protocol names directly (http, tcp, icmp) while capture filters use port numbers and protocol keywords.

**Challenge: SSH payload appears as gibberish**

When analyzing SSH traffic, I initially thought Wireshark was displaying the data incorrectly because the payload showed as random hex with no readable text. I later understood this is correct behavior - SSH encryption is working as designed. The key insight was that only the initial key exchange (Message Code 20) is visible in plaintext, and everything after that is intentionally encrypted. This taught me to differentiate between "Wireshark isn't working" and "encryption is working".

**Challenge: Subnet mask misconfiguration diagnosis**

After deliberately setting the wrong subnet mask (/30 instead of /24), I initially looked for the problem at the application layer (is the web server running?). I then worked down through the layers: Transport (is TCP connecting?), Network (is routing correct?), and finally Link (is ARP resolving?). The `route -n` command showed the destination wasn't in the local route, and `arp -n` showed no MAC address for the target. This reinforced the importance of bottom-up troubleshooting - if Layer 2/3 are broken, Layer 7 problems are irrelevant.

**Challenge: Understanding why UDP has no retransmission**

I initially questioned why UDP doesn't just add basic reliability features like acknowledgments. The answer became clear when analyzing live video streaming - retransmitting a lost video frame after the next 10 frames have already arrived provides no value. The frame is displayed for 1/30th of a second, and by the time the retransmission arrives, the viewer has moved on. UDP's "fire and forget" approach makes sense for time-sensitive data where old data is worthless.

## 9. Key Takeaways

**Layer boundaries matter for security.** HTTP operates at Layer 7, but an attacker at Layer 2 (ARP spoofing) can intercept it. Understanding which layer protects what is critical. Encryption must happen at the right layer - TLS at Layer 6 protects against Layer 2/3 attacks, but ARP spoofing still works until you implement link-layer security (802.1X, DAI).

**TCP's reliability is expensive.** Every packet requires an ACK, sequence numbers consume bandwidth, and retransmissions add latency. This overhead is acceptable for file transfers and web browsing but unacceptable for VoIP and gaming. Choosing between TCP and UDP requires understanding the application's tolerance for packet loss vs latency.

**TTL reveals more than just hop count.** Beyond preventing routing loops, TTL exposes OS identity. A single response packet with TTL=64 immediately identifies Linux, while TTL=128 indicates Windows. This passive fingerprinting works without sending any active probes to the target.

**Encryption doesn't hide everything.** SSH protects payload content, but the key exchange negotiation is visible. An observer knows: that SSH is being used, which cipher was negotiated, the packet sizes and timing, and both endpoints' IPs. Traffic analysis remains possible even when content is encrypted.

**ARP's lack of authentication is a fundamental weakness.** Any device can claim ownership of any IP address on the local segment. This is why enterprise networks implement port security, DHCP snooping, and Dynamic ARP Inspection. The assumption that "everyone on the LAN is trusted" is incorrect.

**Wireshark reveals the gap between theory and practice.** Textbooks explain the OSI model cleanly, but real packets show TCP options that aren't in the standard, window scaling negotiations that vary by OS, and retransmission algorithms that differ between implementations. Hands-on packet analysis teaches how networks actually work, not how they theoretically should work.

**Troubleshooting requires layer-by-layer analysis.** When connectivity fails, start at Layer 1 (is the cable plugged in?) and work up. Checking application logs before verifying IP routing wastes time. The subnet mask misconfiguration exercise demonstrated this - the routing table (Layer 3) showed the problem immediately, eliminating the need to debug HTTP (Layer 7).

## 10. Disclaimer

This lab was performed on isolated virtual machines (Kali Linux and OWASP-BWA) in a controlled local network environment for educational purposes only. No production systems were analyzed. All packet captures were performed on systems under my direct control. Traffic analysis techniques demonstrated in this lab should only be applied to networks where you have explicit authorization. Unauthorized network sniffing or protocol analysis may violate laws and organizational policies.