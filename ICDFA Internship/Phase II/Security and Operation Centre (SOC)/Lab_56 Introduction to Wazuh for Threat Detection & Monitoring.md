# Introduction to Wazuh for Threat Detection & Monitoring

## Executive Summary

This lab established a fully operational Wazuh Security Information and Event Management (SIEM) environment from scratch. A standalone Ubuntu VirtualBox instance was configured as the Wazuh Manager and dashboard server. A Windows 10 host was enrolled as a monitored endpoint (agent) and File Integrity Monitoring (FIM) was configured to detect real-time file system changes on a designated sensitive directory.

**Key outcomes:**

- Wazuh Manager deployed on Ubuntu VM (IP: `192.168.43.137`) with authenticated dashboard access on port `443` and local API on port `55000`
- Windows 10 agent (`FABELT`, Agent ID: `004`) enrolled, authenticated, and confirmed **Active** in the dashboard
- FIM configured on `C:\Users\user\SensitiveFiles` with `realtime="yes"` — monitoring live file events
- Three test events generated and detected in real-time: **file added**, **file modified**, **file deleted**
- Alert details inspected: cryptographic checksum changes (MD5, SHA1, SHA256), size changes, timestamps, and user attribution captured per alert
- SOC analysis questions answered demonstrating analyst-level understanding of false positive management, ransomware detection via FIM, and centralized SIEM advantages

## Lab Environment

| Component | Detail |
|---|---|
| **Wazuh Manager Host** | Ubuntu VM — VirtualBox, IP: `192.168.43.137` |
| **Wazuh Version** | v4.12.0 |
| **Dashboard Access** | `https://192.168.43.137` (self-signed certificate) |
| **API Port** | `55000` |
| **Windows Agent Host** | Windows 10 Pro (10.0.19045.6466) — IP: `192.168.43.17` |
| **Agent Name** | `FABELT` (Agent ID: `004`) |
| **Monitored Directory** | `C:\Users\user\SensitiveFiles` |
| **FIM Mode** | Real-time (`realtime="yes"`) |
| **Network** | Host-only VirtualBox network — isolated lab environment |


## Part 1 — Accessing the Wazuh Dashboard

### Setup Steps

After Wazuh Manager installation completed on the Ubuntu VM:

1. Retrieved the Ubuntu VM IP address:
```bash
ip addr show
# Result: 192.168.43.137
```

2. Opened a web browser and navigated to:
```
https://192.168.43.137
```

3. Accepted the browser security warning — expected behavior for a self-signed certificate in a lab environment

4. Logged in using the credentials provided at the end of the installation script output (default username: `admin`)

### Methodology

Configured a standalone Ubuntu VirtualBox instance with hardware bypass flags and resolved internal API credential and YAML mapping desyncs between the Wazuh manager and dashboard services. The Wazuh services successfully stabilized, allowing:

- Authenticated dashboard access via HTTPS
- Fully responsive local API connection on port `55000`

This established a fully operational, centralized security monitoring lab environment ready for endpoint agent deployment.

### Dashboard — Initial State

Upon login, the Wazuh Overview dashboard displayed:

| Metric | Value |
|---|---|
| **Active Agents** | 0 (pre-agent enrollment) |
| **Disconnected Agents** | 1 |
| **Critical Severity Alerts** | 0 (Rule level 15 or higher) |
| **High Severity Alerts** | 0 (Rule level 12–14) |
| **Medium Severity Alerts** | 3 (Rule level 7–11) |
| **Low Severity Alerts** | 23 (Rule level 0–6) |

---

> 📸 **[SCREENSHOT 1 — Insert Here]**
> **Caption:** Wazuh Dashboard Overview page at `https://192.168.43.137` showing Agents Summary panel (Active: 0, Disconnected: 1) and Last 24 Hours Alerts panel (Critical: 0, High: 0, Medium: 3, Low: 23). Dashboard confirmed accessible with authenticated session.
> **Source:** Your Wazuh dashboard overview screenshot from the PDF (Part 1).

