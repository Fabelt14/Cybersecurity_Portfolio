# Basic Network Configuration: Setting Up and Troubleshooting

## Overview

This lab explored fundamental networking concepts by testing connectivity and analyzing traffic between a Kali Linux machine and an OWASP Broken Web Applications VM. The exercises mapped theoretical OSI and TCP/IP layer concepts to practical command-line tools (ping, traceroute, netstat, arp, nmap) and packet analysis with Wireshark. The goal was to understand how different network layers interact during real communication and how to troubleshoot connectivity issues.

## Objectives

- Map OSI model layers to their TCP/IP equivalents
- Use ICMP ping to verify host reachability
- Trace network paths with traceroute to identify routing failures
- Analyze active TCP connections and connection states
- Compare TCP and UDP protocol behavior through scanning
- Resolve IP addresses to MAC addresses using ARP
- Capture and analyze network traffic at the packet level with Wireshark

## Lab Environment

- **Attacker Machine:** Kali Linux (192.168.92.4)
- **Target Machine:** OWASP Broken Web Applications VM (192.168.92.3)
- **Network Configuration:** Both VMs on same NAT network (192.168.92.0/24)
- **Network Interface:** eth1

## Tools Used

- ping
- traceroute
- netstat
- nmap
- arp
- Wireshark

## Methodology

### Step 1: OSI and TCP/IP Model Mapping

Before running any network commands, I documented the seven OSI layers and their functions, then mapped them to the simpler four-layer TCP/IP model.

**OSI Model Layers:**

**Layer 7 - Application:** Provides network services directly to user applications like web browsers, email clients, and file transfer tools. Protocols include HTTP, FTP, and SMTP.

**Layer 6 - Presentation:** Translates data between application format and network format. Handles encryption (SSL/TLS), compression, and encoding (JPEG, ASCII).

**Layer 5 - Session:** Manages connections between applications. Establishes, maintains, and terminates sessions between devices.

**Layer 4 - Transport:** Ensures reliable data transfer with flow control and error correction. TCP provides guaranteed delivery with acknowledgments, UDP provides fast unreliable delivery.

**Layer 3 - Network:** Handles logical addressing (IP addresses) and routing. Determines the best path for data to travel across networks.

**Layer 2 - Data Link:** Manages physical addressing (MAC addresses) and error detection on the local network segment. Operates on switches.

**Layer 1 - Physical:** Transmits raw bits over physical media (copper cables, fiber optic, radio waves). Operates on hubs and cables.

**TCP/IP Model Mapping:**

The TCP/IP model consolidates the OSI layers into four practical layers:

- **Application Layer** combines OSI Layers 7, 6, and 5 (Application, Presentation, Session)
- **Transport Layer** matches OSI Layer 4 (Transport)
- **Internet Layer** matches OSI Layer 3 (Network)
- **Network Access Layer** combines OSI Layers 2 and 1 (Data Link, Physical)

**Why the mapping matters:** When troubleshooting network issues, you need to know which layer is failing. If ping works but HTTP does not, the problem is at Layer 7 (application), not Layer 3 (network). Understanding the layer structure helps isolate failures quickly.

### Step 2: ICMP Connectivity Test with Ping

I used ping to verify the OWASP VM is reachable on the network. Ping sends ICMP Echo Request packets and waits for ICMP Echo Reply packets.

```bash
ping -c 5 192.168.92.3
```



![Ping results showing 5 successful replies from OWASP VM](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/12_01%20Ping%20results%20showing%205%20successful%20replies%20from%20OWASP%20VM.jpg)

**Result:** All 5 packets transmitted, all 5 received, 0% packet loss. Round-trip times ranged from 0.409ms to 0.677ms with an average of 0.478ms.

**What this proves:** The target VM (192.168.92.3) is online and responsive at the Network Layer (Layer 3). The network path between my Kali machine and the OWASP VM is functional.

**Which OSI layer does ping operate at?**

Ping operates at Layer 3 (Network Layer) of the OSI model. Although you type the ping command in a shell (which is an application), the actual work is performed by the Internet Control Message Protocol (ICMP). ICMP does not use port numbers, which are a Layer 4 (Transport Layer) feature. ICMP is responsible for logical addressing and connectivity testing between IP addresses, which are strictly Layer 3 functions.

