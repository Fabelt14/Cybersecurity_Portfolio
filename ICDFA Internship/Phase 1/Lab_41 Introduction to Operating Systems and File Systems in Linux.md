# Introduction to Operating Systems and File Systems in Linux

## Overview

This lab covered the foundational concepts of Linux as used in digital forensics work. The exercises moved through ten areas: understanding Linux's role in forensics, the Virtual File System layer, path variables, essential CLI commands, file searching, networking tools, file management, shell scripting, and package management. The goal was to build practical familiarity with how Linux organizes and manages data, which directly supports forensic investigation work.

## Objectives

- Understand why Linux is the preferred operating system for digital forensics
- Explain how the Virtual File System (VFS) allows Linux to interact with multiple file system types
- Navigate and explain the Linux hierarchical directory structure
- Configure and modify the PATH environment variable
- Create, copy, delete, and search for files using CLI commands
- Use networking tools to gather system and network intelligence
- Compress and decompress archives for evidence packaging
- Write and execute a basic shell script for system information gathering
- Install and update software using the APT package manager

## Lab Environment

- **Machine:** Kali Linux (kernel 6.17.10+kali1, x86_64)
- **RAM:** 1.9 GiB total, 935 MiB used
- **Storage:** 79 GB primary disk (/dev/sda1), 41% used
- **Network:** eth0 (10.0.2.15, NAT), lo (127.0.0.1, loopback)
- **Shell:** Bash

## Tools Used

- ifconfig (network interface inspection)
- ping (connectivity testing)
- traceroute (network path tracing)
- nmap (port scanning)
- netcat (TCP connection testing)
- wget (command-line file download)
- find (file searching)
- zip / unzip (archive management)
- bash (shell scripting)
- apt (package management)

## Methodology

### Part 1: Linux for Digital Forensics

Before touching a terminal, the lab started with a conceptual question: why Linux specifically for forensics?

**The core reason is control.** Linux is open-source, which means every tool an investigator uses can be audited down to its source code. In a court case, opposing counsel can challenge whether a tool modified the evidence it was analyzing. With Linux and open-source forensic utilities, the analyst can prove exactly how the tool works and that it made no unauthorized changes to the data.

The command-line interface matters too. GUI tools abstract away what is actually happening to data. CLI tools like `dd` copy raw bytes directly from storage media, sector by sector, without the operating system interpreting or caching anything. That raw access is what forensic imaging requires.

**The honest challenges:** Linux struggles with closed, proprietary file systems. Apple's APFS uses encryption and metadata structures that Linux tools do not always parse correctly. An analyst investigating a recent MacBook may need specialized tools or a macOS environment for certain tasks. The CLI also requires training. A single wrong flag on a write operation can corrupt evidence. The skill floor is real.

| Advantage | Challenge |
|---|---|
| Open-source tools allow code-level verification of evidence handling | Closed-source and encrypted file systems (like APFS) are not always fully supported |
| Powerful CLI utilities enable automation, text searching, and raw data access | CLI-heavy environment increases the risk of user error without proper training |

### Part 2: Virtual File System and Directory Structure

**What the VFS does:** When an application calls `open()` or `read()`, it does not know or care whether the file sits on an EXT4 partition, an NTFS drive, or a FAT32 USB stick. The Virtual File System layer in the Linux kernel receives that generic call and translates it into the specific instructions each underlying file system understands. The application talks to one consistent interface. The VFS handles the translation underneath.

This matters in forensics because investigators regularly mount evidence drives with file systems from Windows (NTFS), older cameras (FAT32), or network-attached storage (XFS). The VFS is why the same Linux tools work across all of them without modification.

**Directory structure:** Linux organizes everything under a single root directory (`/`). There are no drive letters. Every file, device, and mounted volume lives somewhere in this single tree.

![Linux file system hierarchy diagram showing root and subdirectories](screenshots/linux_filesystem_hierarchy.png)



Key directories relevant to forensic work:

`/home` stores each user's personal files and application preferences. On a compromised machine, this is where investigators look for browser history, downloaded files, and user-created documents.

`/etc` holds system-wide configuration files in plain text. The `/etc/passwd` file lists every user account. `/etc/shadow` stores password hashes. `/etc/cron.d` shows scheduled tasks. Attackers who maintain persistence often modify files here.

`/usr` contains installed programs, libraries, and documentation. Malware sometimes installs itself here to appear as a legitimate application.

`/var` holds data that changes during system operation: logs in `/var/log`, mail spools, and database files. Forensic log analysis starts here.

### Part 3: PATH Variable

The PATH variable tells the shell where to look for executable programs. When a user types `ls`, the shell does not search the entire filesystem. It checks only the directories listed in PATH, in order, until it finds a matching executable.

