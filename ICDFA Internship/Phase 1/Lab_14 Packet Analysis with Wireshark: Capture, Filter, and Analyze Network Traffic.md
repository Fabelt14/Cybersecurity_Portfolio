# Packet Analysis with Wireshark: Capture, Filter, and Analyze Network Traffic

## Overview

This lab dissected network communication at every layer of the TCP/IP model using Wireshark to capture live traffic between a Kali Linux system and an OWASP Broken Web Applications VM. The goal was to understand how protocols function in practice, not just in theory. By analyzing real packets, I learned why TCP is reliable, why UDP is fast, how encryption protects SSH but leaves HTTP exposed, and how to troubleshoot connectivity failures by working through the protocol stack layer by layer.

## Objectives

- Map the TCP/IP model to actual packet structures in Wireshark
- Analyze TCP handshakes, sequence numbers, and retransmission behavior
- Compare connection-oriented TCP with connectionless UDP
- Identify security risks in unencrypted HTTP traffic versus encrypted SSH
- Use ICMP for network diagnostics and understand TTL behavior
- Perform OS fingerprinting through TCP/IP stack analysis
- Troubleshoot connectivity failures using layered protocol inspection
- Understand ARP's role in bridging Layer 3 (IP) and Layer 2 (MAC addressing)

## Lab Environment

- **Attacker Machine:** Kali Linux (192.168.92.4)
- **Target:** OWASP Broken Web Applications VM (192.168.92.3)
- **Network:** Host-only network (192.168.92.0/24) for isolated communication

## Tools Used

- Wireshark (GUI packet analyzer with protocol dissection)
- tcpdump (command-line packet capture)
- Nmap (OS fingerprinting through TCP/IP stack analysis)
- ping (ICMP connectivity testing)
- netstat/ss (connection state monitoring)
- route (routing table inspection)
- arp (Layer 2 address resolution monitoring)

## Methodology

### Exercise 1: TCP/IP Model vs OSI Model

The OSI model is a theoretical framework that defines networking in seven precise layers. The TCP/IP model is what the internet actually uses. It combines OSI's upper layers (Application, Presentation, Session) into a single Application layer because separating data formatting from protocol logic adds complexity without practical benefit.

**Layer mapping:**

| TCP/IP Layer | Corresponding OSI Layers | Purpose |
|--------------|-------------------------|---------|
| Application | Application, Presentation, Session | HTTP, SSH, DNS - user-facing protocols |
| Transport | Transport | TCP/UDP - reliability vs speed trade-off |
| Internet | Network | IP routing, ICMP diagnostics |
| Link | Data Link, Physical | Ethernet frames, MAC addressing, wire transmission |

The TCP/IP model prioritizes efficiency. Combining layers means fewer encapsulation steps and faster processing. This is why real networks use TCP/IP while network engineers study OSI for conceptual understanding.

### Exercise 2: TCP Handshake and Reliability Mechanisms

To see how TCP establishes connections, I captured traffic while browsing to the OWASP VM's web interface. Wireshark showed the three-way handshake that precedes every HTTP request.

**Handshake breakdown:**

1. **SYN (Synchronize):** Kali sends a packet with the SYN flag set, requesting a connection. This includes an initial sequence number that the server will use to track data order.

2. **SYN-ACK (Synchronize-Acknowledge):** The OWASP VM responds with both SYN and ACK flags set. It acknowledges Kali's sequence number and provides its own initial sequence number.

3. **ACK (Acknowledge):** Kali sends a final ACK to confirm the connection is established. Only after this third packet does HTTP data transmission begin.

**TCP flags and their purposes:**

- **SYN:** Connection initiation - "I want to talk"
- **ACK:** Acknowledgment - "I received your data"
- **FIN:** Graceful connection termination - "I'm done, goodbye"
- **RST:** Immediate connection reset - "Something is wrong, abort now"
- **PSH:** Push buffered data to application immediately - "Don't wait for more data, deliver what you have"
- **URG:** Urgent data pointer - "This packet contains high-priority data"

