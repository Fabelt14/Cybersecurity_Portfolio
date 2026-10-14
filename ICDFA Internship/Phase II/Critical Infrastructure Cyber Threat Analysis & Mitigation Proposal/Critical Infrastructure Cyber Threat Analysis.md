# Critical Infrastructure Cyber Threat Analysis
## & Security Assessment Proposal
### Niger-Delta Petrochemical & Fertilizer Ltd. (NDPF)

---

> **Classification:** TLP:AMBER — For Authorised Recipients Only
> **Document Type:** Sector Threat Analysis & IT/OT Security Assessment Proposal
> **Analyst:** Cyber Threat Intelligence & ICS Security Analyst
> **Date:** 10 October 2026
> **Engagement Reference:** NDPF-ICS-2026-001
> **Status:** Proposal — Awaiting Client Authorisation

---

## Analyst's Preface

This document represents a shift in perspective that every cybersecurity professional should experience.

Most security work focuses on protecting digital assets — servers, databases, web applications, credentials. The consequences of failure in those environments are severe: data theft, financial loss, reputational damage. But they are recoverable. A leaked database can be rotated. Stolen credentials can be reset. A compromised web server can be rebuilt.

Critical infrastructure is different.

When the target is a chemical plant, a power grid, or a water treatment facility, the consequences of a successful cyberattack are not measured in data records or downtime hours. They are measured in toxic gas plumes, pipeline ruptures, thermal explosions, and casualties. The 2017 TRITON attack — the first known cyberweapon designed to trigger industrial disasters — did not target a database. It targeted the Safety Instrumented System of a Saudi petrochemical plant: the last line of defence preventing a catastrophic explosion.

This report was produced as part of a structured investigation into the cyber threat landscape facing Nigeria's chemical sector, culminating in a professional security assessment proposal for Niger-Delta Petrochemical & Fertilizer Ltd. (NDPF). Understanding how to protect this category of infrastructure — and how to assess it without accidentally causing the very disaster you are trying to prevent — is a discipline that extends well beyond conventional IT security.

---

## Part 1 — Sector Intelligence Brief: Nigeria's Chemical Industry

### 1.1 — Sector Overview

The Chemical Sector forms the physical foundation of Nigeria's industrial economy. Unlike enterprise IT environments that process digital data, the chemical sector bridges the digital and physical worlds — where software commands move real volumes of volatile, hazardous substances through pressurised pipelines and high-temperature reactors.

Nigeria's chemical industry is structured across four primary product segments:

| Segment | Description | Nigerian Examples |
|---|---|---|
| **Basic Chemicals** | Petrochemicals (ethylene, plastics feedstocks) and industrial gases | Petroleum refineries and industrial gas plants in the Niger Delta and free trade zones |
| **Specialty Chemicals** | Adhesives, industrial sealants, advanced water-treatment compounds | Chemical suppliers to local manufacturing and construction sectors |
| **Agricultural Chemicals** | Fertilizers (Urea, NPK) and crop pesticides | Dangote Fertilizer Plant, Indorama Eleme Fertilizers (Port Harcourt) |
| **Consumer Products** | Detergents, soaps, paints, household chemicals | Unilever Nigeria, CAP Plc manufacturing lines |

**Why this sector is classified as critical infrastructure:**

The chemical sector does not operate in isolation. A disruption at a fertilizer plant does not merely halt chemical production — it cascades:

- **Agriculture:** Nigeria's farming sector depends on continuous domestic fertilizer supply for food security. A prolonged NDPF shutdown directly threatens crop yield cycles.
- **Healthcare:** Pharmaceutical manufacturers require chemical precursors for drug production. Interruption threatens medicine supply chains.
- **Public utilities:** State water boards cannot safely treat public drinking water without a steady supply of chlorine and purification chemicals. A cyberattack on a chemical plant can therefore trigger a public health emergency far removed from the plant itself.

This interconnectedness is precisely what makes the sector a high-value target for both state-sponsored saboteurs and financially motivated ransomware operators.

