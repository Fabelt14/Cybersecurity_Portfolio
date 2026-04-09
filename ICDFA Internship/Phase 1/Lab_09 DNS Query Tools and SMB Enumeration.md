# DNS Query Tools and SMB Enumeration

**Student Name:** Asekun Fatai
**Student ID:** 2025/INT/12158
**Course:** Kali Linux Tools and System Security
**Instructor:** Mr. Aminu Idris
**Date:** 07 January 2026

---

## 1. Engagement Overview

A reconnaissance assessment was conducted in two phases against two separate
targets. The first phase focused on external DNS reconnaissance against the public
domain `google.com` to demonstrate DNS query techniques applicable to any target
domain. The second phase performed active SMB enumeration against a local
Metasploitable2 instance to extract user accounts, share names, and service
information. The approach was black-box for DNS and grey-box for SMB, conducted
entirely within an isolated Kali Linux lab environment.

---

## 2. Objectives

- Map the DNS infrastructure of a target domain using multiple query tools to
  extract progressively detailed records
- Identify mail server configurations, TXT records, and TTL values that inform
  attack planning
- Enumerate SMB services on a target host to extract user accounts, share names,
  and OS fingerprinting data
- Demonstrate how combined DNS and SMB data creates a complete pre-exploitation
  intelligence picture
- Identify misconfigurations in SMB share access controls and assess associated risk

---

## 3. Scope

**In-Scope:**
- DNS reconnaissance against `google.com` (public domain, passive query only)
- SMB enumeration against local Metasploitable2 instance (isolated lab network)
- Protocols targeted: DNS (UDP/53), SMB (TCP/445, TCP/139)

**Out-of-Scope:**
- Active exploitation of any identified service or account
- Any systems outside the local lab network
- Brute force or password attacks against enumerated user accounts

**Authorization Statement:**
> All DNS queries in this assessment were performed against a public domain using
> standard query tools for educational analysis only. SMB enumeration was conducted
> against Metasploitable2, an intentionally vulnerable virtual machine deployed in an
> isolated lab environment. No production systems or unauthorized infrastructure
> were accessed at any point. All activities were performed for educational purposes
> under authorized course instruction.

---

## 4. Methodology

### Phase 1 — Reconnaissance (DNS Queries)
DNS queries were performed against `google.com` using three tools in sequence:
`nslookup`, `host`, and `dig`. Each tool was used to extract progressively more
detailed information. `nslookup` was run first for basic A and AAAA record
resolution. `host` was run second to retrieve mail server (MX) records alongside
IP addresses. `dig` was run last with explicit record type flags (`MX`, `TXT`) to
extract TTL values, header flags, and organizational TXT records.

### Phase 2 — Mapping / Spidering (SMB Service Discovery)
`enum4linux` was run against the Metasploitable2 target IP to perform a
comprehensive unauthenticated SMB enumeration. The tool was used to extract the
full user list with RID values, enumerate available shares with their types and
comments, and fingerprint the operating system and Samba service version.

### Phase 3 — Vulnerability Identification
Enumerated share permissions were reviewed to identify globally accessible shares.
The `tmp` share was identified as readable and listable without authentication. The
full user list was assessed for accounts that could be targeted in a password
spraying attack.

### Phase 4 — Exploitation (Not Performed)
Exploitation was outside the defined scope of this assessment. All identified
vulnerabilities were documented for remediation rather than active exploitation.

### Phase 5 — Validation
DNS query outputs from all three tools were cross-referenced to confirm record
accuracy and identify information gaps between tools. SMB enumeration output
was reviewed to confirm the completeness of the user and share lists.

### Phase 6 — Documentation
All command outputs, screenshots, and tool results were recorded throughout
both phases to support the findings documented in this report.

---

## 5. Vulnerability Summary

| ID | Vulnerability | Severity | Affected Endpoint |
|----|--------------|----------|-------------------|
| 01 | SMB Null Session Unauthenticated Enumeration | High | Metasploitable2 - TCP/445 |
| 02 | World-Readable SMB Share (tmp) | High | \\Metasploitable2\tmp |
| 03 | Full User Account List Exposed via SMB | High | Metasploitable2 - SMB |
| 04 | DNS Infrastructure and Mail Provider Disclosure | Medium | google.com DNS Records |
| 05 | Organizational TXT Record Exposure | Low | google.com TXT Records |

---

## 6. Detailed Findings

---

### Finding 01 — SMB Null Session Unauthenticated Enumeration

#### Severity
High

#### Affected Endpoint
Metasploitable2 target — TCP port 445 (SMB), TCP port 139 (NetBIOS)