**How TCP maintains reliability:**

![TCP sequence numbers and acknowledgments in packet capture](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/14_01%20TCP%20sequence%20numbers%20and%20acknowledgments%20in%20packet%20capture.jpg)

- Every byte transmitted gets a unique sequence number. If Kali sends bytes 1-1000 in one packet and bytes 1001-2000 in the next, the OWASP VM can detect if the second packet arrives before the first. The VM holds packet 2 in a buffer while it waits for packet 1, then assembles them in the correct order.

- **Retransmission timer:** After sending a packet, Kali starts a timer. If no ACK arrives before the timer expires, Kali assumes the packet was lost and retransmits it. This creates reliability without requiring the underlying network to guarantee delivery. Even on unreliable networks with 10% packet loss, TCP applications work correctly because lost packets are automatically resent.

- **Why this matters:** Downloaded files arrive intact, not corrupted. Web pages load completely, not with missing images. SSH commands execute in the order typed, not scrambled. TCP's complexity at the transport layer simplifies application development because programmers can assume reliable, ordered delivery.

### Exercise 3: IP Packet Header Analysis

To understand Layer 3 routing, I inspected the IP header of packets sent to the OWASP VM.

![IP packet header showing source, destination, TTL, and protocol fields](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/14_02%20IP%20packet%20header%20showing%20source%2C%20destination%2C%20TTL%2C%20and%20protocol%20fields.jpg)


**Key header fields:**

- **Source IP:** 192.168.92.4 (Kali machine)
- **Destination IP:** 192.168.92.3 (OWASP VM)
- **TTL (Time to Live):** 64

---

- **Source and Destination IPs** tell routers where the packet is going and where replies should be sent. Without these, the internet would be impossible. Every router along the path examines the destination IP, consults its routing table, and forwards the packet toward the next hop.

- **TTL prevents routing loops:** Every time a packet passes through a router, the TTL decrements by 1. If TTL reaches 0, the router discards the packet and sends an ICMP Time Exceeded message back to the source. This prevents packets from circulating forever if routing tables contain errors.

- **Routing in this lab:** Both machines are on the same subnet (192.168.92.0/24), so there are zero hops between them. Kali's routing table shows that 192.168.92.3 is directly reachable via the eth1 interface. No gateway is needed. The packet goes straight from Kali's network interface to the OWASP VM's network interface over the virtual switch.

This direct routing means TTL doesn't change because no routers are involved. If I were accessing a website on the internet, the TTL might start at 64 and arrive at the destination with a TTL of 50 after passing through 14 routers.

### Exercise 4: Application Layer Protocol Comparison

Wireshark reveals what happens at Layer 7 when applications communicate. I compared HTTP (unencrypted) with SSH (encrypted) to understand why encryption matters.

**HTTP analysis - complete exposure:**

![HTTP request showing cleartext URL, headers, and cookies](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/14_03%20HTTP%20request%20showing%20cleartext%20URL%2C%20headers%2C%20and%20cookies.jpg)

The HTTP request to /WebGoat/attack exposed everything:

- **URL path:** /WebGoat/attack - tells an eavesdropper exactly which vulnerable application I'm testing
- **User-Agent:** Mozilla/5.0 (X11; Linux x86_64) Firefox/140.0 - reveals my operating system and browser version
- **Cookies:** JSESSIONID and PHPSESSID - session identifiers that could be stolen for session hijacking
- **Referrer:** Previous page visited, creating a browsing history trail

The server's 401 Unauthorized response included:

- **Server header:** Apache-Coyote (reveals server software)
- **WWW-Authenticate:** Basic realm="Web Goat Application" (requests credentials)
- **Content-Type:** text/html
- **Content-Encoding:** gzip (compression before transmission)

**Security impact:** Anyone sniffing the network sees the complete conversation. If I had submitted login credentials, they would appear in plaintext in the HTTP POST body. This is why HTTP is unsuitable for sensitive data.

