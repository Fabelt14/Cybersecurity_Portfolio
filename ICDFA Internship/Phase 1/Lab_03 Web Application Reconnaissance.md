# Web Application Reconnaissance on Live Domains and OWASP BWA

---

## 1. Overview

This lab covers web application reconnaissance techniques performed against three live Nigerian domains and a local OWASP Broken Web Applications (OWASP BWA) virtual machine. The goal was to map the attack surface of each target by identifying IP addresses, domain ownership, DNS records, running technologies, subdomains, open directories, and open ports before any exploitation attempt.

Reconnaissance is the first step in any security engagement. You cannot test what you do not know, and you cannot defend what you have not mapped. This lab documents how that mapping process works in practice.

---

## 2. Objectives

- Resolve IP addresses for target domains using ping
- Perform WHOIS lookups to identify domain ownership and hosting infrastructure
- Query DNS records (A, MX, NS) to understand how each domain is configured
- Fingerprint web technologies using WhatWeb
- Enumerate subdomains using Sublist3r and Knockpy, then compare results
- Discover exposed directories using Dirb
- Scan open ports and services using Nmap
- Run a vulnerability scan using Nikto
- Build a custom Bash reconnaissance tool
- Explain the GET/POST request model and how DNS resolves domain names

---

## 3. Lab Environment

| Component | Detail |
|---|---|
| Attacker Machine | Kali Linux |
| Target (Live Domains) | icdfa.edu.ng, nasarawastate.gov.ng, icdfa.org.ng |
| Target (Local) | OWASP BWA (192.168.92.3) |
| Network Type | Host-only / NAT (for OWASP BWA) |
| Virtualization | Oracle VirtualBox |

---

## 4. Tools Used

- ping
- whois
- nslookup
- WhatWeb
- Sublist3r
- Knockpy
- Dirb
- Nmap
- Nikto
- Bash scripting

---

## 5. Methodology

### Part 1: IP Resolution, WHOIS, and DNS Lookup

#### Step 1: Identify IP Addresses

The first task was to resolve the IP address of each domain. I used `ping -c 5` rather than a dedicated tool because ping confirms both name resolution and basic reachability in one step. The packet loss result tells you whether there is a stable connection to the host, useful context before running heavier tools.

**icdfa.edu.ng**

![Image](link)

icdfa.edu.ng resolved to **131.153.147.186**, hosted on wghservers.com (Whogohost infrastructure). Zero percent packet loss confirmed a stable connection at the time of the scan.

**nasarawastate.gov.ng**

![Image](link)

nasarawastate.gov.ng resolved to **181.215.243.156**, hosted on qservers.net (Qservers infrastructure). Zero percent packet loss confirmed reachability.

**icdfa.org.ng**

![Image](link)

icdfa.org.ng resolved to **131.153.147.186**: the same IP address as icdfa.edu.ng. This tells us both domains share the same server, which is a common setup when an organization runs multiple domain extensions pointing to one hosting account.

**Summary**

| Domain | IP Address |
|---|---|
| icdfa.edu.ng | 131.153.147.186 |
| nasarawastate.gov.ng | 181.215.243.156 |
| icdfa.org.ng | 131.153.147.186 |

---

#### Step 2: WHOIS Lookups

WHOIS lookups go one level deeper than IP resolution. The goal here was to find out who manages the IP block, which organization owns the infrastructure, and whether any registrant details are publicly exposed. I ran WHOIS against the resolved IP addresses rather than the domain names because IP-based WHOIS gives infrastructure ownership details, not just registrar records.

**icdfa.edu.ng: WHOIS on 131.153.147.186**

![Image](link)

![Image](link)

The IP block is managed by ARIN (American Registry for Internet Numbers). The parent organization is Secured Servers LLC. The lookup referred to an rwhois server at securedservers.com, which identified the specific network block as a /29 subnet (131.153.147.184 – 131.153.147.191). The registrant is listed as "Private Customer", a standard configuration hosting providers use to shield end-user identity from public WHOIS results.

**nasarawastate.gov.ng: WHOIS on 181.215.243.156**

![Image](link)

The IP block is also managed by ARIN, with the parent organization listed as ORG-HTL23-RIPE. The registrant is again listed as "Private Customer," consistent with the same privacy configuration seen on the ICDFA domains.

**icdfa.org.ng: WHOIS on 131.153.147.186**

![Image](link)