**Why this distinction matters:** Ping bypasses firewalls that only block TCP/UDP ports because ICMP does not use ports. A server can respond to ping while blocking all TCP services. Conversely, a server can be fully functional at Layer 4 (accepting TCP connections) but have ICMP disabled, causing ping to fail even though the server works fine.

### Step 3: Path Tracing with Traceroute

Traceroute maps the network path by sending packets with incrementing TTL (Time To Live) values. Each router along the path decrements TTL by 1 and sends back an ICMP Time Exceeded message when TTL reaches 0. This reveals every router hop between source and destination.

```bash
traceroute 192.168.92.3
```



![Traceroute showing single hop to OWASP VM](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/12_02%20Traceroute%20showing%20single%20hop%20to%20OWASP%20VM.jpg)



**Result:** The trace completed in 1 hop with response times of 2.862ms and 2.111ms. The single hop is 192.168.92.3, which is the destination itself.

**Why only 1 hop?** Both VMs are on the same NAT network (192.168.92.0/24), so packets travel directly from source to destination without passing through any intermediate routers. In a real internet scenario, traceroute would show 10-20 hops as packets traverse multiple ISP routers.

**What traceroute reveals in network troubleshooting:**

Traceroute pinpoints exactly where connectivity breaks. If you can reach hop 5 but hop 6 shows asterisks (* * *), hop 6 is either down or blocking ICMP. This tells you the failure point is at that specific router, not your machine or the destination.

- **Routing loops:** If traceroute shows the same two IP addresses repeating indefinitely (hop 8: 10.0.1.1, hop 9: 10.0.2.1, hop 10: 10.0.1.1, hop 11: 10.0.2.1), packets are bouncing between two routers that have incorrect routing tables. This is called a routing loop and causes packets to circulate until TTL expires.

- **Asymmetric routing:** Sometimes the path to a destination is different from the return path. Traceroute only shows the forward path. If responses come back through different routers, troubleshooting becomes more complex.

### Step 4: Active Connection Analysis with Netstat

To see which connections are currently open between my machine and the OWASP VM, I used netstat to list all TCP connections involving the target IP.

```bash
netstat -an | grep 192.168.92.3
```



![Netstat showing TCP connections in ESTABLISHED and TIME_WAIT states](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/12_03%20Netstat%20showing%20TCP%20connections%20in%20ESTABLISHED%20and%20TIME_WAIT%20states.jpg)



**Result:** Three TCP connections appeared:

**Connection 1:** 192.168.92.4:47826 → 192.168.92.3:80 (TIME_WAIT)  
**Connection 2:** 192.168.92.4:35618 → 192.168.92.3:80 (ESTABLISHED)  
**Connection 3:** 192.168.92.4:35624 → 192.168.92.3:80 (ESTABLISHED)

**Source IP:** 192.168.92.4 (my Kali machine)  
**Destination IP:** 192.168.92.3 (OWASP VM)  
**Source ports:** 47826, 35618, 35624 (ephemeral ports randomly assigned by the OS)  
**Destination port:** 80 (HTTP web server)

**Connection states explained:**

- **ESTABLISHED:** The connection is active and transferring data. Both machines have completed the three-way handshake and are exchanging HTTP traffic. These are my active browser tabs communicating with the web server.

- **TIME_WAIT:** The connection recently closed but is being kept in a temporary state for 30-120 seconds. This prevents old packets from interfering with new connections using the same port numbers. After the timeout expires, the connection disappears from netstat output.

- **How the Transport Layer manages this communication:**

The Network Layer (Layer 3) finds the IP address (192.168.92.3), but the Transport Layer (Layer 4) determines which specific application receives the data. Port 80 ensures packets go to the web server software, not SSH (port 22) or another service.

Before any data flows, TCP performs a three-way handshake:

1. **SYN:** My machine sends "I want to connect" to port 80
2. **SYN-ACK:** OWASP VM replies "OK, I'm ready"
3. **ACK:** My machine confirms "Connection established"

After the handshake completes, the connection enters ESTABLISHED state and data transfer begins. TCP guarantees every packet is acknowledged and retransmits lost packets automatically.

### Step 5: TCP vs UDP Protocol Comparison

To understand the fundamental differences between TCP and UDP, I ran Nmap scans using both protocols against the same target and compared speed and reliability.