**SSH analysis - encrypted payload protection:**

![SSH key exchange showing negotiated encryption method](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/14_04%20SSH%20key%20exchange%20showing%20negotiated%20encryption%20method.jpg)

- SSH traffic starts with an unencrypted Key Exchange where both sides agree on encryption parameters. Wireshark can dissect this initial handshake, showing the negotiated algorithm (diffie-hellman-group-exchange-sha256). This is not a vulnerability because no sensitive data is transmitted yet.

- After key exchange completes, all subsequent SSH traffic appears as random hex data. Wireshark shows "Encrypted packet (len=XXX)" but cannot decrypt the contents. The actual commands typed, server responses, and any data transferred are protected.

- **Why encryption works:** The SSH key exchange uses Diffie-Hellman to establish a shared secret without transmitting it over the network. Both sides independently derive the same symmetric encryption key, then use it to encrypt all traffic with AES. An eavesdropper sees the key exchange parameters but cannot compute the final key without access to the private keys stored on each machine.

- **Application Layer's role:** This layer translates user actions into network protocols. Clicking a link becomes an HTTP GET request. Typing a command in an SSH terminal becomes encrypted SSH protocol messages. The Application Layer also handles authentication (401 Unauthorized in HTTP, password prompts in SSH) and data formatting (gzip compression, HTML rendering).

### Exercise 5: TCP Error Handling and Retransmission

To see how TCP handles packet loss, I simulated drops by briefly disabling the network interface mid-transfer.

**What happens when packets drop:**

When an ACK doesn't arrive within the timeout period, the sender assumes the packet was lost. The connection appears frozen from the user's perspective because the Application Layer is waiting for the Transport Layer to confirm delivery. No new data can be sent until the missing packet is acknowledged because sequence numbers must remain continuous.

**Positive Acknowledgment with Retransmission (PAR):**

Every TCP packet sent starts a retransmission timer. If the timer expires before an ACK arrives, the sender retransmits the packet. The timer duration adapts based on measured round-trip time. On fast local networks, timeouts are measured in milliseconds. On intercontinental connections with satellite hops, timeouts can be seconds.

**Sequence number recovery:**


![TCP retransmission showing duplicate sequence numbers](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/14_05%20TCP%20retransmission%20showing%20duplicate%20sequence%20numbers.jpg)


- Sequence numbers enable the receiver to detect missing packets. If bytes 1-500 arrive, then bytes 1001-1500 arrive, the receiver knows bytes 501-1000 are missing. Instead of discarding packet 2, the receiver buffers it and sends an ACK specifically requesting bytes 501-1000. This selective acknowledgment tells the sender exactly what to retransmit.

- **Why reliability matters:** TCP guarantees ordered delivery. If a web page requires three packets (HTML, CSS, JavaScript), they might arrive out of order over the internet. TCP's sequence numbers ensure the browser receives them in the correct order, preventing rendering errors. Database transactions using TCP maintain ACID properties because SQL commands execute in the exact order sent.

### Exercise 6: ICMP for Network Diagnostics

ICMP operates at Layer 3 alongside IP. It doesn't carry application data. It carries network status messages.


![ICMP echo request and reply packets showing Type and Code fields](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/14_06%20ICMP%20echo%20request%20and%20reply%20packets%20showing%20Type%20and%20Code%20fields.jpg)


**ICMP packet structure:**

- **Type:** 8 (Echo Request) or 0 (Echo Reply)
- **Code:** 0 (no error subcategory)
- **Checksum:** 0x56cf - detects transmission errors
- **Identifier:** Matches requests with replies
- **Sequence Number:** Increments with each ping, allowing round-trip time measurement

**Diagnostic use:** Ping confirms basic connectivity. If an ICMP Echo Reply arrives, the target is reachable, the network path works, and the target's IP stack is functioning. If no reply arrives, the problem could be anywhere: network cable unplugged, firewall blocking ICMP, target machine powered off, routing misconfiguration.

