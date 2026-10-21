# Incident Response & Digital Forensics Report
## TechSecure Solutions Web Server Compromise (INC-2025-001)

---
> **Report Type:** Post-Incident Forensic Analysis & Remediation Report
>
> **Analyst:** Incident Response & Digital Forensics Analyst
>
> **Incident Reference:** TECHSECURE-2025-001
>
> **Report Date:** 03 October 2026
>
> **Incident Status:** 🔴 Contained (Awaiting Full Eradication and Rebuild)
>
> **Attributed Actor:** APT28 (Fancy Bear / GRU) High Confidence

---

## Executive Summary

On **5–6 November 2025**, TechSecure Solutions' public-facing web server (`web-server-01`, `192.168.1.50`) suffered a critical security breach attributed with high confidence to **APT28 (Fancy Bear)**, a Russian state-sponsored threat group operating under GRU Military Intelligence.

The attacker exploited an unrestricted file upload vulnerability in the client portal to plant a malicious web shell, gain full command execution over the server, and install a Cobalt Strike Beacon for persistent access. Over a **12-hour and 15-minute dwell time**, the adversary conducted systematic reconnaissance, staged target data, and exfiltrated approximately **2.3 GB** of sensitive assets before automated EDR detection triggered a SOC alert at 02:15 UTC.

**What was stolen:**
- Customer database backup containing client PII (1.2 GB)
- Client contact records: names, emails, phone numbers (45 MB)
- Financial transaction records (320 MB)
- Employee credentials file: plaintext passwords (2.3 MB)
- Application source code repository (850 MB)

**The immediate business exposure:** Customer credential data is now in the hands of a state-sponsored threat actor. Downstream phishing attacks against TechSecure clients using stolen contact data should be treated as imminent. Regulatory disclosure obligations under applicable data protection frameworks are triggered.

**Three actions requiring executive decision within 24 hours:**

1. **Authorise full network isolation** of `web-server-01` and initiate a clean rebuild from a gold image, the compromised server cannot be trusted in any state until fully rebuilt
2. **Initiate mandatory credential resets** for the `IUSR_TECHSECURE` service account and all accounts whose credentials appear in the exfiltrated `passwords.txt`
3. **Engage legal counsel** to assess regulatory disclosure obligations under applicable data protection law, the breach timeline, data categories, and volume are documented in this report

---

## Incident Information

| Field | Value |
|---|---|
| **Incident Reference** | TECHSECURE-2025-001 |
| **Incident Severity** | 🔴 Critical |
| **Detection Time** | 2025-11-06 02:30:00 UTC |
| **Estimated Compromise Time** | 2025-11-05 14:00:00 UTC |
| **Total Dwell Time** | 12 hours 15 minutes |
| **Affected System** | `web-server-01.techsecure.local` (`192.168.1.50`) |
| **System Type** | Windows Server 2019 (IIS 10.0) |
| **Attack Vector** | Unrestricted File Upload Vulnerability (Client Portal) |
| **Attributed Actor** | APT28 (Fancy Bear / GRU) |
| **Data Exfiltrated** | ~2.3 GB (Customer PII, Credentials, Financial Records, Source Code) |
| **Containment Status** | Host isolated (eradication and rebuild in progress) |

---

## Part 1: Initial Incident Assessment

### 1.1 - Severity Classification

**Severity Level: 🔴 Critical**

The critical severity classification is driven by five concurrent conditions, any single one of which would independently justify escalation to the highest response tier:

- **Condition 1: Active compromise of a public-facing production server.** The attacker successfully executed arbitrary operating system commands via a web shell under the `IUSR_TECHSECURE` service account context. The server was fully under adversary control.

- **Condition 2: Confirmed data exfiltration of regulated data categories.** Approximately 2.3 GB of data, including customer PII, plaintext credentials, financial records, and source code was confirmed transferred to external attacker-controlled infrastructure between 22:00 and 01:00 UTC.

- **Condition 3: Active Command and Control channel established.** A Cobalt Strike Beacon (disguised as `svchost.exe`) established an encrypted HTTPS C2 channel with 60-second beacon intervals to `185.220.101.45:443`, providing the attacker persistent interactive access beyond the initial web shell.

