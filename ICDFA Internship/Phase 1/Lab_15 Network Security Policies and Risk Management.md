# Network Security Policies and Risk Management

## Overview

This exercise developed a comprehensive network security policy framework for a financial institution by conducting a structured risk assessment and mapping controls to the NIST Cybersecurity Framework. The goal was to move beyond generic security advice and create specific, actionable policies that address real threats to critical assets. Each control was chosen based on the actual vulnerabilities present in the environment, not just compliance checkbox requirements.

## Objectives

- Identify and classify critical organizational assets by sensitivity and business impact
- Conduct quantitative risk assessment using likelihood and impact ratings
- Map vulnerabilities to specific threats for each asset category
- Develop layered security policies using NIST CSF control families
- Design incident response procedures tailored to specific breach scenarios
- Create targeted security awareness training programs for different employee roles

## Lab Environment

- **Industry Context:** Financial institution handling PII, customer financial data, and payment card information
- **Regulatory Requirements:** PCI DSS, GDPR/NDPR compliance mandatory
- **Asset Categories:** 8 critical asset types identified (PII, financial data, IP, third-party data, core banking systems, databases, network infrastructure, endpoints)
- **Framework Applied:** NIST Cybersecurity Framework (CSF) for control selection

## Tools Used

- NIST Cybersecurity Framework (Identify, Protect, Detect, Respond, Recover)
- Risk assessment matrix (Likelihood × Impact = Risk Rating)
- Access control models (Zero Trust Architecture, least privilege, separation of duties)
- Encryption standards (AES-256 for data at rest, TLS for data in transit)

## Methodology

### Asset Identification

Before assessing risk, I cataloged every asset that requires protection. Financial institutions hold data that criminals actively target and regulators heavily scrutinize. Missing an asset category during risk assessment means missing the threats that target it.

**Personal Identifiable Information (PII):** Names, addresses, Social Security numbers, government-issued ID numbers. This data enables identity theft and is subject to GDPR/NDPR notification requirements. A breach triggers mandatory disclosure to regulatory authorities and affected individuals.

**Customer Financial Data:** Bank account numbers, credit/debit card numbers, transaction histories, loan records. This is the target of account takeover attacks and fraudulent transactions. PCI DSS compliance requires strict controls on this data, with quarterly audits and potential loss of card processing privileges if breached.

**Intellectual Property:** Proprietary trading algorithms, financial models, strategic business plans. While not subject to breach notification laws, IP theft destroys competitive advantage. A competitor obtaining trading algorithms could replicate strategies, devaluing the institution's proprietary methods.

**Third-Party and Partner Data:** Information shared with vendors, credit bureaus, fintech partners. Financial institutions don't operate in isolation. APIs connect to payment processors, credit reporting agencies, and third-party service providers. A vendor breach can compromise internal systems through trusted connections.

**Core Banking Systems:** Backend systems processing daily transactions, deposits, withdrawals. These are the operational heart of the institution. Ransomware that encrypts core banking systems halts all customer transactions, creating systemic financial instability.

**Database Management Systems:** Where sensitive information is stored at rest. Databases hold the aggregated sum of all customer data. SQL injection or unauthorized access to a DBMS means access to millions of records simultaneously, not just individual account breaches.

**Network Equipment:** Firewalls, routers, switches, VPNs managing traffic and perimeter security. These devices control what traffic enters and exits the network. Compromising a firewall grants "god mode" access to monitor all network traffic, bypassing every other security control.

**Employee Workstations and Mobile Devices:** Laptops and phones used for remote work or accessing internal systems. These are the weakest link because they leave the protected network perimeter. Employees connect to public Wi-Fi, click phishing links, and physically lose devices. Each endpoint is a potential entry point.

### Risk Assessment Matrix

I assessed each asset using a 1-3 scale for both likelihood and impact, then calculated risk rating by multiplying these values. This quantitative approach prioritizes resources toward the highest-risk scenarios instead of treating all threats equally.

![Risk assessment matrix showing likelihood, impact, and risk ratings for each asset](screenshots/risk_assessment_matrix.png)

**Personal Identifiable Information - HIGH RISK (Likelihood: 3, Impact: 3, Rating: 9)**

**Threats:** Data breaches, identity theft, dark web sales. PII has a thriving black market. Stolen Social Security numbers sell for $1-2 each in bulk, names with full credit histories sell for $30-50. Criminals buy this data to open fraudulent credit accounts.

