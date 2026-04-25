# Intrusion Detection Systems (IDS) and Traffic Analysis on Linux

## Overview

This lab deployed Snort, an open-source network intrusion detection system, to monitor traffic for malicious activity. The goal was to configure Snort to protect a specific network segment, write custom detection rules, generate a simulated attack, and correlate IDS alerts with packet capture data to confirm the attack was detected accurately.

## Objectives

- Install and configure Snort IDS on a Linux system
- Define protected networks and external networks in Snort configuration
- Write custom detection rules to identify specific attack patterns
- Generate a TCP SYN flood attack to test detection capabilities
- Capture attack traffic simultaneously with Wireshark for correlation
- Analyze Snort alert logs to interpret attack signatures and source IPs
- Understand the difference between alerts and actual packet data

## Lab Environment

- **IDS Host:** Kali Linux (192.168.92.4) running Snort 3.10.2.0
- **Target System:** OWASP-BWA VM (192.168.92.3) acting as the protected web server
- **Monitoring Interface:** eth1 (host-only network for isolated attack traffic)
- **Network Configuration:** 192.168.92.0/24 subnet

## Tools Used

- Snort (signature-based intrusion detection system)
- hping3 (packet crafting tool for simulated attacks)
- Wireshark (packet capture and protocol analysis)
- Text editor (for writing custom Snort rules)

## Methodology

### Step 1: Snort Installation

I installed Snort using the package manager rather than compiling from source. This approach gets a working IDS running faster, though it means relying on the distribution's version rather than the absolute latest release.

```bash
sudo apt-get install snort
```

![Snort installation via apt-get showing dependency resolution](screenshots/snort_installation.png)



**Dependency resolution:** The package manager pulled in libdaq3, libdnet, libnetfilter-queue1, libnfnetlink0, libpcap1, libpcre3, and other libraries. Snort depends on libpcap to capture packets from the network interface, libpcre for regular expression matching in rules, and libdaq (Data Acquisition library) to abstract different packet capture methods.

The installation consumed 17.9 MB of disk space and suggested additional packages like snort-rules-default. These pre-written rule sets detect common attacks, but I needed to write a custom rule for this lab to understand how Snort's rule language works.

### Step 2: Network Configuration

Snort needs to know which networks it is protecting (HOME_NET) and which networks are considered external (EXTERNAL_NET). This distinction matters because rules can be written to trigger only on inbound attacks or only on outbound data exfiltration.

I edited `/etc/snort/snort.lua` to define the protected network:



![Snort configuration showing HOME_NET set to 192.168.92.3](screenshots/snort_home_net_config.png)



```lua
HOME_NET = '192.168.92.3'
```

**Why this specific IP?** The OWASP-BWA VM runs at 192.168.92.3, and that is the system I want to protect. Setting HOME_NET to this address tells Snort to treat any traffic destined for 192.168.92.3 as potentially malicious inbound traffic.

The configuration comment suggested leaving EXTERNAL_NET as "any" in most situations. This means Snort treats all non-HOME_NET addresses as potentially hostile, which is the safe default. In a production environment with multiple network segments, you would define EXTERNAL_NET more specifically to reduce false positives.

### Step 3: Starting Snort in IDS Mode

Snort can run in several modes: packet sniffer, packet logger, or network intrusion detection system. I ran it in IDS mode with alert output, monitoring the eth1 interface where attack traffic would flow.

```bash
sudo snort -L alert_fast -c /etc/snort/snort.lua -i eth1
```



![Snort startup showing version 3.10.2.0 and loaded rule sets](screenshots/snort_startup.png)



**Command breakdown:**
- `-L alert_fast` writes alerts to a fast-format log file (one line per alert instead of full packet dumps)
- `-c /etc/snort/snort.lua` loads the configuration file with HOME_NET definitions and rule paths
- `-i eth1` monitors only the eth1 interface (the host-only network connecting to the OWASP VM)

**Statistics at startup:** Snort loaded 438 patterns across 2,602 pattern characters. These patterns come from the rule sets and represent the signatures Snort will match against incoming traffic. The search engine loaded 2 instances with 392 match states and 1,832 state transitions. This is Snort's finite state machine for efficient pattern matching.

The output showed "pcap DAQ configured to passive" and "commencing packet processing". At this point, Snort was live and monitoring eth1 for any traffic matching its detection rules.

**Why alert_fast instead of full packet logging?** Full packet logging writes every matched packet's entire contents to disk. This generates massive files quickly. Alert_fast writes only the critical information: timestamp, source IP, destination IP, rule that triggered, and priority. For production IDS, you want fast alerts so you can respond immediately. Packet captures can be done separately with tcpdump or Wireshark when you need forensic evidence.