---

### 1.2 — Technical Architecture: How a Chemical Plant Functions

To understand how a chemical plant can be attacked, it is first necessary to understand what it is built from. Walking into NDPF's fertilizer facility is fundamentally different from entering a bank's data centre. Both environments contain computers. But NDPF's computers control gas valves, pressurised reactors, and chemical mixing processes that operate at temperatures and pressures incompatible with human survival if uncontrolled.

This environment is built from a specialised combination of **Information Technology (IT)** and **Operational Technology (OT)**:

---

![Diagram showing the seven-layer architecture of a chemical facility from physical sensors at Level 0 through PLCs at Level 1, DCS at Level 2-3, IDMZ at Level 3.5, and Enterprise ERP at Level 4 — based on the Purdue Enterprise Reference Architecture model](images/part1_facility_architecture.png)

*Figure 1 — Chemical facility architecture based on the Purdue Enterprise Reference Architecture (PERA). Data flows upward from physical sensors and actuators through PLCs and DCS to enterprise systems, crossing the IDMZ security boundary between OT and IT. The IDMZ is the most critical security control point in the entire facility.*

---

**Layer-by-Layer Component Breakdown:**

**Layer 0 — Physical Process (Sensors, Actuators, IIoT)**

At the base of the architecture are the devices in direct physical contact with the chemicals. **Sensors** are the plant's sensory nervous system — continuously measuring temperature inside boilers, pressure inside gas pipelines, and flow rates through transfer valves. **Actuators** are the motor functions — physically opening valves or activating pumps in response to received commands.

Modern facilities augment these with **Industrial Internet of Things (IIoT)** devices: internet-connected sensors that transmit real-time operational data. While IIoT dramatically improves operational visibility, it simultaneously creates a new attack surface — each connected sensor is a potential entry point into the physical process environment.

**Layer 1 — Direct Control (Programmable Logic Controllers — PLCs)**

PLCs are ruggedised industrial computers installed directly adjacent to machinery on the factory floor. They receive sensor readings and execute immediate physical decisions autonomously. If a sensor reports that a chemical mixing tank is exceeding safe temperature thresholds, the PLC immediately commands the actuator to open the cooling valve — without waiting for human confirmation.

**Critical security characteristic:** PLCs were engineered in an era before cybersecurity was a design consideration. Most operate on a model of **implicit trust** — they execute any valid network command received from the control network without requiring authentication, encryption, or multi-factor verification. An attacker who successfully places themselves on the OT network can issue arbitrary commands to PLCs indistinguishable from legitimate operator instructions.

**Layer 2–3 — Supervisory Control (Distributed Control System — DCS)**

While a PLC controls a single machine or zone, a large chemical facility operates hundreds of interdependent machines simultaneously. The **DCS** coordinates the entire production line — ensuring that the chemical mixing occurring in one process area is correctly timed and sequenced with packaging, loading, and quality control operations in other areas.

The DCS is the primary target for attackers seeking to cause large-scale disruption: compromising it means losing coordinated control over the entire production environment simultaneously.

**Layer 2 — Human Interface (Human-Machine Interfaces — HMIs)**

**HMIs** are the visual interfaces through which human engineers monitor and interact with the plant. Rather than walking the factory floor to read physical gauges, operators observe a digital representation of the entire facility — live temperature readings, pressure levels, valve states, and flow rates — displayed on control room screens.

HMIs are a particularly important attack surface because they serve a dual function: they display plant status to human operators AND transmit control commands back to the DCS. An attacker who manipulates HMI displays can simultaneously mislead human operators while issuing malicious commands — exactly the technique used in the TRITON attack.

**Independent Fail-Safe — Safety Instrumented System (SIS)**

The SIS is a completely independent hardware and software system with one exclusive purpose: **emergency shutdown**. If the main DCS fails and a chemical reactor reaches critical explosive pressure, the SIS automatically overrides all other systems and initiates a safe shutdown sequence.