#### Description
The SMB service on the Metasploitable2 host accepts null session connections,
meaning an unauthenticated attacker can establish an anonymous SMB session
without providing any credentials. This allowed `enum4linux` to connect and
extract the complete user account list, share names, share types, share comments,
workgroup membership, and the Samba service version. Null session access exists
because the Samba configuration does not restrict anonymous connections.

#### Proof of Concept

`enum4linux` run against the Metasploitable2 target returning full user list and
share enumeration without any credentials supplied:

![enum4linux Output - Full User List with RID Values](image.jpg)

Share enumeration output from the same null session:

![enum4linux Output - Share Names, Types, and Comments](image.jpg)

Users extracted via unauthenticated null session (partial list):
```
user:[games]    rid:[0x3f2]
user:[nobody]   rid:[0x1f5]
user:[bind]     rid:[0x4ba]
user:[proxy]    rid:[0x402]
user:[syslog]   rid:[0x4b4]
user:[www-data] rid:[0x42a]
user:[root]     rid:[0x3e8]
user:[msfadmin] rid:[0xbb8]
user:[mysql]    rid:[0x4c2]
user:[postgres] rid:[0x4c0]
user:[ftp]      rid:[0x4be]
user:[tomcat55] rid:[0x4c4]
```

Service fingerprint returned by enum4linux:
```
IPC$ - IPC Service (metasploitable server (Samba 3.0.20-Debian))
```

#### Impact
A complete list of valid usernames is one of the most valuable assets an attacker
can obtain before attempting authentication attacks. With confirmed usernames
including `root` and `msfadmin`, an attacker can run targeted password spraying
campaigns against SSH, FTP, and other services without wasting attempts on
non-existent accounts. The Samba version (3.0.20) is publicly known to be
vulnerable to critical remote code execution exploits.

#### Remediation
- Disable null session access in the Samba configuration:
  `restrict anonymous = 2`
- Upgrade Samba from version 3.0.20 to a currently supported release to
  eliminate known RCE vulnerabilities
- Restrict SMB access to authenticated users only using the `valid users`
  directive in `smb.conf`
- Block TCP ports 139 and 445 at the network perimeter for any hosts that do
  not require SMB access from external networks

---

### Finding 02 — World-Readable SMB Share (tmp)

#### Severity
High

#### Affected Endpoint
`\\Metasploitable2\tmp` — Disk share, globally accessible

#### Description
The `tmp` share on the Metasploitable2 host was found to be globally readable and
listable by any connecting client, including unauthenticated null sessions. The share
comment "oh noes!" in the enumeration output is consistent with a known
Metasploitable2 misconfiguration. In real-world environments, temporary directories
frequently contain configuration files, log data, scripts, and credentials left behind
by running processes or administrators.

#### Proof of Concept

Share listing from `enum4linux` confirming `tmp` share type and accessibility:

![SMB Share Listing - tmp Share World-Readable](image.jpg)

Shares enumerated:
```
Sharename    Type    Comment
---------    ----    -------
print$       Disk    Printer Drivers
tmp          Disk    oh noes!
opt          Disk
IPC$         IPC     IPC Service (metasploitable server (Samba 3.0.20-Debian))
ADMIN$       IPC     IPC Service (metasploitable server (Samba 3.0.20-Debian))
```

#### Impact
Read access to the `tmp` share allows an attacker to retrieve any files placed there
by system processes or users. In a real scenario this could include database
connection strings, application configuration files with hardcoded credentials,
shell history files, and temporary authentication tokens. Write access to the same
share would allow an attacker to plant a reverse shell or malicious script for
execution by a privileged process.

#### Remediation
- Remove world-readable permissions from the `tmp` share immediately
- Apply explicit `valid users` and `read list` restrictions in `smb.conf` to limit
  access to named accounts only
- Audit all other shares (`print$`, `opt`) for similarly permissive access controls
- Implement regular automated audits of SMB share permissions to detect
  permission drift

---

### Finding 03 — Full User Account List Exposed via SMB

#### Severity
High

#### Affected Endpoint
Metasploitable2 — SMB RID cycling via null session

#### Description
Through unauthenticated null session access, `enum4linux` performed RID cycling
to enumerate every user account on the system. Over 30 user accounts were
returned including service accounts (`www-data`, `mysql`, `postgres`, `ftp`,
`tomcat55`) and privileged accounts (`root`, `msfadmin`). The RID values
returned alongside each username confirm these are local system accounts, not
domain accounts.

#### Proof of Concept

Full user list returned by `enum4linux` RID cycling against the target:

![enum4linux Full User List - RID Values](image.jpg)

Selected high-value accounts from the enumerated list:
```
user:[root]      rid:[0x3e8]
user:[msfadmin]  rid:[0xbb8]
user:[mysql]     rid:[0x4c2]
user:[postgres]  rid:[0x4c0]
user:[www-data]  rid:[0x42a]
user:[ftp]       rid:[0x4be]
user:[tomcat55]  rid:[0x4c4]
```

#### Impact
Confirmed username lists remove the guesswork from credential attacks. An
attacker with this list can run password spraying across SSH (port 22), FTP
(port 21), and Tomcat management interfaces using only the exact valid account
names returned. The presence of `root` as an enumerable account confirms the
system permits root login, which combined with a weak password would yield
immediate full administrative control.

#### Remediation
- Disable RID cycling by setting `restrict anonymous = 2` in `smb.conf`
- Disable direct root login over SSH: `PermitRootLogin no` in `sshd_config`
- Enforce account lockout policies across all services that accept password
  authentication
- Remove or disable all service accounts that are not actively required
  (e.g., `games`, `nobody`, `irc`)

---

### Finding 04 — DNS Infrastructure and Mail Provider Disclosure

#### Severity
Medium

#### Affected Endpoint
`google.com` DNS records — publicly queryable

#### Description
Using `nslookup`, `host`, and `dig` in sequence, the following information was
extracted from public DNS records without any authentication or special access:
the target's IPv4 and IPv6 addresses, the mail server provider (smtp.google.com),
HTTP service bindings, and the TTL value of the A record. The `host` command
confirmed that Google's mail is handled by `smtp.google.com` with a priority of 10,
identifying the exact mail platform the organization uses.

#### Proof of Concept

`nslookup` output returning A and AAAA records for `google.com`:

![nslookup Output - IPv4 and IPv6 Addresses](image.jpg)

`host` output returning IP addresses and MX record:

![host Output - IP Address and Mail Server](image.jpg)

`dig` output returning A record with TTL value and DNS header flags:

![dig Output - A Record with TTL 233 and Header Flags](image.jpg)

Records extracted:
```
A record:    142.251.39.206  (nslookup)
             142.250.179.110 (host / dig — TTL: 233)
AAAA record: 2a00:1450:4006:80f::200e
MX record:   google.com mail is handled by 10 smtp.google.com
TTL value:   233 seconds
Flags:       qr (query response), rd (recursion desired), ra (recursion available)
```

#### Impact
The MX record disclosure confirms which mail platform the organization uses,
allowing a targeted attacker to craft phishing emails designed to bypass the
specific spam filters deployed by that provider. The TTL value of 233 seconds
is directly useful for timing DNS cache poisoning attempts, as it defines how
long a poisoned record would be cached before being refreshed.

#### Remediation
- MX records are a required public DNS record and cannot be hidden without
  breaking mail delivery. Mitigation focuses on the mail platform itself rather
  than the DNS record.
- Implement DMARC, DKIM, and SPF policies to reduce the effectiveness of
  phishing campaigns that exploit knowledge of the mail provider
- Monitor for lookalike domains that abuse knowledge of the mail infrastructure
  for spear phishing campaigns

---

### Finding 05 — Organizational TXT Record Exposure

#### Severity
Low

#### Affected Endpoint
`google.com` TXT DNS records — publicly queryable

#### Description
Querying TXT records for `google.com` using `dig google.com TXT` returned
multiple verification and policy records that fingerprint the services and
third-party platforms the organization uses. Records included Google Site
Verification tokens, a DocuSign verification token, and an MS (Microsoft)
verification token, confirming the use of multiple third-party SaaS platforms.

#### Proof of Concept

`dig` output showing multiple TXT records including verification tokens and
service identifiers:

![dig TXT Record Output - Verification Tokens and Service Records](image.jpg)

Selected TXT records observed:
```
google.com. 3600 IN TXT "google-site-verification=4ibFUgB-..."
google.com. 3600 IN TXT "MS=E4A60B9A82BB9070BCE15412F02916..."
google.com. 3600 IN TXT "docusign=05958488-4752-4ef2-95eb-..."
google.com. 3600 IN TXT "apple-domain-verification=30afl8c..."
```

#### Impact
TXT records confirm which third-party platforms an organization uses for
document signing, site verification, and identity management. This intelligence
allows an attacker to craft highly credible social engineering attacks that
impersonate trusted services already in use by the target — for example, a
phishing email mimicking a DocuSign document request directed at the
organization's staff.

