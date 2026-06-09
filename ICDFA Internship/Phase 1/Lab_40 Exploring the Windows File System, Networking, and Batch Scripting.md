# Exploring the Windows File System, Networking, and Batch Scripting

## Overview

This lab covered three core areas of Windows system operation: file system management, network reconnaissance using built-in command-line tools, and batch scripting for task automation. The work was done on a Windows 10 Pro machine (hostname: FABELT) using Command Prompt and File Explorer. Each part builds on the previous one, moving from basic file handling to network analysis to scripting system information collection.

## Objectives

- Create and manage folders and files using the Windows file system
- Copy and delete files and verify the results
- Use Command Prompt networking tools to gather system and network information
- Share a folder over a local network and understand Windows sharing permissions
- Search the file system using wildcard patterns to locate specific file types
- Write and execute a batch script that automates system enumeration
- Add a script to the system PATH so it runs from any directory

## Lab Environment

- **Machine:** HP EliteBook 840 G1 (hostname: FABELT)
- **Operating System:** Microsoft Windows 10 Pro (Build 19045)
- **Network Adapters:** Ethernet 4 (192.168.137.1) and Wi-Fi 4 (192.168.43.17)
- **Tools:** File Explorer, Command Prompt, Windows Sharing, Notepad, System PATH environment variables

## Tools Used

- File Explorer (folder and file management)
- Command Prompt (ipconfig, ping, tracert, netstat)
- Windows Network Sharing (SMB folder sharing)
- Notepad (batch script creation)
- Windows Environment Variables (PATH configuration)

## Methodology

### Part 1: Folder and File Creation

The first task was to create a working directory and populate it with text files. I created a folder named `workspace4` on the desktop using File Explorer.



![workspace4 folder visible on desktop among other icons](images/part1_workspace4_desktop.png)

Inside `workspace4`, I created three text files: file1.txt, file2.txt, and file3.txt. Each file was opened in Notepad and given a brief description of its purpose before saving.



![workspace4 folder showing file1.txt, file2.txt, file3.txt all at 1KB](images/part1_files_created.png)



**Size change from 0KB to 1KB:** An empty file on Windows shows as 0KB because no data occupies disk space. The moment text is written and saved, the file occupies at least one allocation unit on the NTFS file system, which rounds up to 1KB. This is the minimum reportable file size in File Explorer's detail view, even if the actual content is only a few bytes.

### Part 2: File Copy and Deletion

The lab asked me to copy file1.txt and file2.txt into the workspace4 folder. Since both files were already created there in Part 1, this step confirmed the files were in place.

![workspace4 showing three files present](images/part2_files_in_workspace4.png)



I then deleted file3.txt by selecting it and pressing Delete. File Explorer moved it to the Recycle Bin and the folder now shows only file1.txt and file2.txt.



![workspace4 after deletion showing only file1.txt and file2.txt](images/part2_after_deletion.png)



**Why deletion matters for forensics:** Deleting a file in Windows does not immediately erase the data from disk. The NTFS master file table marks the space as available, but the actual bytes remain until overwritten by new data. This is why digital forensics tools like Autopsy can recover deleted files from a disk image. The Recycle Bin adds another layer, holding the deleted file at `C:\$Recycle.Bin` until the bin is emptied.

### Part 3: Networking Commands

#### ipconfig

Running `ipconfig` reveals the IP addressing configuration of every network adapter on the machine.



![ipconfig output showing Ethernet 4 and Wi-Fi 4 adapter details](images/part3_ipconfig_output.png)



**What the output shows:**

- **Ethernet adapter Ethernet:** Media disconnected. No cable is plugged in.
- **Ethernet 4 (192.168.137.1):** Active Ethernet adapter. The 192.168.137.x range is typically created by Windows when Internet Connection Sharing (ICS) is enabled, used to share the Wi-Fi connection with other devices over Ethernet.
- **Wi-Fi 4 (192.168.43.17):** The active wireless adapter connected to a hotspot or router. Default gateway is 192.168.43.5, which is the router's IP address. All internet-bound traffic exits through this gateway.
- **IPv6 link-local addresses (fe80::...):** Automatically assigned by Windows for local network communication without needing a DHCP server. These only work within the same network segment.

For a penetration tester, `ipconfig` is the starting point for understanding what networks a compromised machine can reach. A machine with both Ethernet and Wi-Fi adapters may bridge two separate network segments, making it useful as a pivot point.

#### ping www.google.com

Ping sends four ICMP Echo Request packets to a destination and measures how long each reply takes.



![ping www.google.com output showing 4 successful replies with TTL 113](images/part3_ping_google.png)



**Output breakdown:**

- Google's IP resolved to 142.251.154.119
- All 4 packets received replies (0% packet loss)
- Response times: 71ms, 78ms, 347ms, 172ms (average 167ms)
- TTL of 113 means the packet started with a TTL of 128 (Windows default) and passed through 15 routers before reaching the machine

The 347ms spike on packet 3 is worth noting. It does not indicate packet loss, just temporary congestion at one of the intermediate hops. A sustained pattern of high latency across all 4 packets would indicate network congestion or a routing problem.

