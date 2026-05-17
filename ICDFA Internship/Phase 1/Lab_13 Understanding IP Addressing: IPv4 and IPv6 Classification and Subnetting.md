# Network Interface Analysis and Packet Capture

## Overview

This lab examined how network interfaces operate at the system level by analyzing traffic between a Kali Linux attack machine and an OWASP Broken Web Applications VM. The goal was to understand how data moves through network layers, how to capture and analyze packets, and how to troubleshoot connectivity issues by manipulating interface states.

## Objectives

- Identify active network interfaces and their assigned IP addresses
- Capture and analyze packets on specific interfaces using tcpdump
- Monitor network statistics to assess interface health and active connections
- Use Wireshark to dissect protocol structures and TCP/IP model layers
- Understand ICMP ping mechanics and TCP three-way handshakes
- Troubleshoot connectivity by disabling and re-enabling interfaces
- Monitor bandwidth usage to detect congestion or unusual activity
- Apply advanced packet filters to isolate specific traffic patterns

## Lab Environment

- **Attacker Machine:** Kali Linux with three network interfaces (eth0, eth1, lo)
- **Target:** OWASP Broken Web Applications VM (192.168.92.3)
- **Network Configuration:** Host-only network for isolated VM communication, NAT network for internet access

## Tools Used

- ifconfig (interface configuration and status)
- tcpdump (command-line packet capture)
- netstat (network statistics and active connections)
- Wireshark (GUI-based protocol analyzer)
- iftop (real-time bandwidth monitoring)

## Methodology

### Exercise 1: Network Interface Discovery

I started by identifying which network interfaces exist on the Kali system and what role each one serves. Running `ifconfig` revealed three active interfaces:

- **eth0 (10.0.2.15):** Primary interface configured for NAT, allowing the VM to access the internet through the host machine's network connection. This is the default route for external traffic.

- **eth1 (192.168.92.4):** Secondary interface on a host-only network that connects Kali directly to the OWASP-BWA VM (192.168.92.3). This isolated network prevents attack traffic from leaking to the real network.

- **lo (127.0.0.1):** Loopback interface used for internal system communication. Applications can send data to 127.0.0.1 to test network code without involving physical hardware. The loopback interface never leaves the machine.

The separation between eth0 and eth1 is critical for security labs. Internet traffic (updates, research) goes through eth0, while attack traffic against vulnerable targets stays isolated on eth1. This prevents accidentally attacking production systems.

### Exercise 2: Packet Capture on Target Interface

To see what traffic flows between Kali and the OWASP VM, I captured packets specifically on the eth1 interface using tcpdump. The command `sudo tcpdump -i eth1` listens only to traffic on that interface and ignores everything on eth0 or lo.

![tcpdump capture showing HTTP, TCP, UDP, and DHCP packets](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/13_01%20tcpdump%20capture%20showing%20HTTP%2C%20TCP%2C%20UDP%2C%20and%20DHCP%20packets.jpg)

- **Protocol breakdown:** The capture showed HTTP (web requests), TCP (connection-oriented traffic), UDP (connectionless traffic), and DHCP (IP address assignment). All of this traffic involved communication between 192.168.92.4 (Kali) and 192.168.92.3 (OWASP-BWA).

![Wireshark protocol hierarchy showing traffic distribution](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/13_02%20Wireshark%20protocol%20hierarchy%20showing%20traffic%20distribution.jpg)

- **Network interface role in the OSI model:** When my browser sends an HTTP request, the operating system hands that data to the network interface. The interface operates at Layer 2 (Data Link) of the OSI model. It takes the IP packet from Layer 3, wraps it in an Ethernet frame with source and destination MAC addresses, and transmits it as electrical signals on the wire.

When receiving packets, the interface does the reverse. It listens for frames addressed to its MAC address (or broadcast frames), strips off the Ethernet header, checks for transmission errors using the Frame Check Sequence, and passes the IP packet up to the operating system for processing. If the destination MAC address does not match, the interface discards the frame without wasting CPU cycles.

### Exercise 3: Network Statistics and Connection States

Network statistics reveal whether interfaces are functioning properly and what connections are active. Running `netstat -i` shows per-interface packet counts and error rates.

![netstat -i output showing RX/TX packet counts and error rates](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/13_03%20netstat%20-i%20output%20showing%20RX%20or%20TX%20packet%20counts%20and%20error%20rates.jpg)

