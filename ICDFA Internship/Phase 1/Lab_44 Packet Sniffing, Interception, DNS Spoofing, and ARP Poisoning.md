# Packet Sniffing, Interception, DNS Spoofing, and ARP Poisoning

## Overview

This lab covered two connected attack techniques used in local network exploitation. The first half focused on packet sniffing and interception using Wireshark and tcpdump to capture credentials and session data from an intentionally vulnerable banking application. The second half escalated to active attacks: ARP poisoning to establish a Man-in-the-Middle position, followed by DNS spoofing to redirect a victim to a fake login page and harvest credentials in a controlled environment.

## Objectives

- Compare how different sniffing tools (tcpdump, Wireshark, NetworkMiner) capture and process network traffic
- Capture and extract sensitive data (credentials, session tokens) from unencrypted traffic
- Analyze legitimate versus spoofed DNS responses at the packet level
- Execute an ARP poisoning attack to intercept traffic between a victim and the router
- Configure DNS spoofing to redirect a target domain to an attacker-controlled IP
- Host a fake login page to demonstrate credential harvesting risk
- Identify the forensic signatures that distinguish spoofed traffic from legitimate traffic

## Lab Environment

- **Attacker Machine:** Kali Linux (network interface eth0)
- **Victim Machine:** Windows host on the same LAN
- **Target Application:** vulnbank.org (intentionally vulnerable banking application)
- **Network:** Local area network with attacker at 192.168.42.135, victim at 192.168.42.101, router/gateway at 192.168.42.129

## Tools Used

- Wireshark 4.6.0
- tcpdump 4.99.5
- Bettercap v2.41.5
- Apache (web server for hosting fake login page)
- HTML/CSS (fake login page)

## Methodology

### Part 1: Packet Sniffing and Interception

Before capturing any traffic, I confirmed both tools were correctly installed and checked version numbers to rule out compatibility issues later.



![Wireshark and tcpdump version confirmation](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_01A%20Wireshark%20and%20tcpdump%20version%20confirmation.jpg)

![Wireshark and tcpdump version confirmation](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_01B%20Wireshark%20and%20tcpdump%20version%20confirmation.jpg)

**Comparing sniffing tools:**

- Tcpdump operates close to the kernel using the Berkeley Packet Filter. Filters like `tcp port 80` get applied directly inside the kernel, so non-matching packets get discarded before they reach userspace. This keeps tcpdump fast and lightweight because it does minimal decoding, printing raw text lines or dumping straight to a pcap file.

- Wireshark works differently. It uses dissectors, which are small modules that each understand one protocol layer. The Ethernet dissector reads MAC addresses, the IP dissector reads source and destination addresses, the TCP dissector reconstructs sequence numbers, and the HTTP or DNS dissector decodes the actual payload into readable text. This is why clicking a packet in Wireshark expands a layered tree structure instead of a single line of text.

- NetworkMiner takes a completely different approach. Instead of organizing data by packet, it organizes by host and artifact. It performs automatic TCP stream reassembly, meaning it reconstructs full files (images, emails, documents) from fragmented packet data and writes the completed file to disk. This makes NetworkMiner useful for extracting evidence quickly without manually following streams packet by packet.

**Capturing live traffic:**

I targeted vulnbank.org, an intentionally vulnerable banking application, browsing it from a Windows host while capturing traffic on Kali.


![VulnBank dashboard accessed during capture](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_02%20VulnBank%20dashboard%20accessed%20during%20capture.jpg)


Filtering the capture for HTTP traffic revealed the login POST request in plaintext.

![Wireshark capture showing HTTP traffic and frame details](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_03%20Wireshark%20capture%20showing%20HTTP%20traffic%20and%20frame%20details.jpg)

**Extracted sensitive data:**

The login request exposed the username and password directly in the JSON body, with no encryption protecting the credentials in transit.

![Extracted username, password, and session cookie](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_04%20Extracted%20username%2C%20password%2C%20and%20session%20cookie.jpg)

- Username: Prime0x
- Password: 12345
- Session cookie: JWT token returned in the Set-Cookie header

The JWT token is particularly dangerous because possessing it allows session fixation. An attacker does not need the password at all if they can capture this token and replay it before it expires.

**Header analysis:**

Documenting source and destination addressing confirmed the traffic path:

- Source IP: 192.168.42.6
- Destination IP: 172.67.134.11
- Source MAC: ba:cc:f4:e8:c7:67
- Destination MAC: 22:bb:15:bc:18:a0

The MAC addresses matter for a different reason than the IP addresses. While IP addresses identify the logical destination, MAC addresses identify the physical next hop on the local network. This distinction becomes critical later in the ARP poisoning phase, since spoofing MAC-to-IP mappings is exactly how the Man-in-the-Middle position gets established.