---

---

## Part 2 — Installing the Wazuh Agent (Windows)

### Agent Download and Installation

The Wazuh Windows Agent MSI installer (v4.12.0) was downloaded from the official Wazuh packages repository:

```
https://packages.wazuh.com/4.x/windows/wazuh-agent-4.12.0-1.msi
```

The MSI file was executed on the Windows 10 host and installed with default settings. After installation, the Wazuh Agent Manager was launched from the Start Menu.

### Initial Agent State (Pre-Registration)

Immediately after installation, before authentication key import:

| Field | Value |
|---|---|
| **Wazuh Version** | v4.12.0 |
| **Agent Auth Key** | Not imported (0) — 0 |
| **Status** | Not Running — requires authentication key import |
| **Manager IP** | `0.0.0.0` (not yet configured) |
| **Authentication Key** | `<insert_auth_key_here>` (placeholder) |

---

> 📸 **[SCREENSHOT 2 — Insert Here]**
> **Caption:** Wazuh Agent Manager GUI on Windows host showing initial unconfigured state — "Agent: Auth key not imported. (0) – 0", "Status: Require import of authentication key – Not Running", Manager IP field showing `0.0.0.0`, Authentication key field showing `<insert_auth_key_here>`. Wazuh v4.12.0 visible in top bar.
> **Source:** Your Wazuh Agent initial state screenshot from the PDF (Part 2).

---

---

## Part 3 — Agent-Manager Registration

### Step 1 — Create Agent Entry on Ubuntu Manager

On the Ubuntu VM terminal, the agent management tool was launched:

```bash
sudo /var/ossec/bin/manage_agents
```

**Agent Registration Process:**

```
Wazuh v4.12.0 Agent manager.
The following options are available:
    (A)dd an agent (A).
    (E)xtract key for an agent (E).
    (L)ist already added agents (L).
    (R)emove an agent (R).
    (Q)uit.

Choose your action: A,E,L,R or Q: A

Adding a new agent (use '\q' to return to the main menu).
Please provide the following:
    * A name for the new agent: Windows-SOC-Lab
    * The IP Address of the new agent: 192.168.43.42
Confirm adding it?(y/n): y
Agent added with ID 003.
```

**Key Extraction:**

```
Choose your action: A,E,L,R or Q: E

Available agents:
    ID: 003, Name: Windows-SOC-Lab, IP: 192.168.43.42
Provide the ID of the agent to extract the key (or '\q' to quit): 003

Agent key information for '003' is:
MDAzIFdpbmRvd3MtU09DLUxhYiAxOTIuMTY4LjQzLjQyIDk2ODkwOGU5YmI3MzY1YjMzODk5MDY...
```

---

> 📸 **[SCREENSHOT 3 — Insert Here]**
> **Caption:** Ubuntu terminal showing `sudo /var/ossec/bin/manage_agents` output — Wazuh v4.12.0 Agent Manager menu, action "A" selected, agent name "Windows-SOC-Lab" entered, IP "192.168.43.42" assigned, confirmed with "y", result: "Agent added with ID 003."
> **Source:** Your manage_agents Add agent terminal screenshot from the PDF (Part 3, Step 1a).

---

> 📸 **[SCREENSHOT 4 — Insert Here]**
> **Caption:** Ubuntu terminal showing manage_agents with action "E" (Extract key) selected — Available agents showing ID: 003, Name: Windows-SOC-Lab, IP: 192.168.43.42. Agent key information for '003' displayed as a long Base64-encoded authentication string.
> **Source:** Your manage_agents Extract key terminal screenshot from the PDF (Part 3, Step 1b).

---

### Step 2 — Apply Authentication Key on Windows Agent

On the Windows host, the Wazuh Agent Manager was opened and configured:

1. **Opened** Wazuh Agent Manager from Start Menu
2. **Pasted** the extracted key into the **Authentication key** field
3. **Entered** the Ubuntu VM IP (`192.168.43.137`) in the **Manager IP** field
4. **Clicked Save** then **Restart** to apply the configuration

**Final Agent Configuration:**

| Field | Value |
|---|---|
| **Agent Name** | FABELT |
| **Agent ID** | 004 |
| **IP Assignment** | any |
| **Status** | **Running** ✅ |
| **Manager IP** | `192.168.43.137` |
| **Authentication Key** | `MDA0IEZBQkVMVCBhbnkgZTh...` (truncated) |

---

> 📸 **[SCREENSHOT 5 — Insert Here]**
> **Caption:** Wazuh Agent Manager GUI on Windows host showing successful registration — Agent: FABELT (004) – any, Status: Running, Manager IP: 192.168.43.137, Authentication key: MDA0IEZBQkVMVCBhbnkgZTh... (populated), bottom bar showing "Restarted" and "Revision rc1".
> **Source:** Your Wazuh Agent running state screenshot from the PDF (Part 3, Step 2).

---

### Verification — Agent Active in Dashboard

After restarting the agent service, the Wazuh Dashboard Agents page was checked. The `FABELT` agent appeared with a green **active** status dot within approximately one minute.

**Agents List — Dashboard Confirmation:**

| ID | Name | IP Address | Group | OS | Cluster Node | Version | Status |
|---|---|---|---|---|---|---|---|
| 004 | FABELT | 192.168.43.17 | default | Microsoft Windows 10 Pro 10.0.19045.6466 | node01 | v4.12.0 | ● **active** |

---

> 📸 **[SCREENSHOT 6 — Insert Here]**
> **Caption:** Wazuh Dashboard Agents page showing filter "status=active" with agent FABELT (ID: 004, IP: 192.168.43.17, Group: default, OS: Microsoft Windows 10 Pro 10.0.19045.6466, Cluster: node01, Version: v4.12.0, Status: ● active) confirmed with green active indicator. "Agents (1)" count shown at top.
> **Source:** Your Wazuh Agents dashboard screenshot from the PDF (Part 3, Verification).

---

---

## Part 4 — Implementing File Integrity Monitoring (FIM)

### What is File Integrity Monitoring?

File Integrity Monitoring (FIM) is a security control that detects unauthorized file system changes by continuously monitoring specified directories for additions, modifications, and deletions. Wazuh's FIM engine (syscheck) computes cryptographic hashes (MD5, SHA1, SHA256) of monitored files at baseline and re-computes them on each change event — triggering an alert whenever a mismatch is detected.

FIM is a critical detective control for:
- Detecting ransomware (mass file modification events)
- Identifying unauthorized configuration changes
- Catching insider threats planting or deleting files
- Compliance requirements (PCI DSS Requirement 11.5, HIPAA, SOC 2)

---

### Step 1 — Edit Agent ossec.conf Configuration

On the Windows host, the agent configuration file was opened as Administrator:

```
C:\Program Files (x86)\ossec-agent\ossec.conf
```

The `<syscheck>` section was located and the following real-time monitoring directive was added:

```xml
<!-- File integrity monitoring -->
<syscheck>

    <directories realtime="yes">C:\Users\user\SensitiveFiles</directories>

    <disabled>no</disabled>

</syscheck>
```

**Configuration Parameters Explained:**

| Parameter | Value | Effect |
|---|---|---|
| `realtime="yes"` | Enabled | Uses Windows `ReadDirectoryChangesW` API to detect changes instantly — not on scan cycle |
| `<disabled>no</disabled>` | FIM active | Ensures syscheck module is not disabled |
| Directory path | `C:\Users\user\SensitiveFiles` | Specific directory under real-time monitoring |

**Why `realtime="yes"` matters:** Without this attribute, syscheck only scans on a configured schedule (default: every 12 hours). With `realtime="yes"`, Wazuh uses the Windows native `ReadDirectoryChangesW` kernel API to receive instant filesystem change notifications — enabling detection within seconds of an event.

