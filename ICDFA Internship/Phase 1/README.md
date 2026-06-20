# ICDFA Internship Phase 1 - Lab Exercises

## Overview
Phase 1 of the ICDFA internship covers foundational cybersecurity skills including Linux fundamentals, scripting, web reconnaissance, network analysis, web application security testing, cryptography, and digital forensics.

---

## Lab Summary Table

| # | Lab Name | Description | Tools Used |
|---|----------|-------------|-----------|
| 1 | [Lab_01 Linux OS Fundamentals](Lab_01%20Linux%20OS%20Fundamentals.md) | Command line basics, directory management, file permissions, and log analysis using the Linux terminal | mkdir, touch, chmod, cat, grep |
| 2 | [Lab_02 Bash Scripting Fundamentals](Lab_02%20Bash%20Scripting%20Fundamentals.md) | Writing Bash scripts for automation including user input, conditionals, loops, functions, and system monitoring | Bash, vim, chmod |
| 3 | [Lab_03 Web Application Reconnaissance](Lab_03%20Web%20Application%20Reconnaissance.md) | Information gathering on live domains and vulnerable web apps including IP resolution, WHOIS, DNS queries | nslookup, dig, whois, curl |
| 4 | [Lab_04 Wireshark Network Protocol Analysis](Lab_04%20Wireshark%20Network%20Protocol%20Analysis.md) | Packet capture and analysis including filtering, dissection, stream reconstruction, and threat identification | Wireshark, tcpdump |
| 5 | [Lab_05 Wireshark Case Study: Challenge](Lab_05%20Wireshark%20Case%20Study:%20Challenge.md) | Real-world packet analysis challenges and forensic investigation scenarios using captured PCAP files | Wireshark, PCAP files |
| 6 | [Lab_06 Web Application Security Testing with Burp Suite and OWASP ZAP](Lab_06%20Web%20Application%20Security%20Testing%20with%20Burp%20Suite%20and%20OWASP%20ZAP.md) | Web vulnerability scanning, proxy interception, and automated scanning techniques | Burp Suite, OWASP ZAP |
| 7 | [Lab_07 IoT & Enterprise Network Forensic Investigation Using Wireshark](Lab_07%20IoT%20&%20Enterprise%20Network%20Forensic%20Investigation%20Using%20Wireshark.md) | Advanced network forensics on IoT and enterprise environments with protocol analysis | Wireshark, tcpdump |
| 8 | [Lab_08 Reverse Shell via Netcat Using DVWA Command Execution](Lab_08%20Reverse%20Shell%20via%20Netcat%20Using%20DVWA%20Command%20Execution.md) | Exploiting command injection vulnerabilities to establish reverse shells | netcat, DVWA, bash |
| 9 | [Lab_09 DNS Query Tools and SMB Enumeration](Lab_09%20DNS%20Query%20Tools%20and%20SMB%20Enumeration.md) | DNS reconnaissance and SMB protocol enumeration to identify network resources and services | nslookup, dig, smbclient, enum4linux |
| 10 | [Lab_10 Anonymity Testing with Tor and Proxychains](Lab_10%20Anonymity%20Testing%20with%20Tor%20and%20Proxychains.md) | Testing anonymous network access and traffic routing through proxy chains | Tor, proxychains |
| 11 | [Lab_11 Password Cracking with John the Ripper](Lab_11%20Password%20Cracking%20with%20John%20the%20Ripper.md) | Password hash cracking and dictionary/brute-force attack techniques | John the Ripper, hashcat |
| 12 | [Lab_12 Basic Network Configuration: Setting Up and Troubleshooting](Lab_12%20Basic%20Network%20Configuration:%20Setting%20Up%20and%20Troubleshooting.md) | Network interface configuration, IP assignment, gateway setup, and connectivity troubleshooting | ifconfig, ip, netstat, ping |
| 13 | [Lab_13 Understanding IP Addressing: IPv4 and IPv6 Classification and Subnetting](Lab_13%20Understanding%20IP%20Addressing:%20IPv4%20and%20IPv6%20Classification%20and%20Subnetting.md) | IP address classes, CIDR notation, subnetting calculations, and IPv6 fundamentals | ipcalc, sipcalc |
| 14 | [Lab_14 Packet Analysis with Wireshark: Capture, Filter, and Analyze Network Traffic](Lab_14%20Packet%20Analysis%20with%20Wireshark:%20Capture,%20Filter,%20and%20Analyze%20Network%20Traffic.md) | Advanced packet capture techniques, filtering, display filters, and traffic analysis workflows | Wireshark, tcpdump |
| 15 | [Lab_15 Network Security Policies and Risk Management](Lab_15%20Network%20Security%20Policies%20and%20Risk%20Management.md) | Security policy development, risk assessment frameworks, and security policy implementation | Risk matrices, NIST framework |
| 16 | [Lab_16 Simulating Network Routing and VLAN Configuration in Linux](Lab_16%20Simulating%20Network%20Routing%20and%20VLAN%20Configuration%20in%20Linux.md) | Virtual LAN setup, routing table manipulation, and inter-VLAN communication | route, vconfig, iptables |
| 17 | [Lab_17 Applied Network Security Analysis: Colonial Pipeline Ransomware Case Study](Lab_17%20Applied%20Network%20Security%20Analysis:%20Colonial%20Pipeline%20Ransomware%20Case%20Study.md) | Real-world analysis of the Colonial Pipeline ransomware attack using network forensics | Wireshark, forensic analysis tools |
| 18 | [Lab_18 Intrusion Detection Systems (IDS) and Traffic Analysis on Linux](Lab_18%20Intrusion%20Detection%20Systems%20(IDS)%20and%20Traffic%20Analysis%20on%20Linux.md) | IDS/IPS deployment and configuration including Snort and Suricata | Snort, Suricata, tcpdump |
| 19 | [Lab_19 Network Security Protocols and Configuration](Lab_19%20Network%20Security%20Protocols%20and%20Configuration.md) | Secure protocol implementation including TLS/SSL, SSH, IPSec, and VPN configuration | OpenSSL, ssh, openvpn |
| 20 | [Lab_20 Session Management Vulnerabilities](Lab_20%20Session%20Management%20Vulnerabilities.md) | Identifying and exploiting session hijacking, fixation, and management flaws in web applications | Burp Suite, Firefox Dev Tools |
| 21 | [Lab_21 Secure Session Management Practices](Lab_21%20Secure%20Session%20Management%20Practices.md) | Implementation of secure session tokens, cookie flags, and session timeout mechanisms | Web frameworks, cookie editors |
| 22 | [Lab_22 Cross-Site Request Forgery (CSRF) Protection](Lab_22%20Cross-Site%20Request%20Forgery%20(CSRF)%20Protection.md) | Understanding CSRF vulnerabilities and implementing CSRF tokens and protections | Burp Suite, DVWA, web frameworks |
| 23 | [Lab_23 Session Hijacking CTF](Lab_23%20Session%20Hijacking%20CTF.md) | Capture-the-flag challenge focusing on session hijacking techniques and defense | Wireshark, session cookies, HTTP interception |
| 24 | [Lab_24 User Management](Lab_24%20User%20Management.md) | Linux user account creation, group management, sudo configuration, and access control | useradd, userdel, groupadd, usermod, visudo |
| 25 | [Lab_25 Linux File and Directory Permissions](Lab_25%20Linux%20File%20and%20Directory%20Permissions.md) | Understanding and configuring file/directory permissions, ownership, and access control | chmod, chown, chgrp, ls -l |
| 26 | [Lab_26 Strengthening System Security on Linux Servers](Lab_26%20Strengthening%20System%20Security%20on%20Linux%20Servers.md) | Security hardening techniques, firewall rules, SELinux policies, and system lockdown | iptables, firewall-cmd, SELinux |
| 27 | [Lab_27 Comprehensive Journey into Secure Communication](Lab_27%20Comprehensive%20Journey%20into%20Secure%20Communication.md) | Encryption, digital signatures, public key cryptography, and secure communication protocols | OpenSSL, GPG, SSH, TLS |
| 28 | [Lab_28 Post-Quantum Cryptography and Future Proofing Security](Lab_28%20Post-Quantum%20Cryptography%20and%20Future%20Proofing%20Security.md) | Post-quantum encryption algorithms and preparing security infrastructure for quantum computing | liboqs, post-quantum algorithms |
| 29 | [Lab_29 Manual and Automated SQL Injection Testing](Lab_29%20Manual%20and%20Automated%20SQL%20Injection%20Testing.md) | SQL injection vulnerability detection and exploitation in web applications | Burp Suite, sqlmap, DVWA |
| 30 | [Lab_30 XSS Vulnerabilities in DVWA](Lab_30%20XSS%20Vulnerabilities%20in%20DVWA.md) | Cross-Site Scripting vulnerability exploitation including stored and reflected XSS | Burp Suite, Firefox Dev Tools, DVWA |
| 31 | [Lab_31 File Upload Vulnerabilities in DVWA](Lab_31%20File%20Upload%20Vulnerabilities%20in%20DVWA.md) | File upload security testing and bypass techniques including magic byte validation | Burp Suite, file manipulation tools |
| 32 | [Lab_32 Security Misconfiguration in DVWA](Lab_32%20Security%20Misconfiguration%20in%20DVWA.md) | Identifying and exploiting common server misconfigurations, exposed files, and sensitive data | Burp Suite, directory scanners |
| 33 | [Lab_33 Insecure Direct Object References (IDOR) in DVWA](Lab_33%20Insecure%20Direct%20Object%20References%20(IDOR)%20in%20DVWA.md) | Authorization bypass through direct object reference manipulation | Burp Suite, parameter tampering tools |
| 34 | [Lab_34 Introduction to Web Technologies and Understanding HTTP and HTTPS](Lab_34%20Introduction%20to%20Web%20Technologies%20and%20Understanding%20HTTP%20and%20HTTPS.md) | HTTP/HTTPS fundamentals, request/response cycles, and protocol security | Burp Suite, curl, Firefox Dev Tools |
| 35 | [Lab_35 Web Development Components and Security Threats](Lab_35%20Web%20Development%20Components%20and%20Security%20Threats.md) | Understanding web architecture, component vulnerabilities, and common security threats | OWASP documentation, code review |
| 36 | [Lab_36 Introduction to Databases and SQL](Lab_36%20Introduction%20to%20Databases%20and%20SQL.md) | Database fundamentals, SQL queries, schema design, and database security principles | MySQL, SQLite, SQL clients |
| 37 | [Lab_37 Hands-on Secure Coding and Advanced Web Application Vulnerability Analysis](Lab_37%20Hands-on%20Secure%20Coding%20and%20Advanced%20Web%20Application%20Vulnerability%20Analysis.md) | Secure coding practices and advanced vulnerability assessment techniques | Code editors, static analysis tools |
| 38 | [Lab_38 Foundational Concepts: Number Systems, Date Formats, ASCII Codes, Disk Drives, and Computer Forensics](Lab_38%20Foundational%20Concepts:%20Number%20Systems,%20Date%20Formats,%20ASCII%20Codes,%20Disk%20Drives,%20and%20Computer%20Forensics.md) | Number systems (binary, hex, octal), ASCII, timestamps, and forensic fundamentals | hex editors, forensic tools |
| 39 | [Lab_39 Understanding Computer Systems, Forensics Challenges, and Storage Structures](Lab_39%20Understanding%20Computer%20Systems,%20Forensics%20Challenges,%20and%20Storage%20Structures%0AOverview.md) | Computer architecture, storage systems, file systems, and forensic investigation methodology | Autopsy, FTK, file system tools |
| 40 | [Lab_40 Exploring the Windows File System, Networking, and Batch Scripting](Lab_40%20Exploring%20the%20Windows%20File%20System,%20Networking,%20and%20Batch%20Scripting.md) | Windows file system structure, networking concepts, and batch scripting for system administration | CMD, PowerShell, batch scripts |
| 41 | [Lab_41 Introduction to Operating Systems and File Systems in Linux](Lab_41%20Introduction%20to%20Operating%20Systems%20and%20File%20Systems%20in%20Linux.md) | Linux OS architecture, file system hierarchy, inodes, and system processes | Linux tools, file system commands |
| 42 | [Lab_42 Creating a Simple Website and Capturing Network Traffic](Lab_42%20Creating%20a%20Simple%20Website%20and%20Capturing%20Network%20Traffic.md) | Building basic web servers and capturing HTTP traffic for analysis | Apache, nginx, Wireshark, tcpdump |
| 43 | [Lab_43 Capturing and Analyzing HTTP Traffic with Embedded Images](Lab_43%20Capturing%20and%20Analyzing%20HTTP%20Traffic%20with%20Embedded%20Images.md) | Advanced HTTP traffic analysis including multipart content and image requests | Wireshark, curl, web servers |
| 44 | [Lab_44 Packet Sniffing, Interception, DNS Spoofing, and ARP Poisoning](Lab_44%20Packet%20Sniffing,%20Interception,%20DNS%20Spoofing,%20and%20ARP%20Poisoning.md) | Network attack techniques including packet sniffing, DNS spoofing, and ARP poisoning | tcpdump, Wireshark, ettercap, dnsspoof |
| 45 | [Lab_45 Digital Forensics Investigation of a USB Drive](Lab_45%20Digital%20Forensics%20Investigation%20of%20a%20USB%20Drive.md) | USB drive forensic investigation, file recovery, and evidence preservation | Autopsy, FTK, dd, forensic tools |