Because the SIS is the last barrier between a process failure and a physical disaster, it has become the highest-value target for nation-state attackers seeking to cause irreversible physical harm. Disabling the SIS means removing the only automated protection against catastrophe.

> **Historical Context — TRITON (2017):** The TRITON malware was the first known cyberweapon specifically engineered to target Safety Instrumented Systems. In 2017, an attributed nation-state threat actor infiltrated a Saudi petrochemical plant and injected malicious code directly into the memory registers of Triconex SIS controllers, attempting to disable the facility's emergency fail-safes. The attack was only discovered because a logic error in the malware caused an unintended safety trip before the destructive payload activated (CCDCOE, 2021; MITRE ATT&CK, 2024).

**Layer 4 — Enterprise (ERP Systems)**

The ERP manages the business side: customer orders, procurement, financial reporting, and human resources. In modern converged facilities, the ERP communicates with the DCS — production schedules flow automatically from the business layer down to the factory floor. This integration creates the primary vulnerability pathway: a threat actor who compromises the corporate ERP has a potential route into the OT environment.

**The Security Boundary — Industrial Demilitarized Zone (IDMZ)**

Because directly connecting corporate IT infrastructure to factory floor OT systems creates an unacceptable security exposure, secure facilities implement an **IDMZ** — a restricted intermediate network segment through which all IT-to-OT data must pass. Firewalls, data diodes, and strict access control lists govern what can traverse this boundary.

The IDMZ is the most important security control in the entire facility. It is also, in practice, the most commonly misconfigured.

---

### 1.3 — Operational Workflow: IT/OT Convergence in Practice

The following workflow illustrates how a production order at NDPF travels from a business decision through multiple network layers to physical chemical process execution — and where each transition creates a potential security vulnerability.

**Scenario: NDPF receives a large Urea fertilizer export order**

```
LEVEL 4 — CORPORATE IT
────────────────────────────────────────────────────────────────────
The customer's order is processed in the corporate ERP system.
The ERP generates a production schedule specifying volumes, grades,
and delivery timelines.

        ↓ [Data crosses IDMZ — security checkpoint]

LEVEL 3.5 — IDMZ (SECURITY BOUNDARY)
────────────────────────────────────────────────────────────────────
The production schedule must pass through the IDMZ firewall.
In a correctly configured facility, only specific data types from
approved source addresses are permitted to cross here.
Third-party maintenance vendors access the OT environment via
VPN gateways in this zone.

⚠️  This is the primary attack surface. A misconfigured IDMZ
or compromised vendor VPN credential bypasses all downstream
OT security controls simultaneously.

        ↓ [Enters OT environment]

LEVEL 2-3 — DISTRIBUTED CONTROL SYSTEM
────────────────────────────────────────────────────────────────────
The DCS receives the production schedule.
Engineers monitor the entire plant via HMI screens in the
control room, adjusting process parameters as required.
The DCS determines which reactor vessels and pipeline systems
to activate.

        ↓

LEVEL 1 — PROGRAMMABLE LOGIC CONTROLLERS
────────────────────────────────────────────────────────────────────
The DCS sends specific temperature, pressure, and flow-rate
commands to the PLCs stationed at each reactor vessel.
PLCs execute immediately — no authentication required.

        ↓

LEVEL 0 — PHYSICAL PROCESS
────────────────────────────────────────────────────────────────────
Actuators open nitrogen and hydrogen feed valves.
Reactors begin Haber-Bosch synthesis under controlled
temperature and pressure.
IIoT sensors continuously report process conditions back
up the chain to engineers.

        ↓ [If process conditions become unsafe]

FAIL-SAFE — SAFETY INSTRUMENTED SYSTEM (SIS)
────────────────────────────────────────────────────────────────────
If pressure sensors detect conditions approaching critical
thresholds and the DCS fails to respond, the SIS independently
triggers emergency shutdown — sealing all valves and cutting
power to prevent explosion.
```

**The security implication of this workflow is direct:** an attacker does not need to physically access the factory floor. They need to compromise the IDMZ — the single digital checkpoint separating the corporate internet from the industrial control systems driving a high-pressure chemical facility.