![Image](link)

The results mirror the icdfa.edu.ng lookup exactly, confirming both domains sit on the same /29 subnet managed by Secured Servers LLC.

---

#### Step 3: DNS Lookups (A, MX, NS Records)

DNS queries expose how a domain routes traffic, web requests, email, and name resolution. I queried three record types for each domain. The A record confirms the IP address. The MX record shows whether the domain handles its own email. The NS record identifies which hosting company manages DNS for the domain.

**icdfa.edu.ng**

![Image](link)

- A record: 131.153.147.186
- MX record: No mail servers returned. The organization uses external mail services rather than hosting email on this domain.
- NS record: nsc.go54.com and nsd.go54.com. managed by Go54 (formerly WhoGoHost)

**nasarawastate.gov.ng**

![Image](link)

- A record: 181.215.243.156
- MX record: Mail exchanger = 0, meaning no dedicated mail servers are configured under this domain
- NS record: ns77.qservers.net and ns78.qservers.net. managed by Qservers

**icdfa.org.ng**

![Image](link)

- A record: 131.153.147.186
- MX record: No mail servers returned
- NS record: nsc.go54.com and nsd.go54.com, same as icdfa.edu.ng, confirming both domains are under the same DNS management

**Summary of DNS Findings**

| Domain | IP Address | Mail Servers | Name Servers |
|---|---|---|---|
| icdfa.edu.ng | 131.153.147.186 | 0 | nsc.go54.com, nsd.go54.com |
| nasarawastate.gov.ng | 181.215.243.156 | 0 | ns77.qservers.net, ns78.qservers.net |
| icdfa.org.ng | 131.153.147.186 | 0 | nsc.go54.com, nsd.go54.com |

---

### Part 2: Technology Fingerprinting with WhatWeb

Before scanning, I identified the OWASP BWA machine's IP address as **192.168.92.3**.

WhatWeb fingerprints web technologies by analyzing HTTP headers, server banners, and page content. I ran both a basic scan and an aggressive scan to see whether increasing the scan intensity would reveal additional information.

**Basic Scan**

![Image](link)

**Aggressive Scan**

![Image](link)

Both scans returned the same core output: identical server banners, HTTP headers, web server version, and framework details. The aggressive scan produced more verbose output when run with the `-v` flag, but the underlying findings were the same. Since the lab task specified an aggressive scan only (not a verbose comparison), the meaningful conclusion is that the target exposes its full technology stack even at basic scan intensity, which is itself a finding.

---

### Part 3: Website Enumeration and Information Gathering

![Image](link)

**Server Banners and HTTP Headers**

- Server Banner: Apache/2.2.14 (Ubuntu)
- Detailed Headers: mod_mono/2.4.3, mod_perl/2.0.4, mod_python/3.3.1, mod_ssl/2.2.14
- Security Header: OpenSSL/0.9.8k

**Web Server, Framework, and CMS**

- Web Server: Apache 2.2.14 running on Ubuntu Linux
- Framework: PHP 5.3.2, Python 2.6.5, Perl 5.10.1, JavaScript 1.3.2

**Visible Misconfigurations and Information Leakage**

The server exposes its full software stack through the response header, PHP version, Python version, Perl version, and OpenSSL version are all publicly visible. This is a misconfiguration because version numbers tell an attacker exactly which CVEs to look up.

The WhatWeb scan also harvested multiple email addresses from the application: admin@metacorp.com, bob@ateliergraphique.com, test@thebodgeitstore.com, among others. Exposed internal email addresses are useful for phishing and social engineering; they give an attacker real targets rather than guesses.

---

### Part 4: Subdomain Enumeration and Tool Comparison

Subdomains are entry points that organizations often forget to secure. Subdomain enumeration maps the full scope of a target's web presence, not just the main domain, but every application, portal, or service running under it.

I ran two tools against each domain and compared their results.

#### icdfa.edu.ng

**Sublist3r**

![Image](link)

Sublist3r uses passive enumeration: it queries search engines (Google, Baidu, Yahoo, Bing) and SSL certificate databases to find subdomains without sending requests directly to the target. It found **30 unique subdomains**.

**Knockpy**

![Image](link)

Knockpy uses active brute-force enumeration: it tests a wordlist of potential subdomain names directly against the domain's DNS. It also validates each result by checking HTTP/HTTPS status, TLS certificates, and IP addresses. It found **33 subdomains**.

