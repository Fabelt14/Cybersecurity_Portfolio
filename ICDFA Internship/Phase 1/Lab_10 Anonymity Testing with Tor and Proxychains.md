# Anonymity Testing with Tor and Proxychains

## Overview

This lab tested anonymity tools used to hide source IP addresses during reconnaissance and security testing. Tor routes traffic through multiple encrypted relays before exiting to the internet, while Proxychains forces applications to use proxy servers. The goal was to verify IP masking works correctly, understand the trade-offs between anonymity and functionality, and identify scenarios where these tools break.

## Objectives

- Configure Tor and Proxychains to anonymize network traffic
- Compare proxy chain modes and their resilience to failures
- Verify IP address masking through external services
- Test anonymity consistency across different applications (curl, Firefox, Nmap)
- Measure performance impact of routing scans through Tor
- Identify limitations and risks of anonymity networks

## Lab Environment

- **Attacker Machine:** Kali Linux
- **Anonymity Tools:** Tor (v0.4.8.16-1), Proxychains
- **Testing Services:** httpbin.org/ip (external IP lookup)
- **Network:** Direct internet connection (baseline) vs Tor network

## Tools Used

- Tor
- Proxychains
- curl
- Firefox browser
- Nmap
- systemctl

## Methodology

### Step 1: Installation and Service Verification

I started by installing Tor and Proxychains using apt. Both were already at the latest versions on Kali Linux, so the installation confirmed existing packages rather than downloading new ones.



![Tor and Proxychains installation confirmation](screenshots/tor_proxychains_install.png)



After installation, I started the Tor service with `sudo service tor start` and checked its status with `systemctl status tor`. The output showed "active (exited)" with process 3197, which means Tor successfully initialized and is running in the background.

![Tor service status showing active state](screenshots/tor_service_status.png)



**Why verify service status:** If Tor is not actually running, Proxychains will try to connect to 127.0.0.1:9050 and fail silently. The connection will fall back to direct routing, exposing my real IP without any error message. Always confirm the service is active before testing anonymity.

### Step 2: Understanding Proxy Chain Modes

Before using Proxychains, I examined the three chain modes in `/etc/proxychains.conf` to understand how they handle proxy failures.



![Proxychains configuration showing chain mode options](screenshots/proxychains_config.png)



**Strict Chain:** Forces traffic through proxies in exact order (A→B→C). If proxy B is down, the entire connection fails. This provides predictable routing but no fault tolerance. Use when you need to ensure traffic passes through specific nodes in sequence.

**Dynamic Chain:** Also follows the proxy list in order, but automatically skips unresponsive proxies. If B is down, traffic routes through A→C instead. Connection succeeds as long as at least one proxy works. This is more reliable than strict chain but less predictable.

**Random Chain:** Selects proxies randomly from the list for each connection. One request might use A→C, the next C→A→B, another just B alone. This makes traffic patterns extremely difficult to correlate because there is no consistent routing path. Best for evading traffic analysis.

I enabled dynamic chain for this lab because it provides fault tolerance while maintaining some order.

### Step 3: Baseline IP Identification

Before testing anonymity, I established my real IP address by querying httpbin.org/ip without any proxying. This gives me a reference point to confirm whether Tor is actually masking my identity.

```bash
curl https://httpbin.org/ip
```



![Real IP address shown as 102.89.68.117](screenshots/baseline_real_ip.png)



**Result:** My real IPv4 address is 102.89.68.117. This is associated with my ISP and my actual geographic location in Nigeria. Any connections from this IP can be directly traced back to my network.

### Step 4: Anonymity Verification with Proxychains

I ran the same IP check command through Proxychains to route it through Tor:

```bash
proxychains curl https://httpbin.org/ip
```



![Proxychains showing Tor exit IP as 109.70.100.6](screenshots/proxychains_tor_ip.png)