---

## Part 2 — Threat Intelligence: Primary Adversary Profiles

### 2.1 — Threat 1: Nation-State Sabotage (Advanced Persistent Threats)

**Motivation:** Physical destruction, long-term espionage, environmental terrorism, geopolitical leverage
**Target within the facility:** SIS, PLCs, DCS — the physical process control layer
**Primary TTPs:** Living-off-the-Land (LotL), process injection, OT-specific malware

Nation-state actors target chemical facilities not for financial gain but to cause physical harm or to gather long-term intelligence on industrial capabilities. Their objectives range from triggering an industrial disaster to quietly exfiltrating chemical production formulas and process data for state competitive advantage.

**Attack Methodology:**

Because traditional antivirus and endpoint detection tools monitor IT network behaviour, sophisticated nation-state actors deliberately avoid deploying conventional malware on IT systems where detection is most likely. Instead, they employ **Living-off-the-Land (LotL)** techniques — hijacking legitimate, built-in administrative tools already present on the corporate network (`PowerShell`, `WMI`, `PsExec`, legitimate IT management protocols) to move laterally without triggering signature-based alerts.

Once through the IDMZ, the attacker deploys **lightweight, obfuscated malware** engineered to appear as routine Windows update files or standard system maintenance scripts. **Process injection** is used to embed malicious commands within legitimate network traffic, making the attacker's activity indistinguishable from authorised engineering operations in network log analysis.

**The ultimate objective** is the factory floor. By compromising PLCs or the SIS, the attacker can:

- Secretly alter chemical mixing parameters (e.g., incrementally increasing reactor pressure beyond safe tolerances)
- While simultaneously **forcing the HMI to display falsified "normal" readings** to human operators in the control room

The operators see a functioning plant. The plant is heading toward catastrophic failure.

**Historical Reference — TRITON Attack (2017):**

The TRITON attack demonstrated that this is not a theoretical threat. A nation-state actor infiltrated a Saudi Arabian petrochemical facility and directly injected malicious code into the **SIS memory registers** of Triconex safety controllers — specifically attempting to disable the emergency shutdown system. Had the attack succeeded, operators would have lost the ability to execute an emergency shutdown during a process excursion (CCDCOE, 2021; MITRE ATT&CK, Campaign C0030, 2024).

---

![MITRE ATT&CK campaign C0030 mapping for TRITON showing initial access via spearphishing, lateral movement through the IDMZ, and final payload injection into Safety Instrumented System memory registers](images/part2_triton_attack.png)

*Figure 2 — TRITON attack chain mapped to MITRE ATT&CK for ICS. The attack progressed through standard IT initial access techniques before crossing the IT/OT boundary to target the SIS directly. The same attack pathway — spearphishing → lateral movement → IDMZ bypass → OT exploitation — applies to NDPF's current architecture.*

---

### 2.2 — Threat 2: Financially Motivated Extortion (Ransomware)

**Motivation:** Financial gain through operational disruption
**Target within the facility:** DCS servers, HMI workstations — the visibility and control layer
**Primary TTPs:** Supply chain compromise, VPN credential theft, ransomware deployment

While nation-state actors seek to cause destruction covertly, ransomware operators seek maximum operational disruption to extract payment. Chemical facilities are high-value ransomware targets because halting production costs millions of dollars per day — creating enormous financial pressure to pay quickly.

**Attack Methodology:**

Ransomware operators targeting OT environments have identified a reliable bypass route for the IDMZ: **third-party maintenance contractors**. Chemical plants routinely grant external ICS integrators and equipment vendors remote VPN access through the IDMZ for legitimate troubleshooting and maintenance purposes. These vendor accounts frequently use weaker authentication controls than internal accounts and are subject to less rigorous security monitoring.

By compromising a vendor's credentials — through phishing, credential stuffing, or purchasing stolen credentials on darkweb markets — attackers walk directly through the open VPN tunnel into the operational network. No IDMZ exploitation is required.