**TCP Scan:**
```bash
nmap -sT 192.168.92.3
```



![Nmap TCP scan completing in 7.99 seconds](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/12_04%20Nmap%20TCP%20scan%20completing%20in%207.99%20seconds.jpg)



**Result:** Scan completed in 7.99 seconds. Detected services: ssh (22/tcp), http (80/tcp), netbios-ssn (139/tcp), imap (143/tcp), https (443/tcp), microsoft-ds (445/tcp), commplex-link (5001/tcp), http-proxy (8080/tcp), blackice-icecap (8081/tcp).

**UDP Scan:**
```bash
sudo nmap -sU 192.168.92.3
```



![Nmap UDP scan taking 17 minutes 52 seconds](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/12_05%20Nmap%20UDP%20scan%20taking%2017%20minutes%2052%20seconds.jpg)



**Result:** Scan completed in 1024.46 seconds (17 minutes 52 seconds). Detected services: netbios-ns (137/udp), dhcp (67/udp), netbios-dgm (138/udp).

**TCP vs UDP speed comparison:**

- TCP scan: 7.99 seconds  
- UDP scan: 1024.46 seconds  
**UDP is 128x slower than TCP**

**Why UDP scans are so slow:**

TCP uses a three-way handshake, so Nmap immediately knows if a port is open (receives SYN-ACK) or closed (receives RST). The response is instant.

UDP is connectionless and does not send acknowledgments. When Nmap sends a UDP packet to a port, three things can happen:

1. **Open port:** Application responds with UDP data (fast detection)
2. **Closed port:** Target sends ICMP Port Unreachable (fast detection)
3. **Filtered port:** No response at all (Nmap must wait for timeout)

Most UDP ports are filtered by firewalls and never respond. Nmap must wait several seconds per port before marking it as "open|filtered". Scanning 1000 UDP ports with 5-second timeouts takes over an hour.

**Reliability comparison:**

- **TCP is reliable:** Three-way handshake confirms both sides are ready. Every packet is numbered and acknowledged. Lost packets are automatically retransmitted. Data arrives in order.

- **UDP is unreliable:** No handshake, no acknowledgments, no retransmission. Packets can be lost, duplicated, or arrive out of order. The application must handle all error correction.

**When to use each protocol:**

- **TCP use cases:** Web browsing (HTTP/HTTPS), email (SMTP/IMAP), file transfer (FTP/SSH), anything requiring guaranteed delivery.

- **UDP use cases:** DNS queries (speed over reliability), video streaming (late packets are useless), online gaming (current state matters more than old positions), VoIP calls (real-time audio cannot wait for retransmission).

**Services discovered per protocol:**

| TCP Services | UDP Services |
|---|---|
| ssh (22) | netbios-ns (137) |
| http (80) | dhcp (67) |
| netbios-ssn (139) | netbios-dgm (138) |
| imap (143) | |
| https (443) | |
| microsoft-ds (445) | |
| commplex-link (5001) | |
| http-proxy (8080) | |
| blackice-icecap (8081) | |

### Step 6: MAC Address Resolution with ARP

IP addresses (Layer 3) identify devices across networks, but local communication requires MAC addresses (Layer 2). ARP (Address Resolution Protocol) translates IP addresses to MAC addresses.

```bash
arp -a | grep 192.168.92.3
```



![ARP showing MAC address C8:f7:33:67:d6:17 for OWASP VM](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/12_06%20ARP%20showing%20MAC%20address%20for%20OWASP%20VM.jpg)



**Result:** 192.168.92.3 is associated with MAC address C8:f7:33:67:d6:17 on interface eth1.

**How ARP works:**

When my machine wants to send packets to 192.168.92.3, it first checks its ARP cache to see if it already knows the MAC address. If not found, it broadcasts an ARP Request to the entire local network: "Who has IP 192.168.92.3? Tell 192.168.92.4."

Every device on the network receives the broadcast, but only 192.168.92.3 responds with an ARP Reply: "I am 192.168.92.3 and my MAC address is C8:f7:33:67:d6:17."

My machine caches this mapping for a few minutes so it does not need to repeat the ARP request for every packet.

**Why ARP is necessary:**

