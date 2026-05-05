# ICDFA Internship Phase 1 - Lab Exercises

## Overview
Phase 1 of the ICDFA internship covers foundational cybersecurity skills including Linux fundamentals, scripting, web reconnaissance, network analysis, and web application security testing.

---

## Lab Summary Table

| # | Lab Name | Description | Tools Used |
|---|----------|-------------|-----------|
| 1 | [Lab_01 Linux OS Fundamentals](Lab_01%20Linux%20OS%20Fundamentals.md) | Command line basics, directory management, file permissions, and log analysis using the Linux terminal | mkdir, touch, chmod, ls, find, grep, wc, cat, tr |
| 2 | [Lab_02 Bash Scripting Fundamentals](Lab_02%20Bash%20Scripting%20Fundamentals.md) | Writing Bash scripts for automation including user input, conditionals, loops, functions, and system monitoring | nano, bash, chmod, date, free, ping, awk |
| 3 | [Lab_03 Web Application Reconnaissance](Lab_03%20Web%20Application%20Reconnaissance.md) | Information gathering on live domains and vulnerable web apps including IP resolution, WHOIS, DNS queries, subdomain enumeration, and directory discovery | ping, whois, nslookup, WhatWeb, Sublist3r, Knockpy, Dirb, Nmap, Nikto |
| 4 | [Lab_04 Wireshark Network Protocol Analysis](Lab_04%20Wireshark%20Network%20Protocol%20Analysis.md) | Packet capture and analysis including filtering, dissection, stream reconstruction, and threat identification in network traffic | Wireshark, Tshark, tcpdump |
| 5 | [Lab_05 Wireshark Case Study: Challenge](Lab_05%20Wireshark%20Case%20Study:%20Challenge.md) | Real-world packet analysis challenges and forensic investigation scenarios using captured PCAP files | Wireshark, Tshark |
| 6 | [Lab_06 Web Application Security Testing with Burp Suite and OWASP ZAP](Lab_06%20Web%20Application%20Security%20Testing%20with%20Burp%20Suite%20and%20OWASP%20ZAP.md) | Web vulnerability scanning and exploitation techniques using security testing frameworks | Burp Suite, OWASP ZAP, Firefox |
| 7 | [Lab_07 IoT & Enterprise Network Forensic Investigation Using Wireshark](Lab_07%20IoT%20&%20Enterprise%20Network%20Forensic%20Investigation%20Using%20Wireshark.md) | Advanced network forensics on IoT and enterprise network traffic including protocol analysis and threat detection | Wireshark, Tshark |
| 8 | [Lab_08 Reverse Shell via Netcat Using DVWA Command Execution](Lab_08%20Reverse%20Shell%20via%20Netcat%20Using%20DVWA%20Command%20Execution.md) | Exploiting command injection vulnerabilities to establish reverse shells and gain system access | Netcat (nc), DVWA, Bash |
| 9 | [Lab_09 DNS Query Tools and SMB Enumeration](Lab_09%20DNS%20Query%20Tools%20and%20SMB%20Enumeration.md) | DNS reconnaissance and SMB protocol enumeration to identify network resources and services | nslookup, dig, host, enum4linux, smbclient |
| 10 | [Lab_10 Anonymity Testing with Tor and Proxychains](Lab_10%20Anonymity%20Testing%20with%20Tor%20and%20Proxychains.md) | Testing anonymous network access and traffic routing through proxy chains | Tor, Proxychains, Firefox, curl |
| 11 | [Lab_11 Password Cracking with John the Ripper](Lab_11%20Password%20Cracking%20with%20John%20the%20Ripper.md) | Password hash cracking and dictionary/brute-force attack techniques | John the Ripper, hashcat, shadow files |
| 12 | [Lab_12 Basic Network Configuration: Setting Up and Troubleshooting](Lab_12%20Basic%20Network%20Configuration:%20Setting%20Up%20and%20Troubleshooting.md) | Network interface configuration, IP assignment, routing, and network diagnostics | ifconfig, ip, route, ping, traceroute, netstat |
| 13 | [Lab_13 Understanding IP Addressing: IPv4 and IPv6 Classification and Subnetting](Lab_13%20Understanding%20IP%20Addressing:%20IPv4%20and%20IPv6%20Classification%20and%20Subnetting.md) | IP address classes, CIDR notation, subnetting, and IPv6 addressing schemes | ipcalc, ip, subnetting calculators |
| 14 | [Lab_14 Packet Analysis with Wireshark: Capture, Filter, and Analyze Network Traffic](Lab_14%20Packet%20Analysis%20with%20Wireshark:%20Capture,%20Filter,%20and%20Analyze%20Network%20Traffic.md) | Comprehensive packet analysis covering capture, filtering, stream follow, and protocol dissection | Wireshark, Tshark, tcpdump |
| 15 | [Lab_15 Network Security Policies and Risk Management](Lab_15%20Network%20Security%20Policies%20and%20Risk%20Management.md) | Security policy development, risk assessment frameworks, and security governance | Documentation tools, Policy templates |
| 16 | [Lab_16 Simulating Network Routing and VLAN Configuration in Linux](Lab_16%20Simulating%20Network%20Routing%20and%20VLAN%20Configuration%20in%20Linux.md) | Virtual LAN setup, routing table manipulation, and network simulation | ip, vconfig, route, brctl, Cisco packet tracer |
| 17 | [Lab_17 Applied Network Security Analysis: Colonial Pipeline Ransomware Case Study](Lab_17%20Applied%20Network%20Security%20Analysis:%20Colonial%20Pipeline%20Ransomware%20Case%20Study.md) | Real-world case study analysis of ransomware attack indicators and network forensics | Wireshark, threat intelligence tools |
| 18 | [Lab_18 Intrusion Detection Systems (IDS) and Traffic Analysis on Linux](Lab_18%20Intrusion%20Detection%20Systems%20(IDS)%20and%20Traffic%20Analysis%20on%20Linux.md) | IDS/IPS deployment and configuration with Snort/Suricata for network threat detection | Snort, Suricata, tcpdump, Wireshark |
| 19 | [Lab_19 Network Security Protocols and Configuration](Lab_19%20Network%20Security%20Protocols%20and%20Configuration.md) | Secure protocol implementation including TLS/SSL, SSH, IPSec, and VPN configuration | OpenSSL, SSH, IPSec tools, OpenVPN |
| 20 | [Lab_20 Session Management Vulnerabilities](Lab_20%20Session%20Management%20Vulnerabilities.md) | Identifying and exploiting session hijacking, fixation, and management flaws in web applications | Burp Suite, DVWA, session analysis tools |
| 21 | [Lab_21 Secure Session Management Practices](Lab_21%20Secure%20Session%20Management%20Practices.md) | Implementation of secure session tokens, cookie flags, and session timeout mechanisms | Web frameworks, cookie analysis tools |
| 24 | [Lab_24 User Management](Lab_24%20User%20Management.md) | Linux user account creation, group management, sudo configuration, and access control | useradd, userdel, groupadd, usermod, visudo |
| 25 | [Lab_25 Linux File and Directory Permissions](Lab_25%20Linux%20File%20and%20Directory%20Permissions.md) | Understanding and configuring file/directory permissions, ownership, and access control | chmod, chown, chgrp, ls, find |
| 26 | [Lab_26 Strengthening System Security on Linux Servers](Lab_26%20Strengthening%20System%20Security%20on%20Linux%20Servers.md) | Security hardening techniques, firewall rules, SELinux policies, and server lockdown procedures | iptables, firewalld, SELinux, auditd, fail2ban |
| 27 | [Lab_27 Comprehensive Journey into Secure Communication](Lab_27%20Comprehensive%20Journey%20into%20Secure%20Communication.md) | Encryption, digital signatures, public key cryptography, and secure messaging implementations | OpenSSL, GnuPG (GPG), SSH, TLS, cryptographic libraries |
| 28 | [Lab_28 Post-Quantum Cryptography and Future Proofing Security](Lab_28%20Post-Quantum%20Cryptography%20and%20Future%20Proofing%20Security.md) | Post-quantum encryption algorithms and preparing security systems for quantum computing threats | liboqs, Kyber, Dilithium, NIST PQC candidates |
| 29 | [Lab_29 Manual and Automated SQL Injection Testing](Lab_29%20Manual%20and%20Automated%20SQL%20Injection%20Testing.md) | SQL injection vulnerability detection and exploitation in web applications | SQLmap, Burp Suite, DVWA, SQL command tools |
| 30 | [Lab_30 XSS Vulnerabilities in DVWA](Lab_30%20XSS%20Vulnerabilities%20in%20DVWA.md) | Cross-Site Scripting vulnerability exploitation including stored and reflected XSS | Burp Suite, Firefox DevTools, DVWA, BeEF |
| 31 | [Lab_31 File Upload Vulnerabilities in DVWA](Lab_31%20File%20Upload%20Vulnerabilities%20in%20DVWA.md) | File upload security testing and bypass techniques including magic byte validation | Burp Suite, DVWA, file analysis tools |
| 32 | [Lab_32 Security Misconfiguration in DVWA](Lab_32%20Security%20Misconfiguration%20in%20DVWA.md) | Identifying and exploiting common server misconfigurations, exposed files, and sensitive data | Burp Suite, Nikto, DVWA, directory enumeration tools |
| 33 | [Lab_33 Insecure Direct Object References (IDOR) in DVWA](Lab_33%20Insecure%20Direct%20Object%20References%20(IDOR)%20in%20DVWA.md) | Authorization bypass through direct object reference manipulation in web applications | Burp Suite, DVWA, parameter tampering tools |

---

## Lab Categories

### Linux & System Administration
- Lab 01: Linux OS Fundamentals
- Lab 24: User Management
- Lab 25: Linux File and Directory Permissions
- Lab 26: Strengthening System Security on Linux Servers

### Scripting & Automation
- Lab 02: Bash Scripting Fundamentals

### Network Security
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

### Cryptography & Secure Communication
- Lab 27: Comprehensive Journey into Secure Communication
- Lab 28: Post-Quantum Cryptography

### Web Application Security
- Lab 06: Burp Suite and OWASP ZAP
- Lab 20: Session Management Vulnerabilities
- Lab 21: Secure Session Management Practices
- Lab 29: SQL Injection Testing
- Lab 30: XSS Vulnerabilities
- Lab 31: File Upload Vulnerabilities
- Lab 32: Security Misconfiguration
- Lab 33: Insecure Direct Object References (IDOR)

### Anonymity & Advanced Topics
- Lab 08: Reverse Shell via Netcat
- Lab 10: Anonymity Testing with Tor and Proxychains
- Lab 11: Password Cracking with John the Ripper

---