- **Condition 4: Persistence survives reboots.** A registry Run key (`HKLM\Software\Microsoft\Windows\CurrentVersion\Run\WdiServiceHost`) ensures the Cobalt Strike Beacon re-executes automatically on every system restart the server cannot be declared clean without active forensic eradication followed by a rebuild.

- **Condition 5: State-sponsored threat actor.** APT28's confirmed involvement elevates the threat ceiling significantly, the actor has the resources, mandate, and operational patience to maintain access across months if not fully eradicated, and has a documented history of weaponising stolen data for follow-on intelligence operations.

---

### 1.2 - Scope Assessment

| Dimension | Assessment |
|---|---|
| **Systems Confirmed Compromised** | `web-server-01.techsecure.local` (192.168.1.50) |
| **Compromise Zone** | Web application directory `C:\inetpub\wwwroot\` and memory space of `w3wp.exe` (PID: 1204) |
| **Lateral Movement Confirmed** | Not confirmed at this stage (SMB scanning observed in network traffic, but no confirmed pivot to internal hosts) |
| **Internal Network Exposure** | Suspected (adversary conducted SMB port scanning from web-server-01 suggesting lateral movement intent) |
| **Data at Risk** | Customer PII, portal login credentials, transaction records, proprietary consulting data from `D:\ClientData` |

> **Scope Limitation:** 
>
>The current confirmed scope is limited to `web-server-01`. However, SMB network scanning activity originating from the compromised host should be treated as a lateral movement preparation, all internal hosts reachable from `192.168.1.50` must be included in the post-containment sweep.


### 1.3 - Business Impact Assessment

| Impact Category | Assessment |
|---|---|
| **Reputational Damage** | **Severe:** TechSecure operates as a security service provider. A confirmed breach of customer data fundamentally undermines client trust and creates immediate competitive vulnerability |
| **Downstream Client Risk** | **High:** Exfiltrated PII, email addresses, and plaintext credentials can be immediately weaponised in targeted phishing campaigns against TechSecure's clients. Downstream victim notifications may be required |
| **Regulatory Liability** | **Critical:** Breach of PII and financial records triggers mandatory disclosure obligations under applicable data protection frameworks (GDPR, local privacy laws). 72-hour notification windows may already be running |
| **Legal Exposure** | **High:** Plaintext storage of employee credentials (evidenced by `passwords.txt`) and the unrestricted file upload vulnerability suggest potential negligence liability if regulatory investigation follows |
| **Operational Disruption** | **Medium:** Web server must be taken offline for forensic analysis, full eradication, and rebuild client portal unavailability during this period |
| **Source Code Exposure** | **High:** Exfiltration of `application_source.zip` (850 MB) exposes TechSecure's proprietary application logic, enabling targeted application-layer attacks against the rebuilt system and potentially against clients running the same software |


## Part 2: Attack Timeline Reconstruction

The following timeline was reconstructed from correlated evidence across IIS access logs, Windows Security Event Logs (Event ID 4688 and 4657), EDR process telemetry, PCAP captures, and firewall logs.

![Lab-61 Attack Chain](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%20II/Screenshots/61_Attack_Chain.png)

**Total dwell time before detection:** 12 hours 15 minutes

**Data exfiltration window:** 3 hours (22:00–01:00 UTC)

**Time from Cobalt Strike deployment to SOC alert:** 2 minutes and 15 seconds

## Part 3: Evidence Collection and Preservation

### 3.1 - Evidence Inventory

| # | Evidence Type | Collected From | Location | Size | Hash (SHA-256) | Status |
|---|---|---|---|---|---|---|
| 1 | Memory Dump | Live system RAM | `/evidence/memory.dmp` | 16 GB | See CoC documentation | ✅ Collected |
| 2 | System Drive Image | C: drive | `/evidence/C_drive.img` | 500 GB | See CoC documentation | ✅ Collected |
| 3 | IIS Logs Archive | IIS log directory | `/evidence/iis_logs.tar` | 2 GB | See CoC documentation | ✅ Collected |
| 4 | Network PCAP | Network capture | `/evidence/network.pcap` | 500 MB | See CoC documentation | ✅ Collected |
| 5 | Malware Samples | Web root / memory | `/evidence/malware/` | 50 MB | See CoC documentation | ✅ Collected |

### 3.2 - Analysis Priority Order

Evidence was triaged and analysed in the following sequence, based on volatility (most volatile first) and investigative priority:

| Priority | Artifact | Rationale |
|---|---|---|
| **1st** | Memory Dump (`memory.dmp`) | **Most volatile:** captures live process state of `svchost.exe` (PID: 2847) and open network sockets before system shutdown. Irreplaceable after reboot. |
| **2nd** | Network PCAP (`network.pcap`) | Documents C2 communication patterns and exfiltration payload confirmation, critical for data breach scope assessment. |
| **3rd** | IIS & Event Logs (`iis_logs.tar`) | Maps the complete attacker command execution history and the initial HTTP-based entry vector. |
| **4th** | System Drive Image (`C_drive.img`) | Captures persistent artifacts registry modifications, file system changes, and deleted file recovery from unallocated space. |
| **5th** | Malware Samples (`svchost.exe`, `shell.aspx`) | Static and dynamic reverse engineering of both artefacts to complete the malware capability profile. |

### 3.3 - Chain of Custody

| Field | Value |
|---|---|
| **Evidence ID** | TECHSECURE-2025-001 |
| **Description** | IIS Web Server Forensic Artifact Bundle :- `web-server-01.techsecure.local` |
| **Collection Time** | 2025-11-06 03:00:00 UTC |
| **Collected By** | Lead Incident Response Analyst |
| **Source Location** | `web-server-01.techsecure.local` (192.168.1.50) |
| **Condition at Collection** | Live system (memory dump taken before shutdown; drive imaged post-isolation) |
| **Integrity Hash (SHA-256)** | `1a2b3c4d...` (full hash in evidence repository) |
| **Transferred To** | Digital Forensics Laboratory |
| **Storage Location** | Evidence Repository / Secure Read-Only Forensic Workstation |

**Chain of Custody Flow:**

```
web-server-01 (Evidence Collection @ 03:00 UTC)
        ↓