### Part 2: Analyzing Pre-Captured DNS Spoofing Traffic

Before launching a live attack, I analyzed a sample pcap file showing what a DNS spoofing attack looks like at the packet level.


![DNS query and spoofed response packets in Wireshark](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_05%20DNS%20query%20and%20spoofed%20response%20packets%20in%20Wireshark.jpg)

![DNS query and spoofed response packets in Wireshark](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_05B%20DNS%20query%20and%20spoofed%20response%20packets%20in%20Wireshark.jpg)


- **Packet 1:** The victim (192.168.88.135) sends a standard DNS query to Google's public resolver (8.8.8.8) asking for the IP address of facebook.com.

- **Packet 2:** The legitimate response arrives, resolving facebook.com to its real CDN address (31.13.66.36) through a CNAME chain.

- **Packet 3:** A second, unrequested response arrives claiming to be from 8.8.8.8, but this one resolves facebook.com to 127.0.1.1, a loopback address. This is the spoofed packet. The attacker raced the legitimate DNS server and won, forcing the victim's machine to accept the forged answer instead of waiting for the real one.

### Part 3: Executing the DNS Spoofing Attack

With the theory confirmed against sample data, I moved to executing the attack live using Bettercap.

![Bettercap version confirmation](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_06%20Bettercap%20version%20confirmation.jpg)

**Step 1: Capture the ARP table before the attack.**

I checked the victim's ARP table first to establish a baseline. The router (192.168.42.129) showed its correct MAC address, distinct from the Kali machine's MAC address.

![Victim ARP table before the attack](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_07%20Victim%20ARP%20table%20before%20the%20attack.jpg)

**Step 2: Launch ARP poisoning.**

I started Bettercap on the eth0 interface, enabled network probing to discover live hosts on the LAN, then targeted the victim's IP address with the ARP spoofing module.

![Bettercap ARP spoofing module running against the victim](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_08%20Bettercap%20ARP%20spoofing%20module%20running%20against%20the%20victim.jpg)

**Step 3: Confirm the poisoned ARP table.**

Checking the victim's ARP table again after the attack showed the router's IP address now mapped to the Kali machine's MAC address. The victim's operating system believes it is talking to the router, but every packet is physically going to the attacker first.

![Victim ARP table after the attack showing the MAC address change](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_09%20Victim%20ARP%20table%20after%20the%20attack%20showing%20the%20MAC%20address%20change.jpg)

**Step 4: Configure DNS spoofing.**

With the MITM position established, I configured Bettercap's DNS spoofing module to intercept any query for vulnbank.com and respond with the attacker's IP address instead of the real one.

![Bettercap DNS spoof configuration targeting vulnbank.com](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_10%20Bettercap%20DNS%20spoof%20configuration%20targeting%20vulnbank.com.jpg)


**Step 5: Host the fake login page.**

I built a simple HTML/CSS clone of the VulnBank login page and hosted it locally so any victim redirected by the DNS spoof would land on a convincing fake.

![Fake VulnBank login page source code and rendered result](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_11A%20Fake%20VulnBank%20login%20page%20source%20code%20and%20rendered%20result.jpg)

![Fake VulnBank login page source code and rendered result](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_11B%20Fake%20VulnBank%20login%20page%20source%20code%20and%20rendered%20result.jpg)

**Step 6: Verify the attack in Wireshark.**

Analyzing the resulting traffic confirmed the full attack chain. Packets 691 and 692 show the victim's DNS query for vulnbank.org heading toward the router. Because of the ARP poisoning, these packets physically arrived at Kali first. Packet 693 is the forged response, claiming to come from the router's IP but containing the attacker's IP in the answer field.

![Wireshark capture showing victim query and spoofed response](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_12%20Wireshark%20capture%20showing%20victim%20query%20and%20spoofed%20response.jpg)



**The critical forensic detail:** The Source IP in packet 693 claims to be the router (192.168.42.129), but the Ethernet frame's Source MAC address belongs to the Kali machine. A legitimate DNS response would have the router's true MAC address at Layer 2. This mismatch between the claimed Layer 3 identity and the actual Layer 2 hardware address is the definitive signature of a spoofed packet, and it cannot be faked without also compromising the actual gateway.

### Part 4: Confirming the Attack Chain Logic

- ARP poisoning and DNS spoofing are not independent attacks. ARP poisoning operates at Layer 2 and exists purely to establish physical interception. DNS spoofing operates at Layer 7 and exists to manipulate what the victim's browser connects to. Neither one is useful alone for this scenario. Without ARP poisoning, the attacker has no way to see the victim's DNS query in the first place. Without DNS spoofing, intercepting the traffic does nothing because the victim would still resolve the correct IP address from the real DNS server.

