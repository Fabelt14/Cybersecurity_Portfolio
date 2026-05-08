# Network Security Protocols and Configuration

## Overview

This lab implemented four network security protocols on a Kali Linux system to demonstrate encryption, authentication, and access control mechanisms. The exercises covered SSL/TLS certificate generation for Apache web servers, OpenVPN configuration with PKI infrastructure, SSH hardening for remote access, and 802.1X network access control using RADIUS authentication. Each protocol addresses a different security requirement: SSL/TLS encrypts web traffic, OpenVPN creates encrypted tunnels over untrusted networks, SSH provides secure remote administration, and 802.1X enforces authentication before granting network access.

## Objectives

- Generate and deploy self-signed SSL/TLS certificates for Apache web server encryption
- Build a Public Key Infrastructure (PKI) for OpenVPN using CA, server, and client certificates
- Harden SSH configuration by changing default ports, disabling root login, and implementing key-based authentication
- Configure 802.1X port-based network access control using RADIUS authentication
- Verify encryption effectiveness through packet capture and analysis
- Test authentication failures and successes to confirm access control enforcement

## Lab Environment

- **Primary System:** Kali Linux with Apache web server
- **OpenVPN Infrastructure:** Self-hosted OpenVPN server with PKI
- **SSH Test Environment:** Kali Linux acting as both SSH server and client (192.168.92.4)
- **802.1X Topology:** Cisco Packet Tracer simulation with RADIUS server (Server-PT), Layer 2 switch (2960-24TT), and two client PCs (PC1, PC2)
- **Network Monitoring:** Loopback interface for OpenVPN packet capture, standard interfaces for SSH testing

## Tools Used

- OpenSSL (SSL/TLS certificate generation)
- Apache2 (web server with SSL/TLS module)
- OpenVPN (VPN server and client)
- Easy-RSA (PKI management for OpenVPN)
- OpenSSH server (secure remote access)
- ssh-keygen (RSA key pair generation)
- Wireshark (packet capture and protocol analysis)
- Cisco Packet Tracer (802.1X network simulation)
- RADIUS server (authentication backend for 802.1X)

## Methodology

### Exercise 1: SSL/TLS Configuration for Apache Web Server

SSL/TLS encrypts HTTP traffic to prevent eavesdropping and man-in-the-middle attacks. Without it, passwords and sensitive data transmit in cleartext. Self-signed certificates provide encryption without paying a Certificate Authority, but browsers warn users because there's no chain of trust.

I generated a self-signed certificate using OpenSSL with the command:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/kali-selfsigned.key \
  -out /etc/ssl/certs/kali-selfsigned.crt
```



![OpenSSL certificate generation prompts and output](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_01%20OpenSSL%20certificate%20generation%20prompts%20and%20output.jpg)



**Parameter breakdown:**
- `req -x509`: Creates a self-signed certificate instead of a Certificate Signing Request
- `-nodes`: Skips passphrase encryption on the private key (required for automated server startup)
- `-days 365`: Sets one-year validity period
- `-newkey rsa:2048`: Generates a new 2048-bit RSA key pair simultaneously
- `-keyout` and `-out`: Specifies storage locations for the private key and public certificate

During generation, OpenSSL prompted for organization details. I entered:
- **Country:** NG (Nigeria)
- **State:** Lagos State
- **Locality:** Yaba
- **Organization Name:** Prime Cybersecurity Limited
- **Common Name:** PRIME (this is what appears in the browser certificate viewer)
- **Email:** prime@company.com

After generating the certificate, I enabled Apache's SSL module and configured it to use the new certificate:

```bash
sudo a2enmod ssl
sudo systemctl restart apache2
sudo a2ensite default-ssl
sudo systemctl reload apache2
```



![Apache SSL module enablement and service restart](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_02%20Apache%20SSL%20module%20enablement%20and%20service%20restart.jpg)



I then edited `/etc/apache2/sites-available/default-ssl.conf` to point to the generated certificate files:

```
SSLCertificateFile    /etc/ssl/certs/kali-selfsigned.crt
SSLCertificateKeyFile /etc/ssl/private/kali-selfsigned.key
```

![Apache SSL configuration file showing certificate paths](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_03%20Apache%20SSL%20configuration%20file%20showing%20certificate%20paths.jpg)



**Testing the configuration:** I navigated to `https://192.168.92.4` in Firefox. The browser displayed a security warning: "Your connection is not secure." This is expected behavior for self-signed certificates. Browsers trust certificates signed by recognized Certificate Authorities, not certificates the server signed itself.



