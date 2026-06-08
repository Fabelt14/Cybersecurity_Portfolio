Looking at the lab content, this is a conceptual/theory-based lab rather than a hands-on technical lab. It covers computer architecture, forensics challenges, HDD components, disk partitions, boot process, and file systems. No tools were used, no commands were run. This needs a slightly different structure from the usual labs.

Let me read the GitHub template to confirm the right approach.

---

# Understanding Computer Systems, Forensics Challenges, and Storage Structures

## Overview

This lab covers the foundational concepts that digital forensics work depends on. Before investigating a compromised system, a forensic analyst needs to understand how computers process data, where that data lives, how disks are organized, and how the system boots. Each section builds context for why digital investigations are difficult and what an analyst must understand before touching evidence.

## Objectives

- Describe how core hardware components interact during program execution
- Identify the challenges that make digital forensics investigations difficult
- Analyze a real-world ransomware case and map its forensic obstacles
- Explain HDD physical structure and calculate total disk sectors
- Calculate disk partition sizes and available storage
- Outline the PC boot sequence and identify where forensic artifacts exist at each stage
- Compare file system structures and recommend appropriate use cases

## Lab Environment

- **Machine:** Personal computer (Windows)
- **Lab Type:** Conceptual and calculation-based
- **Tools Used:** None (theory and analysis exercises)

## Tools Used

None. This lab focuses on foundational knowledge rather than hands-on tool usage.

## Methodology

### Part 1: Computer Architecture and Data Flow

Understanding how a computer processes data is the starting point for any forensic investigation. An analyst who does not know where data lives cannot know where to look for evidence.

**Core components:**

**CPU (Central Processing Unit):** The processor that fetches and executes instructions. Every computation runs through here. From a forensics perspective, the CPU's current execution state at the time of an incident can contain clues about what process was running.

**RAM (Random Access Memory):** Volatile short-term memory that stores active data and running program instructions. It wipes completely when power is removed. This is critical for forensics because malware increasingly runs entirely in RAM to avoid leaving files on disk. If an analyst powers off a compromised machine without capturing a memory dump first, that evidence disappears permanently.

**Motherboard:** The main circuit board that connects all components and distributes power. It provides the communication pathways between the CPU, RAM, and storage devices.

**Storage Devices:** Non-volatile hardware (SSD, HDD) that holds data persistently. This is where the file system lives, where deleted files may be recovered from, and where most forensic artifacts reside.

**Input/Output Devices:** Peripherals (keyboard, mouse, monitor, speakers) that allow users to interact with the computer. From a forensics standpoint, USB I/O devices can be entry points for malware or data exfiltration.

**Data flow during program execution:**

A user launches a program. The operating system reads the program file from the storage device and loads its instructions into RAM. The CPU fetches those instructions from RAM one at a time, executes them, and writes results back to RAM. When the program needs to save output permanently, it writes back to the storage device. The motherboard handles all inter-component communication throughout this process.

![Diagram showing CPU-RAM-storage data flow during program execution](screenshots/computer_architecture_data_flow.png)



### Part 2: Why Digital Forensics is Hard

Three challenges consistently obstruct digital investigations.

**Challenge 1: Data Volatility**

Attackers increasingly use fileless malware that runs entirely inside RAM. No files touch the disk, so traditional antivirus tools that scan file systems miss it entirely. When the system powers off, RAM clears. All evidence vanishes.

*Real consequence:* A first responder who shuts down a compromised machine before capturing memory destroys the only evidence of what was running.

*Solution:* Capture a live memory image before powering off. Tools like DumpIt or FTK Imager can dump the full contents of RAM into an image file while the system is still running. This preserves process lists, network connections, encryption keys, and malicious code that would otherwise disappear.

**Challenge 2: Encryption**

Ransomware groups encrypt victim files using strong cryptographic algorithms. The data is still physically present on the disk, but without the decryption key it is unreadable ciphertext. Forensic tools can see the encrypted file but cannot recover the content.

*Real consequence:* The entire investigation stalls. Analysts cannot read logs, emails, or documents. The organization cannot operate.

*Solution:* Maintain offline, isolated backups that ransomware cannot reach. If backups exist, the organization wipes the affected system and restores from backup. If backups do not exist, the only options are paying the ransom, waiting for law enforcement to seize the threat actor's infrastructure, or hoping for a free decryptor release.

**Challenge 3: Log Tampering**