---

## Lab Categories

### Linux & System Administration
- Lab 01: Linux OS Fundamentals
- Lab 24: User Management
- Lab 25: Linux File and Directory Permissions
- Lab 26: Strengthening System Security on Linux Servers
- Lab 40: Exploring the Windows File System, Networking, and Batch Scripting
- Lab 41: Introduction to Operating Systems and File Systems in Linux

### Scripting & Automation
- Lab 02: Bash Scripting Fundamentals
- Lab 40: Exploring the Windows File System, Networking, and Batch Scripting

### Network Security & Protocol Analysis
- Lab 03: Web Application Reconnaissance
- Lab 04: Wireshark Network Protocol Analysis
- Lab 05: Wireshark Case Study
- Lab 07: IoT & Enterprise Network Forensics
- Lab 09: DNS Query Tools and SMB Enumeration
- Lab 12: Basic Network Configuration
- Lab 13: IP Addressing and Subnetting
- Lab 14: Packet Analysis with Wireshark
- Lab 15: Network Security Policies and Risk Management
- Lab 16: Network Routing and VLAN Configuration
- Lab 17: Colonial Pipeline Case Study
- Lab 18: Intrusion Detection Systems (IDS)
- Lab 19: Network Security Protocols
- Lab 42: Creating a Simple Website and Capturing Network Traffic
- Lab 43: Capturing and Analyzing HTTP Traffic with Embedded Images
- Lab 44: Packet Sniffing, Interception, DNS Spoofing, and ARP Poisoning

