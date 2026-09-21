# Operation Crimson Dawn - APT Simulation

## Executive Summary

Between the reconnaissance phase and the final exfiltration attempt, a sophisticated threat actor group designated **Crimson Dawn** conducted a multi-stage intrusion against our organisation's network infrastructure. The attack progressed through six distinct phases — from initial network probing, through persistent backdoor installation on a public-facing web server, to a lateral movement attempt targeting an internal Windows workstation, and culminating in an attempt to exfiltrate a compressed archive of sensitive credentials.

**The single most important finding:** Our network's DMZ (the boundary zone between our internet-facing servers and the internal corporate network) failed to block the attacker's pivot. A server that should have been fully isolated was able to directly reach internal workstations — which means a breach of one internet-facing asset creates a direct path to our most sensitive internal systems.

**Risk Rating:** 🔴 Critical

**Two actions require immediate executive approval and resource commitment:**

1. **Enforce strict network segmentation** between the DMZ and internal corporate network to prevent any future lateral movement from a compromised perimeter server.
2. **Deploy Endpoint Detection and Response (EDR)** across all servers and workstations to detect and block malicious behaviour in real time — before data leaves the network.

Without these two investments, a real-world version of this attack would result in the public leak of corporate credentials, triggering regulatory penalties, client loss, and reputational damage that would take years to recover from.

---

## Incident Overview

| Field | Detail |
|---|---|
| **Incident Name** | Operation Crimson Dawn |
| **Threat Actor** | Crimson Dawn (Simulated Hacktivist Group) |
| **Attack Objective** | Intellectual property theft and public data leak for reputational damage |
| **Primary Target** | Internal Windows workstation (FABELT — 192.168.43.17) |
| **Initial Entry Point** | DMZ Web Server (Prime-Ubuntu — 192.168.43.216) |
| **Detection Platform** | Wazuh SIEM with Sysmon telemetry |
| **Incident Status** | Contained — lateral movement blocked, exfiltration attempt failed |
| **Environment** | Simulated lab — Wazuh Manager, Ubuntu Web Server, Windows endpoint |

---

## Attack Timeline

The following timeline reconstructs the complete attack chain as observed in Wazuh telemetry and system logs.

```
Phase 1 — Reconnaissance
        │
        ▼
Crimson Dawn conducts network port scanning against the DMZ web server
(Prime-Ubuntu at 192.168.43.216). SSH service on TCP port 22 identified.
Multiple failed authentication attempts logged against both real and
non-existent user accounts.
        │
        ▼
Phase 2 — Exploitation & Persistence
        │
        ▼
Attacker gains access to the web server.
Malicious payload dropped to /tmp/.image.pdf.exe
(hidden prefix + double extension for evasion).
Payload registered as a root-level cron job — executes every minute,
writing C2 heartbeat to /tmp/.hidden-log.
Wazuh FIM captures both events (Rule 554 and Rule 550).
        │
        ▼
Phase 3 — Lateral Movement Attempt
        │
        ▼
From the compromised Prime-Ubuntu server, attacker executes psexec.py
(Impacket) targeting Windows ADMIN$ share over TCP port 445 (SMB).
Authentication fails — STATUS_LOGON_FAILURE (0xc000006d).
Wazuh captures Windows Event ID 4625 (Logon Failure) on the FABELT endpoint.
        │
        ▼
Phase 4 — Data Staging & Exfiltration Attempt
        │
        ▼
On the Windows workstation, attacker creates C:\SensitiveFiles\passwords.txt.
Directory compressed into exfil.tar.gz using native tar.exe.
PowerShell Invoke-WebRequest attempts outbound HTTP transfer to 192.168.43.216.
Connection dropped — no listener active on destination port.
Secondary attempt via curl.exe — triggers Sysmon Event IDs 1 and 3.
Wazuh captures command-line arguments, destination IP, and outbound connection.
        │
        ▼
Phase 5 — Command & Control Channel
        │
        ▼
C2 infrastructure centralised at 192.168.43.216 (Prime-Ubuntu server).
Serves as both the automated backdoor heartbeat destination and
the exfiltration endpoint. Custom Wazuh rule (ID 100001, Level 12)
deployed to alert on any future communication with this IP.
```