**Result:** The exit IP appeared as 109.70.100.6, completely different from my real IP. The Proxychains debug output showed:
- Config file loaded from /etc/proxychains.conf
- Dynamic chain mode active
- Connection routed through 127.0.0.1:9050 (Tor's SOCKS5 proxy)
- Final destination: httpbin.org:443

**What this confirms:** My traffic successfully traveled through the Tor network before reaching httpbin.org. The website only sees the Tor exit node's IP (109.70.100.6), not my real IP (102.89.68.117). My geographic location and ISP identity are hidden from the target.

### Step 5: Tor Circuit Persistence and IP Rotation

To understand how Tor manages connections over time, I ran the same IP check multiple times in quick succession.

```bash
proxychains curl https://httpbin.org/ip
proxychains curl https://httpbin.org/ip
```



![Multiple requests showing IP rotation](screenshots/tor_circuit_rotation.png)



**First request:** 185.220.101.24  
**Second request:** 192.42.116.210

**Result:** The exit IP changed between requests. This happened because Tor builds new circuits periodically to prevent long-term tracking. Each circuit uses a different set of relays and a different exit node.

**Why IP rotation is not guaranteed on every single request:** Tor tries to maintain the same circuit for a short period (usually 10 minutes) to keep sessions stable. If my IP changed every 5 seconds while logged into a website, the server would detect multiple IPs accessing the same session and flag it as suspicious or terminate the session entirely. Tor balances anonymity with usability by rotating circuits gradually, not constantly.

### Step 6: Forcing Circuit Renewal

To test whether I can manually force a new exit IP, I restarted the Tor service and immediately checked my IP again.

```bash
sudo service tor restart
proxychains curl https://httpbin.org/ip
```

**Result:** The exit IP changed after the restart. This confirms that restarting Tor destroys the current circuit and builds a new one with different relays and a different exit node. However, restarting the service is disruptive. In real operations, you can send a NEWNYM signal to Tor (`sudo pkill -HUP tor`) to request a new circuit without fully restarting the service.

### Step 7: Browser-Based Anonymity Consistency

Command-line tools prove Tor works at the network layer, but browsers have additional components (WebRTC, DNS, plugins) that can leak identity. I tested Firefox to confirm the browser also routes through Tor correctly.

I configured Firefox to use a SOCKS5 proxy at 127.0.0.1:9050, then navigated to a "What is my IP" website.



![Firefox showing Tor exit IP matching curl results](screenshots/firefox_tor_ip.png)



**Result:** Firefox displayed the same Tor exit IP as the curl test. This confirms consistency across applications. If curl showed a Tor IP but Firefox showed my real IP, that would indicate a browser leak (DNS leaking outside the proxy, WebRTC exposing local IP, etc.).

**Why consistency matters:** If different applications use different IP addresses simultaneously, an attacker can correlate the timing of connections to de-anonymize me. For example, if curl connects to httpbin.org at 19:13:16 from IP 185.220.101.24, and Firefox connects to the same site at 19:13:17 from IP 102.89.68.117, an attacker can match the timestamps and deduce that both connections came from the same person. Consistency proves there are no leaks.

### Step 8: Scanning Through Tor (Performance Impact)

To measure the trade-off between anonymity and speed, I ran an Nmap scan against an AWS EC2 instance both with and without Tor.

**Scan without Tor:**
```bash
nmap -sT -Pn 44.228.249.3
```

**Result:** Completed in 6.63 seconds

**Scan through Tor:**
```bash
proxychains nmap -sT -Pn 44.228.249.3
```



![Nmap scan through Tor taking 31.43 seconds](screenshots/nmap_tor_scan_time.png)



**Result:** Completed in 31.43 seconds

**Analysis:** Tor added approximately 24.8 seconds of overhead for scanning just two open ports (80/tcp and 443/tcp). This is a 474% increase in scan time. Scaling this to a full network scan with thousands of ports across multiple hosts would take hours instead of minutes.

**Why Tor is slow for scanning:**
1. **Multi-hop latency** - Traffic passes through at least three Tor relays (entry, middle, exit) before reaching the target. Each hop adds round-trip time.
2. **TCP-only limitation** - Tor only supports TCP. Nmap must use `-sT` (TCP Connect scan) which completes a full three-way handshake for every port. Stealth scans like SYN scan (`-sS`) send a single packet and analyze the response without completing the connection, which is faster and harder to detect.
3. **No UDP support** - Proxychains cannot route UDP traffic through Tor, so DNS scans (`-sU`) and ICMP pings are impossible.

**Detection risk:** TCP Connect scans through Tor are easier to detect than direct SYN scans. The target's firewall logs show complete connection attempts from the Tor exit node's IP, which is often already flagged as suspicious. Direct SYN scans from a clean IP are stealthier because they appear as random network noise rather than deliberate probing.

### Step 9: Risk and Limitation Analysis

**Exit Node Visibility Risk**

Tor encrypts traffic between me and the exit node, but the exit node itself can see unencrypted traffic after it leaves the Tor network. If I visit an HTTP website (not HTTPS), the exit node operator can capture my passwords, session cookies, and any data I submit.

**Real-world scenario:** In 2007, a security researcher named Dan Egerstad ran Tor exit nodes and captured embassy email credentials because diplomats were accessing webmail over HTTP through Tor. The exit node saw everything in plaintext.

**Mitigation:** Only visit HTTPS websites when using Tor. The TLS encryption protects data from the exit node to the destination server. The exit node can see which website I'm visiting (the domain name in the Server Name Indication field), but not the content of the communication.

**DNS Leak Risk**

If DNS requests bypass Tor and go directly to my ISP's DNS server, my ISP sees every website I look up. Even though the actual HTTP traffic is anonymized, the DNS log creates a complete browsing history.

**Timing correlation attack:** An investigator can match the timestamp of my DNS query for "example.com" (visible to my ISP) with the timestamp of a Tor connection to example.com's IP address (visible to the website). This correlation reveals my identity.

**Mitigation:** Configure DNS to route through Tor as well. In `/etc/proxychains.conf`, uncomment `proxy_dns` to force DNS queries through the SOCKS proxy. Alternatively, use a DNS-over-HTTPS resolver that respects the Tor circuit.

**Authenticated Account Risk**

Logging into personal accounts (Gmail, Facebook, bank) while using Tor creates a permanent link between my anonymous Tor circuit and my real identity. The moment I authenticate, the service provider knows exactly who I am.

**Additional risk:** Many platforms flag Tor logins as suspicious because Tor exit nodes are often associated with malicious activity. Google may require additional verification, Facebook may lock the account, banks may freeze transactions. The account itself becomes evidence of Tor usage.

**Operational security principle:** Never mix anonymous and authenticated activities on the same circuit. If reconnaissance requires anonymity, do not log into personal accounts during that session.

## Findings

**Successful IP Masking**

- Real IP (102.89.68.117) was successfully hidden behind Tor exit nodes (109.70.100.6, 185.220.101.24, 192.42.116.210). External services only saw the exit node's IP, not my actual location or ISP.

**Circuit Rotation Behavior**

- Tor automatically rotates exit IPs over time to prevent long-term tracking, but maintains circuits for approximately 10 minutes to preserve session stability. Manual circuit renewal (service restart or NEWNYM signal) forces immediate IP change.

**Performance Degradation**

- Nmap scans through Tor are 474% slower than direct scans (31.43 seconds vs 6.63 seconds for two ports). This makes Tor impractical for large-scale reconnaissance.

**Protocol Limitations**

- Tor only supports TCP traffic through SOCKS5. UDP scans, ICMP pings, and raw packet crafting are impossible through Proxychains. Nmap must use TCP Connect scan (`-sT`) instead of stealth SYN scan.

**Exit Node Visibility**

- Exit nodes can see all unencrypted (HTTP) traffic. HTTPS is mandatory to protect data from exit node operators.

**DNS Leak Potential**

- Without proper configuration, DNS queries bypass Tor and expose browsing history to ISP. The `proxy_dns` option in Proxychains configuration prevents this leak.

## Challenges Faced

- **Proxychains debug output confusion:** When running Proxychains for the first time, the terminal displayed several lines of debug information (config file path, DLL init messages, proxy chain selection) before showing the actual curl output. I initially thought these were errors, but they are normal status messages. The actual command output appeared after the "127.0.0.1:9050" connection confirmation. Reading the output carefully revealed that `[proxychains]` prefixes indicate debug messages, not errors.

- **Nmap compatibility with Tor:** My first attempt to scan through Tor used `proxychains nmap -sS` (SYN scan), which failed with "socket operation failed" errors. I learned that Tor only supports TCP, not raw sockets required for SYN scans. The solution was switching to `-sT` (TCP Connect scan), but this made the scan slower and easier to detect. This taught me that anonymity tools constrain attack techniques.

- **Browser proxy configuration persistence:** After testing with Firefox, I forgot to remove the SOCKS5 proxy settings. When I tried to browse normally later, websites timed out because Tor was no longer running. Firefox gave no clear error about the proxy failure. I had to manually check Network Settings and clear the SOCKS proxy configuration. Lesson: Always document configuration changes and revert them after testing.

## Key Takeaways

- **Anonymity requires verification at every layer.** Testing with both curl and Firefox confirmed no leaks occurred. If I had only tested curl, a browser DNS leak would have gone undetected. Always validate anonymity across all applications that will be used during operations.

- **Circuit persistence balances anonymity and functionality.** Tor does not rotate IPs on every request because constant IP changes break authenticated sessions and trigger security alerts. Understanding this trade-off helps set realistic expectations for anonymity tools.

- **Exit nodes are trusted third parties.** The Tor network provides excellent anonymity from the target and from my ISP, but the exit node operator is a blind spot. Always use HTTPS to protect data in transit from the exit node to the destination.

- **Anonymity adds significant overhead.** A 474% increase in scan time is acceptable for targeted reconnaissance where stealth matters more than speed, but Tor is unsuitable for large-scale network mapping. Anonymity tools should be used strategically, not universally.

- **DNS leaks break anonymity silently.** Without `proxy_dns` enabled, my ISP sees every domain I look up even though HTTP traffic is hidden. DNS logging creates timing correlation opportunities that can de-anonymize Tor users.

## Disclaimer

This lab was performed on an authorized Kali Linux system for educational purposes only. External IP lookup services (httpbin.org) and scan targets were accessed legally. No unauthorized networks were tested. Tor and Proxychains were used solely to understand anonymity mechanisms and limitations, not for malicious activity.