**Result: Knockpy found more subdomains for icdfa.edu.ng.**

---

#### nasarawastate.gov.ng

**Sublist3r**

![Image](link)

Sublist3r found **42 unique subdomains**.

**Knockpy**

![Image](link)

Knockpy found **69 subdomains**.

**Result: Knockpy found significantly more subdomains for nasarawastate.gov.ng.**

---

#### icdfa.org.ng

**Sublist3r**

![Image](link)

Sublist3r found **37 unique subdomains**.

**Knockpy**

![Image](link)

Knockpy found **1 subdomain**.

**Result: Sublist3r found far more subdomains for icdfa.org.ng.**

The reversal here is worth noting. Knockpy's brute-force approach depends heavily on how many subdomains are actually resolvable via DNS. For icdfa.org.ng, most subdomains exist in SSL certificate records (which Sublist3r picks up) but do not resolve in DNS, so Knockpy finds almost nothing. This is a good example of why using a single tool gives an incomplete picture.

---

**Tool Comparison**

| | Sublist3r | Knockpy |
|---|---|---|
| Method | Passive: queries search engines and SSL databases | Active: brute-forces DNS with a wordlist |
| Validation | Limited: may list non-resolvable domains | Strong: validates HTTP/HTTPS status, TLS certs, and IP |
| Best used for | Quick, broad reconnaissance | Deep enumeration of resolvable subdomains |

---

### Part 5: Directory Enumeration with Dirb

Directory enumeration tests whether the web server exposes internal folders that should not be publicly accessible. I ran Dirb against the OWASP BWA machine at http://192.168.92.3 using the default wordlist.

![Image](link)

![Image](link)

Dirb scanned 4612 generated words and discovered multiple accessible directories. The most significant finding was that directory listing is enabled across several paths, meaning anyone who knows (or guesses) a directory path can browse its contents like a file manager. One notable example is /assets/. A directory listing of this kind is a direct path to sensitive file exposure.

---

### Part 6: Port Scanning and Vulnerability Detection

#### Nmap Scan

I ran a basic Nmap scan against 192.168.92.3 to map all open ports and identify what services are running on each.

![Image](link)

**Open Ports and Services**

| Port | Service |
|---|---|
| 22 | SSH |
| 80 | HTTP |
| 139 | NETBIOS-SSN |
| 143 | IMAP |
| 443 | HTTPS |
| 445 | Microsoft-DS |
| 5001 | Complex-Link |
| 8080 | HTTP-Proxy |
| 8081 | BlackIce-Icecap |

The number of open ports on this machine is itself a finding. A production server with this many exposed services has a wide attack surface. Each open port is a potential entry point.

---

#### Nikto Scan

Nikto is a web vulnerability scanner. It checks the server against a database of known misconfigurations, outdated software, and exposed files. I ran it against the HTTP service on port 80.

![Image](link)

![Image](link)

Nikto confirmed that the server is running outdated software versions that are no longer receiving security patches: Apache 2.2.14, PHP 5.3.2, and OpenSSL 0.9.8k. Each of these has documented CVEs. Nikto also found exposed directories including /icons, /images, and /test, and confirmed that wp-config.php (which contains WordPress database credentials) is accessible, a critical exposure.

---

### Part 7: Bash Reconnaissance Tool

To bring the individual tools together, I wrote a Bash script that acts as a simple menu-driven reconnaissance launcher. The script prompts the user to input a target IP address or domain, then presents four tool options: WhatWeb, Nmap, Dirb, and Knockpy. Once the user selects an option, the script runs the corresponding tool against the target.

![Image](link)

![Image](link)

The left image shows the script code. The right image shows a successful execution, Nmap run against 131.153.147.186, and Knockpy run against icdfa.edu.ng.

The value of building this script is not automation for its own sake. It is about understanding the reconnaissance workflow well enough to chain the tools in a logical order and being able to explain why each tool runs when it does.

---

### Part 8: GET vs POST Requests

A **GET request** retrieves data from a server. When a user searches for a term in a browser, views an article, or loads a product page, the browser sends a GET request. The request data is visible in the URL.

A **POST request** sends data to the server to create or update a record. When a user fills out a registration form and clicks submit, the browser sends a POST request. The data travels in the request body, not the URL, though this does not make it inherently secure.

