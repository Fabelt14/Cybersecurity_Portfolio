# Anonymity Testing with Tor and Proxychains

---

## 1. Engagement Overview

This lab assessed the anonymity properties and operational limitations of routing
network traffic through the Tor network using Proxychains on Kali Linux. The
assessment covered Tor service configuration, IP masking verification, circuit behavior under repeated requests, Nmap scanning performance through Tor, and DNS leak risk. Testing was conducted in a live internet-connected environment
using `httpbin.org` for IP verification and a public AWS host (`44.228.249.3`) for scan comparison. Tor v0.4.8.16 and Proxychains-ng 4.17 were used throughout.

---

## 2. Objectives

- Confirm that Tor successfully masks the real source IP address during
  outbound connections
- Measure and compare scan performance with and without Tor to quantify
  the operational cost of anonymised scanning
- Observe Tor's dynamic circuit management behavior across repeated
  connections
- Verify that browser and command-line traffic both exit through the same
  Tor node to confirm there are no tool-level leaks
- Document the technical limitations of Tor and Proxychains in a
  penetration testing context, including protocol restrictions and exit node risks

---

## 3. Scope

**In-Scope:**
- Tor service on local Kali Linux machine (`127.0.0.1:9050`)
- Proxychains configuration file (`/etc/proxychains.conf`)
- Outbound HTTP/HTTPS connections to `httpbin.org` via `curl`
- Nmap TCP scan against public AWS host `44.228.249.3`
- Firefox browser IP verification via whatismyip

**Out-of-Scope:**
- Dark web (.onion) service access
- Active exploitation of any scanned host
- UDP-based scanning or ICMP traffic (technically not supported by Tor)

**Authorization Statement:**
> All testing in this assessment was conducted using publicly accessible IP
> verification services and a public AWS host for scan comparison purposes.
> No systems were exploited and no unauthorized access was attempted.
> All activities were performed in an isolated Kali Linux lab environment for
> educational purposes under authorized course instruction.

---

## 4. Methodology

### Phase 1 — Tool Installation and Service Configuration
Tor and Proxychains were installed using `apt`. The Tor service was started
via `sudo service tor start` and its status was confirmed with
`systemctl status tor`. The Proxychains configuration at `/etc/proxychains.conf`
was reviewed, with `dynamic_chain` enabled and the SOCKS5 proxy set to
`127.0.0.1:9050`. DNS leak prevention was confirmed active via the
`proxy_dns` directive in the same config file.

### Phase 2 — Baseline IP Identification
The real public IP address was recorded using two methods: the
`whatismyip.com` web interface and a direct `curl https://httpbin.org/ip`
command. Both returned `102.89.68.117`, confirming the baseline non-anonymous
IP associated with the local ISP.

### Phase 3 — Anonymity Verification
Proxychains was prepended to the same curl command:
`proxychains curl https://httpbin.org/ip`. The returned IP was compared
against the baseline to confirm the traffic exited through a Tor node rather
than the local ISP connection.

### Phase 4 — Circuit Behavior Testing
The proxychains curl command was repeated multiple times without restarting
Tor to observe whether exit IP addresses remained consistent or rotated across
separate connections. Results were compared to assess Tor's circuit reuse
behavior.

### Phase 5 — Browser Consistency Check
Firefox was used to access a browser-based IP check service while Proxychains
was active. The displayed IP was compared against the curl output to confirm
no application-level leak existed between browser and command-line traffic.

### Phase 6 — Nmap Scan Performance Comparison
An Nmap TCP Connect scan was run against `44.228.249.3` twice: once through
Proxychains and once directly, using identical flags (`-sT -Pn`). Completion
times were recorded for both runs to quantify the performance cost of scanning
via Tor.

### Phase 7 — Risk and Limitation Analysis
Tor exit node risks, DNS leak vectors, and the consequences of using Tor with
authenticated personal accounts were assessed based on observed behavior and
technical properties of the Tor protocol.

---

## 5. Vulnerability Summary

> **Note:** This lab is an operational security and anonymity assessment rather
> than a vulnerability scan against a target system. The findings below document
> confirmed anonymity behaviors, measurable limitations, and residual risks
> identified during testing.

| ID | Finding | Severity | Context |
|----|---------|----------|---------|
| 01 | Tor Successfully Masks Real IP | Informational | proxychains curl - httpbin.org |
| 02 | Exit IP Rotates Across Requests (Circuit Reuse) | Informational | Repeated curl via Proxychains |
| 03 | Nmap Scan 4.7x Slower Through Tor (31.43s vs 6.63s) | Medium | Nmap -sT -Pn against 44.228.249.3 |
| 04 | Tor Restricted to TCP - UDP and ICMP Not Supported | Medium | Proxychains Nmap -sU / ping |
| 05 | Unencrypted HTTP Traffic Readable at Exit Node | High | Tor exit node risk - HTTP traffic |
| 06 | DNS Leak Possible if proxy_dns Not Enabled | High | /etc/proxychains.conf - DNS config |

