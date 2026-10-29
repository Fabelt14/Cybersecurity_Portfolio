<div align="center">

# 🛡️ Fatai Asekun (Prime) Cybersecurity Portfolio

**Penetration Tester | SOC & Threat Intelligence | Malware Analysis | Digital Forensics**

*60+ hands-on, fully documented labs spanning offensive security, defensive operations, malware reverse engineering, and digital forensics, built during a 12-month Ethical Hacking & Digital Forensics program at ICDFA.*

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/asekun-fatai/)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat&logo=github&logoColor=white)](https://github.com/Fabelt14)
[![X](https://img.shields.io/badge/X-000000?style=flat&logo=x&logoColor=white)](https://x.com/Prime_shell)
[![Email](https://img.shields.io/badge/Email-D14836?style=flat&logo=gmail&logoColor=white)](mailto:asekunfatai14@gmail.com)

</div>

---

## 📋 Quick Navigation

- [👤 About Me](#-about-me)
- [🎯 Key Skills](#-key-skills--competencies)
- [⭐ Where to Start - Flagship Labs](#-where-to-start---flagship-labs)
- [🦠 Malware Analysis & Reverse Engineering](#-malware-analysis--reverse-engineering)
- [🚨 Security Operations Center (SOC) & Threat Detection](#-security-operations-center-soc--threat-detection)
- [🕵️ Threat Intelligence & Incident Response](#️-threat-intelligence--incident-response)
- [🌐 Web Application Security](#-web-application-security)
- [🔗 Network Security & Protocol Analysis](#-network-security--protocol-analysis)
- [🔐 Cryptography & Secure Communication](#-cryptography--secure-communication)
- [🖥️ Linux & System Administration](#️-linux--system-administration)
- [💾 Digital Forensics & Evidence Handling](#-digital-forensics--evidence-handling)
- [⚙️ Scripting, Password Attacks & Anonymity](#️-scripting-password-attacks--anonymity)
- [🔧 Technical Toolkit](#-technical-toolkit)
- [🛡️ Lab Environment](#️-lab-environment--disclaimer)
- [📬 Get In Touch](#-get-in-touch)

---

## 👤 About Me

I'm a cybersecurity personnel in Nigeria, building toward a penetration testing career through deliberate, hands-on practice rather than credential accumulation alone. I completed a 12-month Ethical Hacking & Digital Forensics internship at ICDFA, structured in two phases: 

- **Phase 1** covered foundational offensive and defensive security across Linux, networking, web application vulnerabilities, and cryptography.
- **Phase 2** went deeper into **SOC operations (Wazuh)**, **malware reverse engineering (static and dynamic analysis of real ransomware/RAT/dropper samples)**, and **threat intelligence / attribution**.

Every lab in this repository is a real exploit chain I walked through, a system I hardened, or a real malware sample I unpacked, not a theory summary. Each write-up documents the methodology, the exact commands/payloads used, the proof-of-concept evidence, the business impact, and the remediation, in a format modeled on professional penetration test and incident response reports.

**Target roles:** 
- Junior Penetration Tester
- SOC Analyst (Tier 1/2)
- Malware Analyst
- Threat Intelligence Analyst
- Security Researcher

---

## 🎯 Key Skills & Competencies

### 🔴 Offensive Security
- Web application exploitation (SQLi, XSS, CSRF, IDOR, file upload bypass, auth/session attacks)
- Command injection & reverse shells
- Password cracking (John the Ripper, rule-based & brute-force attacks)
- Network reconnaissance & enumeration (DNS, SMB, service discovery)
- MITM attacks (ARP poisoning, DNS spoofing)
- Anonymity tooling (Tor/Proxychains)

### 🟠 Malware Analysis & Reverse Engineering
- Static analysis (PE header/IAT/entropy inspection, hashing, IMPHASH pivoting)
- Dynamic analysis & unpacking (x32dbg, VirtualAlloc/VirtualProtect breakpoints, OEP recovery)
- Behavioral analysis (Process Monitor, Regshot, FakeNet-NG)
- MITRE ATT&CK mapping
- IOC extraction and threat classification (ransomware, RATs, droppers, trojans)

### 🟢 Defensive Security & SOC Operations
- SIEM deployment & threat hunting (Wazuh)
- File Integrity Monitoring (FIM)
- Windows Event Log analysis (4624/4625/4688/4104)
- Active response & automated containment
- IDS/IPS (Snort) rule writing and alert triage
- Network forensics (Wireshark, PCAP reconstruction, TCP stream analysis)

### 🔵 Threat Intelligence & Incident Response
- IOC categorization & infrastructure pattern analysis
- Threat actor profiling and attribution (confidence assessment, alternative-hypothesis testing)
- Cyber kill chain reconstruction
- Purple-team exercises (attack execution + SOC investigation)
- Case-study analysis of real-world incidents (Colonial Pipeline)

### 🟣 Digital Forensics
- Disk imaging & chain of custody (FTK Imager, `dd`)
- File system forensics (NTFS/FAT/EXT4, deleted file recovery)
- The Sleuth Kit / Autopsy (inode-level recovery: `icat`, `blkcat`, `blkls`)
- Evidence documentation aligned to legal/chain-of-custody standards

### ⚫ Core Technical Competencies
- Linux & Windows system administration
- Bash/PowerShell scripting
- TCP/IP, subnetting, routing, VLANs
- Database security (SQL)
- Cryptography & PKI
- OWASP Top 10
- Risk assessment & NIST CSF-aligned policy writing

---

## ⭐ Where to Start - Flagship Labs

These labs best demonstrate end-to-end capability:

| Lab | Why It Stands Out |
|---|---|
| **[Operation Silent Intrusion](./ICDFA%20Internship/Phase%20II/Network%20Security%20Operations/Lab_48%20Operation%20Silent%20Intrusion.md)** | Full PCAP-based incident reconstruction: SQLi → admin compromise → web shell upload, written as a client-facing forensic report |
| **[Advanced Threat Intelligence & Cyber Kill Chain Analysis](./ICDFA%20Internship/Phase%20II/Security%20and%20Operation%20Centre%20(SOC)/Lab_58%20Advanced%20Threat%20Intelligence%20%26%20Cyber%20Kill%20Chain%20Analysis.md)** | Purple-team exercise: executed a full multi-stage intrusion myself, then investigated and contained it in Wazuh |
| **[Reverse Engineering of Malware Sample 3 (Lokibot Trojan)](./ICDFA%20Internship/Phase%20II/Malware%20Analysis%20%26%20Exploit%20Development/Lab_55%20Reverse%20Engineering%20of%20Malware%20Sample%203%20(Lokibot%20Trojan).md)** | Defeated anti-debugging, unpacked a fileless AutoIt-wrapped credential stealer, dumped the payload from live memory |
| **[Threat Actor Profiling & Attribution](./ICDFA%20Internship/Phase%20II/Threat%20Intelligence/Lab_59%20Threat%20Actor%20Profiling%20%26%20Attribution.md)** | Full intelligence report: infrastructure analysis, MITRE mapping, and a structured, confidence-rated APT attribution |
| **[Manual and Automated SQL Injection Testing](./ICDFA%20Internship/Phase%201/Lab_29%20Manual%20and%20Automated%20SQL%20Injection%20Testing.md)** | Union-based, error-based, and boolean-based SQLi, manual exploitation validated against SQLMap output |
| **[Digital Forensics Investigation of a USB Drive](./ICDFA%20Internship/Phase%201/Lab_45%20Digital%20Forensics%20Investigation%20of%20a%20USB%20Drive.md)** | Full 4-phase forensic investigation (identification → presentation) recovering deleted evidence of an extortion scheme |

---

## 🦠 Malware Analysis & Reverse Engineering

*Static and dynamic analysis of real-world malware samples (ransomware, RATs, droppers, infostealers) in an isolated Flare VM environment.*

| Lab | Sample / Focus | Key Techniques | Documentation |
|---|---|---|---|
| 50 | Windows API in Malware (Research) | API abuse patterns, WannaCry/Blaster/Stuxnet case studies, API call-sequence detection theory | [📄 View](./ICDFA%20Internship/Phase%20II/Malware%20Analysis%20%26%20Exploit%20Development/Lab_50%20In-Depth%20Research%20on%20Windows%20API%20and%20Its%20Use%20in%20Malware%20Analysis.md) |
| 51 | Unidentified PE32 Downloader | peStudio, FLOSS, IMPHASH pivoting, entropy mapping, self-deletion & C2 reconstruction, MITRE mapping | [📄 View](./ICDFA%20Internship/Phase%20II/Malware%20Analysis%20%26%20Exploit%20Development/Lab_51%20Advanced%20Malware%20Static%20Analysis.md) |
| 52 | TeslaCrypt Ransomware | Custom packer detection, x32dbg unpacking (VirtualAlloc/VirtualProtect breakpoints), OEP recovery, Process Hacker memory dump | [📄 View](./ICDFA%20Internship/Phase%20II/Malware%20Analysis%20%26%20Exploit%20Development/Lab_52%20TeslaCrypt%20Ransomware%20Analysis.md) |
| 53 | Nim-Compiled RAT/Dropper | Full PE structure teardown, dynamic API resolution detection, TLS callback analysis, VirusTotal/MalAPI.io correlation | [📄 View](./ICDFA%20Internship/Phase%20II/Malware%20Analysis%20%26%20Exploit%20Development/Lab_53%20Advanced%20RAT%20Static%20Analysis%20Assignment.md) |
| 54 | `financials-xls.exe` (UPX-packed Dropper) | UPX unpacking, Procmon filtering, Regshot diffing, FakeNet-NG network simulation, registry persistence discovery | [📄 View](./ICDFA%20Internship/Phase%20II/Malware%20Analysis%20%26%20Exploit%20Development/Lab_54%20Unpacking%20and%20Analysis%20of%20Sample%202.md) |
| 55 | Lokibot Trojan (AutoIt-wrapped) | Exe2Aut decompilation, `IsDebuggerPresent` bypass in x32dbg, VirtualAlloc breakpoint payload staging, live memory dump | [📄 View](./ICDFA%20Internship/Phase%20II/Malware%20Analysis%20%26%20Exploit%20Development/Lab_55%20Reverse%20Engineering%20of%20Malware%20Sample%203%20(Lokibot%20Trojan).md) |

**Tools:** PEStudio, PEbear, Detect It Easy (DIE), FLOSS, Ghidra, x32dbg/x64dbg, Process Hacker, Process Monitor, Regshot, FakeNet-NG, Exe2Aut, VirusTotal, MalAPI.io

---

## 🚨 Security Operations Center (SOC) & Threat Detection

*Building and operating a SIEM, and IDS deployment.*

| Lab | Focus | Key Techniques | Documentation |
|---|---|---|---|
| 56 | Wazuh SIEM Deployment & FIM | Manager/agent deployment, real-time File Integrity Monitoring, alert rule IDs, checksum-based tamper detection | [📄 View](./ICDFA%20Internship/Phase%20II/Security%20and%20Operation%20Centre%20(SOC)/Lab_56%20Introduction%20to%20Wazuh%20for%20Threat%20Detection%20%26%20Monitoring.md) |
| 57 | Threat Hunting & Incident Response | Event correlation (4624/4625/4688/4104), PowerShell Script Block Logging, automated active-response firewall containment | [📄 View](./ICDFA%20Internship/Phase%20II/Security%20and%20Operation%20Centre%20(SOC)/Lab_57%20Threat%20Hunting%20%26%20Incident%20Response%20with%20Wazuh.md) |
| 58 | Advanced Threat Intel & Kill Chain (Purple Team) | Self-executed multi-stage intrusion (brute force → persistence → lateral movement → exfiltration) then investigated via Wazuh | [📄 View](./ICDFA%20Internship/Phase%20II/Security%20and%20Operation%20Centre%20(SOC)/Lab_58%20Advanced%20Threat%20Intelligence%20%26%20Cyber%20Kill%20Chain%20Analysis.md) |
| 18 | Intrusion Detection Systems (IDS) | Snort 3 custom rule writing, TCP SYN flood detection, Wireshark/Snort alert correlation | [📄 View](./ICDFA%20Internship/Phase%201/Lab_18%20Intrusion%20Detection%20Systems%20(IDS)%20and%20Traffic%20Analysis%20on%20Linux.md) |

**Tools:** Wazuh (Manager + Agent), Snort 3, Sysmon, PowerShell Script Block Logging, Wireshark, hping3

---

## 🕵️ Threat Intelligence & Incident Response

*Intelligence reporting and PCAP-based forensic incident reconstruction.*

| Lab | Focus | Key Techniques | Documentation |
|---|---|---|---|
| 59 | Threat Actor Profiling & Attribution | Infrastructure clustering, MITRE ATT&CK full matrix, Admiralty-code confidence rating, structured attribution vs. alternative hypotheses | [📄 View](./ICDFA%20Internship/Phase%20II/Threat%20Intelligence/Lab_59%20Threat%20Actor%20Profiling%20%26%20Attribution.md) |
| 48 | Operation Silent Intrusion | PCAP reconstruction of SQLi → admin brute force → web shell upload against a live e-commerce target | [📄 View](./ICDFA%20Internship/Phase%20II/Network%20Security%20Operations/Lab_48%20Operation%20Silent%20Intrusion.md) |
| 49 | Operation Shadow Drop | Double-extension file upload bypass, reverse shell over 8080, exfiltration disguised as HTTPS (port 443) | [📄 View](./ICDFA%20Internship/Phase%20II/Network%20Security%20Operations/Lab_49%20Operation%20Shadow%20Drop.md) |
| 17 | Colonial Pipeline Ransomware Case Study | MFA/segmentation failure analysis, Zero Trust & IT/OT isolation mitigation strategy | [📄 View](./ICDFA%20Internship/Phase%201/Lab_17%20Applied%20Network%20Security%20Analysis:%20Colonial%20Pipeline%20Ransomware%20Case%20Study.md) |

---

## 🌐 Web Application Security

*OWASP Top 10 exploitation against DVWA, Mutillidae II, and live reconnaissance targets.*

| Lab | Vulnerability / Topic | Key Skills | Documentation |
|---|---|---|---|
| 03 | Web Application Reconnaissance | WHOIS, DNS enumeration, subdomain enumeration (Sublist3r vs. Knockpy), Nikto, Dirb | [📄 View](./ICDFA%20Internship/Phase%201/Lab_03%20Web%20Application%20Reconnaissance.md) |
| 06 | Burp Suite & OWASP ZAP | Proxy interception, spidering, automated scanning, manual XSS confirmation | [📄 View](./ICDFA%20Internship/Phase%201/Lab_06%20Web%20Application%20Security%20Testing%20with%20Burp%20Suite%20and%20OWASP%20ZAP.md) |
| 08 | Command Injection → Reverse Shell | DVWA command execution bypass, Netcat reverse shell, privilege/kernel exploit assessment | [📄 View](./ICDFA%20Internship/Phase%201/Lab_08%20Reverse%20Shell%20via%20Netcat%20Using%20DVWA%20Command%20Execution.md) |
| 20 | Session Management Vulnerabilities | Session fixation, session hijacking via Burp, cookie flag auditing | [📄 View](./ICDFA%20Internship/Phase%201/Lab_20%20Session%20Management%20Vulnerabilities.md) |
| 21 | Secure Session Management Practices | HttpOnly/SameSite hardening, server-side session invalidation, timeout enforcement | [📄 View](./ICDFA%20Internship/Phase%201/Lab_21%20Secure%20Session%20Management%20Practices.md) |
| 22 | CSRF Protection | Live CSRF attack simulation, token-based defense implementation and bypass testing | [📄 View](./ICDFA%20Internship/Phase%201/Lab_22%20Cross-Site%20Request%20Forgery%20(CSRF)%20Protection.md) |
| 23 | Session Hijacking CTF (Juice Shop) | JWT token replay after logout, account takeover via token injection, JWT payload analysis | [📄 View](./ICDFA%20Internship/Phase%201/Lab_23%20Session%20Hijacking%20CTF.md) |
| 29 | Manual & Automated SQL Injection | UNION/error/boolean-based SQLi, authentication bypass, SQLMap validation, cleartext password discovery | [📄 View](./ICDFA%20Internship/Phase%201/Lab_29%20Manual%20and%20Automated%20SQL%20Injection%20Testing.md) |
| 30 | XSS (Stored, Reflected, DOM) | Filter bypass (case variation, `img onerror`), client-side validation bypass, CSP remediation | [📄 View](./ICDFA%20Internship/Phase%201/Lab_30%20XSS%20Vulnerabilities%20in%20DVWA.md) |
| 31 | File Upload Vulnerabilities | MIME bypass, magic-byte + double-extension bypass, Weevely web shell, RCE confirmation | [📄 View](./ICDFA%20Internship/Phase%201/Lab_31%20File%20Upload%20Vulnerabilities%20in%20DVWA.md) |
| 32 | Security Misconfiguration | Default creds, directory listing, verbose errors, missing security headers | [📄 View](./ICDFA%20Internship/Phase%201/Lab_32%20Security%20Misconfiguration%20in%20DVWA.md) |
| 33 | IDOR | Sequential ID enumeration, authorization-vs-authentication analysis, First American Financial breach parallel | [📄 View](./ICDFA%20Internship/Phase%201/Lab_33%20Insecure%20Direct%20Object%20References%20(IDOR)%20in%20DVWA.md) |
| 34 | HTTP / HTTPS Fundamentals | Header analysis, TLS certificate chain validation, CORS/Origin enforcement | [📄 View](./ICDFA%20Internship/Phase%201/Lab_34%20Introduction%20to%20Web%20Technologies%20and%20Understanding%20HTTP%20and%20HTTPS.md) |
| 35 | Web Dev Components & Threats | Front-end/back-end architecture, SQLi/XSS/CSRF practical tie-in, Node.js server build | [📄 View](./ICDFA%20Internship/Phase%201/Lab_35%20Web%20Development%20Components%20and%20Security%20Threats.md) |
| 36 | Databases & SQL | Schema design, foreign keys, JOINs, referential integrity, ACID compliance | [📄 View](./ICDFA%20Internship/Phase%201/Lab_36%20Introduction%20to%20Databases%20and%20SQL.md) |
| 37 | Secure Coding & Advanced Vulnerability Analysis | SSRF, XXE, insecure deserialization, race conditions (TOCTOU), JWT alg-confusion, full RCE attack chain | [📄 View](./ICDFA%20Internship/Phase%201/Lab_37%20Hands-on%20Secure%20Coding%20and%20Advanced%20Web%20Application%20Vulnerability%20Analysis.md) |

---

## 🔗 Network Security & Protocol Analysis

*Packet-level analysis, MITM attacks, and network architecture.*

| Lab | Focus | Key Skills | Documentation |
|---|---|---|---|
| 04 | Wireshark Protocol Analysis | TCP handshake, HTTP/SSH comparison, credential exposure, protocol hierarchy | [📄 View](./ICDFA%20Internship/Phase%201/Lab_04%20Wireshark%20Network%20Protocol%20Analysis.md) |
| 05 | Wireshark Case Study: Challenge | Full attack reconstruction from PCAP: file upload → reverse shell → `/etc/passwd` exfil | [📄 View](./ICDFA%20Internship/Phase%201/Lab_05%20Wireshark%20Case%20Study:%20Challenge.md) |
| 07 | IoT & Enterprise Forensic Investigation | Brute force, SQLi, unrestricted file upload chain across a simulated e-commerce breach | [📄 View](./ICDFA%20Internship/Phase%201/Lab_07%20IoT%20%26%20Enterprise%20Network%20Forensic%20Investigation%20Using%20Wireshark.md) |
| 09 | DNS Query Tools & SMB Enumeration | `nslookup`/`host`/`dig`, `enum4linux` null-session enumeration, user/share exposure | [📄 View](./ICDFA%20Internship/Phase%201/Lab_09%20DNS%20Query%20Tools%20and%20SMB%20Enumeration.md) |
| 12 | Basic Network Configuration & Troubleshooting | OSI/TCP-IP mapping, ping/traceroute/netstat, TCP vs UDP scan timing | [📄 View](./ICDFA%20Internship/Phase%201/Lab_12%20Basic%20Network%20Configuration:%20Setting%20Up%20and%20Troubleshooting.md) |
| 13 | IPv4/IPv6 Addressing & Subnetting | Interface analysis, packet capture, bandwidth monitoring, advanced filtering | [📄 View](./ICDFA%20Internship/Phase%201/Lab_13%20Understanding%20IP%20Addressing:%20IPv4%20and%20IPv6%20Classification%20and%20Subnetting.md) |
| 14 | Packet Analysis with Wireshark (Advanced) | TCP reliability internals, OS fingerprinting via Nmap, ARP mechanics, layered troubleshooting | [📄 View](./ICDFA%20Internship/Phase%201/Lab_14%20Packet%20Analysis%20with%20Wireshark:%20Capture,%20Filter,%20and%20Analyze%20Network%20Traffic.md) |
| 15 | Network Security Policy & Risk Management | NIST CSF control mapping, quantitative risk matrix, 8-asset policy framework | [📄 View](./ICDFA%20Internship/Phase%201/Lab_15%20Network%20Security%20Policies%20and%20Risk%20Management.md) |
| 16 | Network Routing & VLAN Simulation | Linux network namespaces, static routing, iptables FORWARD chain, subnet boundary math | [📄 View](./ICDFA%20Internship/Phase%201/Lab_16%20Simulating%20Network%20Routing%20and%20VLAN%20Configuration%20in%20Linux.md) |
| 19 | Network Security Protocols | SSL/TLS cert generation, OpenVPN PKI build, SSH hardening, 802.1X/RADIUS access control | [📄 View](./ICDFA%20Internship/Phase%201/Lab_19%20Network%20Security%20Protocols%20and%20Configuration.md) |
| 42 | Simple Website + Traffic Capture | Apache deployment, TCP 3-way handshake dissection, loopback vs. external traffic comparison | [📄 View](./ICDFA%20Internship/Phase%201/Lab_42%20Creating%20a%20Simple%20Website%20and%20Capturing%20Network%20Traffic.md) |
| 43 | HTTP Traffic with Embedded Images | File signature extraction (magic bytes), IP/TCP overhead calculation, hexdump analysis | [📄 View](./ICDFA%20Internship/Phase%201/Lab_43%20Capturing%20and%20Analyzing%20HTTP%20Traffic%20with%20Embedded%20Images.md) |
| 44 | Packet Sniffing, DNS Spoofing & ARP Poisoning | Bettercap MITM, ARP cache poisoning, DNS spoof, Layer2/Layer3 mismatch forensic detection | [📄 View](./ICDFA%20Internship/Phase%201/Lab_44%20Packet%20Sniffing,%20Interception,%20DNS%20Spoofing,%20and%20ARP%20Poisoning.md) |

---

## 🔐 Cryptography & Secure Communication

*Classical to post-quantum cryptography, applied hands-on in Python/OpenSSL.*

| Lab | Focus | Key Skills | Documentation |
|---|---|---|---|
| 27 | Comprehensive Journey into Secure Communication | Caesar/Vigenère ciphers, AES modes (ECB/CBC/GCM), RSA, Diffie-Hellman, PKI trust chains, hash length-extension attacks, zero-knowledge proofs | [📄 View](./ICDFA%20Internship/Phase%201/Lab_27%20Comprehensive%20Journey%20into%20Secure%20Communication.md) |
| 28 | Post-Quantum Cryptography | ML-KEM / ML-DSA key generation, OQS/OpenSSL 3.x providers, hybrid migration strategy | [📄 View](./ICDFA%20Internship/Phase%201/Lab_28%20Post-Quantum%20Cryptography%20and%20Future%20Proofing%20Security.md) |

---

## 🖥️ Linux & System Administration

*Hardening, access control, and system internals.*

| Lab | Focus | Key Skills | Documentation |
|---|---|---|---|
| 01 | Linux OS Fundamentals | Directory structures, octal permissions, log analysis via `grep`/`wc`/piping | [📄 View](./ICDFA%20Internship/Phase%201/Lab_01%20Linux%20OS%20Fundamentals.md) |
| 24 | User Management | Account lifecycle, group policy, account locking, secure deletion practices | [📄 View](./ICDFA%20Internship/Phase%201/Lab_24%20User%20Management.md) |
| 25 | File & Directory Permissions | Numeric notation, ownership delegation, least-privilege directory design | [📄 View](./ICDFA%20Internship/Phase%201/Lab_25%20Linux%20File%20and%20Directory%20Permissions.md) |
| 26 | Strengthening System Security on Linux Servers | ACLs, role-based sudo, SSH key hardening, 2FA (PAM), password aging, `auditd` | [📄 View](./ICDFA%20Internship/Phase%201/Lab_26%20Strengthening%20System%20Security%20on%20Linux%20Servers.md) |
| 41 | Operating Systems & File Systems in Linux | VFS layer, PATH persistence/attacker abuse, shell scripting for forensic triage | [📄 View](./ICDFA%20Internship/Phase%201/Lab_41%20Introduction%20to%20Operating%20Systems%20and%20File%20Systems%20in%20Linux.md) |

---

## 💾 Digital Forensics & Evidence Handling

*Foundational forensic theory through to real disk imaging and file recovery.*

| Lab | Focus | Key Skills | Documentation |
|---|---|---|---|
| 38 | Number Systems, Dates, ASCII, Disk Drives & Forensics | Binary/hex conversion, Unix epoch, extended ASCII, disk geometry, chain-of-custody principles | [📄 View](./ICDFA%20Internship/Phase%201/Lab_38%20Foundational%20Concepts:%20Number%20Systems,%20Date%20Formats,%20ASCII%20Codes,%20Disk%20Drives,%20and%20Computer%20Forensics.md) |
| 39 | Computer Systems, Forensics Challenges & Storage Structures | Data volatility, encryption as a forensic obstacle, boot-process artifacts, file-system comparison | [📄 View](<./ICDFA%20Internship/Phase%201/Lab_39%20Understanding%20Computer%20Systems,%20Forensics%20Challenges,%20and%20Storage%20Structures%0AOverview.md>) |
| 40 | Windows File System, Networking & Batch Scripting | NTFS allocation units, `ipconfig`/`tracert`/`netstat`, SMB share risk, `.bat` automation | [📄 View](./ICDFA%20Internship/Phase%201/Lab_40%20Exploring%20the%20Windows%20File%20System,%20Networking,%20and%20Batch%20Scripting.md) |
| 45 | Digital Forensics Investigation of a USB Drive | 4-phase investigation model, Autopsy deleted-file recovery, keyword search, intent analysis | [📄 View](./ICDFA%20Internship/Phase%201/Lab_45%20Digital%20Forensics%20Investigation%20of%20a%20USB%20Drive.md) |
| 46 | Autopsy vs. The Sleuth Kit (TSK) | TSK architecture layers, `fls`/`istat`/`icat`/`blkcat`/`blkls` inode-level recovery comparison | [📄 View](./ICDFA%20Internship/Phase%201/Lab_46%20Comparative%20Analysis%20of%20Autopsy%20and%20The%20Sleuth%20Kit%20(TSK).md) |
| 47 | Disk Imaging Techniques (USB Acquisition) | FTK Imager (Windows) vs. `dd` (Linux), MD5/SHA1 hash verification, network acquisition trade-offs | [📄 View](./ICDFA%20Internship/Phase%201/Lab_47%20Disk%20Imaging%20Techniques:%20Acquisition%20Methods%20for%20USB%20Drives.md) |

---

## ⚙️ Scripting, Password Attacks & Anonymity

*Automation and advanced attacker tradecraft.*

| Lab | Focus | Key Skills | Documentation |
|---|---|---|---|
| 02 | Bash Scripting Fundamentals | Conditionals, loops, functions, network/memory monitoring scripts | [📄 View](./ICDFA%20Internship/Phase%201/Lab_02%20Bash%20Scripting%20Fundamentals.md) |
| 10 | Anonymity Testing with Tor & Proxychains | SOCKS5 routing, circuit rotation, exit-node risk, Nmap-over-Tor performance cost | [📄 View](./ICDFA%20Internship/Phase%201/Lab_10%20Anonymity%20Testing%20with%20Tor%20and%20Proxychains.md) |
| 11 | Password Cracking with John the Ripper | Wordlist/incremental/rule-based attacks, NTLM cracking, hash-algorithm risk comparison | [📄 View](./ICDFA%20Internship/Phase%201/Lab_11%20Password%20Cracking%20with%20John%20the%20Ripper.md) |

---

## 📈 Lab Portfolio Statistics

```
╔══════════════════════════════════════════════════════╗
║            COMPLETE LAB PORTFOLIO BREAKDOWN           ║
╚══════════════════════════════════════════════════════╝

Total Documented Labs: 64+

ICDFA Phase 1 (Foundations)............... 47 labs
ICDFA Phase 2 (Advanced Specialization).... 12 labs
  ├── Malware Analysis & Exploit Dev...... 6 labs
  ├── Network Security Operations......... 2 labs
  ├── Security Operations Center (SOC).... 3 labs
  └── Threat Intelligence................. 1 lab
Cisco Networking (CCNA Track)..............  5 labs

Coverage:
  Offensive Security ............. ✓
  Defensive Security / SOC ........ ✓
  Malware Analysis & RE ........... ✓
  Threat Intelligence ............. ✓
  Digital Forensics ............... ✓
  Cryptography ..................... ✓
  Risk Assessment & Policy ........ ✓
  OWASP Top 10 ..................... ✓
```

---

## 🔧 Technical Toolkit

### 🎯 Penetration Testing & Exploitation
- Burp Suite
- OWASP ZAP
- Metasploit Framework
- Netcat
- SQLMap
- Weevely

### 🦠 Malware Analysis & Reverse Engineering
- PEStudio
- PEbear
- Detect It Easy (DIE)
- FLOSS
- Ghidra
- x32dbg / x64dbg
- Process Hacker
- Process Monitor (Procmon)
- Regshot
- FakeNet-NG
- Exe2Aut
- VirusTotal
- MalAPI.io
- Flare VM

### 🚨 SOC, SIEM & Threat Detection
- Wazuh (Manager + Agent, FIM, Active Response)
- Snort 3
- Sysmon
- Wireshark
- tshark

### 🔍 Reconnaissance & Enumeration
- Nmap
- Recon-ng
- SpiderFoot
- WhatWeb
- Sublist3r
- Knockpy
- Dirb
- whois / dig / nslookup
- enum4linux

### 🖥️ Linux & System Administration
- bash/zsh
- iptables/ufw
- SELinux
- auditd
- systemd
- network namespaces

### 🪟 Windows & Forensics
- FTK Imager
- Autopsy
- The Sleuth Kit (TSK)
- `dd`
- Batch/PowerShell scripting
- Event log & registry analysis

### 🔑 Password & Hash Analysis
- John the Ripper
- hashcat
- Hydra

### 🌐 Network Attack & Anonymity Tools
- Bettercap
- arpspoof/dnsspoof
- Tor
- Proxychains

### 🛠️ Infrastructure & Virtualization
- VirtualBox
- Kali Linux
- Flare VM

### 📚 Development & Documentation
- Git/GitHub
- Bash/Python
- Markdown

---

## 🛡️ Lab Environment & Disclaimer

All labs were conducted in **controlled, isolated virtual environments** (host-only/NAT VirtualBox or VMware networks, Flare VM for malware analysis) against intentionally vulnerable targets (DVWA, Mutillidae II, OWASP BWA/Juice Shop, Metasploitable), provided PCAP files, or deliberately authorized self-built lab infrastructure. No production systems, unauthorized networks, or live malicious infrastructure were accessed at any point. Every lab write-up includes its own scope and authorization statement.

| Component | Detail |
|---|---|
| **Host OS** | Windows 11 / Kali Linux |
| **Hypervisor** | VirtualBox / VMware Workstation |
| **Target Systems** | DVWA, Mutillidae II, Metasploitable, OWASP BWA/Juice Shop, isolated Windows/Ubuntu VMs |
| **Malware Analysis Environment** | Flare VM, fully network-isolated |
| **Compliance** | All activity documented, ethically conducted, authorized by course instruction |

---

## 📬 Get In Touch

I'm actively seeking mentorship, collaboration, and entry-level opportunities in penetration testing, SOC operations, and malware analysis.

| Platform | Profile |
|---|---|
| **LinkedIn** | [Fatai Asekun](https://www.linkedin.com/in/asekun-fatai/) |
| **GitHub** | [Fabelt14](https://github.com/Fabelt14) |
| **X (Twitter)** | [@Prime_shell](https://x.com/Prime_shell) |
| **Email** | [asekunfatai14@gmail.com](mailto:asekunfatai14@gmail.com) |