**TTL's dual purpose:**

In normal IP packets, TTL prevents routing loops. In ICMP specifically, TTL enables traceroute. By sending packets with TTL=1, TTL=2, TTL=3, traceroute forces each router along the path to send back an ICMP Time Exceeded message. Each message reveals one router's IP address, mapping the entire route from source to destination.

**Why ICMP differs from TCP:** ICMP doesn't use the three-way handshake because it's not establishing a session. Ping is a simple request-reply. No state needs to be maintained. The overhead of SYN, SYN-ACK, ACK would be wasteful for a single diagnostic packet.

### Exercise 7: UDP - Speed Over Reliability

UDP removes all of TCP's reliability mechanisms to minimize latency and overhead.

![UDP packet structure showing minimal header with only ports and length](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/14_07%20UDP%20packet%20structure%20showing%20minimal%20header%20with%20only%20ports%20and%20length.jpg)


**TCP vs UDP packet structure comparison:**

- **TCP header:** Source port, destination port, sequence number, acknowledgment number, header length, flags (SYN/ACK/FIN/RST/PSH/URG), window size, checksum, urgent pointer, options. Total minimum size: 20 bytes.

- **UDP header:** Source port, destination port, length, checksum. Total size: 8 bytes.

TCP's header is more than twice as large because it carries all the state information needed for reliable delivery. UDP discards this overhead because it doesn't guarantee delivery.

**Why UDP doesn't ensure reliability:**

- UDP has no concept of acknowledgments. It sends the packet to the IP layer and forgets about it. If the packet gets lost, UDP doesn't know and doesn't care. The application must detect packet loss and handle it, if loss matters.

- UDP has no sequence numbers. Packets can arrive out of order and the receiver has no way to reorder them. Again, the application must handle this if order matters.

- UDP has no retransmission timer. There is nothing to time and nothing to retransmit.

**When UDP is the right choice:**

- **Live streaming:** If a video frame is lost, retransmitting it three seconds later is useless because the video has already moved on. Better to skip the frame and continue with newer data. TCP's retransmission would cause buffering and stuttering.

- **Online gaming:** A player's position update that is 500ms old is worthless. The player has already moved. Retransmitting stale position data causes rubber-banding where characters jump around the screen. UDP delivers the latest position immediately, discarding outdated packets.

- **VoIP (Voice over IP):** Human ears tolerate occasional audio dropouts better than delayed speech. A missing 20ms audio packet causes a brief click. Retransmitting that packet 200ms later creates echoes and makes conversation impossible.

- **DNS queries:** A single DNS query fits in one UDP packet. The client sends a query, waits a few seconds, and retransmits if no response arrives. No need for TCP's connection overhead for this simple request-response.

**How UDP manages transmission without reliability guarantees:**

UDP doesn't manage transmission. It delivers packets to the IP layer and immediately returns control to the application. The application is responsible for implementing reliability if needed. Some applications build their own reliability on top of UDP (QUIC protocol used by HTTP/3), others accept packet loss as acceptable (video streaming).

### Exercise 8: OS Fingerprinting Through TCP/IP Stack Analysis

Different operating systems implement the TCP/IP stack with subtle variations. Nmap exploits these differences to identify what OS a target is running without ever logging in.

![Nmap scan results showing OS detection based on network responses](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/14_08%20Nmap%20scan%20results%20showing%20OS%20detection%20based%20on%20network%20responses.jpg)


**Key fingerprinting indicators:**

- **TTL values:** Linux typically uses 64, Windows uses 128, some network equipment uses 255. The OWASP VM's packets arrived with TTL=64, strongly suggesting a Linux-based system.

- **TCP window size:** The SYN-ACK packet showed Win=5840 in one case and Win=1024 in another. Different operating systems and versions use different default window sizes. Windows 10 often uses 8192, older Linux kernels use 5840. These values are hardcoded in the kernel and change only with OS updates.