Once inside the OT environment, **self-propagating ransomware** is deployed against DCS servers and HMI workstations. The physical PLCs and gas valves may remain entirely functional, but if the DCS is encrypted and the HMI is dark, human operators have lost all visibility into the chemical processes running at pressure. With no ability to monitor or control volatile reactions, the only safe option is an immediate emergency manual shutdown — stopping production entirely.

The attacker then demands a ransom for the decryption keys. Every day of downtime at a chemical facility of NDPF's scale represents significant revenue loss, creating pressure to pay rather than rebuild.

---

## Part 3 — Vulnerability Analysis: Why These Attacks Succeed

### 3.1 — Vulnerability 1: Flat Networks and Poor IDMZ Configuration

**Severity:** 🔴 Critical

In theory, the IDMZ is the impenetrable checkpoint between the corporate network and the factory floor. In practice, many chemical facilities operate **flat networks** — environments where network segmentation exists in name but is poorly enforced in configuration, allowing traffic to flow too freely between IT and OT segments.

The most common mechanism by which flat networks manifest in chemical facilities is through **third-party remote access**. Plants routinely grant external VPN access to ICS integrators, equipment vendors, and maintenance contractors. When a vendor's credentials are stolen, the attacker inherits a pre-authorised, properly configured VPN tunnel into the operational network. The IDMZ — which is designed to stop unauthorised traffic — cannot distinguish between a legitimate vendor engineer and an attacker using that engineer's stolen credentials.

**At NDPF specifically:** The combination of managing multiple production lines (fertilizer synthesis, gas processing) across a large facility increases the number of external vendors requiring remote access — and therefore the number of potential credential theft targets.

---

### 3.2 — Vulnerability 2: Legacy Control Systems and Implicit Trust

**Severity:** 🔴 Critical

The PLCs and DCS hardware driving NDPF's production were engineered to prioritise one thing above all else: **physical reliability and continuous uptime**. Cybersecurity was not a design consideration when most of these systems were built.

The result is **implicit trust architecture**: PLCs execute any valid network command they receive from the control network without requiring authentication, encryption, or access verification. There are no usernames, no passwords, no multi-factor authentication prompts between a DCS command and a PLC actuating a gas valve.

From a physical engineering perspective, this made sense — every millisecond of authentication delay is a millisecond during which a runaway chemical process continues unchecked. From a cybersecurity perspective, it means that any attacker who successfully enters the OT network segment has unrestricted command authority over every PLC on that network.

**Combined threat multiplier:** Implicit trust + flat network = any actor who compromises a single vendor VPN credential gains full unrestricted command access to every PLC on NDPF's factory floor.

---

### 3.3 — Vulnerability 3: Shadow OT and IIoT Cloud Sprawl

**Severity:** 🟠 High

To modernise operational monitoring, many facilities retrofit **IIoT sensors** onto legacy chemical equipment — enabling remote monitoring of tank pressure, pipeline temperature, and flow rates. However, these small-form-factor sensors frequently lack the processing capacity for robust firmware updates or strong encryption standards.

More critically, to provide convenient real-time data access for engineering managers and executives, these sensors are frequently configured to transmit data directly to **external cloud dashboards via cellular networks** — completely bypassing the facility's internal firewalls and the IDMZ security boundary.

This creates **Shadow OT**: unmanaged, unmonitored digital entry points that exist outside the facility's security perimeter on the public internet. An attacker who discovers and exploits a vulnerable IIoT sensor endpoint gains a direct path into physical process monitoring systems — entirely circumventing the IDMZ that the facility's security team believes is protecting them.

---

![Vulnerability diagram showing three attack pathways into the NDPF facility — flat network IDMZ bypass via compromised vendor VPN, implicit trust PLC command execution once inside OT, and Shadow OT cloud bypass via cellular-connected IIoT sensors — with severity ratings](images/part3_vulnerability_map.png)

*Figure 3 — NDPF attack surface map. Three distinct vulnerability classes provide three independent pathways to the operational technology environment. Each can be exploited independently; combined, they represent a layered attack surface that requires a layered defensive response.*