- **Interface health check:** The output showed eth0 transmitted 587 packets and received 31 packets with zero errors or dropped packets. Eth1 transmitted 451 packets and received 1,663 packets, also with zero errors. The loopback interface (lo) transmitted and received 10 packets each. These clean statistics indicate the interfaces are functioning correctly with no physical layer issues.

To see active connections, I ran `netstat` without the `-i` flag:

![netstat output showing active UNIX domain sockets](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/13_04%20netstat%20output%20showing%20active%20UNIX%20domain%20sockets.jpg)

- **Active connections:** The output showed multiple UNIX domain sockets in CONNECTED state. These are inter-process communication channels used by system services like pipewire (audio) to talk to each other. UNIX sockets do not involve network interfaces at all. They are file-based communication channels that bypass the TCP/IP stack entirely for efficiency.

**Why this matters for monitoring:**

- **Reliability:** High error counts or dropped packets indicate physical problems. Faulty cables produce CRC errors, failing network cards produce frame errors, and buffer overflows produce dropped packets. Zero errors means the hardware is working.

- **Traffic volume:** ESTABLISHED connection counts show load. If netstat shows 500 simultaneous connections to a web server, that server is under heavy load. Zero connections to a service that should be running means the service crashed or a firewall is blocking access.

- **Security forensics:** Connection states reveal attack patterns. A flood of SYN_SENT states without corresponding ESTABLISHED states indicates a SYN flood attack. TIME_WAIT states confirm TCP teardowns completed properly, preventing old packets from interfering with new connections.

### Exercise 4: Protocol Analysis with Wireshark

Wireshark provides a GUI for detailed protocol inspection. After capturing traffic to the OWASP VM, I analyzed the packets to identify protocols in use and understand how the TCP/IP model maps to real network communication.

- **Protocol identification:** The capture showed TCP as the dominant transport protocol, with HTTP running on top of it. Every HTTP request uses TCP to guarantee reliable delivery.

**Packet dissection:**

![Wireshark packet details showing source/destination IPs, ports, and TCP flags](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/13_05%20Wireshark%20packet%20details%20showing%20source%20and%20destination%20IPs%2C%20ports%2C%20and%20TCP%20flags.jpg)

- **Source IP:** 192.168.92.4 (Kali machine)
- **Destination IP:** 192.168.92.3 (OWASP VM)
- **Destination Port:** 80 (standard HTTP port)
- **Source Ports:** 58758, 37144 (randomly assigned by the OS for each connection)
- **TCP Flags:** SYN (connection initiation), ACK (acknowledgment), FIN (connection termination)

- **TCP/IP model in action:** The captured traffic demonstrates how the TCP/IP model wraps data in successive layers. At the Application Layer, the browser generates an HTTP GET request. The Transport Layer (TCP) adds port numbers (source: 58758, destination: 80) and sequence numbers for reliability. The Internet Layer (IP) adds source and destination IP addresses for routing. Finally, the Link Layer adds MAC addresses for delivery to the next hop.

When the packet reaches the OWASP VM, each layer strips its header and passes the remaining data up to the next layer. The MAC addresses get stripped at Layer 2, IP addresses at Layer 3, TCP ports at Layer 4, and finally the HTTP content reaches the web server application at Layer 7.

### Exercise 5: Ping and TCP Handshake Analysis

To understand round-trip packet behavior, I captured ICMP ping traffic and observed the request-reply pattern.

![tcpdump showing ICMP echo request and echo reply pairs](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/13_06%20tcpdump%20showing%20ICMP%20echo%20request%20and%20echo%20reply%20pairs.jpg)

- **Ping mechanics:** For every ICMP Echo Request packet sent from 192.168.92.4, the OWASP VM immediately responds with an ICMP Echo Reply. This one-to-one correspondence confirms the network path is working and measures round-trip time. Ping does not use the TCP three-way handshake because ICMP operates at Layer 3 (same level as IP), not Layer 4.

- **TCP three-way handshake:** For TCP connections (like HTTP), the handshake works differently:

1. **SYN:** Client sends SYN flag to initiate connection
2. **SYN-ACK:** Server responds with both SYN and ACK flags
3. **ACK:** Client sends final ACK to confirm connection established

Only after this handshake completes does data transmission begin. Ping skips this entirely because it does not need reliability guarantees.

**OSI/TCP layer involvement:**

- For ping: Physical Layer (wire), Data Link Layer (Ethernet frames), Network Layer (IP routing and ICMP). Layers 4-7 are not used.

- For HTTP over TCP: All layers from Physical through Application are involved. TCP at Layer 4 handles reliability, HTTP at Layer 7 provides the actual request/response semantics.