#### tracert www.google.com

Traceroute maps every router between the local machine and the destination by sending packets with incrementally increasing TTL values.



![tracert www.google.com output showing 11 hops to Google](images/part3_tracert_google.png)



**Route analysis:**

- Hop 1 (2ms): The local router/gateway at 192.168.43.5
- Hops 2-5: ISP infrastructure, latency climbs as packets travel through the provider's backbone
- Hops 6, 7, 8: "Request timed out." These routers are configured to drop ICMP packets rather than respond to them. This is a deliberate security configuration common in ISP and enterprise networks to prevent network mapping. The route still completes because ICMP blocking on intermediate hops does not stop the packets from passing through.
- Hops 9-11: Google's infrastructure, ending at 142.251.157.119

**Security relevance:** Traceroute is a reconnaissance tool. Running it against a target network reveals the ISP being used, geographic routing patterns, and sometimes internal IP ranges if a router leaks private addressing. Security-conscious organizations block ICMP on their edge devices specifically to prevent outsiders from mapping their network topology.

#### netstat -a

`netstat -a` lists all active TCP connections and all ports currently listening for incoming connections.



![netstat -a output showing ESTABLISHED, LISTENING, CLOSE_WAIT, TIME_WAIT states](images/part3_netstat_output.png)



**Connection states visible:**

- **LISTENING (0.0.0.0:49666 through 49670):** Windows system services waiting for connections on high-numbered ports. These are RPC (Remote Procedure Call) endpoints used internally by Windows components.
- **ESTABLISHED (192.168.43.17:various → remote:443):** Active HTTPS connections to cloud services. Port 443 is encrypted HTTPS traffic.
- **CLOSE_WAIT:** The remote server already sent a FIN to close the connection, but the local application has not closed its side yet. Usually means an application is slow to clean up connections.
- **TIME_WAIT:** Connection is fully closed but the port is being held briefly to ensure any delayed packets from the old connection do not interfere with new connections using the same port.

For an attacker doing post-exploitation, `netstat -a` reveals what services are running locally, what external systems the machine is actively communicating with, and which ports are open for new connections. For a defender, unexpected ESTABLISHED connections to unknown external IPs are worth investigating.

#### Network Folder Sharing

I shared the `workspace4` folder with other users on the local network by right-clicking the folder and accessing Properties, then the Sharing tab.



![workspace4 sharing confirmation showing UNC path \\FABELT\Users\user\Desktop\workspace4](images/part3_folder_shared.png)



Steps taken:
1. Right-clicked `workspace4` on the desktop and selected Properties
2. Navigated to the Sharing tab and clicked Share
3. Selected Everyone from the dropdown, clicked Add
4. Changed Permission Level to Read/Write, then clicked Share

The folder became accessible via the UNC path `\\FABELT\Users\user\Desktop\workspace4`. Any machine on the same network segment can now access this folder using that path.

**Security implications of Read/Write for Everyone:** This permission grants any user on the network full read and write access without authentication. An attacker on the same network could browse the share, read sensitive files, or write malicious files that the owner might later execute. Proper network sharing restricts access to specific named users or groups, requires authentication, and follows the principle of least privilege.

#### File System Search

To locate all text files on the machine, I used the Windows search function with a wildcard pattern.



![File Explorer search results showing txt files found across multiple directories](images/part3_txt_search_results.png)

Steps taken:
1. Opened This PC from the desktop
2. Clicked the search bar in the top-right corner of File Explorer
3. Typed `*.txt` and pressed Enter

The search returned 17 .txt files located across system directories including `C:\Windows` (ntbtlog.txt, default.help.txt, ThirdPartyNotices.txt) and other locations. The `*` wildcard matches any filename, so `*.txt` finds every file with a .txt extension regardless of name.

**Forensics application:** This same technique applies to evidence collection. Searching `*.docx`, `*.pdf`, or `*.xlsx` on a seized machine quickly finds documents of interest without browsing every folder manually. Investigators can also search for specific file signatures rather than extensions since extensions can be renamed to hide file types.

### Part 4: Batch Scripting

#### Creating sys_info.bat

A batch file is a plain text file containing a sequence of Command Prompt commands that execute in order when the file is run. I created `sys_info.bat` in Notepad with the following content:

```batch
@echo off
echo System Information:
systeminfo
pause
```



![sys_info.bat file visible in File Explorer](images/part4_sysinfo_bat_created.png)



**Line-by-line explanation:**

- `@echo off`: Stops the terminal from printing each command before executing it. Without this, every command would appear twice: once when printed and once when the output appears. The `@` prevents `echo off` itself from being printed.
- `echo System Information:`: Prints a label to the terminal so the output has a clear header.
- `systeminfo`: Runs the built-in Windows command that collects OS version, hardware details, network configuration, hotfix history, and more.
- `pause`: Holds the terminal window open after execution completes. Without this, the Command Prompt window closes immediately and the output disappears before it can be read.

#### Making the Batch File Callable from Any Directory