---

## Part 4 — Security Assessment Proposal

### Client: Niger-Delta Petrochemical & Fertilizer Ltd. (NDPF)
### Engagement Type: IT/OT Convergence Security Assessment & Penetration Test
### Reference: NDPF-ICS-2026-001

---

### 4.1 — Engagement Scope

Defining precise boundaries is not a bureaucratic formality in ICS security assessments — it is a safety requirement. Active exploitation techniques that are routine in enterprise IT penetration testing can trigger unintended physical consequences in an OT environment. A network packet that would merely crash a web server can cause a PLC to drop a safety command during an active chemical process. Accordingly, this engagement applies strict technical constraints that differ materially from a standard IT penetration test.

**In-Scope: Active Testing Permitted**

| Network Layer | Assets Included |
|---|---|
| **Level 4 — Corporate IT** | ERP servers, employee workstations, corporate email infrastructure, Active Directory |
| **Level 3.5 — IDMZ** | External-facing firewalls, third-party vendor VPN gateways, data historian servers bridging IT and OT |
| **Cloud & IIoT Interfaces** | Cloud-based monitoring dashboards and cellular data endpoints associated with tank pressure and temperature sensors |

**Out-of-Scope: Passive Assessment Only — No Active Exploitation**

| Network Layer | Assets Excluded from Active Testing | Rationale |
|---|---|---|
| **Levels 0–3 — OT Environment** | Active PLCs, DCS servers under production load, HMI workstations, SIS controllers | Active exploitation, payload detonation, or aggressive scanning of live OT systems can trigger valve malfunctions, unintended process excursions, or emergency shutdowns. These systems will be assessed using passive monitoring methods only. |

> **Safety Note:** This scope boundary reflects industry best practice for ICS security assessments as defined by ICS-CERT and NIST SP 800-82r3. The boundary is not a limitation on assessment quality — it is a recognition that the risk of disrupting an active chemical process during a security test is categorically unacceptable. Passive OT assessment methods yield equivalent security intelligence without operational risk.

---

### 4.2 — Engagement Objectives

The four primary objectives of this assessment are derived directly from the threat and vulnerability analysis in Parts 2 and 3. Each objective maps to a specific, identified attack pathway:

**Objective 1 — Validate IT/OT Network Segmentation**

Determine whether true network segmentation exists between the corporate ERP environment and the IDMZ, and between the IDMZ and the OT environment. Specifically, assess whether a "flat network" condition allows unrestricted lateral movement from a compromised corporate workstation down to the DCS layer without triggering security controls.

*Threat mapped:* Nation-state APT lateral movement; ransomware operator propagation

**Objective 2 — Identify Shadow OT and Cloud Leakage**

Discover unmanaged IIoT devices that bypass the IDMZ via cellular or direct internet connections and could serve as direct external backdoors into the physical process environment. Map all cloud dashboards receiving OT sensor data and assess their authentication and access control posture.

*Threat mapped:* Shadow OT exploitation; unauthenticated remote access to process monitoring

**Objective 3 — Assess Third-Party Vendor Access Controls**

Test the resilience of external contractor VPN gateways against credential stuffing, session hijacking, and privilege escalation. Determine whether vendor access accounts are governed by the same least-privilege principles applied to internal staff, and whether multi-factor authentication is enforced.

*Threat mapped:* Ransomware initial access via compromised vendor credentials

**Objective 4 — Evaluate Detection and Response Capability**

Determine whether NDPF's current security monitoring infrastructure can detect Living-off-the-Land (LotL) techniques — specifically lateral movement executed using legitimate administrative tools (`PowerShell`, `WMI`, `PsExec`) — before an attacker reaches the IDMZ. Assess SIEM alert fidelity and SOC response time for simulated OT-adjacent threat scenarios.

*Threat mapped:* Nation-state LotL evasion; post-compromise dwell time before detection

---

### 4.3 — Assessment Methodology