### Cryptography & Secure Communication
- Lab 27: Comprehensive Journey into Secure Communication
- Lab 28: Post-Quantum Cryptography

### Web Application Security
- Lab 06: Burp Suite and OWASP ZAP
- Lab 20: Session Management Vulnerabilities
- Lab 21: Secure Session Management Practices
- Lab 22: Cross-Site Request Forgery (CSRF) Protection
- Lab 23: Session Hijacking CTF
- Lab 29: SQL Injection Testing
- Lab 30: XSS Vulnerabilities
- Lab 31: File Upload Vulnerabilities
- Lab 32: Security Misconfiguration
- Lab 33: Insecure Direct Object References (IDOR)
- Lab 34: Introduction to Web Technologies and HTTP/HTTPS
- Lab 35: Web Development Components and Security Threats
- Lab 36: Introduction to Databases and SQL
- Lab 37: Hands-on Secure Coding and Advanced Vulnerability Analysis

### Anonymity & Advanced Topics
- Lab 08: Reverse Shell via Netcat
- Lab 10: Anonymity Testing with Tor and Proxychains
- Lab 11: Password Cracking with John the Ripper

### Computer Forensics & Digital Investigation
- Lab 38: Foundational Concepts (Number Systems, Date Formats, ASCII, Disk Drives, Forensics)
- Lab 39: Understanding Computer Systems and Storage Structures
- Lab 45: Digital Forensics Investigation of a USB Drive

---

## Quick Navigation
- **Beginner Track**: Labs 01-05 (Linux & Network Basics)
- **Intermediate Track**: Labs 06-28 (Web Security & Cryptography)
- **Advanced Track**: Labs 29-45 (Vulnerabilities, Forensics & Attack Techniques)