---

> 📸 **[SCREENSHOT 7 — Insert Here]**
> **Caption:** Notepad editor showing ossec.conf with the `<syscheck>` section visible — `</sca>` tag above, then `<!-- File integrity monitoring -->` comment, then `<syscheck>` opening tag, then `<directories realtime="yes">C:\Users\user\SensitiveFiles</directories>`, then `<disabled>no</disabled>` — confirming real-time FIM configuration saved.
> **Source:** Your ossec.conf Notepad screenshot from the PDF (Part 4, Step 1).

---

### Step 2 — Restart the Agent

After saving `ossec.conf`, the Wazuh agent service was restarted to apply the new FIM configuration. This was done via the Wazuh Agent Manager GUI (Restart button) or through the Windows Services console (`services.msc`).

**Why restart is required:** Wazuh reads `ossec.conf` at startup only. Configuration changes do not take effect until the service restarts and re-reads its configuration file, re-establishing the `ReadDirectoryChangesW` monitoring handles for newly added directory paths.

---

---

## Part 5 — SOC Verification & Alert Analysis

### Step 1 — Create the Monitored Folder

The monitored directory was created on the Windows host at:

```
C:\Users\user\SensitiveFiles
```

This folder was confirmed present alongside standard Windows user directories (OneDrive, Pictures, PyCharmProjects, Saved Games, Searches, test).

---

> 📸 **[SCREENSHOT 8 — Insert Here]**
> **Caption:** Windows File Explorer showing `This PC > Prime (C:) > Users > user` directory listing — SensitiveFiles folder visible with creation date "9/6/2026 4:29 AM" as a File folder type, alongside other standard directories (OneDrive, Pictures, PyCharmProjects, Saved Games, Searches, test).
> **Source:** Your Windows File Explorer showing SensitiveFiles folder from the PDF (Part 5, Step 1).

---

### Step 2 — Generate Test Events

Three deliberate file system events were created inside `C:\Users\user\SensitiveFiles` to verify FIM detection:

| Event | Action | File | Purpose |
|---|---|---|---|
| 1 | **Created** | `secret_plan.txt` | Test file addition detection |
| 2 | **Modified** | `secret_plan.txt` | Test checksum change detection |
| 3 | **Deleted** | `secret_plan.txt` / `new text document.txt` | Test file deletion detection |

---

> 📸 **[SCREENSHOT 9 — Insert Here]**
> **Caption:** Windows File Explorer showing `This PC > Prime (C:) > Users > user > SensitiveFiles` directory containing `secret_plan.txt` — Date modified: 9/6/2026 4:51 AM, Type: Text Document, Size: 1 KB — confirming the test file was created successfully inside the monitored folder.
> **Source:** Your SensitiveFiles folder with secret_plan.txt screenshot from the PDF (Part 5, Step 2).

---

### Step 3 — Investigate Alerts in the Wazuh Dashboard

The Wazuh Dashboard Security Events page was opened and filtered by `rule.groups: "syscheck"` to display only File Integrity Monitoring alerts.

**FIM Recent Events — Dashboard View:**

| Time | Path | Action | Rule Description | Rule Level | Rule ID |
|---|---|---|---|---|---|
| Sep 6, 2026 @ 04:51:35.892 | `c:\users\user\sensitivefiles\secret_plan.txt` | **modified** | Integrity checksum changed. | 7 | 550 |
| Sep 6, 2026 @ 04:50:53.125 | `c:\users\user\sensitivefiles\secret_plan.txt` | **added** | File added to the system. | 5 | 554 |
| Sep 6, 2026 @ 04:50:53.057 | `c:\users\user\sensitivefiles\new text document.txt` | **deleted** | File deleted. | 7 | 553 |
| Sep 6, 2026 @ 04:50:45.197 | `c:\users\user\sensitivefiles\new text document.txt` | **added** | File added to the system. | 5 | 554 |