---

## 6. Detailed Findings

---

### Finding 01 — Tor Successfully Masks Real IP Address

#### Severity
Informational

#### Affected Component
Outbound HTTP connections via `proxychains curl`

#### Description
When `proxychains` was prepended to the curl command, all outbound traffic
was routed through the Tor SOCKS5 proxy at `127.0.0.1:9050` before reaching
`httpbin.org`. The IP returned by the service was a Tor exit node address, not
the local ISP address. The exit node IP (`109.70.100.6`) belongs to a Tor relay
operator in a different country from the real source, confirming the ISP identity
and geographic location were both masked.

#### Proof of Concept

Baseline IP confirmed without Tor:



![curl httpbin.org/ip Without Tor - Real IP 102.89.68.117](image.jpg)



Proxychains curl output showing Tor exit node IP instead of real IP:



![proxychains curl httpbin.org/ip - Exit Node IP 109.70.100.6](image.jpg)



Output comparison:
```

Without Tor:  "origin": "102.89.68.117"
With Tor:     "origin": "109.70.100.6"
```

The Proxychains terminal also confirmed the routing path:
```
[proxychains] Dynamic chain ... 127.0.0.1:9050 ... httpbin.org:443
```

#### Impact
Tor routing successfully prevents a destination server from identifying the
original source IP address. From the server's perspective, the connection
originates from the Tor exit node. This is the expected behavior and confirms
the basic anonymity function works as intended.

#### Remediation
This is a confirmed working control, not a vulnerability. To maintain this
protection in practice, all traffic must be routed through Proxychains without
exception. Any tool or application that does not respect the proxy configuration
will leak the real IP.

---

### Finding 02 — Exit IP Rotates Across Repeated Requests

#### Severity
Informational

#### Affected Component
Tor circuit management - repeated `proxychains curl` sessions

#### Description
Running the same `proxychains curl https://httpbin.org/ip` command multiple
times without restarting Tor returned different exit IPs across separate
sessions. The IPs observed across runs included `109.70.100.6`,
`185.220.101.24`, and `192.42.116.210`. This confirms that Tor builds new
circuits periodically rather than permanently committing traffic to a single
exit node. The rotation is managed automatically by Tor's circuit management
and does not require any manual intervention.

#### Proof of Concept

Three consecutive proxychains curl runs showing different exit IPs:



![Three Consecutive proxychains curl Requests - Rotating Exit IPs](image.jpg)



Exit IPs observed:
```

Run 1: "origin": "109.70.100.6"
Run 2: "origin": "185.220.101.24"
Run 3: "origin": "192.42.116.210"
```

#### Impact
IP rotation prevents a destination server from building a consistent picture of
activity from a single address. However, rotation is not guaranteed on every
single request. Tor reuses circuits for a period to avoid breaking active sessions.
If a session requires login state or persistent connection, the circuit is kept alive
longer, meaning the exit IP stays fixed for that session's duration.

#### Remediation
For operational use, forcing a new Tor identity via `sudo service tor restart`
guarantees a new circuit and exit node when strict IP change is needed. This
should be done between distinct activities rather than mid-session.

---

### Finding 03 — Nmap Scan 4.7x Slower Through Tor

#### Severity
Medium

#### Affected Component
Active scanning via `proxychains nmap -sT -Pn 44.228.249.3`

#### Description
The same Nmap TCP Connect scan was run against `44.228.249.3` with and
without Proxychains. The scan through Tor completed in 31.43 seconds. The
same scan run directly completed in 6.63 seconds. The latency penalty
introduced by routing through multiple Tor relays before reaching the target
produced a 4.7x slowdown on a single-host scan of a small port range. Scanning
thousands of ports across a full subnet through Tor would produce proportionally
longer times, making large-scale reconnaissance impractical in time-sensitive
engagements.

#### Proof of Concept

Nmap scan through Proxychains:



![Nmap via Proxychains - 31.43 Seconds Scan Time](image.jpg)



Direct Nmap scan without Proxychains for comparison:



![Nmap Direct - 6.63 Seconds Scan Time](image.jpg)



Both scans returned identical open ports, confirming the results were accurate
despite the time difference:
```

PORT     STATE  SERVICE
80/tcp   open   http
587/tcp  open   submission
```