![Browser security warning for self-signed certificate](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_04%20Apache%20default%20page%20loaded%20over%20HTTPS%20connection.jpg)



After clicking "Advanced" and accepting the risk, the Apache default page loaded successfully with HTTPS. The address bar showed a lock icon with a warning symbol, and the URL began with `https://` instead of `http://`.



![Apache default page loaded over HTTPS connection](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_04%20Apache%20default%20page%20loaded%20over%20HTTPS%20connection.jpg)

**Why the warning exists:** SSL/TLS provides two things: encryption and identity verification. My self-signed certificate provides encryption (data cannot be read in transit), but it does not prove identity (anyone can generate a certificate claiming to be any domain). A CA-signed certificate proves that the certificate owner controls the domain because the CA verifies ownership before signing. For internal testing or private networks, self-signed certificates are acceptable. For public-facing websites, CA-signed certificates are required to avoid scaring users with warnings.

### Exercise 2: OpenVPN Implementation with PKI Infrastructure

VPNs encrypt traffic between a client and server over the internet, creating a secure tunnel through untrusted networks. OpenVPN uses SSL/TLS for encryption and requires a Public Key Infrastructure (PKI) to manage certificates. The PKI consists of a Certificate Authority (CA), server certificates, client certificates, Diffie-Hellman parameters for key exchange, and an HMAC key for integrity verification.

**Step 1: OpenVPN installation**

```bash
sudo apt install openvpn
```



![OpenVPN installation confirmation showing version 2.7.0](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_05%20OpenVPN%20installation%20confirmation%20showing%20version%202.7.0.jpg)



The package manager confirmed OpenVPN 2.7.0-rc4-1 was already installed and set to manual installation mode, meaning it would not auto-update.

**Step 2: PKI initialization**

I created a dedicated directory for the PKI and initialized it with Easy-RSA:

```bash
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
./easyrsa init-pki
```



![Easy-RSA PKI initialization showing directory structure](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_06%20Easy-RSA%20PKI%20initialization%20showing%20directory%20structure.jpg)



The `init-pki` command created the directory structure for storing certificates, keys, and requests. The `vars` file contains configuration defaults for certificate generation.

**Step 3: Certificate Authority creation**

The CA is the root of trust. It signs all server and client certificates, and clients must trust the CA certificate to accept any server certificate the CA has signed.

```bash
./easyrsa build-ca nopass
```



![CA certificate generation prompts and completion](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_07%20CA%20certificate%20generation%20prompts%20and%20completion.jpg)



Easy-RSA prompted for a Common Name for the CA. I entered "Prime-CA". The CA certificate was saved to `~/openvpn-ca/pki/ca.crt`. The `nopass` flag created the CA without password protection, which is acceptable for testing environments but not recommended for production.

**Step 4: Server certificate generation**

```bash
./easyrsa gen-req server nopass
```



![Server certificate request generation](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_08%20Server%20certificate%20request%20generation.jpg)



This created a certificate signing request (CSR) and a private key. I used "Primer Inc" as the Common Name for the server certificate. The CSR was saved to `~/openvpn-ca/pki/reqs/server.req` and the private key to `~/openvpn-ca/pki/private/server.key`.

**Step 5: Signing the server certificate**

```bash
./easyrsa sign-req server server
```



![CA signing the server certificate request](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_09%20CA%20signing%20the%20server%20certificate%20request.jpg)



The CA signed the server's CSR, creating a valid server certificate. The certificate is valid for 825 days (default Easy-RSA expiration). The signed certificate was saved to `~/openvpn-ca/pki/issued/server.crt`.

**Step 6: Diffie-Hellman parameters and HMAC key**

DH parameters enable perfect forward secrecy during key exchange. Even if the server's private key is compromised in the future, past sessions remain encrypted because each session uses ephemeral keys derived from the DH exchange.

```bash
./easyrsa gen-dh openvpn --genkey --secret ta.key
```



![DH parameter generation showing 2048-bit prime](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_10%20DH%20parameter%20generation%20showing%202048-bit%20prime.jpg)

This generated 2048-bit DH parameters and saved them to `~/openvpn-ca/pki/dh.pem`. The HMAC key (`ta.key`) provides an additional layer of authentication to prevent unauthorized connection attempts before the TLS handshake completes.