![TCP window size variations in captured packets](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/14_09%20TCP%20window%20size%20variations%20in%20captured%20packets.jpg)


- **TCP options and flag handling:** Nmap sends unusual flag combinations (SYN+FIN+URG+PSH) that standard traffic never uses. Each OS responds differently. Some ignore invalid combinations, some send RST, some respond normally. The specific response pattern maps to a known OS fingerprint database.

- **Closed port responses:** When Nmap probes a closed port, the target sends [RST, ACK] to reject the connection. The exact timing and additional TCP options in this rejection vary by OS.

**Why OS detection matters in security assessments:**

- **Vulnerability mapping:** If Nmap identifies "Linux 2.6.x," an attacker can search CVE databases for Linux 2.6 kernel vulnerabilities. Windows RDP exploits won't work against a Linux target. Knowing the OS focuses the attack.

- **Patch verification:** If a network scan shows "Windows Server 2008," but the organization claims all systems are updated to Server 2019, either the scan is wrong or an undocumented legacy system exists. OS fingerprinting validates asset inventories.

- **Intrusion detection tuning:** IDS rules can be written for specific OS behaviors. A Windows machine sending packets with TTL=64 is suspicious and might indicate a compromised system running Linux malware or a misconfigured VM.

### Exercise 9: ARP - Bridging IP and MAC Addressing

ARP solves the problem that IP addresses identify hosts logically, but Ethernet requires MAC addresses physically.

![ARP request broadcast asking who has a specific IP](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/14_10%20ARP%20request%20and%20reply.jpg)


**ARP operation:**

When Kali needs to send a packet to 192.168.92.3, it checks its ARP cache. If no MAC address is cached, it broadcasts an ARP Request: "Who has 192.168.92.3? Tell 192.168.92.4." This broadcast reaches every device on the local network segment.

- The OWASP VM recognizes its own IP in the request and sends a unicast ARP Reply: "192.168.92.3 is at c8:f7:33:67:d6:17." This reply goes directly to Kali, not to the entire network.

- Kali stores this mapping in its ARP cache for several minutes. Subsequent packets to 192.168.92.3 skip the ARP query and directly use the cached MAC address.

**Layer 2 vs Layer 3 addressing:**

- IP addresses (Layer 3) identify hosts across different networks. Routers use IP addresses to make forwarding decisions.

- MAC addresses (Layer 2) identify network interfaces on the same physical segment. Switches use MAC addresses to forward frames to the correct port.

- ARP translates between these two addressing schemes. Without ARP, the Ethernet frame would have no destination MAC address and the switch wouldn't know which port to send it to.

**ARP's role in the lab environment:**

Even though Kali and the OWASP VM are on a virtual network, the same ARP process applies. The virtual switch emulates physical Ethernet. When Kali sends a frame to MAC address c8:f7:33:67:d6:17, the virtual switch delivers it to the OWASP VM's virtual network interface.

**ARP cache poisoning:** Because ARP has no authentication, an attacker on the same network can send fake ARP replies claiming to own any IP address. This redirects traffic through the attacker's machine, enabling man-in-the-middle attacks. Static ARP entries and ARP inspection features on managed switches prevent this attack.

### Exercise 10: Troubleshooting Using Layered Protocol Analysis

To practice systematic troubleshooting, I intentionally broke connectivity by changing Kali's subnet mask from /24 to /30.

**The simulated failure:**

With a /30 mask (255.255.255.252), only 4 IP addresses fit in the subnet: .0 (network), .1, .2, .3 (hosts), .4 (broadcast). Kali was .4 and the OWASP VM was .3. According to the /30 mask, they are on different subnets.

**Effects on communication:**

Kali's routing table updated to reflect the new subnet. The route -n output showed that 192.168.92.3 was no longer in the local subnet. Kali started trying to route packets to a gateway instead of delivering them directly.