### Exercise 6: Interface Troubleshooting

To simulate a network failure and understand recovery, I disabled the eth1 interface while actively communicating with the OWASP VM.

**Disabling the interface:**

```bash
sudo ifconfig eth1 down
```

![ifconfig eth1 down command execution](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/13_07%20ifconfig%20eth1%20down%20command%20execution.jpg)

**Immediate effects:** All active TCP connections to 192.168.92.3 dropped instantly. Attempting to ping the OWASP VM returned 100% packet loss because the operating system no longer had a route to send packets. Refreshing the web page in the browser showed "Unable to connect" errors. Running `netstat -i` confirmed eth1 was DOWN.

**Re-enabling the interface:**

```bash
sudo ifconfig eth1 up
```

![ifconfig eth1 up command execution](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/13_08%20ifconfig%20eth1%20up%20command%20execution.jpg)

**Recovery behavior:** When I brought the interface back UP, the system restored the IP configuration (192.168.92.4). The previously failed web page reloaded successfully after refresh. Ping started working again immediately.

**Practical troubleshooting use:** Network administrators use this technique to isolate software versus hardware failures. If disabling and re-enabling an interface fixes connectivity, the problem was likely a stuck state in the network driver or a stale ARP cache entry. If it does not fix the issue, the problem is physical (bad cable, failed port). This is the "turn it off and back on" of networking, clearing the interface's internal buffers and state tables.

### Exercise 7: Bandwidth Monitoring

To understand current network load, I used `iftop` to monitor real-time bandwidth usage on the eth1 interface while browsing the OWASP VM.

![iftop showing bandwidth usage between Kali and OWASP VM](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/13_09%20iftop%20showing%20bandwidth%20usage%20between%20Kali%20and%20OWASP%20VM.jpg)

- **Current bandwidth usage:** The peak transfer rate was 76.8 Kb (kilobits) and the current rate fluctuated between 1.9 Kb and 4.21 Kb. These are extremely low values. A standard 100 Mbps Ethernet interface can handle 100,000 Kb/s, so the current usage is only 0.076% of available capacity.

- **No congestion detected:** The low bandwidth usage confirms there is no traffic congestion. Web browsing and simple HTTP requests consume minimal bandwidth. Congestion would show up as sustained high usage near the interface's maximum capacity, typically accompanied by increased latency and packet loss.

**Security and capacity planning applications:**

- **Baseline establishment:** Monitoring shows normal traffic patterns. If eth1 suddenly starts transmitting 50 Mbps when it normally uses 5 Kb, something is wrong. This could indicate a compromised machine exfiltrating data or a misconfigured backup job.

- **Capacity planning:** If bandwidth usage consistently approaches interface capacity, the network needs an upgrade. Monitoring trends over time reveals whether 100 Mbps is sufficient or if a 1 Gbps link is required.

- **Attack detection:** Bandwidth monitoring catches data exfiltration attacks. If a workstation suddenly uploads gigabytes to an unknown external IP address, an attacker may be stealing sensitive files. Normal workstation traffic is mostly downloads (web browsing, software updates), not uploads.

### Exercise 8: Advanced Packet Filtering

Without filtering, tcpdump and Wireshark capture every packet on the interface, including broadcast traffic, background system updates, and unrelated browser activity. Advanced filters isolate only the packets relevant to the current investigation.

- **Why filtering matters:** A busy network interface can process thousands of packets per second. Trying to find a specific HTTP request in a capture of 50,000 mixed packets is impractical. Filters eliminate noise and focus on the conversation of interest.

**Example filter use cases:**

- **Troubleshooting failed logins:** Filter for `tcp.port == 22 and ip.addr == 192.168.92.3` to see only SSH traffic to the OWASP VM. This isolates authentication attempts and shows whether the server is sending rejection messages or the connection is timing out.

- **Security audit for unencrypted protocols:** Filter for `http or telnet or ftp` to find all cleartext traffic. Any matches indicate passwords or sensitive data being transmitted without encryption.

- **Latency diagnosis:** Filter for `ip.src == 192.168.92.4 and tcp.flags.syn == 1` to see connection initiation packets, then measure the time until the corresponding SYN-ACK arrives. High latency between SYN and SYN-ACK indicates network congestion or an overloaded server.

- **Retransmission analysis:** Filter for `tcp.analysis.retransmission` to see packets that had to be resent due to loss or corruption. Frequent retransmissions indicate network quality problems.

Filters transform packet captures from raw data dumps into diagnostic tools that answer specific questions about network behavior.