- **Why ARP is exploitable:** The protocol has two structural weaknesses. First, there is no authentication. Any device can claim to own any IP-to-MAC mapping and other devices have no way to verify the claim. Second, ARP is stateless, meaning a machine will accept an unsolicited ARP reply even if it never sent a corresponding request. Attackers exploit this by sending gratuitous ARP replies that overwrite legitimate entries in the victim's cache without ever being asked.

### Part 5: Risk Analysis and Prevention Strategies

Having executed the full attack chain, the next step was reflecting on what makes it work conceptually and how a network defender would actually stop it.

**Breaking down the three components:**

- Man-in-the-Middle is the overarching concept, not a specific technique. It just means an attacker has positioned themselves between two parties who believe they are talking directly to each other. ARP poisoning and DNS spoofing are two specific techniques that achieve this goal, but they operate at completely different layers.

- ARP poisoning works at Layer 2 (Data Link). Its only job is to corrupt the victim's understanding of which MAC address belongs to the gateway, forcing traffic to physically route through the attacker's network card.

- DNS spoofing works at Layer 7 (Application). Its job is to corrupt which IP address a domain name resolves to, redirecting the victim's browser to a server the attacker controls.

**Why the chain only works in this order:** ARP poisoning has to happen first because DNS spoofing depends on the attacker being able to see the victim's DNS query before the real DNS server responds. Without the physical interception ARP poisoning provides, the attacker has no visibility into the query at all and cannot win the race to respond first. This is why the lab structured the attacks sequentially rather than treating them as independent.

**Technical prevention measures:**

- For ARP poisoning, the strongest network-level defense is Dynamic ARP Inspection (DAI) on managed switches. DAI cross-references every ARP packet against a trusted DHCP snooping binding table and drops anything that does not match a known, legitimate IP-to-MAC pairing. For smaller or more static environments, administrators can hardcode static ARP entries for critical infrastructure like the default gateway, which prevents that specific mapping from ever being dynamically overwritten regardless of how many forged replies an attacker sends.

- For DNS spoofing, the equivalent defense is DNSSEC. It adds cryptographic signatures to DNS records so the resolver can verify a response actually came from the authoritative server rather than being forged in transit. DNS over HTTPS and DNS over TLS add a different layer of protection by encrypting the query itself, which prevents an on-path attacker from even reading the query well enough to respond to it, regardless of whether they can intercept the packets.

**User-level and application precautions:**

- Even if ARP poisoning and DNS spoofing both succeed, SSL/TLS certificates provide a last line of defense. The attacker can redirect the victim to their own server, but they cannot present a valid certificate for a domain they do not own. A browser enforcing strict certificate validation will throw a warning the moment the victim lands on the fake page, assuming the user does not click through it.

- HSTS closes a related gap. Without it, an attacker performing the MITM can strip the connection down to plain HTTP before the victim ever gets the chance to see a certificate warning at all. HSTS forces the browser to refuse anything except HTTPS for that domain, removing the attacker's ability to downgrade the connection in the first place.

### Part 6: Forensic Verification and Real-World Risk Assessment

**Confirming attacker and victim identity in the capture:**

Before drawing conclusions from the traffic, I documented the exact addresses involved so the analysis could be tied to specific evidence rather than assumption.

- Victim IP: 192.168.42.101
- Victim MAC: be:a9:58:9c:4f:8f
- Attacker IP: 192.168.42.135
- Attacker MAC: 08:00:27:c5:94:f1

![Attacker MAC address](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_13%20Attacker%20MAC%20address.jpg)

**Three ways to tell a spoofed DNS response from a legitimate one:**

- The first and most reliable method is checking for a Layer 2/Layer 3 mismatch. A legitimate DNS response from the gateway will have the Ethernet frame's source MAC address match the gateway's actual hardware address. In the spoofed packet, the source IP claimed to be the gateway (192.168.42.129) but the source MAC address belonged to the Kali machine. This mismatch cannot occur naturally and is the strongest single indicator of a forged packet.

![Ethernet frame showing claimed gateway IP paired with attacker MAC address](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/44_14%20Ethernet%20frame%20showing%20claimed%20gateway%20IP%20paired%20with%20attacker%20MAC%20address.jpg)

- The second method is payload analysis. A spoofed response resolved vulnbank.org to an internal, local network address (192.168.42.135), while a real public domain should resolve to a publicly routable internet address. Seeing a private RFC 1918 address in the answer field for what should be a public-facing domain is itself a red flag independent of any MAC address check.

- The third method is timing and duplication. Because the attacker sits on the local network, their forged response arrives almost instantly after the query goes out. The legitimate response, traveling across the actual internet, arrives slightly later carrying the same transaction ID but a completely different resolved IP. Seeing two responses to the same query with different answers is direct evidence that one of them is fraudulent, regardless of which one arrived first.

**Reflection on sensitive data extraction:**