**Alert Rule ID Reference:**

| Rule ID | Description | Level |
|---|---|---|
| **550** | Integrity checksum changed (file modified) | 7 — Medium |
| **553** | File deleted from monitored directory | 7 — Medium |
| **554** | New file added to monitored directory | 5 — Low |

---

> 📸 **[SCREENSHOT 10 — Insert Here]**
> **Caption:** Wazuh Dashboard "FIM: Recent events" panel showing four events in the SensitiveFiles directory — secret_plan.txt (modified, Rule 550, Level 7), secret_plan.txt (added, Rule 554, Level 5), new text document.txt (deleted, Rule 553, Level 7), new text document.txt (added, Rule 554, Level 5). All timestamped Sep 6, 2026 between 04:50:45 and 04:51:35.
> **Source:** Your Wazuh FIM Recent Events dashboard screenshot from the PDF (Part 5, Step 3).

---

### Detailed Alert Inspection — File Modified Event

The `secret_plan.txt` modification alert (Rule 550) was clicked to inspect the full alert details:

**File Metadata Panel:**

| Field | Value |
|---|---|
| **File Path** | `c:\users\user\sensitivefiles\secret_plan.txt` |
| **Last Analysis** | Sep 6, 2026 @ 04:51:44.000 |
| **Last Modified** | Sep 6, 2026 @ 04:51:44.000 |
| **User** | Fatai\ Asekun |
| **User ID** | S-1-5-21-959415697-2057280212-4073888110-1001 |
| **Size** | 37 Bytes |
| **MD5** | `af4be7ac5f5cc635fe981769196ebbf8` |
| **SHA1** | `431343d07a0b6ffebbfa4dce3f27db976f7f113b` |
| **SHA256** | `bf763e38365432c43eac52968e23daeecef6d40c38da2283c631cbcb922f3264` |

---

> 📸 **[SCREENSHOT 11 — Insert Here]**
> **Caption:** Wazuh alert detail panel for `c:\users\user\sensitivefiles\secret_plan.txt` showing: Last analysis (Sep 6, 2026 @ 04:51:44.000), Last modified (Sep 6, 2026 @ 04:51:44.000), User (Fatai\ Asekun), User ID (S-1-5-21-959415697-2057280212-4073888110-1001), Size (37 Bytes), MD5 (af4be7ac5f5cc635fe981769196ebbf8), SHA1 (431343d07a0b6ffebbfa4dce3f27db976f7f113b), SHA256 (bf763e38365432c43eac52968e23daeecef6d40c38da2283c631cbcb922f3264).
> **Source:** Your Wazuh alert details panel screenshot from the PDF (Part 5, Step 3 — file detail view).

---

---

## Lab Challenge & Analysis Questions

---

### Question 1 — Alert Triage: What specific details tell you what changed?

**Alert Investigated:** `secret_plan.txt` — Rule 550 (Integrity checksum changed)

**Full Log Entry (`full_log` field):**

```
Changed attributes: size, mtime, md5, sha1, sha256
Size changed from '0' to '37'
Old modification time was: '1788666649', now it is '1788666704'
Old md5sum was: 'd41d8cd98f00b204e9800998ecf8427e'
New md5sum is: 'af4be7ac5f5cc635fe981769196ebbf8'
Old sha1sum was: 'da39a3ee5e6b4b0d3255bfef95601890afd80709'
New sha1sum is: '431343d07a0b6ffebbfa4dce3f27db976f7f113b'
```

**Decoded Alert Analysis:**