## Findings

- **Network interfaces serve distinct roles based on their configuration.** Eth0 handles internet-bound traffic through NAT, eth1 provides isolated communication with vulnerable VMs, and the loopback interface enables internal system communication without involving physical hardware. This separation prevents attack traffic from leaking to production networks.

- **Packet capture reveals protocol structure and communication patterns.** Tcpdump and Wireshark show how the TCP/IP model wraps data in successive layers. HTTP sits on top of TCP, which sits on top of IP, which sits on top of Ethernet frames. Each layer adds its own header information for routing, reliability, or error detection.

- **Interface statistics diagnose physical and logical network problems.** Zero errors and dropped packets indicate healthy hardware. High error counts point to cable issues or failing network cards. Connection state information (SYN_SENT, ESTABLISHED, TIME_WAIT) reveals whether services are running and whether TCP handshakes are completing successfully.

- **ICMP ping and TCP connections use different mechanisms for different purposes.** Ping uses a simple request-reply pattern without reliability guarantees, making it ideal for connectivity testing. TCP uses a three-way handshake and sequence numbers to guarantee ordered, reliable delivery, making it suitable for data transfer.

- **Disabling and re-enabling interfaces clears stuck states.** Bringing an interface down kills all active connections and flushes buffers. Bringing it back up forces the system to reinitialize the driver and request a new IP address if using DHCP. This isolates software failures from hardware failures.

- **Bandwidth monitoring detects anomalies and informs capacity planning.** Normal web browsing consumes minimal bandwidth (kilobits per second). Sudden spikes to megabits or gigabits indicate data exfiltration, misconfigured services, or network congestion. Sustained high usage reveals when an interface upgrade is needed.

- **Packet filters transform captures into diagnostic tools.** Filtering by IP address, port number, protocol, or TCP flags isolates the conversation of interest. This makes it possible to find specific errors, measure latency, audit for unencrypted protocols, or troubleshoot failed connections in captures containing thousands of packets.

## Challenges Faced

- **Distinguishing between interface roles:** Initially, I was confused about why three interfaces were necessary. It became clear that each serves a different purpose: eth0 for internet access, eth1 for isolated lab work, and lo for internal system communication. Understanding this separation helped me grasp why attack traffic should never use the same interface as normal browsing.

- **Interpreting netstat output:** The netstat command showed UNIX domain sockets instead of TCP/IP connections. I initially expected to see TCP connections listed, but UNIX sockets bypass the network stack entirely. This taught me that not all inter-process communication uses TCP/IP, even on networked systems.

- **Correlating OSI model layers to real packets:** The OSI model is abstract, but Wireshark shows exactly which bytes belong to which layer. Seeing the Ethernet header, IP header, TCP header, and HTTP payload as separate sections in the packet dissection made the layered architecture concrete instead of theoretical.

## Key Takeaways

- **Network interfaces are more than just IP addresses.** They operate at Layer 2 of the OSI model, handling MAC addressing, frame encapsulation, error detection, and physical transmission. Understanding this prevents the mistake of thinking IP addresses are the only important identifier.

- **Packet capture is a troubleshooting superpower.** Being able to see exactly what packets are sent, what responses are received, and where communication breaks down eliminates guesswork. Wireshark shows whether a connection fails because the server never responded, because the response was malformed, or because packets are being dropped.

- **Connection states tell the story of network communication.** ESTABLISHED means data is flowing, SYN_SENT means waiting for a response, TIME_WAIT means the connection closed cleanly. These states are visible in netstat and Wireshark and reveal what is happening at the TCP layer.

- **Ping and TCP serve different purposes.** Ping tests basic connectivity with minimal overhead. TCP provides reliable data transfer with handshakes, sequence numbers, and acknowledgments. Use ping to confirm routes work, use TCP for actual data transfer.

- **Interface troubleshooting requires isolating layers.** If toggling an interface fixes the problem, it was software-related (stuck state, bad ARP cache). If toggling does not fix it, the problem is physical (cable, port, hardware). This saves time by ruling out entire categories of failure.

- **Bandwidth monitoring reveals what normal looks like.** Once you know normal traffic patterns (5-10 Kb for idle, 100 Kb for active browsing), deviations become obvious. A workstation uploading 50 Mbps is abnormal and warrants investigation.

## Disclaimer

This lab was performed in a controlled environment using virtual machines configured specifically for security training. The OWASP Broken Web Applications VM is an intentionally vulnerable system designed for educational use. No unauthorized systems were accessed.