Scan times:
```
Via Proxychains: Nmap done: 1 IP address (1 host up) scanned in 31.43 seconds
Direct:          Nmap done: 1 IP address (1 host up) scanned in 6.63 seconds
```

#### Impact
The latency penalty is manageable for single-host, limited-port scans. It becomes
a practical problem in engagements that require scanning wide port ranges or
multiple hosts. A full 65535-port scan that takes 10 minutes directly could take
nearly an hour via Tor. Planning must account for this if anonymised scanning
is a requirement.

#### Remediation
Limit Tor-routed scanning to the minimum necessary scope. Run initial
reconnaissance directly or through a VPS to identify live hosts and relevant port
ranges, then use Tor only for targeted follow-up scans where source anonymity
is specifically required.

---

### Finding 04 — Tor Restricted to TCP Traffic Only

#### Severity
Medium

#### Affected Component
Proxychains + Nmap - UDP scanning and ICMP ping

#### Description
Tor routes TCP traffic only. UDP packets and ICMP traffic cannot be proxied
through Tor or Proxychains. In practice, this means UDP scans (`nmap -sU`)
and standard ping-based host discovery are not available when routing through
Proxychains. Nmap must use `-sT` (TCP Connect) instead of `-sS` (SYN stealth),
which requires completing a full three-way TCP handshake for every port tested.
A completed handshake is more visible in server logs than a half-open SYN
scan, reducing the stealth benefit that anonymised scanning is meant to provide.

#### Proof of Concept

Nmap scan command used through Proxychains:
```

proxychains nmap -sT -Pn 44.228.249.3
```

The `-Pn` flag was required because ICMP ping through Proxychains is not
supported. Without it, Nmap would report the host as down. The `-sT` flag was
mandatory because SYN scans require raw socket access that cannot pass
through a SOCKS5 proxy.

#### Impact
The protocol restriction limits the types of reconnaissance that can be performed
anonymously. UDP services (DNS on port 53, SNMP on port 161, etc.) cannot be
scanned through Tor. Host discovery via ICMP is unavailable, so the port list
and host targets must be predetermined before routing through Proxychains.
The full three-way handshake required by `-sT` also leaves more complete log
entries at the target than a stealth SYN scan would.

#### Remediation
Accept the TCP-only restriction as a design property of Tor rather than a
misconfiguration. UDP enumeration should be conducted through a separate
channel if needed. When using `-sT` through Proxychains, combine it with
timing flags (`-T2` or slower) to reduce the density of log entries at the target.

---

### Finding 05 — Unencrypted HTTP Traffic Readable at Tor Exit Node

#### Severity
High

#### Affected Component
Tor exit node operator visibility - HTTP traffic

#### Description
Tor encrypts traffic between the client and the exit node, but the exit node
decrypts the traffic before forwarding it to the destination server. For HTTP
connections (port 80, unencrypted), the exit node operator can read the full
request content including any credentials, session tokens, or personal data in
the HTTP body. Additionally, a malicious exit node can perform a
man-in-the-middle attack, injecting content into HTTP responses or
downgrading HTTPS connections if the client does not enforce strict TLS
certificate checking. The exit node does not know the source IP, but it can
see everything the destination sees.

#### Proof of Concept

The Nmap scan confirmed port 80/tcp open on the test host:
```

80/tcp  open  http
```

Any HTTP request sent to this or any other HTTP endpoint via Tor exits at the
Tor relay in cleartext before reaching the server. The exit node operator
receives the full plaintext HTTP request.

#### Impact
A user who sends credentials, API keys, or personal data over HTTP while
believing Tor provides complete privacy has a false sense of security. The
geographic and ISP identity is protected, but the content of unencrypted
communications is not. This is a frequent misunderstanding about what Tor
actually protects.

#### Remediation
Use HTTPS for all connections routed through Tor without exception. Enforce
HTTPS by installing browser extensions like HTTPS Everywhere or enabling
strict HTTPS-only mode in Firefox. Avoid submitting credentials or sensitive
data over any HTTP endpoint, regardless of whether Tor is in use.

---

### Finding 06 — DNS Leak Possible Without proxy_dns Directive

#### Severity
High

#### Affected Component
`/etc/proxychains.conf` - DNS resolution behavior

#### Description
By default, DNS resolution can occur outside the proxy chain if the application
performs a DNS lookup before establishing the proxied TCP connection. Without
the `proxy_dns` directive enabled in `/etc/proxychains.conf`, DNS queries are
sent directly to the system resolver over the real network connection, not through
Tor. This means the ISP's DNS server receives a query for every hostname
resolved, creating a log of all destinations visited even when the actual
connection content is hidden. During this lab, `proxy_dns` was confirmed active
in the configuration file, mitigating this risk for the current session.