| Changed Attribute | Old Value | New Value | What It Means |
|---|---|---|---|
| **Size** | 0 bytes | 37 bytes | File went from empty to containing 37 bytes of content |
| **mtime** | 1788666649 | 1788666704 | Modification timestamp advanced by 55 seconds |
| **MD5** | `d41d8cd98f00b204e9800998ecf8427e` | `af4be7ac5f5cc635fe981769196ebbf8` | Old value is the MD5 of an empty file (0 bytes) — new value confirms content was written |
| **SHA1** | `da39a3ee5e6b4b0d3255bfef95601890afd80709` | `431343d07a0b6ffebbfa4dce3f27db976f7f113b` | Old value is SHA1 of empty file — new value confirms content changed |

**Answer:** The alert reveals the change through **cryptographic checksum comparison** and **file size change**. The MD5 hash `d41d8cd98f00b204e9800998ecf8427e` is the universally known hash of an empty (zero-byte) file — its presence as the "old" value confirms the file existed but was empty. The new MD5 `af4be7ac5f5cc635fe981769196ebbf8` proves content was written. Together, the checksum deltas and size change from 0 → 37 bytes provide cryptographic proof of exactly what changed, who changed it, and when.

---

> 📸 **[SCREENSHOT 12 — Insert Here]**
> **Caption:** Wazuh alert detail view showing `agent.name: FABELT`, `decoder.name: syscheck_integrity_changed`, and `full_log` field containing: "Changed attributes: size, mtime, md5, sha1, sha256", "Size changed from '0' to '37'", "Old modification time was: '1788666649', now it is '1788666704'", "Old md5sum was: 'd41d8cd98f00b204e9800998ecf8427e'", "New md5sum is: 'af4be7ac5f5cc635fe981769196ebbf8'", old and new SHA1 values.
> **Source:** Your Wazuh full_log alert detail screenshot from the PDF (Lab Challenge Q1).

---

### Question 2 — False Positive Identification: Which folders generate too many FIM alerts?

Two high-noise directories that would generate an overwhelming volume of false positive FIM alerts in a corporate environment:

**1. `C:\Windows\Temp`**

Windows Temp directories experience constant automated file creation, modification, and deletion by background system processes, Windows Update, installer programs, and user applications. Every software installation, Windows patch, and print job creates temporary files here — triggering a continuous flood of FIM alerts that would completely drown out legitimate security anomalies. An analyst monitoring this directory would spend all their time dismissing routine OS activity rather than investigating real threats.

**2. `C:\Users\<User>\AppData\Local\Google\Chrome\User Data` (Browser cache)**

Web browsers continuously write, update, and clear cache files, cookie databases, session storage, and browsing history in the background during normal web use. Every webpage load writes dozens of cache entries. Every video watched updates media cache blocks. Chrome alone can generate hundreds of file change events per minute during normal browsing — making FIM on this path operationally useless and analytically exhausting.

**Principle:** Effective FIM targets high-value, low-change-frequency paths — system binaries, configuration files, security tool directories, and sensitive data stores. Monitoring high-churn paths creates alert fatigue and reduces the signal-to-noise ratio to near zero.

---

### Question 3 — Proactive Defense: How can Wazuh FIM detect ransomware behavior?

Ransomware exhibits a distinctive file system behavior pattern that Wazuh FIM can detect proactively:

**Ransomware-Specific FIM Detection Patterns:**

| Ransomware Behavior | FIM Detection Signal | Wazuh Response |
|---|---|---|
| **Mass file modification in rapid succession** | Dozens of Rule 550 (checksum changed) alerts firing within seconds across multiple files in the same directory | High-volume syscheck alerts from single agent in short timeframe |
| **File extension changes** | Files renamed from `.docx` → `.docx.locked` or `.docx.WNCRY` trigger rename/delete/add event chains | Rule 553 + 554 pairs for every encrypted file |
| **Shadow copy deletion** | `vssadmin.exe delete shadows` or `wmic shadowcopy delete` execution triggers process monitoring alert | Correlate with syscheck + audit.log alerts |
| **Encrypted file creation flood** | Hundreds of Rule 554 (new file added) alerts as encrypted copies are written | Unusually high FIM event rate from single endpoint |

**Practical Detection Rule Concept:**