Since no gateway exists on this isolated network, all packets failed with "Network is unreachable" errors. Ping returned 100% packet loss. Web browsers showed "Connection refused" because they couldn't even reach the TCP layer.

**Layer-by-layer diagnosis:**

- **Layer 3 (Network Layer):** Running `route -n` showed that the destination IP no longer matched the directly connected route. Kali interpreted 192.168.92.3 as remote and tried to send packets to a default gateway that doesn't exist.

- **Layer 2 (Link Layer):** Running `arp -n` showed no MAC address for 192.168.92.3. ARP requests are broadcast only on the local segment. Since Kali's routing table said the VM was remote, it never broadcast an ARP request. Without a MAC address, the Ethernet frame cannot be constructed.

**Resolution:**

I restored the correct /24 mask (255.255.255.0) using `ifconfig eth1 192.168.92.4 netmask 255.255.255.0`. Immediately, the routing table updated to show 192.168.92.0/24 as directly connected. Kali broadcast an ARP request, received the OWASP VM's MAC address, and communication resumed.

**Recommended troubleshooting tools and methodology:**

| Tool | Primary Use | TCP/IP Layer | Example |
|------|------------|--------------|---------|
| ping | Verify end-to-end connectivity | Network (ICMP) | Does the target respond at all? |
| traceroute | Identify where packets are dropping | Network (IP/TTL) | Which router is the last to respond? |
| Wireshark | Analyze protocol-level errors | All Layers | Are packets being sent with wrong flags? |
| netstat / ss | Confirm service is listening | Transport (TCP/UDP) | Is port 80 actually open? |
| dig / nslookup | Test DNS resolution | Application (DNS) | Does the hostname resolve to the right IP? |
| route | Examine routing table | Network (IP) | Is there a route to the destination? |
| arp | Check Layer 2 address resolution | Link (ARP) | Does the MAC address resolve? |

**Troubleshooting methodology:**

1. **Start at Layer 1:** Is the cable plugged in? Is the link light on? Virtual networks can't have cable problems, but physical networks can.

2. **Check Layer 2:** Does ARP resolve the MAC address? If `arp -n` shows `<incomplete>`, the target isn't responding to ARP requests.

3. **Verify Layer 3:** Does ping work? If ICMP is blocked by firewalls, try pinging the gateway instead to confirm routing.

4. **Test Layer 4:** Does `telnet <target> <port>` connect? This confirms TCP handshake completion even if the actual service is broken.

5. **Examine Layer 7:** If lower layers work but the application fails, capture traffic in Wireshark to see error codes (HTTP 404, 500, etc.).

This bottom-up approach isolates failures. If ping fails but ARP works, the problem is at Layer 3 or above (routing, firewall). If ARP fails, the problem is at Layer 2 or below (switch configuration, wrong VLAN).

## Findings

- **TCP reliability comes from continuous feedback loops.** Every packet sent triggers a timer. Every packet received sends an ACK. Sequence numbers detect missing or reordered data. This overhead makes TCP unsuitable for latency-sensitive applications but perfect for data integrity requirements.

- **UDP sacrifices reliability for minimal latency.** By eliminating handshakes, acknowledgments, and retransmission timers, UDP reduces per-packet overhead from 20+ bytes to 8 bytes. Applications that can tolerate packet loss gain significant performance improvements.

- **Unencrypted protocols expose everything.** HTTP transmits URLs, cookies, credentials, and content in plaintext. Anyone with packet capture access sees the complete conversation. This includes network administrators, ISP employees, and attackers on the same WiFi network. SSH encryption renders the payload unreadable even with full packet captures.

- **OS fingerprinting works because implementations differ.** TTL values, window sizes, and TCP option ordering vary between Windows, Linux, BSD, and network appliances. These variations are implementation details, not protocol requirements. Nmap's fingerprint database maps these patterns to specific OS versions.

- **ARP is the invisible translator between IP and Ethernet.** Users think in IP addresses, but network hardware only understands MAC addresses. ARP requests are broadcast to all devices, replies are unicast to minimize network noise. ARP cache poisoning exploits the lack of authentication in this translation process.