**Viewing the current PATH:**

```bash
echo $PATH
```



![echo $PATH output showing current directory list](screenshots/echo_path_output.png)



The output showed a colon-separated list of directories:
`/home/fatai/.local/bin:/usr/local/sbin:/usr/local/sbin:/sbin:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games:/home/fatai/.dotnet/tools`

**Adding a new directory temporarily:**

```bash
export PATH=$PATH:/home/fatai/scripts
```



![echo $PATH output after export showing new scripts directory appended](screenshots/echo_path_after_export.png)

The updated PATH now includes `/home/fatai/scripts` at the end. Any executable placed in that directory can now be run by name without typing the full path. The `export` keyword makes this change available to any child processes spawned from the current shell session.

**Why "temporarily" matters:** This change only persists for the current session. Once the terminal closes, PATH reverts to its default. To make a path change permanent, it must be written to `~/.bashrc` or `~/.profile`. This is an important forensic detail. An attacker who modified PATH temporarily would not leave a trace in those configuration files, making the change harder to detect after a reboot.

### Part 4: Essential File Operations

**Creating the directory and files:**

```bash
mkdir ~/forensics_lab
touch ~/forensics_lab/file1.txt ~/forensics_lab/file2.txt
```

![mkdir and touch commands creating forensics_lab directory with two files](screenshots/create_forensics_lab.png)



`mkdir` creates the directory. `touch` creates empty files or updates the timestamp of existing files. Running `ls ~/forensics_lab` confirmed both files exist: `file1.txt` and `file2.txt`.

**Copying and deleting:**

```bash
cp ~/forensics_lab/file1.txt ~/forensics_lab/file1_copy.txt
rm ~/forensics_lab/file2.txt
ls ~/forensics_lab
```



![cp and rm commands, then ls showing file1_copy.txt and file1.txt remaining](screenshots/copy_delete_verify.png)



After copying file1.txt and deleting file2.txt, `ls` showed only `file1_copy.txt` and `file1.txt`. The deletion was permanent. Unlike a GUI recycle bin, `rm` does not move files to a recoverable location by default. In a forensic context, this is why deleted file recovery requires specialized tools that read raw disk sectors rather than the file system index.

### Part 5: File Searching

```bash
find ~/forensics_lab -name "*.txt"
```



![find command output showing full paths of file1.txt and file1_copy.txt](screenshots/find_txt_files.png)



The output returned two results:
```
/home/fatai/forensics_lab/file1.txt
/home/fatai/forensics_lab/file1_copy.txt
```

`find` searches recursively through the specified directory and all subdirectories. The `-name "*.txt"` filter matches any filename ending in `.txt`. The output shows full absolute paths, not just filenames.

In a real forensic investigation, this command scales to entire drive mounts. Searching for `*.docx`, `*.pdf`, or specific filenames across a mounted evidence image helps locate documents of interest without opening a file browser. Combining `find` with `-mtime` filters files modified within a specific time window, which is useful for building a timeline of attacker activity.

### Part 6: Networking Commands

**ifconfig - Network interface inspection:**

```bash
ifconfig
```



![ifconfig output showing eth0 with IP 10.0.2.15 and loopback interface](screenshots/ifconfig_output.png)



The output showed eth0 configured with IP address 10.0.2.15 and netmask 255.255.255.0. That netmask means the full internal network range is 10.0.2.0 to 10.0.2.255. From an attacker's perspective, this reveals exactly which IP range to feed into a network scanner to find other machines. From an investigator's perspective, finding this on a compromised machine tells you what network it was on and which other hosts it could have communicated with.

**ping - Connectivity testing:**

```bash
ping -c 5 google.com
```



![ping output showing 5 packets sent, 0 packet loss, round-trip times around 22-25ms](screenshots/ping_google.png)

Five packets sent, five received, 0% packet loss. Round-trip times averaged around 24ms. This confirms stable connectivity between the machine and Google's servers. In a forensic context, ping confirms whether a suspect machine had active internet access at the time of investigation.

**traceroute - Path mapping:**

```bash
traceroute google.com
```



![traceroute output showing first hop at 10.0.2.2 then multiple * * * hops](screenshots/traceroute_google.png)