This engagement employs a **four-phase hybrid methodology** combining active ethical hacking techniques where safe and permitted with passive vulnerability analysis throughout the OT environment.

---

**Phase 1 — External Reconnaissance & OSINT**

*Duration:* 5 business days
*Method:* Passive — no interaction with NDPF systems

Objective: Map NDPF's external digital footprint before any active assessment begins. This phase simulates the reconnaissance a threat actor would conduct using only publicly available open sources.

Activities:
- Passive domain enumeration — identifying all NDPF subdomains, mail servers, VPN gateways, and cloud infrastructure accessible from the public internet
- IIoT sensor dashboard discovery — searching Shodan, Censys, and similar industrial asset search platforms for NDPF-attributed devices exposing OT data to the internet
- Credential exposure assessment — searching breach intelligence platforms and paste sites for NDPF employee or contractor credentials that have appeared in publicly disclosed data breaches
- Cloud storage reconnaissance — identifying misconfigured S3 buckets, Azure blobs, or similar cloud storage assets that may contain NDPF operational data

*Deliverable:* External exposure report mapping all publicly visible NDPF assets and any identified credential or data exposures

---

**Phase 2 — Social Engineering & Phishing Simulations**

*Duration:* 10 business days
*Method:* Active — with explicit prior written authorisation

Objective: Simulate the human-targeted initial access techniques most commonly employed by both ransomware operators and nation-state actors targeting the chemical sector.

Activities:
- **Targeted spearphishing campaign against corporate IT staff** — simulating ransomware operator initial access via business email compromise and malicious attachment delivery
- **Supply chain vendor targeting** — simulating a nation-state or ransomware operator campaign against NDPF's third-party ICS maintenance contractors, targeting the vendor-to-IDMZ access pathway specifically
- **Pretexting assessment** — testing front-desk and operations staff responses to social engineering scenarios requesting VPN credentials or remote access provisioning

> **Ethical constraint:** All phishing simulations use non-destructive payloads (credential capture pages, click-through tracking) with no malicious functionality. All contractor organisations are notified of the scope of vendor-targeting activities prior to the engagement commencing.

*Deliverable:* Social engineering susceptibility report with individual and aggregate click rates, credential submission rates, and SOC detection performance

---

**Phase 3 — Vulnerability Assessment (Active IT, Passive OT)**

*Duration:* 10 business days
*Methods:* Active scanning (IT only) — Passive monitoring (OT)

**IT Network — Active Assessment (Level 4)**

Standard enterprise vulnerability assessment methodology applied to the corporate IT environment:
- Authenticated and unauthenticated vulnerability scanning of ERP servers and corporate workstations
- Active Directory configuration review — identifying Kerberoastable accounts, AS-REP roasting exposure, and misconfigured Group Policy Objects
- IDMZ firewall rule audit — verifying that only authorised traffic types, source addresses, and protocols are permitted to cross the IT/OT boundary

**OT Network — Passive Assessment (Levels 0–3)**

Passive network monitoring tools (Wireshark, Zeek, industrial-specific OT monitoring platforms) will be deployed within authorised IDMZ and OT network segments to capture and analyse traffic without transmitting any packets to fragile PLC or DCS hardware.

Analysis targets:
- **Industrial protocol analysis** — identifying Modbus, DNP3, EtherNet/IP (CIP), and PROFINET traffic; mapping which devices communicate with which PLCs and via which authentication methods
- **Cleartext authentication identification** — detecting legacy industrial protocols transmitting engineering credentials, device configurations, or process commands without encryption
- **Unauthorised device discovery** — identifying unregistered IIoT devices, rogue endpoints, or vendor laptops connected to OT network segments without authorisation
- **SIS isolation verification** — confirming that Safety Instrumented System traffic is segregated from standard DCS and HMI communications as required by IEC 61511

*Deliverable:* Combined vulnerability report covering corporate IT findings (with CVSS scores and exploitability assessments) and OT network traffic analysis (with ICS-specific risk ratings aligned to NIST SP 800-82r3)