By default, Windows only runs a script if you are in the same directory as the file or provide the full path. To run `sys_info.bat` from anywhere, I added `C:\Scripts` to the system PATH environment variable.



![System PATH showing C:\Scripts added alongside Docker path](images/part4_path_configured.png)



The PATH variable is a list of directories Windows searches whenever a command is typed. When I type `sys_info.bat` in any Command Prompt window, Windows checks each folder in PATH in order until it finds the file. Adding `C:\Scripts` means Windows will always find scripts stored there.

#### Executing the Batch File

Running `sys_info.bat` from `C:\Users\user` confirmed the PATH configuration worked. The script executed and printed system enumeration details to the terminal.

![sys_info.bat execution output showing hostname FABELT, OS version, system model, boot time](images/part4_sysinfo_output.png)



**Key output captured:**

- Host Name: FABELT
- OS Name: Microsoft Windows 10 Pro
- OS Version: 10.0.19045 N/A Build 19045
- OS Configuration: Standalone Workstation
- System Manufacturer: Hewlett-Packard
- System Model: HP EliteBook 840 G1
- System Boot Time: 5/18/2026, 3:29:49 PM
- Processor: Intel64 Family 6 Model 69

**Why this matters for security:** The `systeminfo` command is one of the first tools an attacker runs after gaining access to a Windows machine. It reveals the OS version (to identify missing patches), hardware model, install date, and network configuration. Defenders use the same output to audit machines for compliance with baseline security standards. A machine running Windows 10 Build 19045 that should be on a newer build is overdue for updates.

## Findings

**The Windows file system enforces minimum allocation units.** Empty files show as 0KB, but any content rounds up to the minimum reportable size of 1KB in File Explorer. This is an NTFS allocation characteristic, not the actual byte count of the content.

**Deleted files are not immediately gone.** Windows moves deleted files to the Recycle Bin and marks disk space as available without overwriting data. The actual bytes remain recoverable until new data overwrites them, which is why forensic tools can recover deleted files from disk images.

**ipconfig reveals dual network paths.** The machine has both Ethernet (192.168.137.1) and Wi-Fi (192.168.43.17) interfaces active. This matters for penetration testing because a machine connected to two networks can potentially bridge them, making it a pivot point for accessing otherwise unreachable segments.

**Traceroute hops with asterisks (*) are not dead routers.** Hops 6-8 showed request timeouts, but the route completed at hop 11. Those routers are dropping ICMP specifically to prevent network mapping while still forwarding other traffic normally.

**netstat -a shows what a compromised machine is connected to.** ESTABLISHED connections, listening ports, and connection states all reveal what services are running and what external systems the machine communicates with. This is reconnaissance data for attackers and audit data for defenders.

**Granting Read/Write access to Everyone on a shared folder is a security risk.** Any unauthenticated user on the same network can read, write, or delete files in that share. Proper sharing restricts access to named accounts with the minimum required permissions.

**Adding scripts to the system PATH enables automation from any context.** Once `C:\Scripts` was in PATH, `sys_info.bat` ran from any directory without specifying its location. This is the same mechanism attackers use to ensure malicious scripts remain executable even after moving to different directories on a compromised machine.

## Challenges Faced

**File copy task was already satisfied by Part 1.** The lab asked me to copy file1.txt and file2.txt into the workspace4 folder, but the files were already there from the creation step. Rather than copy them to another location and back, I confirmed their presence in the directory and documented why the step was already complete.

**Traceroute asterisks initially looked like failures.** Hops 6, 7, and 8 all showed "Request timed out" with three asterisks. My first thought was that the route was broken at those hops. Looking at the subsequent hops, the route clearly continued and completed successfully, which clarified that those routers were blocking ICMP responses by design rather than being unreachable.

## Key Takeaways

**File system basics are forensics fundamentals.** Knowing that deleted files persist on disk, that file size reports minimum allocation units rather than exact byte counts, and that NTFS tracks metadata separately from content all matter for investigating digital evidence.

**Windows networking commands map directly to attacker and defender workflows.** `ipconfig`, `ping`, `tracert`, and `netstat` are built into every Windows installation and require no additional tools. An attacker uses them immediately after gaining access to understand the environment. A defender uses the same output to audit network exposure and detect unexpected connections.

**Batch scripting turns manual steps into repeatable tools.** The `sys_info.bat` script replaced typing `systeminfo` by hand with a single command that runs from anywhere on the system. The same principle applies to more complex tasks: log collection, scheduled backups, or automated incident response can all be scripted using batch files or PowerShell.

**The PATH variable determines what Windows can find without a full path.** This is why malware that places itself in a PATH directory executes reliably across different contexts. It is also why defenders audit PATH contents when investigating suspicious activity.

**Network shares need least-privilege permissions.** Sharing a folder to Everyone with Read/Write access is convenient but creates real risk on any network with untrusted devices. Production environments restrict share access to specific users or security groups and log access attempts.

## Disclaimer

This lab was performed on a personal Windows machine in a controlled educational environment. All networking commands were run against publicly accessible infrastructure (Google) for connectivity testing only. No unauthorized systems were accessed.