---

## Part 1 — Initial Breach: Reconnaissance and Delivery

### What Happened

The attacker's first move was to identify active network services on our public-facing web server. Using network scanning techniques, they located the SSH service running on TCP port 22. They then executed a series of automated login attempts — submitting invalid passwords against both existing user accounts and non-existent usernames — attempting to brute-force their way into the server.

**Investigation Action:** A Wazuh Dashboard query was created using the filter `rule.groups: "authentication_failed"` targeting the web server agent. This returned a clear record of repeated SSH authentication failures from a single source.

---

![Wazuh Dashboard showing authentication_failed alert filter results — multiple failed SSH login events from both non-existent and valid users captured from the Prime-Ubuntu web server agent](images/part1_authentication_failed.png)

*Figure 1 — Wazuh Dashboard query results for `rule.groups: "authentication_failed"`. The alert log shows repeated SSH login failures — both against non-existent users and a real account with incorrect passwords — confirming the attacker's brute-force reconnaissance phase.*

---

### MITRE ATT&CK Mapping

| Technique ID | Name | Evidence |
|---|---|---|
| **T1595** | Active Scanning — Scanning IP Blocks | Attacker probed network ports to identify the active SSH service on TCP port 22 |
| **T1110** | Brute Force | Repeated invalid login attempts against SSH — both non-existent and real user accounts with wrong passwords |

---

## Part 2 — Establishing Foothold: Exploitation and Persistence

### What Happened

Following initial access, the attacker staged a malicious payload on the compromised web server. The file `/tmp/.image.pdf.exe` was created — designed to look like an image file with a PDF extension, while the hidden prefix (`.`) and double extension (`.pdf.exe`) were deliberate evasion techniques to bypass manual administrative inspection.

The payload was then registered in the system's **cron daemon** under the root user, configured to execute every minute. This script simulated a Command and Control (C2) heartbeat — writing a timestamp to `/tmp/.hidden-log` on each execution cycle. The cron job ensured the attacker's access survived server restarts and terminal disconnections.

**Investigation Action:** Wazuh's **File Integrity Monitoring (FIM)** module was queried using `rule.groups: "syscheck"`. Two critical events were returned from the Prime-Ubuntu endpoint.

---

![Wazuh File Integrity Monitoring results showing Rule 554 (file creation) for /tmp/.image.pdf.exe and Rule 550 (file modification) for /tmp/.hidden-log on the Prime-Ubuntu endpoint](images/part2_fim_alerts.png)

*Figure 2 — Wazuh FIM alerts from the Prime-Ubuntu web server. Rule 554 captured the creation of the hidden malicious payload `/tmp/.image.pdf.exe`. Rule 550 captured repeated automated modifications to `/tmp/.hidden-log`, driven by the cron-executed C2 heartbeat script.*

---

### MITRE ATT&CK Mapping

| Technique ID | Name | Evidence |
|---|---|---|
| **T1036** | Masquerading | Hidden file prefix (`.`) and double extension (`.pdf.exe`) used to disguise executable as a temporary image file and evade manual administrative review |
| **T1053.003** | Scheduled Task/Job: Cron | Malicious payload registered in root-level cron daemon — auto-executes every minute, persisting across reboots and terminal disconnections |

---

## Part 3 — Lateral Movement and Privilege Escalation Attempt

### What Happened

Using the compromised web server as a **pivot point**, the attacker attempted to move deeper into the network — targeting the internal Windows workstation (FABELT at 192.168.43.17). The attack used **psexec.py** from the Impacket suite, which attempts to authenticate to the Windows `ADMIN$` administrative share over SMB (TCP port 445) and gain remote code execution.

The attempt **failed** — the target Windows system returned `STATUS_LOGON_FAILURE (0xc000006d)` due to invalid credentials. However, the attempt itself was captured in Wazuh telemetry as Windows Event ID 4625.