The Network Layer (Layer 3) uses IP addresses to route packets across the internet through multiple routers. But on the local network segment, switches and network cards do not understand IP addresses. They only recognize MAC addresses.

When a frame reaches the local switch, the switch examines the destination MAC address and forwards the frame only to the port connected to that specific MAC. Without ARP, IP packets could reach the correct network but would never be delivered to the final device.

**ARP security risk:**

ARP has no authentication. Any device can send a fake ARP Reply claiming to own any IP address. An attacker on the local network can send "I am 192.168.1.1 (the router) and my MAC is [attacker's MAC]." All traffic intended for the router now goes to the attacker instead. This is called ARP spoofing and is the basis for man-in-the-middle attacks on LANs.

### Step 7: Packet Capture and Analysis with Wireshark

To see exactly what happens at the packet level during network communication, I captured traffic between my Kali machine and the OWASP VM while browsing to a web application.

I started Wireshark, selected the eth1 interface, and began capturing. Then I navigated to http://192.168.92.3/webgoat in Firefox to generate HTTP traffic.



![Wireshark protocol hierarchy showing captured protocols](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/12_07%20Wireshark%20protocol%20hierarchy%20showing%20captured%20protocols.jpg)

**Protocols observed in capture:**

- **Frame (100%):** Every packet captured (Layer 2 Ethernet frames)
- **Ethernet (100%):** MAC-level communication
- **Internet Protocol Version 4 (100%):** IPv4 packets (Layer 3)
- **User Datagram Protocol (0.8%):** UDP traffic for DHCP
- **Dynamic Host Configuration Protocol (0.8%):** DHCP requests/responses
- **Transmission Control Protocol (99.2%):** TCP traffic dominates the capture
- **Hypertext Transfer Protocol (26.1%):** HTTP web traffic
- **Portable Network Graphics (6.5%):** PNG image files transferred
- **Media Type (2.7%):** Other media content
- **Line-based text data (3.1%):** Plain text responses
- **Compuserve GIF (0.4%):** GIF image files

**TCP Three-Way Handshake Identification:**

I filtered the capture for TCP traffic to/from port 80 and found the handshake sequence:

![Wireshark showing TCP three-way handshake and HTTP transaction](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/12_08%20Wireshark%20showing%20TCP%20three-way%20handshake%20and%20HTTP%20transaction.jpg)



**Packet 1:** 192.168.92.4 → 192.168.92.3 | TCP | [SYN] Seq=0 Win=64240  
**Packet 2:** 192.168.92.3 → 192.168.92.4 | TCP | [SYN, ACK] Seq=0 Ack=1 Win=5792  
**Packet 3:** 192.168.92.4 → 192.168.92.3 | TCP | [ACK] Seq=1 Ack=1 Win=64512  
**Packet 4:** 192.168.92.4 → 192.168.92.3 | HTTP | GET /HTTP/1.1

**Handshake breakdown:**

- **SYN packet:** My machine initiates connection with Seq=0 (starting sequence number) and Win=64240 (receive window size in bytes). This tells the server "I want to connect and I can receive up to 64KB of data at once."

- **SYN-ACK packet:** Server accepts with its own Seq=0 and Ack=1 (acknowledging my SYN). Win=5792 means the server can receive 5.7KB at once. The server is ready to communicate.

- **ACK packet:** My machine acknowledges the server's SYN with Ack=1. Connection is now fully established (ESTABLISHED state in netstat).

- **HTTP GET packet:** Now that the TCP connection is open, the actual HTTP request flows. This requests the root page (/) from the web server.

- **Subsequent packets:** The server responds with HTTP 200 OK and sends the HTML page. Wireshark shows the page content, images, CSS, and JavaScript files transferred over the established TCP connection.

**What Wireshark reveals that other tools miss:**

Ping only tells you if a host responds. Traceroute only shows routing paths. Netstat only shows current connection states. Wireshark shows the actual packet contents, protocol headers, and timing of every single frame. You can see:

- Unencrypted passwords in HTTP POST requests
- Session cookies that could be stolen
- Malware command-and-control traffic patterns
- Misconfigured network devices sending invalid packets
- Timing delays indicating network congestion

## Findings

**OSI to TCP/IP Mapping**