After breaching a system, skilled attackers clear their tracks by deleting or modifying server log files. Logs are the primary timeline reconstruction tool in forensics. Without accurate logs, investigators cannot determine when the attacker entered, what they accessed, or how they moved laterally.

*Solution:* Forward logs in real time to a centralized SIEM (Security Information and Event Management) system like Splunk or an ELK stack. If logs ship off the compromised machine immediately, an attacker who deletes local logs has already lost. The SIEM copy remains intact.

**Case Study: Colonial Pipeline Ransomware Attack (May 2021)**

The Colonial Pipeline supplies 45% of the East Coast's fuel. DarkSide, a ransomware group, gained access through a single compromised VPN account belonging to a former employee. The account had no multi-factor authentication and the password had been leaked in a previous breach.

Once inside, DarkSide deployed ransomware that encrypted 100 gigabytes of billing and accounting data. Colonial Pipeline could no longer bill customers. Fearing the malware would spread to operational pipeline controls, management shut down the entire pipeline. Fuel shortages spread across multiple states.

**Forensic obstacles:**

The encryption created an immediate dead end. Analysts could see the encrypted files but could not read them. Without accessible billing records, the company could not operate. Colonial Pipeline had no verified, isolated backups that could restore operations quickly. Faced with an ongoing operational shutdown causing national impact, they paid $4.4 million in Bitcoin to obtain the decryption key.

The FBI later recovered approximately $2.3 million of the ransom by seizing the private key used to access the Bitcoin wallet.

**What this demonstrates:** Encryption is not just a forensic problem. It becomes a business continuity crisis. The ransom payment was not a security decision. It was a business decision made under operational pressure. Forensic analysts had no technical path around the cryptographic barrier because no backups existed and no decryption tool was available.

### Part 3: Hard Disk Drive Structure

Understanding HDD physical components explains why forensic recovery is possible even after deletion.

**Physical components:**

**Platters:** Circular disks made of aluminum, glass, or ceramic coated with a magnetic layer. Data is recorded as binary (1s and 0s) in concentric rings called tracks. Multiple platters stack on a single spindle, each with its own read/write head.

**Spindle:** The central axle that holds platters and spins them at constant speed, typically 5,400 to 15,000 RPM. Higher RPM means the correct data sector reaches the read/write head faster, reducing rotational latency.

**Read/Write Heads:** Tiny magnetic sensors at the ends of actuator arms. They hover nanometers above the spinning platter surface without physically touching it. They detect existing magnetic fields (reading) or generate new ones (writing).

**Actuator Arm:** The mechanical arm that moves read/write heads across platter surfaces to reach any track and sector.



![HDD internal components diagram showing platters, spindle, actuator arm, and read/write heads](screenshots/hdd_components_diagram.png)



**Disk geometry calculation:**

Given: Sectors per track = 500, Number of heads = 10, Cylinders = 12,000

Total sectors = Cylinders × Heads × Sectors per track

Total sectors = 12,000 × 10 × 500 = **60,000,000 sectors**



![Disk geometry calculation showing formula and result](screenshots/disk_geometry_calculation.png)

**Why HDD design affects forensic recovery:**

When a file is "deleted," the OS typically only removes the file's entry from the file system index. The actual data sectors remain on the platters with their magnetic encoding intact. A forensic tool that reads raw sectors, bypassing the file system, can recover those sectors and reconstruct the original file.

SSDs complicate this. They use a feature called TRIM that actively erases data blocks when files are deleted, freeing them for immediate reuse. SSDs also use wear leveling algorithms that move data between cells to extend drive life, scattering fragments across the physical storage. Recovering deleted data from an SSD is significantly harder than from an HDD.

### Part 4: Disk Partitions

**What partitioning does:**

Partitioning divides a single physical drive into separate logical sections, each treated as an independent storage area by the operating system. The partition table (stored at the start of the drive in MBR or GPT format) tells the OS where each partition begins and ends.

Common uses:
- Separate OS files from user data (C: drive vs D: drive on Windows)
- Install multiple operating systems on one physical disk (dual-boot)
- Isolate recovery environments from the main OS
- Apply different file systems to different partitions

**Partition calculation:**

Given: 1 TB (1,000 GB) total drive capacity

| Partition | Size | Calculation | Percentage |
|---|---|---|---|
| System | 200 GB | 200 ÷ 1000 × 100 | 20% |
| Data | 600 GB | 600 ÷ 1000 × 100 | 60% |
| Recovery | 200 GB | 200 ÷ 1000 × 100 | 20% |