**Critical finding:** Despite the authentication failure, this event proves the DMZ web server had unrestricted network-level access to internal workstations over SMB — a fundamental network segmentation failure.

**Investigation Action:** A correlation query was executed in the Wazuh Dashboard: `(data.win.system.eventID: 4624 AND logon.process: "psexec") OR (data.win.system.eventID: 5140) OR (data.win.system.eventID: 4625)`. This returned the logon failure event with full forensic context.

---

![Wazuh Dashboard showing Windows Event ID 4625 Logon Failure alert on the FABELT endpoint — source IP 192.168.43.216, target account Administrator, Logon Type 3 (network SMB)](images/part3_lateral_movement.png)

*Figure 3 — Wazuh alert for Windows Event ID 4625 (Logon Failure) on the FABELT Windows endpoint. Key forensic artifacts: Source IP 192.168.43.216 (the compromised Prime-Ubuntu server), Target Account: Administrator, Logon Type 3 (network-based SMB connection). This confirms an unauthorized lateral movement attempt across the internal network boundary.*

---

### MITRE ATT&CK Mapping

| Technique ID | Name | Evidence |
|---|---|---|
| **T1021.002** | Remote Services: SMB/Windows Admin Shares | psexec.py targeting ADMIN$ share over TCP port 445 — generating STATUS_LOGON_FAILURE and Windows Event ID 4625 |

---

## Part 4 — Data Staging and Exfiltration Attempt

### What Happened