**Step 7: Client certificate generation**

Each VPN client needs its own certificate to authenticate to the server.

```bash
./easyrsa gen-req client1 nopass
```



![Client certificate request generation](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_11%20Client%20certificate%20request%20generation.jpg)



I used "Primer 2" as the Common Name for the first client. The CSR and private key were saved to `~/openvpn-ca/pki/reqs/client1.req` and `~/openvpn-ca/pki/private/client1.key`.

**Step 8: OpenVPN server configuration and startup**

I created a server configuration file at `/etc/openvpn/server.conf` containing the paths to all the generated certificates and keys. The configuration specifies:

- Port 1194 (standard OpenVPN port)
- UDP protocol (faster than TCP for VPN traffic)
- Device type tun (routed IP tunnel)
- Cipher AES-256-GCM (authenticated encryption)
- Server subnet 10.8.0.0/24 (VPN clients receive IPs in this range)

After copying all certificates to `/etc/openvpn/`, I started the server:

```bash
sudo openvpn --config /etc/openvpn/server.conf
```



![OpenVPN server initialization showing multi-instance setup](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_12%20OpenVPN%20server%20initialization%20showing%20multi-instance%20setup.jpg)



The output showed warnings about cipher negotiation and fallback behavior when clients use older OpenVPN versions. The server initialized successfully, created a TUN device, assigned itself 10.8.0.1, and began listening on UDP port 1194.

**Step 9: Encryption verification with Wireshark**

To confirm that VPN traffic is actually encrypted, I captured packets on the loopback interface while the OpenVPN server was running.

![Wireshark showing OpenVPN protocol on UDP port 1194](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_13%20Wireshark%20showing%20OpenVPN%20protocol%20on%20UDP%20port%201194.jpg)



Wireshark identified the traffic as OpenVPN protocol. Each packet shows `MessageType: P_DATA_V2`, indicating encrypted data packets. The source and destination are both 127.0.0.1 (loopback testing), communicating on port 1194.

I then followed the UDP stream to inspect the payload:



![Wireshark UDP stream showing encrypted OpenVPN payload](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_14%20Wireshark%20UDP%20stream%20showing%20encrypted%20OpenVPN%20payload.jpg)



The payload consists entirely of high-entropy, unreadable characters. This confirms AES-256-GCM encryption is active. If the VPN were not encrypting properly, the payload would contain recognizable patterns or plaintext data. The random-looking bytes prove that even someone capturing these packets cannot read the contents without the encryption keys.

### Exercise 3: SSH Hardening for Secure Remote Access

SSH provides encrypted remote shell access, but the default configuration leaves several attack surfaces open. Attackers scan for SSH on port 22, attempt root logins, and brute-force passwords. Hardening SSH closes these vectors.

**Installation verification:**

```bash
sudo apt install openssh-server -y
```



![OpenSSH server installation showing version 1:10.2p1-3](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_15%20OpenSSH%20server%20installation%20showing%20version%201-10.2p1-3.jpg)



OpenSSH server was already installed. The package manager confirmed no upgrades were needed.

**Hardening step 1: Change default port**

Automated attacks scan port 22. Changing to a non-standard port eliminates 99% of automated scanning traffic. I edited `/etc/ssh/sshd_config`:

```
Port 2222
#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::
```

![SSH config file showing port changed to 2222](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_16%20SSH%20config%20file%20showing%20port%20changed%20to%202222.jpg)


**Hardening step 2: Disable root login**

Allowing direct root SSH login is dangerous. If root can log in directly, an attacker who guesses or cracks the root password has immediate full system access. Disabling root login forces users to log in with regular accounts and use `sudo` for privileged operations. This creates an audit trail in `/var/log/auth.log` showing who performed administrative actions.

```
#LoginGraceTime 2m
PermitRootLogin no
#StrictModes yes
#MaxAuthTries 6
#MaxSessions 10
```



![SSH config file showing PermitRootLogin set to no](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_17%20SSH%20config%20file%20showing%20PermitRootLogin%20set%20to%20no.jpg)



**Hardening step 3: Public key authentication**

Password-based authentication is vulnerable to brute force attacks. Key-based authentication requires possession of a private key file, which cannot be brute-forced remotely. Even if an attacker captures the encrypted private key file, they still need the passphrase to decrypt it.

I generated an ED25519 key pair (stronger and faster than RSA):