The distinction matters in reconnaissance because identifying whether a form uses GET or POST tells you how user-supplied data flows into the application, which is relevant for understanding injection surfaces.

---

### Part 9: Information Gathering in Web Application Security

Information gathering, also called reconnaissance or footprinting, is the first step of any security assessment. The principle behind it is straightforward: you cannot test what you do not know.

This phase maps the target's structure: IP addresses, running services, subdomains, open ports, and exposed technologies. The goal is to identify entry points and misconfigurations before any active testing begins. Every finding from Parts 1 through 7 in this lab is a product of this phase.

The practical output of good information gathering is a list of specific, verifiable targets, not guesses.

---

### Part 10: How Websites Work Request-Response Model

**How a website works**

Every interaction with a website follows a request-response cycle. The browser (client) sends a request to the server, which includes an HTTP method (GET, POST, etc.) and headers describing the browser and its capabilities. The server processes the request and sends back a response, which includes a status code (200 OK, 404 Not Found, etc.) and the page content (HTML, CSS, JavaScript).

**What happens when you type a domain name**

The browser first checks its local DNS cache to see if it already knows the IP for that domain. If not, it sends a DNS query to a resolver, usually provided by the ISP. The resolver works through root servers, top-level domain servers, and the authoritative name server until it returns the correct IP address. Once the IP is known, the browser performs a TCP three-way handshake to establish a connection. If the site uses HTTPS, a TLS handshake follows to encrypt the session. Then the request-response cycle begins.

**The role of the DNS server**

Computers work with IP addresses, not domain names. DNS (Domain Name System) is the service that translates human-readable names like icdfa.edu.ng into machine-readable IP addresses like 131.153.147.186. Without DNS, every user would need to memorize the IP address of every site they want to visit. The DNS server receives the query, searches through the resolution chain, and returns the correct IP so the browser can connect.

---

## 6. Findings

| Finding | Detail |
|---|---|
| icdfa.edu.ng and icdfa.org.ng share one IP | Both resolve to 131.153.147.186 on the same /29 subnet |
| All three domains have zero MX records | External email services are in use |
| OWASP BWA exposes full server stack | Apache 2.2.14, PHP 5.3.2, OpenSSL 0.9.8k, all end-of-life |
| Directory listing enabled | /assets, /icons, /images, /test, and others are browsable |
| wp-config.php accessible | WordPress database credentials file is publicly reachable |
| Internal email addresses exposed | admin@metacorp.com, bob@ateliergraphique.com, test@thebodgeitstore.com harvested by WhatWeb |
| Nine open ports on OWASP BWA | Wide attack surface including SSH, HTTP, IMAP, SMB, and HTTP proxies |
| Knockpy vs Sublist3r | Neither tool is strictly better, results depend on how the domain's DNS is configured |

---

## 7. Challenges Faced

- The icdfa.org.ng Knockpy scan returned only one subdomain, while Sublist3r found 37. Initially, this looked like a tool failure. After reviewing the output, it became clear that most of the icdfa.org.ng subdomains exist in SSL certificate records but do not resolve in DNS, so Knockpy's active brute-force approach correctly returned nothing for them. The discrepancy was not a bug; it was a finding.

- During DNS lookups, the MX query for icdfa.edu.ng returned a "communications error" before falling back to a non-authoritative answer. This is a DNS resolver timeout on the lab network, not a problem with the domain itself. Running the query a second time or changing the resolver would resolve it.

---

## 8. Key Takeaways

- Two tools scanning the same target can return different results without either tool being wrong. The difference tells you something about the target's configuration.
- Exposed version numbers in HTTP headers are a misconfiguration, not a minor detail. They hand an attacker a shortcut to known CVEs.
- A /29 subnet hosts eight IP addresses. When two domains share one IP in a /29 block, that block is worth investigating further; other domains may be co-hosted on adjacent addresses.
- Directory listing is not a vulnerability that requires exploitation. An attacker with a browser can browse the files directly.
- Reconnaissance is where the security mindset starts. Every tool in this lab answers a question about the target, and those answers determine what comes next.

---

## 9. Disclaimer

This lab was performed in a controlled environment for educational purposes only. The live domain reconnaissance was conducted using passive and semi-passive techniques (ping, WHOIS, DNS queries, and public subdomain enumeration) that do not interact with the target servers beyond standard network requests. The OWASP BWA machine is a deliberately vulnerable application designed for security training. No unauthorized systems were tested at any point.