**Vulnerabilities:** Insecure storage (unencrypted databases), weak encryption (outdated algorithms like DES or 3DES), human error (sending unencrypted PII via email), phishing (tricking employees into revealing credentials that access PII databases).

**Impact:** Massive regulatory fines (GDPR violations carry fines up to 4% of annual global revenue or €20 million, whichever is higher. NDPR in Nigeria carries similar penalties). Loss of customer trust is unquantifiable but leads to account closures and revenue loss. Legal litigation from affected customers seeking damages for identity theft consequences.

**Why High Likelihood:** Phishing attacks targeting financial institutions are constant. Attackers know these organizations hold valuable PII and actively probe for vulnerabilities. Human error is inevitable at scale—a single employee mistake can expose thousands of records.

**Customer Financial Data - HIGH RISK (Likelihood: 3, Impact: 3, Rating: 9)**

**Threats:** Fraudulent transactions, account takeover (credential stuffing using passwords leaked from other breaches), data leaks (misconfigured S3 buckets exposing transaction histories).

**Vulnerabilities:** Misconfigured access controls (employees having access to accounts they don't need to service), unpatched systems (running outdated software with known CVEs), insecure APIs (authentication flaws allowing unauthorized queries).

**Impact:** Direct financial losses for both the institution (fraudulent wire transfers) and customers (unauthorized debit card charges). PCI DSS violations result in fines ranging from $5,000 to $100,000 per month of non-compliance, plus potential loss of the ability to process card transactions entirely.

**Why High Likelihood:** Financial data is the most directly monetizable. Criminals don't need to resell it—they use it immediately for fraudulent transfers. API vulnerabilities are common because developers prioritize functionality over security during rapid development cycles.

**Intellectual Property - LOW RISK (Likelihood: 1, Impact: 2, Rating: 2)**

**Threats:** Corporate espionage (competitors or nation-state actors stealing trading algorithms), insider threats (disgruntled employees exfiltrating proprietary models before resignation).

**Vulnerabilities:** Lack of Data Loss Prevention (DLP) tools to block bulk file transfers, unmonitored file transfers (employees copying files to personal email or USB drives without detection).

**Impact:** Loss of competitive advantage (competitors replicating successful trading strategies), devaluation of proprietary models (if known to be compromised, the intellectual property loses its strategic value).

**Why Low Likelihood:** Intellectual property theft requires sophisticated, targeted attacks. Unlike PII or financial data that has commodity value on dark web markets, trading algorithms only have value to direct competitors. Most attackers pursue easier targets with faster monetization.

**Third-Party and Partner Data - MEDIUM RISK (Likelihood: 2, Impact: 2, Rating: 4)**

**Threats:** Supply chain attacks (attackers compromising a vendor's systems to pivot into client networks), data breaches via vendors (third-party processor exposing data shared with them).

**Vulnerabilities:** Weak API security (insufficient authentication or authorization on partner-facing APIs), lack of vendor security audits (accepting vendor security claims without verification), over-reliance on partner security (assuming vendors implement controls without validation).

**Impact:** Indirect breach of internal systems via trusted partner tunnels. Many attacks against large financial institutions start by compromising smaller vendors who have trusted network connections. The 2013 Target breach entered through an HVAC vendor's credentials.

**Why Medium Likelihood:** Vendors are attractive targets because they often have weaker security than primary institutions but maintain privileged network access. However, supply chain attacks require more effort than direct attacks, reducing likelihood below the highest tier.

**Core Banking Systems - MEDIUM RISK (Likelihood: 1, Impact: 3, Rating: 3)**

**Threats:** Ransomware attacks (encrypting transaction processing systems), system disruption (DDoS preventing customer access), data manipulation (altering account balances or transaction logs).

**Vulnerabilities:** Software vulnerabilities (unpatched operating systems or applications), misconfigurations (default settings that weaken security), weak internal controls (insufficient separation of duties allowing single administrators excessive privileges).

**Impact:** Inability to process transactions (availability loss) means customers cannot deposit checks, withdraw cash, or make payments. This creates major financial instability as the institution cannot fulfill its core function. Potential systemic risk if the outage triggers broader financial market disruptions.

**Why Low Likelihood, High Impact:** Core banking systems are typically well-protected because organizations recognize their criticality. They run on isolated networks with strict change control. However, if an attack succeeds, the impact is catastrophic. This is a classic "low probability, high consequence" risk.

**Database Management Systems - MEDIUM RISK (Likelihood: 1, Impact: 3, Rating: 3)**

**Threats:** SQL injection (attackers inserting malicious SQL commands to extract or modify data), unauthorized access (compromised DBA credentials granting full database access).

**Vulnerabilities:** Default configurations (using standard 'sa' or 'root' accounts with well-known default passwords), unpatched database engines (running old versions with known SQL injection vulnerabilities), lack of audit logging (no record of who accessed what data).

**Impact:** Full data breach (attacker downloads entire customer database), destruction (DROP TABLE commands deleting financial records), alteration of financial records (changing account balances to facilitate fraud).

**Why Low Likelihood, High Impact:** Database administrators typically understand the criticality of their systems and implement strong controls. However, a single misconfiguration or unpatched vulnerability creates a single point of failure. If breached, the aggregated data loss is massive.

**Network Equipment - MEDIUM RISK (Likelihood: 2, Impact: 2, Rating: 4)**

**Threats:** Eavesdropping (packet sniffing to capture credentials or sensitive data in transit), firewall bypass (exploiting rules to access restricted network segments).

**Vulnerabilities:** Use of default passwords (many network devices ship with admin/admin credentials that are never changed), outdated firmware (running old software with known exploits), unencrypted management traffic (using Telnet instead of SSH to configure devices, transmitting credentials in cleartext).

**Impact:** Attackers gain "god mode" access to monitor all network traffic. Compromising a router means seeing every packet crossing that network segment. This includes credentials for other systems, allowing lateral movement throughout the network.

**Why Medium Likelihood and Impact:** Network equipment is often managed by specialized teams who understand its importance, but it's also complex. A busy network operations center may overlook firmware updates for months. Impact is high but not catastrophic because other security layers (encryption, authentication) still protect specific systems even if network monitoring is compromised.

**Employee Workstations and Mobile Devices - HIGH RISK (Likelihood: 3, Impact: 2, Rating: 6)**

**Threats:** Malware/ransomware infiltration (employees clicking phishing email attachments), phishing (credential harvesting attacks), data leakage (accidental email to wrong recipient), physical loss (leaving laptop in taxi).

**Vulnerabilities:** Human error (inevitable at scale), weak endpoint protection (outdated antivirus or lack of EDR), use of public Wi-Fi (exposing unencrypted traffic to eavesdropping), accidental data exposure (copying sensitive data to personal cloud storage).

**Impact:** Internal network breach (malware installed on endpoint pivoting to internal servers), credential theft (keyloggers capturing usernames and passwords), data exfiltration (copying files before detection), Business Email Compromise fraud (attacker impersonating executives to authorize fraudulent wire transfers).

**Why High Likelihood, Medium Impact:** Endpoints are attacked constantly because they're accessible from the internet and users interact with them. Every employee is a potential victim. However, endpoint compromise alone doesn't grant access to core systems if proper network segmentation and access controls exist. The impact is medium because attackers must successfully pivot from the initial endpoint to reach critical assets.

### Security Policy Development Using NIST CSF

For each high and medium-risk asset, I mapped specific NIST CSF controls to address the identified vulnerabilities. This isn't generic compliance. Each control directly mitigates a known weakness in the risk assessment.

**Personal Identifiable Information - Access Control Policy**

**NIST CSF PR.AA-05 (Access Permissions):** Restrict access and privileges to the minimum necessary using Zero Trust Architecture.

**Implementation:** Only employees with a documented business need can access PII databases. Customer service representatives see data only for accounts they're actively servicing. Reports that aggregate PII for analysis are anonymized. Administrative access to PII databases requires manager approval and periodic revalidation.

**Why This Control:** The vulnerability assessment identified "human error" as a weakness. Reducing the number of employees who can access PII reduces the attack surface for both accidental exposure and malicious insiders. Zero Trust means never assuming an employee should have access just because they're on the internal network.

**Personal Identifiable Information - Data Protection Policy**

**NIST CSF PR.DS-01 (Data-at-Rest Protection):** Apply AES-256 encryption to all PII storage and implement data masking for sensitive ID numbers.

**Implementation:** Database columns containing Social Security numbers, passport numbers, and driver's license numbers are encrypted with AES-256. Application developers never see plaintext values. Customer service interfaces display masked values (XXX-XX-1234) unless the representative explicitly needs to verify the full number for identity confirmation.

**Why This Control:** The vulnerability assessment identified "insecure storage" and "weak encryption." Even if an attacker gains database access through SQL injection or stolen credentials, encrypted data is unreadable without the encryption keys, which are stored separately in a hardware security module (HSM).

**Personal Identifiable Information - Incident Response Procedure**

**NIST CSF RS.AN-01 (Analysis):** Triage leak severity and initiate legal notification protocols for the Data Protection Commission.

**Implementation:** When PII exposure is suspected, the incident response team determines: (1) How many records were exposed, (2) What types of PII were included, (3) Whether the data was encrypted. If unencrypted PII of 500+ individuals is exposed, legal notification to the Data Protection Commission occurs within 72 hours per GDPR Article 33 requirements.

**Why This Procedure:** Breach notification laws have strict timelines. Having a pre-defined playbook prevents delays caused by figuring out whom to notify and how. The 72-hour GDPR deadline is non-negotiable—missing it increases fines.

**Personal Identifiable Information - Training Program**

**NIST CSF PR.AT-01 (General Awareness):** Train personnel to recognize social engineering attempts, report suspicious activity, comply with acceptable use policies, and perform basic cyber hygiene.

**Implementation:** All employees complete annual security awareness training covering: recognizing phishing emails, reporting suspicious phone calls requesting sensitive information, password hygiene (no reuse, use password manager), and clean desk policies for physical documents containing PII.

**Why This Training:** The vulnerability assessment identified "phishing" as a threat vector. Technical controls can't prevent employees from voluntarily giving attackers their credentials. Training is the only defense against social engineering.

**Customer Financial Data - Access Control Policy**

**NIST CSF PR.AA-05 (Access Permissions):** Restrict access and privileges to the minimum necessary.

**Implementation:** Database views limit what each employee role can see. Customer service representatives access only customer names and masked account numbers, not full transaction histories. Fraud investigators access transaction details but cannot initiate transactions. Only dedicated treasury operations staff can authorize outbound wire transfers, and they cannot access customer account details.

**Why This Control:** Role-based access control (RBAC) implements separation of duties. No single employee has both the ability to view customer financial data and the ability to transfer funds. This prevents insider fraud and limits damage from compromised credentials.

**Customer Financial Data - Data Protection Policy**

**NIST CSF PR.DS-11 (Data Integrity):** Enforce strict data integrity checks and encryption for all databases containing customer transaction history.

**Implementation:** Every transaction record includes a cryptographic hash. Any modification to transaction amount, date, or account number changes the hash, flagging the record as potentially tampered. Database replication uses checksums to verify data consistency between primary and backup systems. All transaction logs are append-only—deleting or modifying existing entries is cryptographically prevented.

**Why This Control:** Financial data integrity is as critical as confidentiality. If attackers can modify account balances without detection, the entire financial record becomes unreliable. Cryptographic hashing creates an audit trail that reveals unauthorized modifications.

**Customer Financial Data - Incident Response Procedure**

**NIST CSF RS.MA-01 (Mitigation):** Execute the pre-defined "Data Breach" playbook and isolate affected database segments immediately.

**Implementation:** If unauthorized access to customer financial data is detected, the database segment is immediately isolated by disabling network routes to that database server. This prevents further data exfiltration while the investigation continues. Affected customer accounts are flagged for monitoring, and customers receive notification if fraudulent activity is detected.

**Why This Procedure:** Speed matters in financial data breaches. Every minute of continued access means more data stolen. Pre-defined network isolation procedures allow immediate action without waiting for executive approval, preventing the breach from expanding.

**Intellectual Property - Access Control Policy**

**NIST CSF PR.DS-02 (Data-in-Transit Protection):** Block access to personal email, file sharing, file storage services, and other personal communications from organizational systems.

**Implementation:** Network firewall rules block outbound connections to Gmail, Dropbox, Google Drive, personal OneDrive accounts, and file transfer sites. USB ports are disabled on workstations with access to proprietary algorithms. Employees who need to work remotely access trading models through virtual desktop infrastructure (VDI) where files cannot be downloaded to the local machine.

**Why This Control:** The vulnerability assessment identified "unmonitored file transfers" as a weakness. Blocking personal cloud storage and USB drives prevents both accidental data leakage and intentional IP theft by insiders planning to resign and join competitors.

**Intellectual Property - Data Protection Policy**

**NIST CSF PR.DS-02 (Data-in-Transit Protection):** Deploy DLP tools to block the use of personal email, USB drives, or unauthorized file-sharing services.

**Implementation:** Data Loss Prevention (DLP) software scans outbound email and web uploads for patterns matching proprietary algorithm file types. Attempting to attach a .py file containing trading logic to an email triggers automatic blocking and alerts the security team. Large file transfers (>100MB) to unapproved destinations are blocked and logged.

**Why This Control:** Technical enforcement is more reliable than policy alone. Employees might not realize that copying a file to personal email for convenience violates policy. DLP automatically prevents violations before data leaves the organization.

**Third-Party and Partner Data - Access Control Policy**

**NIST CSF PR.AA-06 (Identity Federation):** Utilize identity federation and apply strict vendor-specific IAM permissions to partner accounts.

**Implementation:** Third-party vendors access internal systems through federated identity management (SAML or OAuth). Vendors authenticate through their own identity provider, but permissions are controlled by the financial institution's IAM system. Each vendor account has explicitly defined permissions limited to only the systems and data required for their contract deliverables.

**Why This Control:** The vulnerability assessment identified "weak API security" as a concern. Identity federation prevents vendors from creating their own accounts with unknown credentials. The institution retains centralized control over what vendors can access and can instantly revoke access if the vendor relationship ends or a security incident occurs.

**Third-Party and Partner Data - Data Protection Policy**

**NIST CSF PR.DS-02 (Data-in-Transit Protection):** Mandate Mutual TLS (mTLS) for all third-party API connections.

**Implementation:** All partner-facing APIs require mutual TLS authentication. Both the client (vendor) and the server (financial institution) present certificates to prove identity. This prevents man-in-the-middle attacks and ensures that only authorized vendors with valid certificates can connect to internal APIs. Certificate revocation immediately blocks vendor access.

**Why This Control:** Standard TLS only authenticates the server. The client (vendor) might be anyone with valid credentials. Mutual TLS authenticates both sides of the connection, preventing credential theft from compromising the API.

**Core Banking Systems - Access Control Policy**

**NIST CSF PR.AA-01 (Identity Management):** Enforce administrative separation of duties to ensure no single sysadmin has total control over core data.

**Implementation:** Modifying customer account balances requires two separate administrators: one to submit the change request and a different administrator to approve it. No single account has both permissions. Database backups are managed by a separate team from those who manage production databases. This prevents a rogue administrator from modifying data and destroying audit logs to cover their tracks.

**Why This Control:** The vulnerability assessment identified "weak internal controls" as a concern. Separation of duties is a fundamental internal control that prevents both fraud and accidental catastrophic errors. It also satisfies audit requirements for financial institutions.

**Core Banking Systems - Data Protection Policy**

**NIST CSF PR.DS-11 (Data Integrity):** Maintain immutable audit logs of all transaction changes.

**Implementation:** Every modification to customer accounts or transaction records is logged to a separate, append-only logging system. Logs include timestamp, user account, IP address, and before/after values. The logging system prevents deletion or modification of existing entries, even by administrators. This creates a verifiable record of all ledger activity that can be audited to detect unauthorized changes.

**Why This Control:** Financial regulations require audit trails for all transactions. Immutable logs prevent attackers or malicious insiders from covering their tracks by deleting evidence. Even if account balances are manipulated, the audit log shows exactly what changed and who changed it.

**Database Management Systems - Access Control Policy**

**NIST CSF PR.AA-05 (Access Permissions):** Disable all default 'sa' or 'root' accounts and require the use of named user accounts for all DBA tasks.

**Implementation:** The default 'sa' account in SQL Server and 'root' account in MySQL are disabled. Every database administrator has a named account tied to their actual identity. All database access is logged to individual accounts, creating accountability. Shared accounts are prohibited.

**Why This Control:** The vulnerability assessment identified "default configurations" as a weakness. Default accounts are the first thing attackers try after gaining access to a database server. Disabling them forces attackers to compromise specific user credentials, which are monitored. Named accounts also create audit trails showing exactly which administrator performed which actions.

**Network Equipment - Access Control Policy**

**NIST CSF PR.AA-03 (Identity Authentication):** Require SSH with MFA for all administrative access to routers, switches, and firewalls.

**Implementation:** Telnet and HTTP management interfaces are disabled on all network devices. Administrative access requires SSH (encrypted) with both password and time-based one-time password (TOTP) from an authenticator app. Failed authentication attempts trigger alerts to the network operations center.

**Why This Control:** The vulnerability assessment identified "unencrypted management traffic (Telnet)" as a vulnerability. Telnet transmits credentials in cleartext. Anyone sniffing the network sees admin passwords. SSH encrypts the session. MFA means stolen passwords alone cannot grant access.

**Network Equipment - Data Protection Policy**

**NIST CSF PR.DS-02 (Data-in-Transit Protection):** Disable Telnet and HTTP protocols in favor of secure alternatives like SSH and HTTPS.

**Implementation:** Configuration standards require SSH for command-line management and HTTPS for web-based management. Any attempt to enable Telnet or HTTP triggers a configuration compliance alert. Automated scanning detects devices violating the policy.

**Why This Control:** Preventing insecure protocols from being enabled is more effective than trying to detect their misuse. Once Telnet is enabled, credentials are already exposed. Disabling it at the configuration level prevents the vulnerability from existing.

**Employee Workstations - Access Control Policy**

**NIST CSF PR.AA-05 (Access Permissions):** Remove local administrative rights from all standard user accounts.

**Implementation:** Employees cannot install software, modify system settings, or disable security tools. Software installation requests go through IT support, who verify the business need and security of the requested application before installation. This prevents both accidental installation of malware and intentional installation of unauthorized tools.

**Why This Control:** The vulnerability assessment identified "human error" and "weak endpoint protection" as vulnerabilities. Many malware infections require administrative privileges to install. Removing admin rights blocks the majority of malware even if users click malicious links.

**Employee Workstations - Data Protection Policy**

**NIST CSF PR.DS-11 (Data Integrity):** Deploy managed endpoint encryption such as BitLocker or FileVault across all corporate laptops.

**Implementation:** Full-disk encryption is enabled on all laptops using BitLocker (Windows) or FileVault (macOS). Encryption keys are escrowed to a central management system. If a laptop is lost or stolen, the data is unreadable without the user's password. Remote wipe capabilities allow IT to destroy the encryption key, rendering the drive permanently unreadable.

**Why This Control:** The vulnerability assessment identified "physical loss" as a threat. Laptops left in taxis or stolen from cars are common. Without encryption, the thief can remove the hard drive and read all stored files. Encryption makes physical theft of the device irrelevant—the data remains protected.

## Findings

**Risk assessment must be quantitative, not qualitative.** Saying an asset is "high risk" without defining likelihood and impact provides no prioritization guidance. Using a 1-3 scale for each dimension and multiplying them creates a numeric ranking that shows which assets require the most investment. PII and customer financial data both scored 9, clearly indicating they need immediate attention.

**Access control policies work in layers.** Role-based access control (RBAC) limits what employees can see. Separation of duties requires multiple people to authorize critical actions. Multi-factor authentication prevents stolen credentials from granting access. Identity federation centralizes control over third-party access. No single control is sufficient. Each layer addresses a different attack vector.

**Data protection requires both encryption and integrity checks.** AES-256 encryption prevents unauthorized reading of data. Cryptographic hashing detects unauthorized modification of data. Both are necessary. An attacker who can't read encrypted data might still modify it. An attacker who can read unencrypted data might alter it. Different controls protect different properties (confidentiality vs. integrity).

**Incident response procedures must have pre-defined triggers and timelines.** "Notify legal if there's a breach" is too vague. The procedure specifies: if unencrypted PII of 500+ individuals is exposed, notify the Data Protection Commission within 72 hours. This removes ambiguity during a crisis when decision-making is impaired by stress.

**Security awareness training must be role-specific.** General employees need phishing awareness. Database administrators need secure configuration training. Network operations staff need firewall rule management training. Procurement staff need supply chain risk awareness. One-size-fits-all training wastes time on irrelevant topics and fails to cover role-specific threats.

**Default configurations are the most common vulnerability.** Default 'sa' accounts on databases, default 'admin' passwords on network equipment, Telnet enabled by default on older devices. Every risk assessment finding included "default configurations" as a vulnerability. Mature security programs have automated compliance scanning that detects deviations from hardened baselines.

**Third-party risk is indirect but significant.** Vendors don't target financial institutions directly—they target the financial institution's vendors who have weaker security but trusted network connections. API security, vendor audits, and identity federation all address the supply chain attack vector that bypasses perimeter defenses.

**Endpoint security is a people problem, not a technology problem.** No antivirus stops determined users from clicking phishing links. Full-disk encryption protects lost laptops. But the root cause is human behavior. Training addresses the source. Technical controls limit the damage when training fails.

## Challenges Faced

**Determining appropriate likelihood ratings without historical data:** Assigning likelihood ratings (1-3) requires knowing attack frequency. In a real assessment, historical security incident logs would inform these ratings. Without that data, I based likelihood on industry threat intelligence: PII and financial data are high-likelihood because financial institutions are constantly targeted. Intellectual property is low-likelihood because it requires sophisticated, targeted attacks that are less common.

**Balancing security controls with operational usability:** Blocking personal email and USB drives prevents data leakage but also prevents employees from working flexibly. Requiring MFA on network equipment improves security but adds friction to urgent troubleshooting. Security policies must acknowledge these trade-offs and justify why the risk reduction outweighs the operational inconvenience.

**Mapping NIST CSF controls to specific vulnerabilities:** The NIST CSF provides 108 controls across 23 categories. Selecting the right control requires understanding what each control actually does, not just reading the title. PR.AA-05 (Access Permissions) applies to PII, financial data, databases, and endpoints, but the implementation is different in each case. The control is a framework, not a specific technical configuration.

**Defining "medium" risk threshold:** Risk ratings ranged from 2 (IP) to 9 (PII/financial data). Where does "high" end and "medium" begin? I used 6+ as high, 3-5 as medium, 1-2 as low. This is arbitrary but necessary for prioritization. In practice, risk appetite is defined by executive leadership based on regulatory requirements and business tolerance for residual risk.

## Key Takeaways

**Risk assessment drives control selection.** Don't implement controls because a compliance checklist says to. Implement them because the risk assessment identified a specific threat that the control mitigates. Every control in this policy maps directly to a vulnerability in the risk assessment matrix. This creates defensible security spending—when executives ask why MFA is required on network equipment, the answer is "because the risk assessment identified default passwords as a vulnerability with medium likelihood and medium impact."

**Separation of duties prevents single points of failure.** No single administrator should be able to both modify data and delete audit logs. No single employee should be able to both view customer data and authorize money transfers. Requiring multiple people to complete sensitive actions prevents both malicious insiders and catastrophic mistakes.

**Encryption is table stakes, not a complete solution.** Every asset category includes encryption controls (AES-256 at rest, TLS in transit). But encryption alone doesn't prevent unauthorized access if the attacker has the decryption keys or accesses the data through authorized applications. Access control, authentication, and audit logging are equally critical.

**Third-party risk management is about controlling the relationship, not trusting the vendor.** Identity federation means vendors don't control their own access. Mutual TLS means vendors must present valid certificates. Vendor security audits verify controls are actually implemented. The policy doesn't assume vendors are secure—it enforces security through technical controls.

**Incident response procedures must have measurable triggers.** "If there's a breach, notify the DPC" is useless during a crisis. The procedure specifies: "If unencrypted PII of 500+ individuals is exposed, notify within 72 hours." This removes decision paralysis and ensures regulatory compliance.

**Security awareness training must match the threat to the role.** Customer service representatives need phishing awareness because they're targeted by credential theft campaigns. Database administrators need secure configuration training because they manage sensitive systems. Network operations staff need firewall rule management training because misconfigured rules create vulnerabilities. Generic "don't click suspicious links" training wastes everyone's time.

## Disclaimer

This policy framework was developed as an academic exercise to demonstrate understanding of risk assessment methodologies and the NIST Cybersecurity Framework. It is not a complete, implementation-ready security policy for a production financial institution. Real-world security policies require legal review, executive approval, regulatory consultation, and continuous updating based on threat intelligence and technology changes. No actual financial institution's data or systems were involved in this exercise.