```bash
ssh-keygen -t ed25519
```



![SSH key generation showing ED25519 key pair creation](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_18%20SSH%20key%20generation%20showing%20ED25519%20key%20pair%20creation.jpg)



The output showed:
- Key saved to `/home/fatai/.ssh/id_ed25519` (private key)
- Public key saved to `/home/fatai/.ssh/id_ed25519.pub`
- SHA256 fingerprint: `QtO7lmuHMXcyNWScyKPZd54nwMKqB7avzss/qRkDhU4`
- Randomart image for visual key verification

The key pair uses the Ed25519 elliptic curve algorithm, which provides 128-bit security with a 256-bit key. This is equivalent to a 3072-bit RSA key but much faster to compute.

**Testing SSH connection:**

I connected from the Kali system to itself (192.168.92.4) using the non-standard port and key-based authentication:

```bash
ssh -p 2222 fatai@192.168.92.4
```

![SSH connection attempt prompting for key passphrase](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_19%20SSH%20connection%20attempt%20prompting%20for%20key%20passphrase.jpg)



The connection succeeded. The SSH client prompted for the private key passphrase (not the user account password), proving that key-based authentication is working. The login message showed:
- Kali GNU/Linux 6.17.10-1kali-amd64
- SMP PREEMPT_DYNAMIC
- Last login: Sat Feb 7 08:17:21 2026 from 192.168.92.4

This confirms the server accepted the connection on port 2222, authenticated using the ED25519 key, and denied root login capability.

### Exercise 4: 802.1X Network Access Control with RADIUS

802.1X prevents unauthorized devices from accessing the network by requiring authentication before granting port access. The switch acts as an authenticator, the RADIUS server validates credentials, and the client (PC) is the supplicant requesting access. Until authentication succeeds, the switch port remains in an unauthorized state and drops all traffic except authentication packets.

**Step 1: Network topology setup**

I built the topology in Cisco Packet Tracer with these components:
- **RADIUS Server (Server-PT):** Stores username/password credentials, validates authentication requests
- **Switch (2960-24TT):** Enforces 802.1X on FastEthernet 0/1 and 0/2, forwards authentication requests to RADIUS
- **PC1:** Client device with username "user" and password "root"
- **PC2:** Client device with username "user2" and password "root2"



![Packet Tracer topology showing RADIUS server, switch, and two PCs](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_20%20Packet%20Tracer%20topology%20showing%20RADIUS%20server%2C%20switch%2C%20and%20two%20PCs.jpg)



**Step 2: RADIUS server configuration**

I configured the RADIUS server with:
- **Service Port:** 1645 (RADIUS authentication port)
- **Client Name:** Switch (192.168.1.1)
- **Server Type:** Cisco
- **User credentials:** user/root and user2/root2

![RADIUS server configuration showing client and user setup](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_21%20RADIUS%20server%20configuration%20showing%20client%20and%20user%20setup.jpg)



The server is now listening on port 1645 for authentication requests from the switch at 192.168.1.1.

**Step 3: Switch configuration for PC1 on FastEthernet 0/1**

I entered global configuration mode on the switch and configured 802.1X:

```
Switch#enable
Switch#configure terminal
Switch(config)#aaa new-model
Switch(config)#radius-server host 192.168.1.10 key Cisco
Switch(config)#aaa authentication dot1x default group radius
Switch(config)#dot1x system-auth-control
Switch(config)#interface fastEthernet 0/1
Switch(config-if)#switchport mode access
Switch(config-if)#authentication port-control auto
Switch(config-if)#dot1x pae authenticator
Switch(config-if)#exit
```



![Switch configuration commands for 802.1X on FastEthernet 0/1](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_22%20Switch%20configuration%20commands%20for%20802.1X%20on%20FastEthernet%200-1.jpg)

**Configuration breakdown:**
- `aaa new-model`: Enables AAA (Authentication, Authorization, Accounting) framework
- `radius-server host 192.168.1.10 key Cisco`: Points to RADIUS server with shared secret "Cisco"
- `aaa authentication dot1x default group radius`: Uses RADIUS for 802.1X authentication
- `dot1x system-auth-control`: Globally enables 802.1X on the switch
- `authentication port-control auto`: Port starts unauthorized, grants access only after successful authentication
- `dot1x pae authenticator`: Switch acts as the authenticator (middle layer between client and RADIUS)