Total allocated: 200 + 600 + 200 = 1,000 GB

Remaining free space: 1,000 - 1,000 = **0 GB**

The drive is fully partitioned with no unallocated space remaining.

**Forensic relevance of partition structure:**

Forensic tools image drives at the sector level, capturing all partitions including recovery partitions that Windows Explorer hides from users. Attackers sometimes hide data in unallocated space between partitions or create small hidden partitions that the OS does not mount. A sector-level image captures everything.

### Part 5: PC Boot Process

Each boot stage leaves forensic artifacts and presents attack opportunities.

**Stage 1: Power On and POST (Power-On Self Test)**

The moment the power button is pressed, the CPU initializes from a hard-coded memory address and begins executing BIOS/UEFI firmware. POST checks that RAM, CPU, and connected storage devices are functional. Hardware failures here produce error codes or audible beeps.

**Forensic relevance:** UEFI firmware can be compromised by firmware-level rootkits (bootkits) that load before the OS and persist across OS reinstalls. Detecting these requires firmware integrity verification, not just OS-level scans.

**Stage 2: BIOS/UEFI Boot Order Check**

The firmware reads its configured boot order and searches for a bootable device in that sequence (USB first, then HDD, for example). It looks for the boot sector at the start of the first partition.

**Forensic relevance:** An attacker can boot from a USB drive to bypass OS-level access controls entirely. This is why full disk encryption (BitLocker, VeraCrypt) matters. Even if an attacker boots from external media, encrypted drives appear as unreadable ciphertext.

**Stage 3: Bootloader Execution**

The firmware hands control to the bootloader (Windows Boot Manager or GRUB on Linux). The bootloader knows where the OS kernel lives on disk and loads it into RAM.

**Forensic relevance:** Bootkits target this stage. A compromised bootloader can load a rootkit before the OS initializes, making the rootkit invisible to OS-level security tools.

**Stage 4: OS Kernel Load**

The bootloader loads the OS kernel into RAM. The kernel initializes device drivers, mounts the file system, and starts system services. At this point, the user sees the login screen.

**Forensic relevance:** Windows creates event logs from this point forward. The first log entries after boot establish the baseline timeline for incident reconstruction.



![PC boot sequence diagram showing each stage from POST to OS load](screenshots/boot_sequence_diagram.png)



**Common boot failures:**

"No Bootable Device Found" means the BIOS cannot locate a bootloader. This happens when the boot order is wrong, the drive connection is loose, or the bootloader itself is corrupted (sometimes from a failed OS update). Fix: check BIOS boot order and verify the drive is physically connected.

Continuous beeping during POST indicates a hardware failure caught before the OS loads, most commonly failed or improperly seated RAM. Fix: reseat the memory sticks or test with known-good RAM.

### Part 6: File Systems

**What a file system does:**

Raw disk storage is just sectors of binary data with no structure. A file system creates the organizational layer that maps file names and folder paths to specific sectors on the disk. Without a file system, there is no way to know where one file ends and another begins, or what any particular block of bytes represents.

**File system comparison:**

| File System | Best Use | Advantages | Disadvantages |
|---|---|---|---|
| FAT32 | USB drives, external media, embedded devices | Near-universal compatibility across Windows, Mac, Linux, consoles, cameras | 4 GB file size limit, no access control, no journaling |
| NTFS | Windows internal drives | File permissions (ACLs), native encryption (EFS), journaling prevents corruption | Mac can read but cannot write without third-party software |
| EXT4 | Linux servers | Fast with large numbers of small files, journaling, Linux permissions, large file support | Windows and Mac cannot read without additional software |

**Recommendations by use case:**

**Personal computer (Windows):** NTFS. File permissions matter when multiple user accounts share one machine. NTFS access control lists prevent one user from reading another's files. Journaling means the file system recovers cleanly from power failures or crashes.

**Linux server:** EXT4. Linux servers handle large numbers of small files (web requests, log files, configuration files). EXT4 is tuned for this access pattern and integrates natively with Linux's permission model.

**Embedded systems (smart TVs, cameras, IoT devices):** FAT32. These devices need a file system that Windows, Mac, and Linux can all read without extra software. They rarely store files larger than 4 GB. The simplicity of FAT32 means it requires minimal processing overhead, which matters on hardware with constrained resources.

**Forensic perspective on file systems:**