#### Proof of Concept

`/etc/proxychains.conf` configuration showing `proxy_dns` enabled and the
SOCKS5 proxy entry pointing to Tor:



![proxychains.conf - proxy_dns and SOCKS5 127.0.0.1:9050 Configuration](image.jpg)



Configuration entries confirmed:
```

proxy_dns
socks5  127.0.0.1  9050
tcp_read_time_out  15000
tcp_connect_time_out  8080
```

A DNS leak in the absence of `proxy_dns` would allow the ISP to log every
resolved hostname, enabling timing correlation between the DNS request
timestamps and Tor connection timestamps to identify visited destinations.

#### Impact
DNS leaks are one of the most common causes of de-anonymisation in
proxy-routed sessions. If an investigator has access to ISP DNS logs and can
observe that a DNS query for a sensitive hostname was made from a specific IP
at the same time an anonymous Tor connection reached that host, correlation is
straightforward. `proxy_dns` forces DNS resolution through Tor, preventing
the ISP from seeing hostname queries.

#### Remediation
Confirm `proxy_dns` is present and uncommented in `/etc/proxychains.conf`
before any anonymised session. Periodically test for DNS leaks using services
such as `dnsleaktest.com` routed through Proxychains to verify no out-of-band
DNS queries are being made during active Tor sessions.

---

## 7. Tools Used

- Kali Linux (lab environment)
- Tor v0.4.8.16-1
- Proxychains-ng 4.17
- curl (IP verification via `httpbin.org/ip`)
- Nmap 7.98 (TCP connect scanning)
- Firefox (browser-based IP verification)

---

## 8. Challenges Encountered

- **Tor circuit reuse made IP comparison across runs inconsistent:** Tor does
  not guarantee a new exit node on every single request. Some consecutive curl
  runs returned the same exit IP, while others rotated. Waiting briefly between
  runs or restarting Tor produced more consistent rotation for the purpose of
  demonstrating the behavior.
- **Nmap host discovery disabled by Tor's ICMP restriction:** Running Nmap
  without the `-Pn` flag caused it to report the target as down, because the
  default ping probe could not pass through Proxychains. Adding `-Pn` bypassed
  the host discovery phase entirely and proceeded directly to port scanning.
- **Systemd service status output cut off in terminal:** The `systemctl status tor`
  output was paginated, requiring pressing the spacebar to view the full output.
  Piping to `cat` or using `--no-pager` produced the full output in one block
  for documentation purposes.

---

## 9. Key Takeaways

- **Tor protects identity at the network level, not the content level:** The
  verified IP change from `102.89.68.117` to `109.70.100.6` confirms identity
  masking works as expected. What Tor does not protect is the content of
  unencrypted connections after they leave the exit node. Treating Tor as a
  full privacy solution without using HTTPS leads to the wrong conclusions
  about what is actually protected.
- **proxy_dns is a required configuration, not optional:** A correctly configured
  IP proxy that still leaks DNS queries provides much weaker protection than it
  appears to. Enabling `proxy_dns` is a one-line configuration change that
  closes a significant de-anonymisation vector.
- **The 4.7x scan slowdown has real operational consequences:** The difference
  between 6.63 and 31.43 seconds on a tiny scan scales badly. Anonymised
  scanning is practical only when the target scope is deliberately small and
  pre-defined. Large-scale discovery and enumeration do not fit within the
  performance constraints Tor imposes.
- **Authenticated sessions destroy anonymity regardless of Tor:** Logging into
  a personal account over Tor gives the service provider a direct link between
  the Tor circuit and a verified identity. This is not a Tor failure — it is the
  correct outcome of providing credentials. Tor cannot anonymise an action the
  user intentionally identifies themselves for.
- **Chain mode selection has a direct tradeoff between reliability and
  detection resistance:** Strict chain is the most predictable but breaks on any
  proxy failure. Dynamic chain keeps the connection alive but reduces the
  number of hops when proxies drop out. Random chain produces the least
  consistent traffic pattern from a logging perspective, at the cost of being
  unpredictable about which path the traffic actually takes.

---

## 10. Disclaimer

> This assessment was conducted within an isolated Kali Linux lab environment
> for educational purposes only. All IP verification was performed against publicly
> accessible services (`httpbin.org`, `whatismyip.com`). The Nmap scan target
> (`44.228.249.3`) is a public AWS infrastructure IP used solely for scan
> timing comparison — no exploitation or unauthorized access was attempted
> against any host. Tor and Proxychains were used within the bounds of normal
> educational security training. These tools must only be used in contexts where
> applicable laws and explicit authorization permit their use.

---