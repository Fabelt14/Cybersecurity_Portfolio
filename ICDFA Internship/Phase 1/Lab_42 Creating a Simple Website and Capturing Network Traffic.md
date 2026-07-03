# Creating a Simple Website and Capturing Network Traffic

## Overview

This lab had two goals running in parallel. The first was to build and host a basic HTML website on Apache running locally on Kali Linux. The second was to capture the network traffic that website generates using Wireshark, then compare what local traffic looks like against traffic going to an external site. Together, both parts show how HTTP communication works at the packet level, from the TCP handshake down to the raw MAC and IP addresses involved.

## Objectives

- Write and deploy a basic HTML page on an Apache web server
- Configure correct file ownership and permissions for web server access
- Capture TCP traffic on the loopback interface using Wireshark
- Identify TCP three-way handshake packets (SYN, SYN-ACK, ACK)
- Extract source/destination ports, IP addresses, sequence numbers, and timestamps from packet captures
- Compare local loopback traffic (127.0.0.1) against external internet traffic
- Observe how MAC addresses appear in external traffic but not loopback traffic

## Lab Environment

- **Machine:** Kali Linux
- **Web Server:** Apache2 (hosted on localhost)
- **Local Target:** http://localhost/index.html
- **External Target:** http://shinyfreshmajesticsmile.neverssl.com/online/
- **Network Interfaces Used:** Loopback (lo) for local traffic, eth0 for external traffic

## Tools Used

- nano (HTML file creation)
- Apache2 (web server)
- systemctl (service management)
- chown / chmod (file ownership and permissions)
- Wireshark (packet capture and analysis)
- Firefox browser

## Methodology

### Step 1: Building the HTML Page

I created the website content in nano and saved it as index.html. The page introduced me as a penetration testing student with a brief summary of completed lab work.

![nano editor showing index.html content](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/42_01%20nano%20editor%20showing%20index.html%20content.jpg)

The HTML included a heading, two paragraph tags, and an unordered list of completed work. Nothing complex, the point was to have a working page that Apache could serve so Wireshark would have real HTTP traffic to capture.

---

### Step 2: Deploying the Page on Apache

Apache only serves files from its document root, which on Kali Linux is `/var/www/html/`. Placing the file anywhere else returns a 404. I copied index.html into that directory and confirmed it landed correctly:

```bash
sudo cp index.html /var/www/html/
ls /var/www/html/
```

![File copy and directory listing showing index.html in /var/www/html/](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/42_02%20File%20copy%20and%20directory%20listing%20showing%20index.html%20in%20var-www-html.jpg)


- **File ownership:** Apache runs as the `www-data` user. If the file is owned by root or any other user, Apache can still read it because of the permissions set, but the correct practice is to set ownership explicitly so the web server has proper authority over its own files.

```bash
sudo chown www-data:www-data /var/www/html/index.html
```

- **File permissions:** I set 644, which gives the owner (www-data) read and write access, while group and others get read-only. Apache only needs to read the file to serve it. Write access for the web server user is intentionally limited to prevent a compromised Apache process from modifying its own files.

```bash
sudo chmod 644 /var/www/html/index.html
```

Running `ls -l` confirmed the result: `-rw-r--r-- 1 www-data www-data 652 Jun 11 12:26 /var/www/html/index.html`


**Why 644 and not 755?** Executable permission on a static HTML file is unnecessary and slightly expands the attack surface. If an attacker uploads a malicious script, executable permissions would let the server run it. 644 removes that possibility entirely.

---

### Step 3: Starting Apache

```bash
sudo systemctl start apache2
sudo systemctl enable apache2
```

![systemctl start and enable output for apache2](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/42_03%20systemctl%20start%20and%20enable%20output%20for%20apache2.jpg)

`start` turns the service on immediately. `enable` creates a systemd symlink so Apache restarts automatically after a reboot. Without `enable`, Apache stops the moment the machine powers off and must be started manually every session.

I opened Firefox, navigated to http://localhost, and the page loaded correctly.