File system choice affects what artifacts are recoverable. NTFS maintains a transaction log ($LogFile) and a change journal ($UsnJrnl) that record file creation, modification, and deletion events. These survive after files are deleted and give forensic analysts a detailed activity timeline. FAT32 has no equivalent. EXT4's journal records metadata changes but not full file content.

When imaging a drive for forensics, identifying the file system type determines which parsing tools to use. A forensic examiner who tries to parse an EXT4 partition as NTFS gets garbage output.

## Findings

**Data volatility is the most time-sensitive forensic challenge.** RAM evidence disappears the moment power is cut. Every other challenge (encryption, tampering) leaves something on disk to work with, even if that work is difficult. Volatile memory capture must happen first, before anything else.

**The Colonial Pipeline attack illustrates how technical failures become operational crises.** A single unprotected VPN account with a leaked password gave attackers access to critical national infrastructure. The forensic obstacle was encryption, but the actual crisis was the absence of isolated backups. Paying $4.4 million was cheaper than continued operational shutdown.

**HDD physical structure enables forensic recovery that deleted files cannot prevent.** Deleting a file removes its file system entry but leaves magnetic data on the platters. Raw sector imaging recovers this data. SSDs with TRIM make this significantly harder.

**Boot process integrity matters for forensics.** Attacks at the firmware or bootloader stage load before the OS and become invisible to OS-level security tools. Secure Boot and firmware integrity verification address this, but many systems run without these protections.

**File system choice determines what artifacts are available for investigation.** NTFS logs detailed file system activity through its change journal. FAT32 logs nothing. This means NTFS-formatted drives give forensic analysts a much richer activity timeline than FAT32 drives.

**Partition structure can hide evidence.** Unallocated space between partitions and hidden partitions that the OS does not mount are invisible to standard file browsers but visible to sector-level forensic imaging tools.

## Challenges Faced

**Connecting theoretical concepts to practical forensics scenarios:** The lab content covered hardware architecture as standalone knowledge. The challenge was mapping each concept to its forensic implication. For example, understanding that RAM is volatile is one thing. Understanding that fileless malware specifically exploits this volatility to evade detection requires thinking from an attacker's perspective and working backwards to the forensic consequence.

**Disk geometry calculation units:** The sector count calculation (60,000,000 sectors) produces a large number that requires context to interpret. Each sector is typically 512 bytes, meaning 60 million sectors represents approximately 30 GB of storage. Converting abstract geometry parameters into practical storage capacity takes deliberate unit tracking.

**File system trade-offs without hands-on testing:** Comparing FAT32, NTFS, and EXT4 from documentation is less intuitive than mounting each file system and observing behavior directly. The 4 GB FAT32 limit is clearly stated, but understanding why that limit exists (FAT32 uses 32-bit cluster addressing, which caps the addressable space) required going beyond the lab materials.

## Key Takeaways

**Power off a machine and you destroy volatile evidence.** The first decision at any incident scene is whether to pull the power or capture memory first. Pulling power protects against remote access but destroys RAM contents. Capturing memory first preserves evidence but keeps an active attacker connected. This decision determines what evidence survives.

**Backup strategy is a forensic tool.** Colonial Pipeline paid $4.4 million because they had no verified offline backups. Organizations with tested, isolated backups treat ransomware as an inconvenience rather than a crisis. Forensic analysts can investigate without time pressure when the business can restore from backup independently.

**Deletion is not erasure on HDDs.** The file system entry disappears, but the magnetic data on the platter remains until overwritten. Forensic imaging at the sector level recovers this data. On SSDs with TRIM, actual erasure happens immediately and recovery becomes much harder.

**Attackers who reach the boot process become nearly invisible.** Firmware rootkits and bootkit malware load before the OS, before antivirus, before any OS-level security control. Detecting and removing them requires specialized tools that operate at the firmware level.

**File system choice shapes the investigation before any incident occurs.** NTFS's change journal is a passive audit trail that records file activity continuously. An organization using NTFS has a built-in activity log that forensic analysts can parse during an investigation. An organization using FAT32 has none.

**The BIOS/UEFI handoff to the bootloader is a critical trust boundary.** If an attacker corrupts this handoff, everything that follows loads in a compromised state. Secure Boot exists specifically to cryptographically verify each component in this chain before trusting it.

## Disclaimer

This lab was completed as part of the Basic Computer Skills for Digital Forensics course at ICDFA. All content is for educational purposes. The Colonial Pipeline case study uses publicly available information from news sources and official government reports.