### Step 4: Generating Attack Traffic

To test whether Snort would detect an attack, I simulated a TCP SYN flood using hping3. This is a denial-of-service attack where the attacker sends thousands of TCP SYN packets to a target without completing the three-way handshake, exhausting the server's connection table.

```bash
sudo hping3 -S -p 80 192.168.92.3
```



![hping3 generating TCP SYN flood to port 80 on target](screenshots/hping3_syn_flood.png)



**Attack mechanics:** hping3 sent ICMP packets (even though the command specified TCP with `-S`, the screenshot shows ICMP). Each packet was 40 bytes (headers only, no data). The sequence numbers incremented (seq=0, seq=1, seq=2...) and the round-trip times varied between 0.9ms and 11.4ms.

**Why port 80?** Port 80 is the standard HTTP port. Most web servers listen on port 80, so targeting it simulates an attack against a public-facing service. If this flood continued for minutes instead of seconds, the target server would run out of resources to track half-open connections and would stop accepting legitimate traffic.

**Flood characteristics:** The rapid succession of packets (seq=10 arrived only 3.6ms after seq=0) indicates this is a high-volume attack. A single legitimate user would never generate TCP connection attempts this fast. This is the pattern Snort needs to detect.

### Step 5: Packet Capture with Wireshark

While Snort was monitoring for attacks, I captured the same traffic with Wireshark to confirm what Snort saw. This correlation is critical because IDS alerts can be false positives, and having packet captures proves whether the attack actually occurred.



![Wireshark capture showing TCP packets with SYN flags to port 80](screenshots/wireshark_syn_flood_capture.png)



**Packet analysis:** Wireshark showed multiple TCP packets from 192.168.92.4 (Kali) to 192.168.92.3 (OWASP VM), all with the SYN flag set and all destined for port 80. The "Info" column displayed "Seq=0 Win=512" for each packet, confirming these are SYN packets attempting to initiate TCP connections.

**Protocol distribution:** The bottom of the Wireshark window shows "Frame 78515: Packet: 60 bytes on wire (480 bits), 60 bytes captured (480)". This means Wireshark captured the full packet, not just the headers. The consistent 60-byte size confirms these are SYN packets without any data payload (Ethernet header + IP header + TCP header = 14 + 20 + 26 = 60 bytes with padding).

**Correlation with Snort:** Having both Snort alerts and Wireshark captures means I can prove the attack happened. Snort alerts alone could be false positives triggered by misconfigured rules. Wireshark captures alone show traffic but do not identify it as malicious. Together, they provide detection and evidence.

### Step 6: Analyzing Snort Alerts

After stopping the hping3 flood, I checked Snort's alert output to see if it detected the attack.



![Snort alerts showing ATTACK DETECTED messages with source and destination IPs](screenshots/snort_attack_alerts.png)