The first hop responded at 10.0.2.2 (the VM's gateway), then subsequent hops returned `* * *` (no response). Routers that do not respond to traceroute probes are configured to drop ICMP packets, which is common on ISP infrastructure. Network administrators use traceroute to pinpoint exactly where a connection fails. If hop 3 is the last responding hop, the problem is between that router and the next one downstream.

**nmap - Port scanning:**

```bash
nmap 127.0.0.1 -p 21
```



![nmap output showing port 21/tcp closed ftp](screenshots/nmap_port21.png)



Port 21 (FTP) returned a `closed` state. Closed means the host is reachable and actively refusing connections because no FTP service is running. This is different from `filtered`, which means a firewall is blocking the probe entirely. To start an FTP service, a system administrator would run `systemctl start vsftpd`. During a penetration test, finding an open FTP port on a server prompts further investigation into anonymous login access and version-specific exploits.

**netcat - TCP connection testing:**

```bash
nc -l -p 12345
```



![Two terminal windows showing netcat listener on left and client connection on right with 'ls' and 'testing connection' data transfer](screenshots/netcat_tcp_test.png)

One terminal ran the listener (`nc -l -p 12345`). A second terminal connected to it (`nc 127.0.0.1 12345`). Text typed in one terminal appeared in the other, confirming the TCP connection worked. Netcat has two primary uses in security work: testing whether a firewall allows traffic through a specific port, and transferring data between machines without setting up a full file transfer service. Attackers also use netcat to establish reverse shells, which is why security tools flag it as a potentially dangerous utility.

**wget - Command-line file download:**

```bash
wget https://pbs.twimg.com/media/DulILzQXcAAkFMV.jpg
```



![wget output showing download progress, 159 KB/s speed, file saved successfully](screenshots/wget_download.png)



The file downloaded at 159 KB/s and saved to the current directory. wget works in environments without a graphical browser and supports resuming interrupted downloads with the `-c` flag. In forensics, wget is used to pull tools or evidence files from a remote server during an investigation. Attackers also use it to download malware payloads onto compromised systems, so wget entries in bash history are worth examining.

### Part 7: File Compression and Archive Management

```bash
zip -r forensics_lab.zip ~/forensics_lab
```



![zip command output showing forensics_lab/ and both txt files being added](screenshots/zip_forensics_lab.png)



The `-r` flag compresses recursively, including all files inside the directory. The output confirmed three items were added: the directory itself and both text files.

```bash
mkdir ~/uncompressed_lab
unzip forensics_lab.zip -d ~/uncompressed_lab
ls ~/uncompressed_lab
```



![unzip output showing files extracting, then ls confirming forensics_lab directory present](screenshots/unzip_verify.png)



Running `ls ~/uncompressed_lab/forensics_lab/` confirmed both `file1.txt` and `file1_copy.txt` were restored correctly. The directory structure was preserved inside the archive.

Compression matters in forensics for two reasons. First, evidence archives need to travel between investigators and courts. Compression reduces transfer size. Second, file archives preserve directory structure and timestamps in a single portable file. Running a hash on the zip file before and after transfer confirms it was not modified in transit.

### Part 8: Shell Script for System Information

Shell scripts automate sequences of commands that would be tedious to type manually. The script created in this lab collects three categories of system information in one execution:

```bash
#!/bin/bash
echo "System Information:"
uname -a
free -h
df -h
```

**Making it executable and running it:**

```bash
chmod +x ~/forensics_lab/sys_info.sh
bash ~/forensics_lab/sys_info.sh
```



![sys_info.sh execution output showing kernel version, memory stats, and disk usage](screenshots/sys_info_output.png)

**Output breakdown:**

`uname -a` returned: `Linux kali 6.17.10+kali1 #1 SMP PREEMPT_DYNAMIC Kali 6.17.10-kali1 (2025-12-08) x86_64 GNU/Linux`

This shows the operating system, hostname, kernel version, build date, and architecture. In a forensic report, this identifies the exact system configuration at the time of investigation.

`free -h` showed 1.9 GiB total RAM, 935 MiB used, 128 MiB free, with 1.1 GiB in buffer/cache. The swap partition had 1.0 GiB total with 4 KiB used, meaning the system was not under heavy memory pressure.

`df -h` showed the primary disk (/dev/sda1) at 79 GB total, 31 GB used (41%). Temporary filesystems (tmpfs) accounted for the remaining mount points.

This script is the starting point for a forensic triage tool. Expanding it to include running processes (`ps aux`), active network connections (`netstat -antp`), and recently modified files (`find / -mtime -1`) gives a complete snapshot of system state at acquisition time.

### Part 9: Software Installation

```bash
sudo apt install terminator
```



![apt install terminator output showing package installation and dependencies](screenshots/apt_install_terminator.png)



APT resolved dependencies automatically, identifying that `gir1.2-keybinder-3.0` was required alongside terminator. The package manager also flagged that `python3-prettytable` was installed as a dependency of a previous package but is no longer needed, suggesting it can be removed with `sudo apt autoremove`.

Terminator is a terminal emulator that supports splitting a single window into multiple panes. For security work, this is useful when running a scan in one pane while monitoring logs in another without switching windows.

### Part 10: System Updates

```bash
sudo apt update && sudo apt upgrade
```

![apt update and upgrade output showing package lists being fetched and packages being upgraded](screenshots/apt_update_upgrade.png)



`apt update` refreshed the package index from configured repositories, fetching updated package lists from sources including kali-rolling. `apt upgrade` then compared installed package versions against the updated index and downloaded newer versions where available.

The `&&` operator runs the second command only if the first succeeds. If `apt update` fails (no internet connection, repository error), `apt upgrade` does not run. This prevents upgrading against a stale package index.

Keeping systems updated matters in both offensive and defensive work. Unpatched packages are one of the most common ways attackers gain initial access. During a penetration test, running `apt list --upgradable` on a compromised machine shows which vulnerabilities the owner failed to patch.

## Findings

**Linux's open-source foundation makes it the right tool for evidence-grade forensics.** Every utility used in this lab can be audited for exactly what it does to data. That auditability is what allows forensic findings to hold up under scrutiny.

**The VFS layer is what makes multi-file-system forensics practical.** Without it, investigators would need a different operating system for every file system type they encounter. The VFS abstraction lets Linux mount NTFS, FAT32, XFS, and EXT4 evidence drives and apply the same investigative tools to all of them.

**Networking commands reveal both system configuration and attacker opportunities.** The IP address and netmask shown by ifconfig directly define the network range an attacker would target. Traceroute maps network topology. Nmap confirms which services are running. Understanding these tools from both sides of the attack is what separates security-aware analysts from basic system administrators.

**PATH variable manipulation is a viable persistence mechanism.** An attacker who prepends a malicious directory to PATH can cause the system to execute their version of a command (like `ls` or `sudo`) instead of the legitimate one. This would not appear in most log files and disappears after reboot unless written to `.bashrc`. Forensic investigators should always check PATH configuration when investigating compromised systems.

**Shell scripts turn repetitive forensic tasks into repeatable processes.** The sys_info.sh script collected kernel version, memory state, and disk usage in one command. A fully developed triage script can document complete system state in seconds, which is critical when working against time on a live system before it shuts down.

**Compression preserves evidence integrity for transport.** Hashing a zip archive before and after transfer confirms it was not tampered with. Directory structure and file timestamps inside the archive are preserved, which supports timeline reconstruction.

## Challenges Faced

**Traceroute hops showing `* * *`:** Most hops after the gateway returned no response. Initially this looked like a network failure. After checking ping (which worked fine), it became clear that the intermediate routers simply drop ICMP probes as a configuration policy. The destination was reachable, the routers were just silent. This distinction between a dead route and a probe-dropping router is important for accurate network troubleshooting.

**Understanding closed vs filtered in nmap:** The port 21 result said `closed`, not `filtered` or `open`. Getting the distinction right matters for penetration testing. Closed means the port is reachable but no service is listening. Filtered means a firewall is preventing the probe from getting through at all. These require different responses during an engagement.

**PATH changes not surviving session end:** After adding `/home/fatai/scripts` to PATH with `export`, closing and reopening the terminal removed the change. Writing environment variable changes to `~/.bashrc` makes them persistent across sessions. This is also worth noting from an attacker's perspective: temporary PATH changes are less detectable precisely because they do not persist.

## Key Takeaways

**Open-source is not just a cost question, it is an evidence integrity question.** When a forensic tool's source code is publicly available, every step of how it handles data can be verified. That verification is what makes findings defensible.

**The `/etc` directory is the first place to look on a compromised machine.** Password files, cron jobs, network configuration, and service definitions all live here in plain text. An attacker who modified system behavior almost certainly touched something in `/etc`.

**Networking commands give you the attacker's map.** ifconfig reveals the internal network range. nmap shows what services are listening. Traceroute reveals network topology. Learning these tools means understanding exactly what an attacker learns during the reconnaissance phase.

**A closed port is not a filtered port.** Nmap distinguishes between these states, and the difference matters. Closed means no service is running but the port is reachable. Filtered means a firewall is in the way. Treating them as the same leads to wrong conclusions during a security assessment.

**Shell scripting turns manual investigation into repeatable, documentable process.** A script that collects system state at acquisition time creates a consistent, reproducible record. Investigators who do this manually risk missing steps or documenting outputs inconsistently across different cases.

**Package management is part of the attack surface.** Unpatched software is how most initial access happens. The same apt commands used to keep a system secure can also show an investigator exactly which vulnerabilities an attacker may have exploited on a target they are examining.

## Disclaimer

This lab was performed on a local Kali Linux virtual machine configured for educational use as part of the ICDFA Basic Computer Skills for Digital Forensics course. All commands were executed in a controlled, isolated environment. No external systems were targeted or accessed without authorization.