![Firefox showing the hosted website at http://localhost](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/42_04%20Firefox%20showing%20the%20hosted%20website%20at%20http%20localhost.jpg)

---

### Step 4: Capturing Local Traffic with Wireshark

With the site confirmed working, I opened Wireshark and selected the loopback interface (lo) before navigating to the page. The loopback interface handles all traffic where both source and destination are 127.0.0.1. Since the browser and the Apache server are both running on the same machine, all communication goes through lo, not eth0. After starting the capture, I navigated to http://localhost/index.html in Firefox. Wireshark immediately started logging packets.

![Wireshark packet list showing TCP and HTTP traffic to localhost](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/42_05%20Wireshark%20packet%20list%20showing%20TCP%20and%20HTTP%20traffic%20to%20localhost.jpg)

---

### Step 5: Analyzing the TCP Three-Way Handshake

The first three packets in the capture show the TCP handshake before any HTTP data moves:

- **Packet 1 (SYN):** Browser sends a connection request to port 80. Sequence number is 0 (relative), meaning this is the opening move with no prior data sent.

- **Packet 2 (SYN-ACK):** Apache acknowledges the connection request and sends its own SYN. Both sides are now synchronizing.

- **Packet 3 (ACK):** Browser acknowledges Apache's SYN. The connection is now established and HTTP can begin.

![Wireshark showing packets 27, 28, 29 as the SYN, SYN-ACK, ACK handshake](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/42_06%20Wireshark%20showing%20packets%2027%2C%2028%2C%2029%20as%20the%20SYN%2C%20SYN-ACK%2C%20ACK%20handshake.jpg)

**Packet details extracted:**

Clicking into the ACK packet (packet 29) revealed:

- **Source IP:** 127.0.0.1
- **Destination IP:** 127.0.0.1
- **Source Port:** 46414 (browser's randomly assigned ephemeral port)
- **Destination Port:** 80 (Apache's listening port)
- **Sequence Number:** 1 (relative)
- **Acknowledgment Number:** 1 (relative)
- **Timestamp:** June 12, 2026 at 14:48:37 UTC (2:48 PM)

![Wireshark frame details showing timestamp 14:48:37 UTC](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/42_09%20Wireshark%20frame%20details%20showing%20timestamp%2015-36-58%20UTC.jpg)

**Why source and destination are both 127.0.0.1:** The browser and Apache are both processes on the same machine. When you connect to localhost, the operating system routes the packet through the loopback interface internally. No physical network card is involved. The traffic never leaves the machine, so both endpoints share the same IP address.

---

### Step 6: Repeating the Capture for an External Site

To compare local loopback traffic against real internet traffic, I switched Wireshark to capture on eth0 and navigated to http://shinyfreshmajesticsmile.neverssl.com/online/. This site intentionally serves over plain HTTP (no SSL) to make it useful for traffic analysis exercises.

**Packet details extracted from external traffic:**

- **Source Port:** 58406 (browser ephemeral port)
- **Destination Port:** 443 (HTTPS, despite the site being HTTP, the initial connection attempt tried 443 first)
- **Sequence Number:** 0 (first SYN packet, as expected)
- **Timestamp:** June 12, 2026 at 15:36:58 UTC (3:36 PM)


![Wireshark packet details showing source port 58406, destination port 443, sequence number 0](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/42_08%20Wireshark%20packet%20details%20showing%20source%20port%2058406%2C%20destination%20port%20443%2C%20sequence%20number%200.jpg)

![Wireshark frame details showing timestamp 15:36:58 UTC](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/42_09%20Wireshark%20frame%20details%20showing%20timestamp%2015-36-58%20UTC.jpg)


**MAC and IP addresses for external traffic:**

- **Source MAC:** c8:f7:33:67:d6:17 (Intel NIC on the Kali machine)
- **Destination MAC:** 52:55:0a:00:02:02 (default gateway/router)
- **Source IP:** 10.0.2.15 (Kali machine's eth0 address)
- **Destination IP:** 34.223.124.45 (remote web server)


![Wireshark Ethernet and IP layer showing source and destination MAC addresses and IPv4 addresses](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/42_10%20Wireshark%20Ethernet%20and%20IP%20layer%20showing%20source%20and%20destination%20MAC%20addresses%20and%20IPv4%20addresses.jpg)


**Why MAC addresses appear here but not in local traffic:** MAC addresses operate at Layer 2 (Data Link). When traffic goes to 127.0.0.1, it never reaches Layer 2 because the operating system handles it entirely within the kernel loopback interface, no Ethernet framing needed. When traffic leaves the machine toward an external IP, it must be wrapped in an Ethernet frame with source and destination MAC addresses so the router knows where to forward it. The destination MAC is the router's MAC (the next hop), not the final server's MAC, because MAC addresses only travel one hop at a time.

## Findings

- **Apache requires correct file placement and ownership to serve pages.** The web root `/var/www/html/` is the only directory Apache serves by default. File ownership set to `www-data` and permissions set to 644 give Apache read access while preventing the server process from modifying its own files, which limits the damage a compromised web server can do.

- **The TCP three-way handshake is visible in every new connection.** Packets 27, 28, and 29 in the local capture showed SYN, SYN-ACK, and ACK in sequence before any HTTP data moved. This handshake takes three round trips before the client can send the first GET request, which is why connection latency matters for web performance.

- **Local loopback traffic uses 127.0.0.1 for both source and destination.** The browser and Apache server are both processes on the same machine. Traffic between them never leaves the kernel, so both endpoints share the same IP address and no MAC addressing is needed.

- **External traffic exposes the full network stack.** Capturing traffic to the external site showed the Kali machine's real IP (10.0.2.15), the remote server's IP (34.223.124.45), the machine's Intel NIC MAC address (c8:f7:33:67:d6:17), and the router's MAC address (52:55:0a:00:02:02) as the next hop. Each layer of the OSI model adds its own header, and Wireshark shows all of them.

- **Sequence numbers start at 0 on the first SYN.** In the external traffic capture, the SYN packet had sequence number 0. This is the opening value before any data has been transmitted. Wireshark displays relative sequence numbers by default to make them easier to read, as raw sequence numbers are 32-bit random values.

## Challenges Faced

- **File served with wrong ownership initially:** After copying index.html to `/var/www/html/`, I initially skipped the chown step. The page still loaded because the default permissions on the file allowed world-read. I ran the chown command after reviewing the correct practice. The lesson is that just because something works does not mean it is configured correctly.

- **Selecting the right interface in Wireshark:** The loopback interface (lo) is not always listed first in Wireshark's interface selector. On the first attempt, I accidentally started capturing on eth0, which showed no traffic when I navigated to localhost. Once I selected lo specifically, the packets appeared immediately. Interface selection is the first thing to verify when a capture shows no results.

## Key Takeaways

- **Knowing where Apache serves files from is basic but non-negotiable.** Every web server has a document root. Placing files outside that root, without additional configuration, means the server cannot find them. This is also a security concern: a misconfigured document root that points to a sensitive directory could expose system files to anyone with a browser.

- **File permissions on a web server are a security control.** 644 for static files means Apache can read and serve the file but cannot write to it. If an attacker exploits a vulnerability in Apache, the 644 permission prevents that process from overwriting the file with malicious content. Permissions are not just for organization; they are a line of defense.

- **The TCP handshake is the foundation of every HTTP connection.** Before a single byte of web content moves, three packets must complete. Understanding this handshake is how you diagnose failed connections. If you see SYN but no SYN-ACK, the server is not responding. If you see SYN-ACK but no ACK, there is a routing problem on the return path.

- **Loopback traffic and external traffic follow different paths through the OS.** Loopback skips the physical network entirely and stays inside the kernel. External traffic goes through the NIC, gets Ethernet framing with MAC addresses, and leaves the machine. Wireshark captures both, but the data present in each packet reflects how far down the stack the packet actually traveled.

- **MAC addresses identify the next hop, not the final destination.** In the external capture, the destination MAC was the router's address, not the web server's address. The router strips that MAC, looks up the destination IP, and rewrites the Ethernet frame with the next router's MAC. This happens at every hop across the internet. IP addresses stay constant across the entire journey. MAC addresses change at every hop.

## Disclaimer

This lab was performed in a controlled environment on a local machine running Kali Linux. The external website accessed (shinyfreshmajesticsmile.neverssl.com) is a publicly available test site designed for traffic capture exercises. No unauthorized systems were accessed or tested.