#### Remediation
- Remove any TXT verification records for services that are no longer actively
  in use to minimize the exposed platform footprint
- Conduct periodic DNS audits to identify stale or unnecessary TXT records
- Security awareness training for staff should include awareness of phishing
  attacks that impersonate internal tools and trusted third-party services

---

## 7. Attack Chain

```
[Phase 1] External DNS Reconnaissance
nslookup google.com
→ IPv4: 142.251.39.206 / IPv6: 2a00:1450:4006:80f::200e identified

host google.com
→ MX record: smtp.google.com (mail provider confirmed)

dig google.com TXT
→ DocuSign, Microsoft, Apple verification tokens returned
→ Third-party platform usage confirmed for social engineering planning

dig google.com MX
→ TTL: 233 seconds noted for DNS spoofing timing
        ↓
[Phase 2] SMB Enumeration (Internal Target)
enum4linux against Metasploitable2
→ Null session accepted — no credentials required
→ 30+ user accounts enumerated including root and msfadmin
→ Shares listed: print$, tmp (world-readable), opt, IPC$, ADMIN$
→ Samba 3.0.20 version fingerprinted
        ↓
[Combined Intelligence Picture]
DNS data identifies mail provider and third-party platforms
→ Enables credible phishing email construction targeting known staff accounts

SMB data provides valid username list (root, msfadmin, postgres, etc.)
→ Enables targeted password spraying against SSH, FTP, Tomcat

tmp share world-readable
→ If write access exists: reverse shell can be planted
→ If configuration files present: credentials can be extracted for lateral movement
```

---

## 8. Tools Used

- Kali Linux (assessment environment)
- `nslookup` (basic DNS A and AAAA record resolution)
- `host` (DNS resolution with MX record and HTTP service binding output)
- `dig` (advanced DNS query with TTL, header flags, MX, and TXT record extraction)
- `enum4linux` (SMB enumeration — users, shares, OS fingerprinting)

---

## 9. Challenges Encountered

- **enum4linux incompatibility with public IPs:** Running `enum4linux` against
  the public IP address resolved from `google.com` returned no results. SMB
  enumeration requires a target that exposes SMB services, which public web
  servers do not. The tool was redirected to the Metasploitable2 local target
  where SMB was confirmed active.
- **DNS record variability:** `nslookup` and `host` returned slightly different
  A record IP addresses for `google.com` across queries (142.251.39.206 vs
  142.250.179.110). This is expected behavior due to Google's anycast DNS
  infrastructure and round-robin load balancing. Both addresses resolve to the
  same organization.
- **TXT record volume:** `dig google.com TXT` returned a large number of
  records requiring manual review to identify which records disclosed actionable
  intelligence versus routine verification tokens.

---

## 10. Key Takeaways

- **DNS and SMB intelligence compound each other:** Neither phase alone tells
  the full story. DNS identifies public infrastructure and mail platforms; SMB
  reveals internal accounts and shares. Combined, they provide a complete
  pre-exploitation map covering both the external perimeter and internal network
  surface.
- **Null sessions are a critical misconfiguration:** Allowing unauthenticated SMB
  connections in 2026 is an unacceptable risk. The single configuration change
  `restrict anonymous = 2` would have prevented the entire SMB enumeration
  phase from succeeding.
- **TXT records are overlooked reconnaissance targets:** Most security teams
  monitor their A and MX records but rarely audit TXT records. Verification tokens
  for DocuSign, Microsoft, and Apple confirm exactly which platforms staff use
  daily, making TXT records a direct input to social engineering attack planning.
- **TTL values have offensive utility:** The 233-second TTL value returned by
  `dig` is not just metadata. It defines the precise window in which a DNS
  cache poisoning attack remains effective before the victim's resolver refreshes
  the record. Understanding this transforms a routine query output into an attack
  timing parameter.
- **Tool progression matters in reconnaissance:** Starting with `nslookup` and
  progressing to `dig` is not redundant. Each tool surfaces information the
  previous one does not. `dig` provides TTL, header flags, and explicit record
  type control that `nslookup` does not expose by default.

---

## 11. Disclaimer

> This assessment was conducted entirely within an isolated lab environment for
> educational purposes. DNS queries were performed against `google.com` using
> standard read-only public DNS lookups that generate no different traffic than a
> normal browser. SMB enumeration was performed exclusively against
> **Metasploitable2**, an intentionally vulnerable virtual machine designed for
> security training. No production systems, live networks, or unauthorized
> infrastructure were accessed or disrupted at any point. All techniques described
> in this report must only be applied against systems for which explicit written
> authorization has been obtained.

---