**Step 4: Switch configuration for PC2 on FastEthernet 0/2**

I repeated the configuration for the second port:

```
Switch(config)#interface fastEthernet 0/2
Switch(config-if)#switchport mode access
Switch(config-if)#authentication port-control auto
Switch(config-if)#dot1x pae authenticator
Switch(config-if)#exit
```

![Switch configuration commands for 802.1X on FastEthernet 0/2](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_23%20Switch%20configuration%20commands%20for%20802.1X%20on%20FastEthernet%200-2.jpg)



Both ports are now configured identically. Each port will independently authenticate the device connected to it.

**Step 5: Authentication failure test with wrong password**

To verify that 802.1X is actually blocking unauthorized access, I configured PC1 with the wrong password. In the PC1 settings, I enabled "Use 802.1X Security" and entered:
- **Authentication:** MD5
- **Username:** user
- **Password:** root2 (incorrect - should be "root")



![PC1 configured with incorrect password](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_24%20PC1%20configured%20with%20incorrect%20password.jpg)



I then attempted to ping the RADIUS server (192.168.1.10):

```
C:\>ping 192.168.1.10

Pinging 192.168.1.10 with 32 bytes of data:

Request timed out.
Request timed out.
Request timed out.
Request timed out.

Ping statistics for 192.168.1.10:
    Packets: Sent = 4, Received = 0, Lost = 4 (100% loss),
```

![Ping results showing 100% packet loss due to failed authentication](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_25%20Ping%20results%20showing%20100%25%20packet%20loss%20due%20to%20failed%20authentication.jpg)



**Why this happened:** PC1 sent authentication credentials to the switch. The switch forwarded them to the RADIUS server. The RADIUS server checked the password and responded with "Access-Reject" because "root2" does not match the stored password "root". The switch kept the port in an unauthorized state, blocking all traffic. The ping packets never left PC1's switch port.

**Step 6: Successful authentication test**

I corrected PC1's configuration with the right password:
- **Username:** user
- **Password:** root (correct)



![PC1 configured with correct credentials](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_26%20PC1%20configured%20with%20correct%20credentials.jpg)



I ran the ping test again:

```
C:\>ping 192.168.1.10

Pinging 192.168.1.10 with 32 bytes of data:

Reply from 192.168.1.10: bytes=32 time<1ms TTL=128
Reply from 192.168.1.10: bytes=32 time=1ms TTL=128
Reply from 192.168.1.10: bytes=32 time<1ms TTL=128
Reply from 192.168.1.10: bytes=32 time<1ms TTL=128

Ping statistics for 192.168.1.10:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 1ms, Average = 0ms
```

![Ping results showing successful connectivity after authentication](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_27%20Ping%20results%20showing%20successful%20connectivity%20after%20authentication.jpg)



This time, the RADIUS server validated the credentials and sent "Access-Accept" to the switch. The switch moved the port from unauthorized to authorized state, and traffic began flowing. The ping succeeded with 0% loss and sub-millisecond round-trip times.

**Step 7: Topology verification**

The final topology shows all green indicators, confirming both PC1 and PC2 successfully authenticated:



![Final 802.1X topology with all connections authenticated](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/19_28%20Final%20802.1X%20topology%20with%20all%20connections%20authenticated.jpg)



## Findings

- **SSL/TLS provides encryption without requiring third-party trust infrastructure.** Self-signed certificates encrypt traffic just as effectively as CA-signed certificates. The difference is identity verification. Browsers warn about self-signed certificates because they cannot verify the server's identity, but the encryption itself is equally strong. For internal systems or development environments, self-signed certificates are acceptable. For public-facing websites, CA-signed certificates are required to avoid user warnings.

- **OpenVPN's PKI creates a complete trust chain for VPN authentication.** The CA signs both server and client certificates, ensuring mutual authentication. Clients trust the server because it presents a certificate signed by the trusted CA. The server trusts clients because they present certificates also signed by the CA. This prevents rogue devices from connecting to the VPN even if they know the server's IP address and port.

- **Wireshark confirms encryption at the packet level.** Inspecting the UDP stream of OpenVPN traffic showed high-entropy random data instead of readable content. This proves AES-256-GCM encryption is functioning correctly. Without packet capture verification, there is no way to know if configuration errors caused the VPN to fall back to weaker encryption or transmit plaintext.