- **Subnet masks change routing behavior without changing IPs.** A /30 mask makes 192.168.92.3 appear remote from 192.168.92.4, even though they're numerically adjacent. The routing table recalculates based on the mask, causing packets to route through nonexistent gateways instead of going direct.

- **Layer-by-layer troubleshooting isolates failures efficiently.** Starting at the Physical layer and working up prevents wasting time debugging application issues when the problem is a misconfigured subnet mask. Each layer depends on the layers below it, so failures propagate upward.

## Challenges Faced

- **Distinguishing between protocol theory and implementation behavior:** The OSI model teaches seven layers, but TCP/IP combines three of them into one Application layer. Initially, I expected to see Session layer activity in Wireshark, but real protocols don't separate session management from application logic. Understanding that TCP/IP is the practical implementation while OSI is the conceptual model clarified this confusion.

- **Interpreting encrypted SSH payloads:** When SSH traffic appeared as random hex data in Wireshark, I initially thought the capture was corrupted. Realizing that this randomness is proof that encryption works shifted my perspective. The inability to analyze encrypted payloads is a feature, not a bug. It demonstrates that even with full network access, an attacker gains nothing from capturing SSH traffic.

- **Understanding why ARP is necessary:** If both machines already have IP addresses, why do they need MAC addresses too? The answer is that IP addresses identify hosts globally across networks, while MAC addresses identify interfaces locally on a single segment. Routers need both: IP to route packets between networks, MAC to deliver frames within a network. ARP is the bridge between these two addressing schemes.

- **Recognizing that sequence numbers are byte counts, not packet counts:** TCP sequence numbers increment by the number of bytes sent, not by the number of packets. Sending 1000 bytes advances the sequence number by 1000, regardless of whether those bytes were split into one packet or ten packets. This allows receivers to detect missing bytes, not just missing packets.

## Key Takeaways

- **TCP and UDP solve different problems.** TCP guarantees reliability at the cost of latency. UDP minimizes latency at the cost of reliability. Neither is better. They serve different use cases. File transfers need TCP. Live video needs UDP. Choosing the wrong protocol creates problems the application can't solve.

- **Encryption must happen before transmission.** HTTP sends data, then SSL/TLS encrypts it. But HTTP by itself is cleartext. SSH encrypts from the start. The protocol, not the application, determines security. A web app can implement perfect authentication, but if it runs over HTTP, the passwords are visible in Wireshark.

- **Packet analysis reveals what applications hide.** The browser shows "Connection refused," but Wireshark shows whether the SYN packet was sent, whether the server responded with RST, or whether the packet never left the network interface. Applications abstract away the protocol details. Packet captures expose those details.

- **Every layer adds headers.** HTTP data gets wrapped in a TCP header, which gets wrapped in an IP header, which gets wrapped in an Ethernet frame. Each layer encapsulates the layer above it. Receivers strip headers in reverse order. This encapsulation allows each layer to operate independently without knowing what the other layers are doing.

- **TTL prevents infinite loops but also enables traceroute.** A single field serves two purposes: preventing routing loops by decrementing to zero, and revealing the network path by intentionally expiring packets at each hop. This dual use demonstrates why protocol design is about trade-offs, not absolutes.

- **OS fingerprinting is passive reconnaissance.** Nmap identifies the operating system without authenticating, without exploiting vulnerabilities, without sending any malicious traffic. Just by observing how the target responds to standard network traffic, an attacker learns what exploits might work. This is why defense-in-depth matters. Even if the OS can't be hidden, hardening the OS prevents exploitation.

## Disclaimer

This lab was performed in a controlled environment using virtual machines configured specifically for security training. The OWASP Broken Web Applications VM is an intentionally vulnerable system designed for educational use. No unauthorized systems were accessed. All packet captures were performed on a host-only network isolated from production systems.