- No credentials, session tokens, or cookies were extracted during the live DNS spoofing portion of this lab. This was intentional. The fake login page included a JavaScript `preventDefault()` call that blocked the actual form submission from firing, so even though the victim was successfully redirected to the spoofed page, nothing typed into it ever left the browser. This was a deliberate safety boundary to keep the live exercise non-destructive while still proving the redirection itself worked. The earlier passive sniffing exercise in Part 1 already demonstrated what real credential extraction looks like against unencrypted traffic, so this second exercise focused specifically on proving the MITM and spoofing mechanics rather than repeating the harvesting step.

**Ethical and legal considerations:**

- In a real attack where the failsafe was not present, the consequences scale quickly. Stolen usernames and passwords lead directly to account takeover, giving an attacker access to banking, corporate, or personal accounts without needing to break any further security controls. If the attacker captures an active session token instead of (or alongside) the password, they can hijack the session entirely, impersonating the victim without ever needing to log in themselves. Because people frequently reuse passwords across services, credentials harvested from one spoofed site often get tested against unrelated accounts through credential stuffing, which means the damage from a single successful attack rarely stays contained to the one application that was targeted.


## Findings

- **Unencrypted traffic exposes credentials in plaintext.** The VulnBank login request transmitted the username, password, and session token without any transport encryption, allowing complete extraction through passive sniffing alone, before any active attack was launched.

- **ARP poisoning successfully redirected all victim traffic through the attacker.** After running Bettercap's ARP spoof module, the victim's ARP table showed the router's IP address mapped to the attacker's MAC address. This confirms the victim's outbound traffic, including DNS queries, was physically routing through Kali instead of the legitimate gateway.

- **DNS spoofing redirected a real domain to an attacker-controlled server.** Configuring Bettercap to intercept queries for vulnbank.com resulted in the victim's browser loading a fake login page while the address bar continued to display the correct domain name, with no visible warning to the user.

- **The Layer 2/Layer 3 mismatch is the only reliable detection signature.** Both the legitimate and spoofed DNS responses claimed the same source IP address. The only way to tell them apart in the packet capture was checking whether the Ethernet frame's source MAC address actually belonged to the claimed sender.

- **Timing reveals the race condition.** The spoofed response from the local attacker arrived faster than the legitimate response traveling across the internet. In the live capture, the forged answer (packet 693) appeared before the real DNS server's response (packet 707), confirming the attacker won the race by virtue of physical proximity.

## Challenges Faced

- **No credentials were captured from the live phishing page.** I deliberately configured the fake login page's JavaScript with `e.preventDefault()` to block the actual HTTP POST request from firing. This meant the simulated credential capture stayed entirely client-side and nothing sensitive ever left the victim's browser during the live exercise. This was a safety decision to avoid handling real credential data even in a lab setting, but it also meant I could not directly demonstrate the final harvesting step the way the earlier sniffing exercise did.

- **Distinguishing two DNS responses with identical claimed sources required careful frame inspection.** At first glance, both the legitimate and spoofed responses in the capture appeared to come from the same source IP. Only by expanding the Ethernet II frame details and comparing the MAC address against the known router MAC did the spoofing become provable rather than assumed.

## Key Takeaways

- **Passive sniffing alone can fully compromise an account if traffic is unencrypted.** No active attack was needed to extract the VulnBank credentials and session token. HTTPS is not optional protection, it is the only thing standing between a sniffer and a stolen account.

- **ARP has no concept of trust, which makes it trivially exploitable on any shared network.** The protocol accepts unsolicited replies and never verifies identity. Any device on the same LAN segment can claim to be the gateway, and nothing in the protocol itself stops this.

- **DNS spoofing is invisible to the user without certificate validation.** The browser's address bar showed the correct domain name throughout the attack. The only thing that would have exposed the fake site to a vigilant user is an HTTPS certificate warning, which makes HSTS and strict certificate checking far more valuable than URL inspection.

- **MAC address verification is the most reliable forensic check against spoofing.** IP addresses can be claimed by anyone in a packet header, but Ethernet frame source addresses are harder to fake convincingly without also controlling the legitimate device. When investigating suspected spoofing, checking Layer 2 against known-good hardware addresses is the first move.

- **Defense requires layering, not a single control.** Dynamic ARP Inspection stops the poisoning at the switch level, DNSSEC and DoH/DoT stop the spoofing at the resolver level, and HSTS plus strict TLS stop credential theft at the application level. Relying on just one of these leaves the other attack surfaces open.

## Disclaimer

This lab was performed in a controlled, isolated environment against an intentionally vulnerable application (vulnbank.org) created for security training. All ARP poisoning and DNS spoofing activity was conducted against a designated lab victim machine on a closed network. No unauthorized systems were accessed, and the fake login page was configured to prevent any actual credential transmission as an additional safety measure.