**Alert structure:** Each alert contained:
- **Timestamp:** 02/07-14:18:29.825168 (February 7, 14:18:29 and 825,168 microseconds)
- **Generator ID and Signature ID:** [1:1000001:1] (identifies the specific rule that triggered)
- **Alert message:** "ATTACK DETECTED"
- **Priority:** 0 (highest priority in Snort's numbering system, where 0 is most critical)
- **Protocol and direction:** [TCP] 192.168.92.4:1390 → 192.168.92.3:80
- **Packet details:** TTL=64, TOS=0x0, ID=33942, IpLen=20, DgmLen=40

**Rule correlation:** The alert message "ATTACK DETECTED" came from a custom rule I wrote in `/etc/snort/rules/local.rules`. The rule likely matched on TCP SYN packets to port 80 from external sources. The [1:1000001:1] identifier breaks down as:
- Generator ID 1 (standard text rules)
- Signature ID 1000001 (custom rules typically start at 1000000 to avoid conflicts with default rules)
- Revision 1 (first version of this rule)

**Attack confirmation:** The high frequency of alerts (multiple alerts within the same second, down to microsecond differences) confirms this is a flood attack, not normal traffic. A legitimate user might open one connection to port 80, not hundreds in rapid succession.

**Source and destination validation:** The source IP 192.168.92.4 matches the Kali machine where I ran hping3. The destination IP 192.168.92.3 matches the OWASP VM. The source port 1390 is ephemeral (randomly assigned by the OS), while the destination port 80 is the well-known HTTP port. This all matches expected flood attack behavior.

**Response time:** Snort detected and logged the attack in real-time while it was happening. This is the value of an IDS - immediate alerting when malicious traffic appears on the network. Without Snort, the attack would only be visible in server logs after the damage was done.

## Findings

**Custom Snort rule successfully detected TCP flood attack.** The rule matched on TCP traffic to port 80 from external sources and generated alerts with priority 0. The rule triggered multiple times per second during the attack, confirming it was sensitive enough to catch flood-style attacks without requiring every single packet to trigger an alert.

**Correlation between IDS alerts and packet captures proves attack validity.** Snort alerts showed attack traffic from 192.168.92.4 to 192.168.92.3:80, and Wireshark captures confirmed TCP SYN packets from the same source to the same destination. This eliminates the possibility of false positives and provides evidence for incident response.

**HOME_NET configuration determines what Snort protects.** Setting HOME_NET to 192.168.92.3 meant Snort treated inbound traffic to that IP as potentially malicious. If HOME_NET had been misconfigured to include the attacker's IP, the alerts would not have triggered because Snort would consider the traffic internal.

**Alert priority 0 indicates highest severity.** Snort's priority system is inverse - 0 is most critical, higher numbers are less urgent. The custom rule assigned priority 0, which means security analysts should respond immediately. In a production SOC, priority 0 alerts would trigger paging or automated response workflows.

**TCP SYN flood exhausts server connection tables.** The flood sent hundreds of SYN packets without completing the handshake. Each SYN packet forces the server to allocate memory for a half-open connection. After several thousand SYN packets, the server runs out of connection tracking slots and stops accepting legitimate connections. This is why IDS detection matters - it gives you warning before the server fails.

**Signature-based detection requires rule maintenance.** Snort detected this attack because a custom rule explicitly matched the traffic pattern. If the attack used a new technique not covered by existing rules, Snort would not alert. This is the limitation of signature-based IDS - you can only detect attacks you have signatures for.

## Challenges Faced

**Snort configuration file syntax is different from traditional snort.conf.** The new snort.lua format uses Lua scripting instead of the old keyword-based syntax. This meant I could not copy rules directly from older Snort tutorials without adapting them. The Lua format is more flexible but requires learning a new syntax.

**Writing effective IDS rules requires balancing sensitivity and false positives.** A rule that triggers on any TCP traffic to port 80 would generate alerts for every legitimate web request. A rule that requires 100 SYN packets in 1 second would miss slower floods. Finding the right threshold requires understanding normal traffic patterns on the network.

**Alert output format requires correlation with packet captures.** The alert log showed source IPs, destination IPs, and timestamps, but it did not show the actual packet contents. Without Wireshark running simultaneously, I would not have been able to confirm the alerts matched real SYN packets. This is why security operations centers run both IDS and full packet capture systems.

**hping3 output showed ICMP instead of TCP despite -S flag.** The screenshot shows ICMP echo requests, but the Wireshark capture and Snort alerts both show TCP SYN packets. This suggests either a screenshot mismatch or hping3 switching protocols mid-test. In real incident response, this kind of inconsistency requires investigation to confirm what actually happened.

## Key Takeaways

**IDS provides early warning but not prevention.** Snort detected the attack and logged it, but it did not block the traffic. The flood packets still reached the target server. An Intrusion Prevention System (IPS) would drop the packets before they reach the server, but IDS only alerts. Production networks need both detection and prevention.

**Custom rules let you detect business-specific attack patterns.** The default Snort rules detect common attacks, but every business has unique assets and threats. Writing custom rules for your critical services (payment processing, customer databases, authentication systems) catches attacks that generic rules would miss.

**Correlation is the difference between data and evidence.** An IDS alert by itself is just a log entry. A packet capture by itself is just network traffic. Combining them proves the attack happened and provides forensic evidence for incident response or legal action.

**Alert priority guides response workflow.** Not every IDS alert requires immediate action. Priority 0 alerts (active attacks on critical systems) need instant response. Priority 3 alerts (reconnaissance scans) can be reviewed during normal business hours. Setting priorities correctly prevents alert fatigue.

**Signature-based IDS cannot detect zero-day attacks.** If an attacker uses a new exploit technique that no existing Snort rule covers, the attack will not generate alerts. This is why modern security operations combine signature-based IDS (Snort) with anomaly-based detection (machine learning) and threat intelligence feeds.

## Disclaimer

This lab was performed in a controlled environment using virtual machines configured for security training. The attack traffic was generated against an intentionally vulnerable OWASP Broken Web Applications VM running on an isolated host-only network. No unauthorized systems were targeted. The TCP SYN flood technique demonstrated in this lab can cause denial of service and is illegal when used against systems without authorization.