Secure Encrypted Transport
        ↓
Forensic Lab Repository (Ingestion & Hash Verification @ 03:30 UTC)
        ↓
Read-Only Forensic Workstation (Analysis — no write access to originals)
```

**Chain of Custody Integrity:** Unbroken. Cryptographic hashes calculated at acquisition were verified to match during lab ingestion — confirming zero modification of evidence between collection and analysis.

### 3.4 - Forensic Imaging

| Drive | Size | Imaging Tool | Hash (MD5) | Status |
|---|---|---|---|---|
| C: (System Drive) | 500 GB | FTK Imager | `a1b2c3d4e5f6...` | ✅ Complete |
| D: (Data Drive) | 1 TB | FTK Imager | `1a2b3c4d5e6f...` | ✅ Complete |

**Why bit-by-bit imaging is required:**

A bit-level physical image creates an exact replica of the entire drive including unallocated space, deleted files, and file system slack space. This matters for three reasons specific to this incident:

- The attacker deleted staging artefacts from `C:\inetpub\wwwroot\` after establishing the Cobalt Strike Beacon, deleted file recovery from unallocated space may reveal additional tools and scripts
- Web shell files (`shell.aspx`) may leave residual content in slack space even after deletion
- File system metadata (MFT entries, $LogFile, $UsnJrnl) preserves creation, modification, and access timestamps that would be overwritten by a logical file copy

A bit-by-bit image also preserves evidence admissibility, the original drive is never modified post-collection, and every analysis action is conducted against verified copies.

## Part 4: Forensic Analysis

### 4.1 - Malware Analysis: svchost.exe (Cobalt Strike Beacon)

#### Static Analysis

| Property | Value |
|---|---|
| **File Name** | `svchost.exe` |
| **File Path** | `C:\inetpub\wwwroot\svchost.exe` (web root, not the legitimate system path) |
| **File Size** | 347,648 bytes (339.5 KB) |
| **MD5** | `a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6` |
| **SHA-256** | `1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p7q8r9s0t1u2v3w4x5y6z7a8b9c0d1e2f` |
| **Digital Signature** | ❌ Unsigned |
| **Compilation Date** | 2025-11-04 18:30:00 UTC (24 hours before deployment) |
| **Packer / Obfuscation** | UPX packer (compresses and obfuscates binary to evade signature-based endpoint detection) |
| **Malware Family** | Cobalt Strike Beacon |

**Static Analysis Findings:**

The filename `svchost.exe` is a deliberate masquerade, the legitimate Windows Service Host process is located at `C:\Windows\System32\svchost.exe`. This copy was placed in `C:\inetpub\wwwroot\` and executed from the web root context, which is immediately anomalous. No legitimate `svchost.exe` binary should ever reside in or execute from the IIS web root directory.

The UPX packer obfuscates the binary's true import table and code structure preventing static signature matching against known Cobalt Strike Beacon patterns at rest on disk. The real malware code is unpacked entirely into memory at runtime.

#### Dynamic Analysis

| Property | Value |
|---|---|
| **Parent Process** | `w3wp.exe` (IIS Worker Process, PID: 1204) |
| **Spawned Process** | `svchost.exe` (PID: 2847) |
| **Command Line** | `svchost.exe -k netsvcs -p -s WdiServiceHost` |
| **Network Behaviour** | HTTPS C2 connection to `185.220.101.45:443` (60-second beacon intervals - AES-256 encrypted) |
| **Registry Modification** | `HKLM\Software\Microsoft\Windows\CurrentVersion\Run` → Value: `WdiServiceHost` |

**Process Execution Anomaly - The Key Detection Signal:**

The parent-child process relationship `w3wp.exe → svchost.exe` is inherently anomalous. The IIS worker process (`w3wp.exe`) handles HTTP requests, it has no legitimate operational reason to ever spawn `svchost.exe`. The combination of:

- Unexpected parent process (`w3wp.exe`)
- Execution from web root (`C:\inetpub\wwwroot\`) instead of `C:\Windows\System32\`
- Unsigned binary
- UPX packing

...produces a detection profile that should trigger immediate EDR behavioural alerts in any well-configured endpoint security deployment.

**Cobalt Strike Beacon Capabilities Confirmed:**

| Capability | Detail |
|---|---|
| **Remote Code Execution** | Executes arbitrary OS commands on behalf of the threat actor via the encrypted C2 channel |
| **C2 Communication** | HTTPS to `attacker-c2.ru` / `185.220.101.45:443` (60-second beacon intervals - AES-256 encryption) |
| **Data Exfiltration** | Encrypts and transfers staged client files outbound through the C2 channel |
| **Persistence** | Registry Run key (`WdiServiceHost`) ensures beacon survives reboots |
| **Lateral Movement** | SMB pivot capability enables remote access to internal network shares |
| **Anti-Analysis** | UPX packing evades static signature detection at rest |

### 4.2 - Web Shell Analysis: shell.aspx

| Property | Value |
|---|---|
| **File Name** | `shell.aspx` |
| **File Path** | `C:\inetpub\wwwroot\shell.aspx` |
| **MD5** | `b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7` |
| **File Type** | ASP.NET Web Application Script |
| **Upload Time** | 2025-11-05 14:00:00 UTC |
| **First Execution** | 2025-11-05 14:20:00 UTC |

**Functionality:** The web shell accepts HTTP GET requests with a `?cmd=` URL query parameter and executes the supplied value as an operating system command under the `IUSR_TECHSECURE` service account context. Output is returned in the HTTP response body, providing the attacker a fully functional remote command shell accessible from any web browser.

**Commands Executed via Web Shell (Reconstructed from IIS Logs):**

| Time (UTC) | Command | Purpose |
|---|---|---|
| 14:20:00 | `whoami` | Confirm execution context and service account identity |
| 14:25:00 | `ipconfig` | Map network interfaces and IP addressing |
| 14:35:00 | `systeminfo` | Enumerate OS version, hotfixes, and hardware profile |
| 14:45:00 | `dir C:\Users` | Identify user accounts on the system |
| 14:50:00 | `net user` | Enumerate all local user accounts and groups |
| 15:00:00 | `tasklist` | List all running processes |
| 15:10:00 | `netstat -ano` | Map active network connections and listening ports |
| 15:20:00 | `wmic logicaldisk get name` | Enumerate all drive letters and storage volumes |
| 20:00:00 | `dir D:\ClientData` | Discover target data directory contents |
| 20:15:00 | `copy D:\ClientData\*.*` | Begin staging all client data for exfiltration |

### 4.3 - YARA Rule Analysis

All five applied YARA rules returned confirmed TRUE matches with HIGH confidence:

| Rule Name | Result | Confidence | Matched Evidence |
|---|---|---|---|
| `Cobalt_Strike_Beacon.yar` | ✅ TRUE | HIGH | Known Cobalt Strike beacon signatures matched in `svchost.exe` (MD5: `a1b2c3d4...`) |
| `Suspicious_Process_Execution.yar` | ✅ TRUE | HIGH | Anomalous parent-child relationship: `w3wp.exe` → `svchost.exe` (Event ID 4688) |
| `Data_Exfiltration_Patterns.yar` | ✅ TRUE | HIGH | Large data transfer (~2.3 GB) to external C2 IP `185.220.101.45:443`  (confirmed in PCAP) |
| `Persistence_Mechanisms.yar` | ✅ TRUE | HIGH | Registry key `HKLM\...\Run\WdiServiceHost` modification (confirmed in Event ID 4657) |
| `APT28_Indicators.yar` | ✅ TRUE | HIGH | Multiple infrastructure, malware, and behavioural TTP indicators matching known APT28 patterns |

**False Positive Assessment:** Zero false positives identified. Each YARA rule match correlates directly with distinct forensic evidence across at minimum three independent sources, EDR process logs, Windows Event Logs, PCAP captures, file system artefacts, and registry modifications all corroborate the same findings.

### 4.4 - Log Analysis

#### IIS Access Log Findings

- **First suspicious activity:** Automated web vulnerability scanning recorded at **12:00:00 UTC**, two hours before initial compromise. The scanning activity matches common web vulnerability scanner fingerprints and directory enumeration patterns.

- **Web shell deployment:** Successful HTTP POST uploading `shell.aspx` to `C:\inetpub\wwwroot\` at **14:00:00 UTC**. Server returned HTTP 200, confirming the upload was accepted without server-side validation.

- **Web shell first access:** HTTP GET to `/shell.aspx?cmd=whoami` at **14:20:00 UTC**, confirming successful web shell execution under `IUSR_TECHSECURE`.

- **Data staging commands:** Directory listing and bulk copy commands observed from **20:00 UTC**, confirming `D:\ClientData` as the primary data staging target.

#### Windows Security Event Log Findings

**Event ID 4688 (Process Creation) Key Events:**

| Time (UTC) | Parent Process | Child Process | Significance |
|---|---|---|---|
| 14:15:00 | `w3wp.exe` (PID: 1204) | `cmd.exe` | Web shell spawning system command interpreter (first execution indicator) |
| 14:15:00 | `cmd.exe` | `shell.aspx` | ASPX web shell executing operating system commands |
| 02:12:45 | `w3wp.exe` (PID: 1204) | `svchost.exe` (PID: 2847) | Cobalt Strike Beacon spawned from IIS worker (critical detection event) |

**Event ID 4657 (Registry Modification) Persistence:**

`HKLM\Software\Microsoft\Windows\CurrentVersion\Run` → Value `WdiServiceHost` → Data: `svchost.exe -k netsvcs -p -s WdiServiceHost`

This registry entry ensures the Cobalt Strike Beacon executes automatically on every system startup, providing reboot-persistent C2 access that survives most standard incident response actions that do not include explicit registry eradication.

## Part 5: Network Traffic Analysis

### 5.1 - C2 Communication Profile

**Beacon Characteristics:**

| Property | Value |
|---|---|
| **Source** | `192.168.1.50:52847` (web-server-01) |
| **Destination** | `185.220.101.45:443` (attacker-c2.ru) |
| **Protocol** | TCP / HTTPS |
| **Encryption** | AES-256 |
| **Beacon Interval** | 60 seconds (periodic and consistent with Cobalt Strike default configuration) |
| **First Observed** | 2025-11-05 22:00:00 UTC |
| **Last Observed** | 2025-11-06 02:30:00 UTC (containment) |

**Detection Note:** The use of HTTPS (TCP port 443) over encrypted channels is a deliberate evasion strategy. Most enterprise outbound firewall policies permit port 443 traffic without inspection. The 60-second beacon interval produces low-volume periodic traffic that blends into normal HTTPS background noise, this pattern is specifically designed to avoid threshold-based volume alerts.

### 5.2 - Exfiltrated Assets (Confirmed)

| Asset | Filename | Size | Sensitivity |
|---|---|---|---|
| Customer database backup | `ClientData_2025-11-05.bak` | 1.2 GB | 🔴 Critical (Full customer PII database) |
| Client contact information | `clients.xlsx` | 45 MB | 🔴 Critical (Names, emails, phone numbers) |
| Financial transaction records | `transactions_2025.csv` | 320 MB | 🔴 Critical (Financial data, regulatory disclosure likely required) |
| Employee credentials | `passwords.txt` | 2.3 MB | 🔴 Critical (Plaintext passwords immediate forced reset required) |
| Application source code | `application_source.zip` | 850 MB | 🔴 High (Proprietary IP - enables targeted application attacks) |

**Total volume exfiltrated:** ~2.3 GB

**Exfiltration window:** 2025-11-05 22:00:00 UTC to 2025-11-06 01:00:00 UTC (3 hours)

## Part 6: Threat Intelligence Integration

### 6.1 - IOC Enrichment

**IP Address: 185.220.101.45**

| Field | Value |
|---|---|
| **Geolocation** | Netherlands |
| **ASN** | AS39798 |
| **Hosting Provider** | Bulletproof Hosting known policy of non-compliance with law enforcement takedown requests |
| **Reputation** | Malicious, confirmed C2 infrastructure |
| **Previous Incidents** | Associated with multiple APT28 espionage campaigns deploying Cobalt Strike Beacons |
| **Backup C2 IPs** | `185.220.102.8`, `185.220.103.25` |

**Domain: attacker-c2.ru**

| Field | Value |
|---|---|
| **Registrar** | NameCheap |
| **Registration Date** | 2025-10-15 (21 days before the attack) |
| **WHOIS Status** | Privacy-protected |
| **SSL Certificate** | Self-signed (valid 2025-10-15 to 2026-10-15) |
| **Associated Domains** | `backup-c2.ru`, `staging-c2.ru` |

**Intelligence Assessment:** 

The 21-day registration gap between domain registration (October 15) and attack execution (November 5) is consistent with deliberate operational preparation, the actor registered infrastructure, configured it, and allowed a brief settling period before use. The `.ru` TLD combined with bulletproof Netherlands hosting is a well-documented APT28 infrastructure pattern.

### 6.2 - Threat Actor Profile: APT28 (Fancy Bear)

| Field | Value |
|---|---|
| **Actor Name** | APT28 |
| **Known Aliases** | Fancy Bear, Strontium, Sednit, Sofacy, Pawn Storm, Tsar Team |
| **Attributed To** | Russian GRU (Military Intelligence Directorate) |
| **Active Since** | Mid-2000s |
| **Primary Motivation** | State-sponsored cyber espionage (strategic intelligence collection) |
| **Known Targets** | Government, military, defence, corporate entities worldwide |
| **Sophistication Level** | HIGH (Advanced Persistent Threat) |

**Tools Confirmed in This Incident:**
- Cobalt Strike Beacon (UPX-packed custom HTTPS variant)
- Custom ASPX web shell (`shell.aspx`)
- Native Windows utilities: `cmd.exe`, `netstat`, `wmic`, `systeminfo`, `net user`


## Part 7: MITRE ATT&CK Full Mapping

| Tactic | Technique ID | Technique Name | Evidence |
|---|---|---|---|
| **Reconnaissance** | T1595.002 | Active Scanning: Vulnerability Scanning | IIS access logs showing automated web vulnerability scan and directory probing from 12:00 UTC |
| **Initial Access** | T1190 | Exploit Public-Facing Application | Web shell `shell.aspx` uploaded via unrestricted file upload endpoint to `C:\inetpub\wwwroot\` |
| **Execution** | T1059.001 / T1059.003 | Command and Scripting Interpreter: PowerShell / Cmd | Web shell executing `cmd.exe` commands via `?cmd=` URL parameters (whoami, ipconfig, net user, dir, copy) |
| **Persistence** | T1543.003 | Create or Modify System Process: Windows Service (Run Key) | Registry value `WdiServiceHost` added under `HKLM\...\CurrentVersion\Run` (Event ID 4657) |
| **Privilege Escalation** | T1548.002 | Abuse Elevation Control Mechanism: UAC Bypass | UAC bypass attempt recorded in Windows Security Event logs |
| **Defense Evasion** | T1036.005 | Masquerading: Match Legitimate Name or Location | `svchost.exe` naming convention used by legitimate Windows process, UPX packing to evade static AV signatures |
| **Credential Access** | T1552.001 | Unsecured Credentials: Credentials In Files | `passwords.txt` (2.3 MB plaintext credential file) exfiltrated from `D:\ClientData` |
| **Discovery** | T1082 | System Information Discovery | `systeminfo`, `ipconfig`, `net user` executed via web shell for host enumeration |
| **Discovery** | T1057 | Process Discovery | `tasklist` and `netstat -ano` executed for process and network connection mapping |
| **Lateral Movement** | T1021.002 | Remote Services: SMB/Windows Admin Shares | SMB port scanning from `192.168.1.50` observed in network traffic, lateral movement preparation |
| **Collection** | T1005 | Data from Local System | `dir D:\ClientData` and `copy D:\ClientData\*.*` bulk staging of client database, financial records, and source code |
| **Exfiltration** | T1041 | Exfiltration Over C2 Channel | ~2.3 GB transferred outbound to `185.220.101.45:443` over AES-256 encrypted HTTPS, confirmed in PCAP |
| **Command & Control** | T1071.001 | Application Layer Protocol: Web | HTTPS-based Cobalt Strike Beacon with 60-second intervals to `attacker-c2.ru` (blends with legitimate HTTPS traffic) |

**Attack Chain Summary:**
```
Vulnerability Scanning (T1595.002)
    → Web Shell Upload (T1190)
        → System Discovery & Staging (T1082, T1057, T1005)
            → Registry Run Key Persistence (T1543.003)
                → Cobalt Strike Beacon Deployment (T1071.001)
                    → AES-256 Encrypted Exfiltration (T1041)