Monitor for: **more than 20 FIM checksum-change events (Rule 550) on a single endpoint within 60 seconds** → trigger Critical alert → auto-isolate endpoint via active response.

This threshold catches ransomware's mass encryption phase before all files are encrypted, while avoiding false positives from normal software installations (which are sequential, not concurrent).

---

### Question 4 — Tool Comparison: How does Wazuh FIM outperform native Windows Event Logs?

**Centralized Aggregation vs. Isolated Silos:**

Windows Event Logs are stored locally on each machine by default. To collect FIM data from 500 endpoints using only Event Viewer, an analyst must manually connect to each machine or build a separate, complex log-forwarding architecture (Windows Event Collector, custom SIEM integration). Wazuh agents forward all FIM events to a single centralized manager automatically — a fleet of 500 endpoints generates a unified alert stream in one dashboard without any additional forwarding infrastructure.

**Advanced Correlation vs. Raw Log Data:**

Windows Event Logs provide raw event records with limited context. Wazuh correlates FIM changes with:

- **Vulnerability assessment data** — if a modified file belongs to a known-vulnerable package, Wazuh flags the combination
- **MITRE ATT&CK framework** — FIM events are automatically tagged with relevant technique IDs (e.g., T1565 — Data Manipulation)
- **Threat intelligence feeds** — file hashes from FIM events can be checked against known malware hash databases
- **Compliance frameworks** — PCI DSS, HIPAA, and GDPR compliance requirements are mapped automatically to FIM alerts

**Automated Severity Scoring:**

Wazuh assigns rule levels (0–15) to every FIM event based on the file path, change type, and correlated context — giving analysts immediate triage priority. Windows Event Logs assign a generic EventID with no automated severity scoring or context enrichment.

**Active Response:**

Wazuh can automatically respond to high-severity FIM events by executing scripts or commands on the affected endpoint — blocking a process, isolating the host from the network, or reverting a file change. Windows Event Logs have no native active response capability.

---

---

## MITRE ATT&CK Mapping — Lab Techniques Demonstrated

| MITRE Tactic | Technique ID | Technique Name | Lab Demonstration |
|---|---|---|---|
| **Defense Evasion / Impact** | T1565.001 | Stored Data Manipulation | FIM detected modification of `secret_plan.txt` — checksum change confirmed tampering |
| **Discovery** | T1083 | File and Directory Discovery | FIM baseline scan enumerates all files in monitored directory — attacker reconnaissance activity would be detected |
| **Persistence** | T1074.001 | Local Data Staging | File creation events (Rule 554) detect when an attacker stages data in a local directory before exfiltration |
| **Impact** | T1485 | Data Destruction | File deletion events (Rule 553) detect deliberate destruction of monitored files |
| **Impact** | T1486 | Data Encrypted for Impact | Ransomware mass-modification pattern detectable via high-volume Rule 550 event clustering |

---

## Key Takeaways

1. **Real-time FIM is a foundational SOC control.** The ability to detect a file modification within seconds — with cryptographic proof of exactly what changed — is something no manual monitoring process can replicate at scale. Wazuh's `realtime="yes"` directive using `ReadDirectoryChangesW` provides near-instant detection.

2. **Checksums are better evidence than timestamps alone.** Timestamps can be forged (timestomping). MD5, SHA1, and SHA256 hashes cannot be faked — if the hash changes, the content changed, period. Wazuh's FIM stores all three for every monitored file, providing multi-layer cryptographic evidence.

3. **The empty file MD5 is a universal indicator.** `d41d8cd98f00b204e9800998ecf8427e` is the MD5 hash of a zero-byte file. Seeing this as an "old" MD5 in a Rule 550 alert means the file existed but was empty — content was then written to it. Knowing this pattern speeds up alert triage significantly.

4. **Alert fatigue is a real threat to SOC effectiveness.** Monitoring high-churn directories like `C:\Windows\Temp` or browser cache paths produces so many false positives that analysts either disable the monitoring or begin ignoring alerts — both outcomes are worse than not monitoring at all. FIM effectiveness depends entirely on correct scope definition.