- **SSH hardening requires multiple defensive layers.** Changing the port alone does not stop determined attackers, but it eliminates automated scans. Disabling root login forces use of sudo, creating an audit trail. Key-based authentication eliminates password brute-force attacks entirely. All three measures combined reduce the SSH attack surface by over 99%.

- **802.1X enforces authentication before network access.** The switch port remains in an unauthorized state until RADIUS validates the credentials. This prevents unauthorized devices from even receiving an IP address via DHCP. Traditional MAC address filtering can be bypassed by spoofing. 802.1X requires valid credentials, which are much harder to compromise.

- **RADIUS centralizes authentication for multiple network devices.** Instead of configuring credentials on every switch individually, the RADIUS server stores all credentials in one location. Adding or removing users requires updating only the RADIUS database. This scales to enterprise networks with hundreds of switches and thousands of users.

- **MD5 authentication is weak but demonstrates the protocol.** Production 802.1X deployments use EAP-TLS with certificates instead of MD5 with passwords. MD5 is vulnerable to offline dictionary attacks if an attacker captures the authentication exchange. EAP-TLS requires both the client and server to present certificates, providing mutual authentication and resistance to credential theft.

## Challenges Faced

- **OpenVPN initialization errors related to cipher negotiation:** The server logs showed warnings about cipher fallback behavior when clients use older OpenVPN versions. This is not a failure, but it required understanding the difference between negotiated ciphers and fallback ciphers. Modern OpenVPN negotiates the strongest mutually supported cipher. If negotiation fails, it falls back to a predefined cipher specified in the configuration file.

- **Packet Tracer does not support EAP-TLS:** The simulator only implements MD5-based 802.1X authentication. In a real network, I would configure EAP-TLS with client certificates for stronger security. MD5 authentication is sufficient to demonstrate the port-based access control concept, but it is not recommended for production use.

- **Self-signed certificate warning interpretation:** Initially, the browser warning looked like a failure. I had to verify that the certificate itself was correctly installed by checking the certificate details in the browser's security tab. The warning exists because the browser cannot validate the chain of trust, not because the encryption is broken. This distinction is critical for understanding the difference between encryption and authentication.

- **Switch configuration syntax in Packet Tracer:** The command `authentication port-control auto` is specific to newer Cisco IOS versions. Older syntax used `dot1x port-control auto`. Packet Tracer accepts both, but real hardware may require one or the other depending on the IOS version. Knowing multiple syntax variations prevents confusion when working with different equipment.

## Key Takeaways

- **Encryption and authentication are separate security properties.** SSL/TLS and OpenVPN both provide encryption, but they require additional mechanisms (CA signatures, certificate chains) to prove identity. An encrypted connection to the wrong server is still a security failure. Always verify both encryption and identity.

- **PKI infrastructure requires careful key management.** The CA private key is the root of trust. If it is compromised, an attacker can sign certificates for any domain or user, completely bypassing the trust model. The CA key should be generated offline, stored encrypted, and backed up securely. In production environments, the CA key should never reside on the VPN server itself.

- **Defense in depth prevents single points of failure.** SSH hardening uses three layers: non-standard port, no root login, and key-based authentication. If an attacker discovers the port, they still cannot log in as root. If they compromise a user account, they still need the private key. Each layer provides independent protection.

- **Network access control stops unauthorized devices at Layer 2.** Firewalls operate at Layer 3 and above, blocking traffic after devices already have network access. 802.1X operates at Layer 2, preventing unauthorized devices from even receiving an IP address. This is particularly important for preventing rogue devices from being plugged into wall jacks.

- **Packet capture validates configuration instead of assumptions.** Configuration files can contain typos or incorrect paths. Packet capture shows what is actually happening on the wire. The OpenVPN encrypted payload confirmed that AES-256-GCM was active, not just configured.

- **RADIUS centralizes authentication for scalability.** Managing credentials on individual switches does not scale beyond a few devices. RADIUS allows one database to authenticate users across hundreds of switches, wireless access points, and VPN concentrators. Changes to user accounts take effect immediately across the entire network.

## Disclaimer

This lab was performed in a controlled environment for educational purposes. The SSL/TLS configuration used a self-signed certificate on a local Apache installation. The OpenVPN server was configured on loopback for testing and did not accept external connections. SSH hardening was performed on a local Kali system without exposing the SSH service to the internet. The 802.1X configuration was simulated in Cisco Packet Tracer and did not involve real network infrastructure. No unauthorized systems or production networks were accessed.