- The OSI model provides a detailed theoretical framework, but TCP/IP is the practical implementation. TCP/IP combines OSI's top three layers (Application, Presentation, Session) into a single Application layer because most protocols handle their own presentation and session management. The bottom two layers (Data Link, Physical) are combined into Network Access because the interface driver handles both.

**ICMP Operates at Layer 3**

- Ping uses ICMP, which has no concept of ports (Layer 4 feature). This makes ping useful for testing basic IP connectivity independent of higher-layer services. A host can respond to ping even if all TCP/UDP services are blocked.

**Traceroute Identifies Routing Failures**

- Single-hop result (directly connected network) versus multi-hop internet paths. Asterisks in traceroute output indicate routers dropping ICMP or timing out. This pinpoints network failures to specific router hops.

**TCP Connection States Reflect Communication Lifecycle**

- ESTABLISHED = active data transfer, TIME_WAIT = recently closed connection held in temporary state. The Transport Layer manages this state machine automatically. Applications do not need to manually track connection status.

**TCP Provides Reliability at the Cost of Speed**

- TCP scan completed in 7.99 seconds with guaranteed accurate results. UDP scan took 1024.46 seconds (128x slower) because lack of acknowledgments forces timeout-based detection. For security scanning, TCP is practical while UDP is prohibitively slow for large networks.

**ARP Bridges IP and MAC Addressing**

- Network Layer routing uses IP addresses, but local delivery requires MAC addresses. ARP resolves this translation automatically. Without ARP, packets would reach the correct network but never find the destination host.

**Wireshark Captures Protocol Details Invisible to Other Tools**

- Observed TCP three-way handshake (SYN, SYN-ACK, ACK) before HTTP GET request. Saw HTTP 200 OK response with full page content. Protocol hierarchy showed 99.2% TCP traffic, 26.1% HTTP application data, and 0.8% DHCP for network configuration. This level of detail is impossible with ping, traceroute, or netstat alone.

## Challenges Faced

- **TCP vs UDP timing confusion:** I initially thought UDP was faster than TCP because UDP does not have handshake overhead. The scan results showed the opposite. UDP scans are massively slower because filtered ports (most UDP ports) never respond, forcing Nmap to wait for timeouts. TCP immediately receives RST (reset) packets from closed ports, so detection is instant. This taught me that protocol simplicity does not equal practical speed.

- **Netstat connection state interpretation:** When I first saw TIME_WAIT in the output, I thought it meant the connection was stuck or broken. After researching, I learned TIME_WAIT is normal and prevents port number collisions. TCP keeps the connection in this state for 30-120 seconds after closing to ensure no delayed packets from the old connection interfere with new connections using the same port numbers. This is a safeguard, not an error.

- **Wireshark filter syntax for TCP handshake:** Finding the three-way handshake in thousands of captured packets was difficult initially. I tried filtering by "tcp" which showed too many results. The correct approach was using "tcp.flags.syn==1 and tcp.flags.ack==0" to isolate SYN packets, then following the TCP stream for that specific connection. This showed me the importance of learning proper Wireshark display filters.

## Key Takeaways

- **Layer isolation helps troubleshooting.** If ping works (Layer 3) but web browsing fails (Layer 7), the problem is not network connectivity. The issue is with the web server application or HTTP protocol. Knowing which layer is failing saves hours of debugging.

- **Traceroute maps the path, not just the destination.** A single asterisk in a 15-hop trace tells you exactly where packets are being dropped. This is far more useful than "connection failed" errors that do not indicate location.

- **TCP reliability is not free.** The three-way handshake, acknowledgments, and retransmissions add overhead. For applications where speed matters more than perfect delivery (video streaming, online gaming), UDP's unreliable nature is actually an advantage.

- **ARP has no authentication and is trivially spoofed.** Any device on the local network can claim to be any IP address by sending fake ARP replies. This enables man-in-the-middle attacks on LANs. Switched networks do not prevent ARP spoofing.

- **Wireshark is the ground truth for network behavior.** Tools like ping and netstat show summaries, but Wireshark shows exactly what is on the wire. When documentation conflicts with packet captures, trust the packets.

## Disclaimer

This lab was performed in a controlled local network environment using authorized VMs. The OWASP Broken Web Applications VM is intentionally vulnerable and designed for security training. No unauthorized networks were accessed. All testing was conducted for educational purposes only.