5. **Wazuh's strength is correlation, not just collection.** Any SIEM can collect logs. Wazuh adds value by correlating FIM events with vulnerability data, compliance frameworks, and threat intelligence automatically — reducing the analyst's manual enrichment burden and enabling faster, more accurate triage.

6. **Ransomware leaves a distinctive FIM signature.** The combination of high-volume Rule 550 alerts (mass checksum changes) appearing within seconds across multiple files in the same directory is operationally distinct from normal software behavior. This pattern can be detected and auto-responded to before encryption completes — if FIM is deployed and monitoring the right directories.

---

## Challenges Encountered and Resolutions

| Challenge | Impact | Resolution |
|---|---|---|
| **Internal API credential and YAML mapping desync between manager and dashboard** | Wazuh dashboard failed to connect to manager API on port 55000 — agent status not visible | Resolved by correcting API credentials and YAML configuration mapping on the Ubuntu VM; services restarted in correct order |
| **Ubuntu VirtualBox hardware bypass flags required** | VirtualBox default settings prevented Wazuh services from starting correctly | Hardware virtualization bypass flags enabled in VirtualBox VM settings before service startup |
| **Self-signed certificate browser warning** | Firefox blocked dashboard access with security warning | Warning accepted as expected behavior — self-signed certificates are standard for lab environments |
| **Agent authentication key import required manual steps** | Windows agent remained in "Not Running" state until key was imported | Key extracted from Ubuntu manager via `manage_agents` (E option), pasted into Windows Agent Manager GUI, Manager IP configured, service restarted |

---

## Screenshot Index

| # | Description | Lab Section |
|---|---|---|
| Screenshot 1 | Wazuh Dashboard Overview — initial state (Active: 0, Medium: 3, Low: 23) | Part 1 |
| Screenshot 2 | Wazuh Agent Manager GUI — initial unconfigured state (Not Running) | Part 2 |
| Screenshot 3 | Ubuntu terminal — manage_agents Add agent (Windows-SOC-Lab, ID 003) | Part 3, Step 1a |
| Screenshot 4 | Ubuntu terminal — manage_agents Extract key for agent 003 | Part 3, Step 1b |
| Screenshot 5 | Wazuh Agent Manager GUI — Running state (FABELT, 192.168.43.137) | Part 3, Step 2 |
| Screenshot 6 | Wazuh Dashboard Agents page — FABELT (004) active status confirmed | Part 3, Verification |
| Screenshot 7 | ossec.conf in Notepad — syscheck section with SensitiveFiles directory | Part 4, Step 1 |
| Screenshot 8 | Windows File Explorer — SensitiveFiles folder created in user directory | Part 5, Step 1 |
| Screenshot 9 | Windows File Explorer — secret_plan.txt (1 KB) inside SensitiveFiles | Part 5, Step 2 |
| Screenshot 10 | Wazuh Dashboard — FIM Recent Events (4 events: added, modified, deleted) | Part 5, Step 3 |
| Screenshot 11 | Wazuh alert detail panel — secret_plan.txt checksums (MD5, SHA1, SHA256) | Part 5, Step 3 |
| Screenshot 12 | Wazuh full_log field — changed attributes: size, mtime, md5, sha1, sha256 | Lab Challenge Q1 |

---

## Disclaimer

This lab was conducted in a controlled, isolated VirtualBox environment for educational purposes as part of the ICDFA Security and Operation Center (SOC) course (2025/INT/12158). The Wazuh Manager was deployed on a local Ubuntu VM with no internet connectivity. The Windows agent monitored only the designated test directory (`C:\Users\user\SensitiveFiles`). No production systems, real user data, or live networks were accessed or monitored at any point. All file events were generated deliberately for the purpose of testing and demonstrating FIM detection capabilities.