On the Windows workstation, the attacker created a target directory (`C:\SensitiveFiles\`) containing `passwords.txt` — simulating a targeted theft of sensitive corporate credentials. The directory was compressed into a gzip archive (`exfil.tar.gz`) using the native `tar.exe` binary — reducing the transfer size and bypassing file content monitoring tools that scan individual files.

Two exfiltration attempts were made:

1. **PowerShell `Invoke-WebRequest`** — attempted to POST the archive to the C2 server (192.168.43.216). The HTTP connection was dropped as no listener was active. Critically, this method *also evaded Sysmon process creation logging* because `Invoke-WebRequest` executes within the PowerShell engine without spawning a separate child process.

2. **`curl.exe`** — a secondary attempt using the external `curl.exe` binary. This successfully triggered Sysmon Event IDs 1 (process creation) and 3 (network connection), capturing both the command-line arguments and the destination C2 IP in Wazuh.

**Investigation Action:** Wazuh was queried for `data.win.system.eventID: 1` (Sysmon process creation) targeting `tar.exe` and `curl.exe` command lines.

---

![Wazuh Dashboard showing Sysmon Event ID 1 process creation alerts for tar.exe (data compression) and curl.exe (exfiltration attempt) with destination IP 192.168.43.216 visible in command-line telemetry](images/part4_exfiltration.png)

*Figure 4 — Wazuh Sysmon telemetry showing the two-stage exfiltration sequence. The top alert captures tar.exe compressing the SensitiveFiles directory. The second alert captures curl.exe initiating an outbound HTTP connection to the C2 server at 192.168.43.216, exposing the destination IP in the command-line arguments.*

---

### MITRE ATT&CK Mapping

| Technique ID | Name | Evidence |
|---|---|---|
| **T1560.001** | Archive Collected Data: Archive via Utility | `tar.exe` used to compress `C:\SensitiveFiles\passwords.txt` into `exfil.tar.gz` before transfer |
| **T1041** | Exfiltration Over C2 Channel | Outbound HTTP transfer attempted via PowerShell `Invoke-WebRequest` and `curl.exe` targeting 192.168.43.216 |

---

## Part 5 — Strategic Countermeasures and Threat Intelligence Integration

### 5a — Custom Wazuh Detection Rule Deployment

A custom detection rule was created on the Wazuh Manager to generate a **Level 12 (High) alert** for any future communication with the known C2 IP address (192.168.43.216). This rule was added to `/var/ossec/etc/rules/local_rules.xml`.

```xml
<group name="local_threat_intel,">
  <rule id="100001" level="12">
    <if_group>web</if_group>
    <match>192.168.43.216</match>
    <description>Connection to known malicious IP (Crimson Dawn C2).</description>
    <mitre>
      <id>T1071.001</id>
    </mitre>
  </rule>
</group>
```

---

![Wazuh local_rules.xml modified with custom rule ID 100001 Level 12 targeting the known Crimson Dawn C2 IP address 192.168.43.216 with MITRE T1071.001 mapping](images/part5a_custom_rule.png)

*Figure 5 — Wazuh Manager local rules file updated with custom rule ID 100001. Any future communication with the known C2 IP (192.168.43.216) will immediately trigger a Level 12 high-severity alert tagged to MITRE T1071.001 (Application Layer Protocol), enabling SOC analysts to detect C2 re-establishment instantly.*

---

### 5b — Vulnerability Detection Scan Results

A proactive vulnerability scan was executed against the compromised web server (Prime-Ubuntu) using the Wazuh Vulnerability Detector module. The scan was filtered using `rule.groups: "vulnerability-detector"`.

**Result:** Zero critical CVEs detected for the installed services. The host software was confirmed as fully patched.

**Analyst Assessment:** The absence of known CVEs indicates Crimson Dawn likely gained initial access through one of three alternative vectors: a **zero-day exploit** against the web service, a successful **credential brute-force attack** via SSH (consistent with the Phase 1 findings), or exploitation of a **system misconfiguration** rather than an unpatched vulnerability. The brute-force evidence from Part 1 makes credential-based access the most probable initial access vector.

---

![Wazuh Vulnerability Detector scan results for the Prime-Ubuntu web server showing zero critical CVEs for the installed FTP service confirming the host is fully patched](images/part5b_vulnerability_scan.png)

*Figure 6 — Wazuh Vulnerability Detector results for the Prime-Ubuntu web server. Zero critical CVEs were returned for the installed service. This confirms the host was fully patched, and supports the assessment that initial access was gained through credential brute-forcing (consistent with Part 1 findings) rather than a known exploitable vulnerability.*

---

## Full Attack Narrative — Cyber Kill Chain Mapping

| Kill Chain Stage | Adversary Action | MITRE ATT&CK |
|---|---|---|
| **Reconnaissance** | Crimson Dawn identified the DMZ web server (Prime-Ubuntu at 192.168.43.216). Conducted network port scanning to locate the active SSH service, then executed repeated brute-force authentication attempts using invalid and non-existent credentials. | T1595 — Scanning IP Blocks; T1110 — Brute Force |
| **Exploitation & Installation** | Attacker dropped `/tmp/.image.pdf.exe` — a hidden payload using a deceptive double extension. Registered in the root cron daemon to execute every minute, writing a C2 heartbeat to `/tmp/.hidden-log`. Wazuh FIM captured file creation (Rule 554) and modification (Rule 550). | T1036 — Masquerading; T1053.003 — Cron |
| **Lateral Movement** | From the compromised Prime-Ubuntu server, attacker executed `psexec.py` targeting the Windows ADMIN$ share (TCP 445) as Administrator. Authentication failed (STATUS_LOGON_FAILURE). Wazuh captured Event ID 4625 on the FABELT endpoint — proving direct network routing from DMZ to internal workstation. | T1021.002 — Remote Services: SMB/Windows Admin Shares |
| **Data Exfiltration** | On the Windows workstation, attacker created `C:\SensitiveFiles\passwords.txt`, compressed it with `tar.exe` into `exfil.tar.gz`, and attempted HTTP transfer to 192.168.43.216 via PowerShell `Invoke-WebRequest` and `curl.exe`. Sysmon captured process creation and outbound network connection events. | T1560.001 — Archive via Utility; T1041 — Exfiltration Over C2 Channel |
| **Command & Control** | Centralised C2 infrastructure at 192.168.43.216 served as both the automated backdoor heartbeat endpoint and the exfiltration destination. Custom Wazuh rule deployed to alert on all future communications with this IP. | T1071.001 — Application Layer Protocol |

---

## Business Impact Assessment

| Impact Area | Assessment |
|---|---|
| **Data at Risk** | `C:\SensitiveFiles\passwords.txt` — corporate credentials and potential intellectual property were staged and compressed for exfiltration |
| **Reputational Risk** | Crimson Dawn operates as a hacktivist group with the explicit goal of publicly leaking stolen data. A successful exfiltration and public release of credentials would severely degrade client trust and invite immediate negative media coverage |
| **Regulatory and Financial Risk** | Exfiltration of credentials or PII triggers immediate regulatory notification obligations and potential fines. External incident response engagements and legal counsel would represent significant unplanned expenditure |
| **Operational Disruption** | Both the DMZ web server and the internal Windows workstation would require forensic imaging, malware eradication, and secure rebuilding — resulting in measurable downtime and productivity loss |
| **Network Trust Boundary Failure** | The confirmed ability of the DMZ server to route SMB traffic directly to internal workstations means **the blast radius of any future DMZ breach is the entire internal network**, not just the perimeter server |

---

## Root Cause Analysis

Four systemic failures enabled this attack chain to progress from initial reconnaissance to a near-successful data exfiltration:

**1 — DMZ Network Segmentation Failure**
The compromised Prime-Ubuntu web server located in the DMZ was able to route SMB traffic (TCP 445) directly to the internal Windows workstation (FABELT). A properly segmented DMZ would have blocked this routing at the firewall layer. This is the most critical finding — it is the gap that converts a contained perimeter breach into a full internal network compromise.

**2 — Insufficient Outbound Egress Filtering**
The internal Windows workstation was permitted to establish unrestricted outbound HTTP connections to an untrusted external IP address. Internal workstations should not be able to initiate arbitrary outbound connections — a proxy or egress firewall configured with an allow-list of known destinations would have blocked the C2 exfiltration traffic entirely.

**3 — Exposed Management Interfaces**
The web server's SSH service (TCP 22) was accessible directly from the internet, enabling the attacker to execute remote brute-force authentication attempts at scale with no rate limiting or geographic restriction. Management ports should sit behind a VPN or Zero Trust gateway.

**4 — Permissive Host Execution Policy**
The attacker successfully staged a malicious payload in the `/tmp` directory and established persistence via cron — indicating the endpoint lacked execution controls. Mounting `/tmp` with the `noexec` flag would have prevented script execution from that directory, eliminating the cron-based persistence vector entirely.

---

## Strategic Recommendations

The following four controls directly address each root cause identified above, ordered by implementation priority.

### Priority 1 — Enforce Network Segmentation (Immediate)

Implement strict firewall Access Control Lists (ACLs) between the DMZ and the internal corporate network. Specifically, **block SMB (TCP 445), RDP (TCP 3389), and all administrative protocols** from originating in the DMZ and reaching internal workstations. This single control would have stopped the lateral movement phase entirely.

> **Business Justification:** Enforcing DMZ network segmentation eliminates the direct path from a perimeter breach to internal systems, reducing the blast radius of any future web server compromise from "entire internal network" to "one isolated server."

---

### Priority 2 — Deploy Endpoint Detection and Response (Immediate)

Upgrade endpoint security from traditional antivirus to an active EDR solution across all servers and workstations. EDR provides real-time behavioural monitoring and blocking — it would have detected and stopped the cron job persistence attempt, the `psexec.py` execution, and the `tar.exe` data staging in real time, before any data left the host.

> **Business Justification:** Deploying EDR mitigates the risk of catastrophic data theft by neutralising internal threats that bypass our perimeter defences, protecting intellectual property and avoiding the regulatory fines that follow a confirmed data breach.

---

### Priority 3 — Implement Outbound Egress Filtering (Short-term)

Deploy a Secure Web Gateway or egress firewall to restrict outbound internet access from internal workstations. Endpoints should only be permitted to communicate with known, categorised destinations — this would have blocked the `curl.exe` and `Invoke-WebRequest` C2 exfiltration attempts at the network layer.

---

### Priority 4 — Secure Management Interfaces via Zero Trust (Short-term)

Remove direct public internet access to administrative services (SSH, RDP) on all DMZ servers. Require administrators to authenticate through a corporate VPN or Zero Trust Network Access (ZTNA) gateway before managing internet-facing infrastructure. This eliminates the brute-force attack surface entirely.

---

## Strategic Analysis

### Alternative Persistence Technique (If Admin Rights Were Gained on Windows)

Had Crimson Dawn successfully authenticated to the Windows workstation with administrative privileges, the most impactful persistence mechanism they could have deployed is **T1543.003 — Create or Modify System Process: Windows Service**.

A malicious Windows service configured to start automatically on boot would execute the payload as the highly privileged `SYSTEM` account — persisting across all reboots before any user logs in, and running with permissions that bypass most user-context security controls. This is a significant escalation from cron-based persistence and would require forensic-level investigation to detect and remediate.

---

### How Proactive Vulnerability Detection Could Have Prevented the Breach

Had the Wazuh Vulnerability Detector been actively monitored prior to this incident, the SOC team would have received alerts on any high-severity CVEs affecting the DMZ web server's installed services — flagging the vulnerable attack surface before Crimson Dawn could exploit it. This shifts the security posture from reactive (detect after breach) to proactive (patch before exploitation), eliminating the attack vector at its source.

---

### Bypassing the Custom IP-Block Rule — Attacker's Counter-Move

The custom Wazuh rule (Rule 100001) blocks communication with the known C2 IP address `192.168.43.216`. An experienced attacker would bypass this within minutes by switching to **domain-based C2** rather than a hardcoded IP.

The attacker registers a domain (e.g., `update-server.com`) and points it to their C2 IP. If that IP gets blocked, they update the domain's DNS `A` record to point to a new, unblocked IP address — sometimes within seconds, using a technique called **Fast Flux DNS**. The blocked rule becomes irrelevant because the malware calls the domain, not the IP directly.

**Defensive counter-measure:** IP-based blocking must be paired with **DNS-based threat intelligence filtering** (blocking malicious domains at the resolver level) and **SSL inspection** to catch domain-fronting evasion techniques.

---

## Challenges Encountered and Resolutions

### Challenge 1 — File Integrity Monitoring Blind Spot

**Challenge:** During the persistence phase, Wazuh failed to generate an alert for the malicious file dropped into `/tmp` because the default FIM configuration excludes `/tmp` from real-time monitoring — due to the high volume of routine temporary system activity that would generate excessive noise.

**Resolution:** The Wazuh agent's `ossec.conf` file was manually modified to add `/tmp` to the real-time monitoring scope: `<directories realtime="yes">/tmp</directories>`. Following an agent restart, Wazuh successfully captured both the file creation (Rule 554) and subsequent modification (Rule 550) events within the `/tmp` directory.

**Lesson:** Default SIEM configurations are designed for noise reduction, not maximum visibility. High-value or high-risk directories that fall outside default scope require manual inclusion — a gap that must be addressed during initial deployment, not after an incident.

---

### Challenge 2 — Process Logging Evasion via Native PowerShell Cmdlets

**Challenge:** The initial exfiltration simulation using PowerShell's `Invoke-WebRequest` did not trigger Sysmon process creation logging (Event ID 1 / Event ID 4688). Because `Invoke-WebRequest` executes internally within the PowerShell engine, it does not spawn a child process — making it invisible to process-creation-based detection rules.

**Resolution:** A secondary exfiltration simulation using `curl.exe` was executed. As an external binary, `curl.exe` spawns a distinct process — successfully triggering Sysmon Event IDs 1 and 3, capturing both the command-line arguments and the destination C2 IP in the Wazuh dashboard.

**Lesson:** Process-creation logging is necessary but not sufficient. Native PowerShell cmdlets, WMI calls, and COM-based execution all bypass it. Complete coverage requires **PowerShell Script Block Logging** (Event ID 4104) alongside Sysmon to capture inline cmdlet execution.

---

### Challenge 3 — Legacy Vulnerability Simulation on Modern Infrastructure

**Challenge:** The objective required installing a deliberately vulnerable FTP service to simulate a CVE-exploitable attack vector. Attempting to install a vulnerable package from 2014 failed — modern Ubuntu package mirrors have purged obsolete versions, forcing the installation of a current, fully patched FTP package.

**Resolution:** Rather than artificially breaking the server environment, the scenario was adapted to reflect a realistic SOC outcome. The vulnerability scan correctly returned zero critical CVEs, demonstrating the Wazuh scanner was functioning accurately. The threat narrative was updated to position the initial breach as credential-based (brute force), consistent with the Part 1 evidence — which is also a more realistic attack vector for modern infrastructure.

**Lesson:** Not all scenarios can be reproduced identically in modern environments. Adapting simulations to realistic constraints while preserving the analytical integrity of findings is a core SOC analyst skill.

---

## Consolidated Indicators of Compromise (IOCs)

| Type | Value | Phase |
|---|---|---|
| **Attacker IP / C2** | `192.168.43.216` | All phases — reconnaissance origin, C2 heartbeat destination, exfiltration endpoint |
| **Target Internal IP** | `192.168.43.17` (FABELT) | Lateral movement target |
| **Malicious File** | `/tmp/.image.pdf.exe` | Exploitation & Persistence — hidden payload with double extension |
| **C2 Heartbeat Log** | `/tmp/.hidden-log` | Automated modification every minute by cron job |
| **Persistence Mechanism** | Root cron job — `echo "C2 Checkin" >> /tmp/.hidden-log` | Scheduled Task/Job: Cron |
| **Exfiltration Archive** | `C:\SensitiveFiles\exfil.tar.gz` | Data staging — compressed credentials |
| **Sensitive Data Targeted** | `C:\SensitiveFiles\passwords.txt` | Data Exfiltration target |
| **Exfiltration Tool 1** | `PowerShell Invoke-WebRequest` | Bypassed Sysmon process logging |
| **Exfiltration Tool 2** | `curl.exe` | Captured by Sysmon Event IDs 1 and 3 |
| **Lateral Movement Tool** | `psexec.py` (Impacket) | SMB/Windows Admin Shares |
| **SMB Target Share** | `ADMIN$` on 192.168.43.17 | Lateral movement target |
| **Wazuh Rule Deployed** | Rule ID 100001, Level 12 | Custom C2 IP detection rule |

---

## Appendix — MITRE ATT&CK Full Matrix

| Phase | Tactic | Technique ID | Technique Name | Evidence |
|---|---|---|---|---|
| 1 | Reconnaissance | T1595 | Active Scanning — Scanning IP Blocks | Network port scan identifying SSH on TCP 22 |
| 1 | Credential Access | T1110 | Brute Force | Repeated SSH login failures against real and non-existent accounts |
| 2 | Defense Evasion | T1036 | Masquerading | `/tmp/.image.pdf.exe` — hidden prefix + double extension |
| 2 | Persistence | T1053.003 | Scheduled Task/Job: Cron | Root cron job executing payload every minute |
| 3 | Lateral Movement | T1021.002 | Remote Services: SMB/Windows Admin Shares | `psexec.py` targeting ADMIN$ — Event ID 4625 captured |
| 4 | Collection | T1560.001 | Archive Collected Data: Archive via Utility | `tar.exe` compressing `SensitiveFiles` into `exfil.tar.gz` |
| 4 | Exfiltration | T1041 | Exfiltration Over C2 Channel | Outbound HTTP to 192.168.43.216 via PowerShell and curl.exe |
| 5 | Command & Control | T1071.001 | Application Layer Protocol | HTTP-based C2 heartbeat and exfiltration channel |

---

> **Disclaimer:** This report documents a structured threat simulation conducted within an isolated lab environment as part of a cybersecurity training exercise. No production systems were accessed. All attacker actions, IP addresses, and file artifacts described in this report are simulated. This document is published for educational and portfolio purposes only.