---

**Phase 4 — "Assume Breach" Penetration Testing**

*Duration:* 10 business days
*Method:* Active — contained to IT and IDMZ only

Objective: Operating under the assumption that the corporate IT network has already been compromised by an initial access technique (simulated from Phase 2), testers will attempt to move laterally toward the OT environment using Living-off-the-Land techniques — the same methods employed by nation-state APTs.

Activities:
- **LotL lateral movement simulation** — using native Windows administrative tools (`PowerShell`, `WMI`, `PsExec`, `net use`) to move between corporate workstations and toward IDMZ-adjacent systems without triggering signature-based IDS/IPS alerts
- **IDMZ firewall penetration attempts** — testing identified firewall rule weaknesses and misconfigurations to determine whether a lateral movement path exists from corporate IT to the IDMZ control layer
- **DCS server access assessment** — if IDMZ penetration is achieved, attempting to establish theoretical command-line access to historian or DCS management servers without issuing any commands to live PLCs or process control functions
- **Detection validation** — measuring SOC alert response time and detection accuracy for all simulated LotL activities

> **Critical constraint:** Phase 4 activity stops at the DCS management server level. Under no circumstances will testers issue commands to live PLCs, actuate valves or pumps, modify process setpoints, or interact with Safety Instrumented Systems in any way. All Phase 4 activity is conducted in coordination with NDPF's Operations Manager and can be terminated by NDPF at any time without penalty.

*Deliverable:* Penetration test report detailing all successful lateral movement paths, IDMZ bypass findings (if any), and SOC detection performance metrics — with prioritised remediation recommendations

---

### 4.4 — Why This Assessment Matters Beyond NDPF

The vulnerabilities identified in this sector analysis — flat networks, implicit trust, Shadow OT, third-party access sprawl — are not unique to NDPF. They are endemic across Nigeria's critical infrastructure landscape, including water treatment facilities, power generation plants, and petroleum pipeline operations.

A successful cyberattack on any component of this ecosystem does not stay contained. Chemical supply disruptions cascade into food security (fertilizer), public health (water treatment chemicals), and energy (petrochemical feedstocks). The interconnected nature of critical infrastructure means that cybersecurity investment at one node protects the stability of multiple downstream sectors simultaneously.

This is why critical infrastructure security represents one of the most strategically important areas within the cybersecurity field — and why the methodology, ethical constraints, and risk management discipline required to conduct ICS security assessments safely demands a different level of preparation than conventional IT penetration testing.

---

## References

| Source | Citation |
|---|---|
| CCDCOE (2021) | NATO Cooperative Cyber Defence Centre of Excellence. *Triton (2017)*. International Cyber Law: Interactive Toolkit. https://cyberlaw.ccdcoe.org/wiki/Triton_(2017) |
| CISA (n.d.) | Cybersecurity and Infrastructure Security Agency. *Chemical Sector*. U.S. Department of Homeland Security. https://www.cisa.gov/topics/critical-infrastructure-security-and-resilience/critical-infrastructure-sectors/chemical-sector |
| Cisco (2021) | *How Does Triton Attack Triconex Industrial Safety Systems?* Cisco Security Blog. https://blogs.cisco.com/security/how-does-triton-attack-triconex-industrial-safety-systems |
| MITRE ATT&CK (2024) | *Triton Safety Instrumented System Attack, Campaign C0030*. https://attack.mitre.org/campaigns/C0030/ |
| NIST (2023) | *Guide to Operational Technology (OT) Security* (Special Publication 800-82r3). U.S. Department of Commerce. https://www.nist.gov/publications/guide-operational-technology-ot-security |
| Palo Alto Networks (n.d.) | *What Is Critical Infrastructure? Sectors and Security*. https://www.paloaltonetworks.com/cyberpedia/what-is-critical-infrastructure |
| Zscaler (n.d.) | *What Is the Purdue Model for ICS Security?* https://www.zscaler.com/resources/security-terms-glossary/what-is-purdue-model-ics-security |