```

## Part 8: Remediation Plan

### Immediate Actions (Execute Within 24 Hours)

| Action | Target | Method | Priority |
|---|---|---|---|
| **Network Isolation** | `web-server-01` (192.168.1.50) | Disconnect from internal network; implement perimeter firewall blocks for `185.220.101.45`, `185.220.102.8`, `185.220.103.25` | 🔴 Critical |
| **Process Termination** | `svchost.exe` PID 2847 | Terminate via EDR remote action and do not kill via standard process manager as beacon may respawn | 🔴 Critical |
| **Web Shell Removal** | `C:\inetpub\wwwroot\shell.aspx` | Delete and verify removal, hash-verify against collected forensic copy post-deletion | 🔴 Critical |
| **Persistence Eradication** | Registry key `HKLM\...\Run\WdiServiceHost` | Delete registry value and verify via `reg query` before declaring clean | 🔴 High |
| **Credential Reset** | `IUSR_TECHSECURE` + all accounts in `passwords.txt` | Force password reset for service account and every credential exposed in the exfiltrated plaintext file | 🔴 High |
| **C2 Domain Block** | `attacker-c2.ru`, `backup-c2.ru`, `staging-c2.ru` | Add to DNS sinkhole and perimeter firewall block list | 🔴 High |

### Short-Term Actions (Execute Within 2 Weeks)

| Action | Method |
|---|---|
| **Deploy WAF with strict upload validation** | Implement Web Application Firewall in front of the client portal; enforce server-side file type validation, allowlist only expected MIME types; block executable extensions (.aspx, .asp, .php, .exe) at the upload endpoint |
| **Disable IIS execute permissions on upload directories** | Configure IIS permissions so all user-accessible upload directories have Execute Permissions explicitly set to None |
| **Block w3wp.exe child process spawning** | Configure EDR behavioural rule to alert and terminate any child processes spawned by `w3wp.exe` or `httpd.exe`, no legitimate web application requires IIS worker process to spawn system shells |
| **Enforce outbound egress filtering** | Restrict web server subnet from establishing arbitrary outbound HTTPS connections with whitelist approved destinations only |
| **Full credential audit** | Enumerate all accounts with access to the affected server and `D:\ClientData` then force rotation for all, not only those appearing in `passwords.txt` |

### Long-Term Actions (Execute Within 60 Days)

| Action | Method |
|---|---|
| **Clean server rebuild from gold image** | Rebuild `web-server-01` from a verified clean base image, do not attempt to reuse the compromised OS installation |
| **Implement least privilege for IIS service account** | Restrict `IUSR_TECHSECURE` file system write access to only essential application directories and remove `D:\ClientData` access entirely |
| **Deploy SIEM persistence monitoring** | Continuous alerting for any modification to `HKLM\...\Run` and `HKCU\...\Run` registry keys and alert on any new unsigned executable added to autostart locations |
| **Implement network anomaly detection** | Deploy NTA (Network Traffic Analysis) with threshold-based alerting for unexpected large-volume outbound HTTPS flows from server subnets |
| **Conduct tabletop exercise** | Simulate an APT28-style attack chain with the SOC team to validate updated detection rules and test the new incident response playbook against the TTPs observed in this incident |

---

## Consolidated Indicators of Compromise (IOCs)

### Malware Hashes

| Type | Hash | File |
|---|---|---|
| MD5 | `a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6` | `svchost.exe` (Cobalt Strike Beacon) |
| SHA-256 | `1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p7q8r9s0t1u2v3w4x5y6z7a8b9c0d1e2f` | `svchost.exe` |
| MD5 | `b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7` | `shell.aspx` (ASPX Web Shell) |

### Network Indicators

| Type | Value | Context |
|---|---|---|
| C2 IP | `185.220.101.45` | Primary (Bulletproof hosting, Netherlands, AS39798) |
| C2 IP | `185.220.102.8` | Backup C2 (same /24 subnet) |
| C2 IP | `185.220.103.25` | Backup C2 (same /24 subnet) |
| Domain | `attacker-c2.ru` | Primary C2 domain (registered 2025-10-15) |
| Domain | `backup-c2.ru` | Associated backup domain |
| Domain | `staging-c2.ru` | Associated staging domain |
| Port | TCP 443 (HTTPS) | C2 and exfiltration channel |

### Host Indicators

| Type | Value | Context |
|---|---|---|
| File Path | `C:\inetpub\wwwroot\shell.aspx` | Web shell (web root) |
| File Path | `C:\inetpub\wwwroot\svchost.exe` | Cobalt Strike Beacon - web root (anomalous) |
| Registry Key | `HKLM\Software\Microsoft\Windows\CurrentVersion\Run\WdiServiceHost` | Persistence mechanism |
| Registry Value | `svchost.exe -k netsvcs -p -s WdiServiceHost` | Beacon autostart command |
| Process | `svchost.exe` PID 2847 | Malicious,spawned from `w3wp.exe` |
| Parent Process | `w3wp.exe` PID 1204 | IIS worker, exploited execution context |
| Service Account | `IUSR_TECHSECURE` | Compromised execution context |
| Source Port | `52847` | Outbound C2 connection source port |

## Detection Opportunities

This incident exposed two detection gaps that, if closed, would have reduced the dwell time from 12 hours 15 minutes to under 15 minutes:

- **Detection Gap 1 - w3wp.exe spawning cmd.exe (14:15 UTC):** An EDR behavioural rule alerting on `w3wp.exe` spawning any command interpreter (`cmd.exe`, `powershell.exe`) would have fired at 14:15 UTC, within 15 minutes of web shell first execution. This alone would have prevented the entire exfiltration phase. **Recommended rule: Alert and kill on any child process of `w3wp.exe` or `httpd.exe` that is a shell or scripting engine.**

- **Detection Gap 2 - Large volume outbound HTTPS from server subnet (22:00 UTC):** A network-based alert for sustained high-volume outbound HTTPS traffic originating from the `192.168.1.x` server subnet would have fired at 22:00 UTC, 30 minutes into the exfiltration window. Even if the web shell went undetected, a volume-based egress alert would have cut the exfiltration short. **Recommended threshold: Alert on any single outbound HTTPS flow from a server subnet exceeding 100 MB.**

---

> **Disclaimer:** This report documents a structured Incident Response and Digital Forensics training exercise. TechSecure Solutions is a fictional organisation. All IP addresses, domain names, file hashes, event log entries, and forensic artefacts described in this report are simulated for educational analysis. No real systems, organisations, or individuals were involved. This document is published as a portfolio artefact demonstrating applied DFIR methodology.
