# Foundational Concepts: Number Systems, Date Formats, ASCII Codes, Disk Drives, and Computer Forensics

## Overview

This lab covered the foundational knowledge that sits underneath every digital forensics investigation. Before an investigator can analyze a disk image, interpret a log file, or verify evidence integrity, they need to understand how computers represent data at the binary level, how timestamps work across systems and time zones, how storage devices are physically organized, and why the tools and procedures used in forensic work exist in the first place. This lab worked through each of those areas through hands-on calculations, comparisons, and system analysis.

## Objectives

- Convert binary and hexadecimal numbers to decimal and explain the process step by step
- Understand why hexadecimal notation is the standard in computing contexts like memory addressing and color representation
- Convert Unix timestamps to human-readable dates and explain why Unix time is the forensic standard for timeline reconstruction
- Distinguish between date format standards and identify where each is appropriate
- Create ASCII reference tables and write Python scripts to convert between characters and their numeric codes
- Understand how extended ASCII expanded character support beyond the English alphabet
- Calculate disk capacity from raw geometry parameters and explain how physical disk structure affects performance
- Compare FAT32, NTFS, and EXT4 file systems across compatibility, security, and performance
- Use `lscpu` to extract forensically relevant system information
- Explain the core challenges in digital forensics: deleted file recovery, evidence integrity, and chain of custody

## Lab Environment

- **Machine:** Kali Linux (Virtual Machine)
- **Hypervisor:** KVM (confirmed via `lscpu`)
- **CPU:** Intel Core i5-4300U @ 1.90GHz, x86_64 architecture
- **Python:** Used for ASCII conversion scripting

## Tools Used

- Linux CLI (`lscpu`)
- Python (ASCII code conversion)
- Manual arithmetic (number system conversions, disk geometry calculations)
- FTK Imager (research and forensic context)
- Autopsy (research and forensic context)

## Methodology

### Part 1: Number Systems and Conversion

#### Binary to Decimal

Binary is the native language of computers. Every file, every password, every image stored on disk exists as a sequence of 1s and 0s. Understanding how to convert binary to decimal is not just an academic exercise. It is the skill that lets an investigator read raw hex dumps and understand what those bytes represent.

The conversion process works by assigning each binary digit a positional value based on powers of 2, starting from the rightmost digit at position 0.

**Conversion: 1010111101**

```
Position:   9    8    7    6    5    4    3    2    1    0
Bit:        1    0    1    0    1    1    1    1    0    1
Value:    512    0  128    0   32   16    8    4    0    1
```

512 + 128 + 32 + 16 + 8 + 4 + 1 = **701**

**Conversion: 11011011**

```
Position:   7    6    5    4    3    2    1    0
Bit:        1    1    0    1    1    0    1    1
Value:    128   64    0   16    8    0    2    1
```

128 + 64 + 16 + 8 + 2 + 1 = **219**



![Python terminal confirming binary to decimal conversions](screenshots/binary_decimal_conversions.png)



#### Hexadecimal to Decimal

Hexadecimal (base 16) maps directly to binary in groups of 4 bits. This is why every memory address, every color code, and every file signature in forensics is written in hex. One hex character always represents exactly 4 bits. Two hex characters always represent exactly 1 byte. The math stays predictable.

**Conversion: 0xABCD**

Each hex digit has a letter value: A=10, B=11, C=12, D=13.

```
Position:   3      2      1      0
Digit:      A      B      C      D
Value:     10     11     12     13
Weight:  4096    256     16      1

(10 × 4096) + (11 × 256) + (12 × 16) + (13 × 1)
= 40960 + 2816 + 192 + 13
= 43981
```

**Why hex beats decimal in computing contexts:**

Memory addresses in a 32-bit system are always exactly 8 hex characters (e.g., 0x0804842B). In decimal, the same address is 134513707. In a debugger displaying hundreds of memory addresses in a column, the decimal version creates a ragged, unreadable column because the digit count fluctuates. The hex version stays perfectly aligned because the length is determined by bit width, not value.

The same logic applies to RGB color codes. One byte always equals two hex characters. White is #FFFFFF (255 red, 255 green, 255 blue). Black is #000000. The format never changes length regardless of the color values. Decimal RGB requires commas and variable digit counts: rgb(255, 255, 255) vs rgb(0, 0, 0). For storing and comparing colors in a database, hex is simpler to parse.

![lscpu output confirming x86_64 architecture](screenshots/lscpu_output.png)



### Part 2: Date Formats and Conversion

#### Unix Timestamp Conversion

Unix timestamp 1626258000 counts the total seconds elapsed since January 1, 1970, at 00:00:00 UTC (the Unix Epoch).

**Manual conversion:**

```
Step 1: Total days
1,626,258,000 ÷ 86,400 = 18,822 days remainder 37,200 seconds

Step 2: Remaining hours
37,200 ÷ 3,600 = 10 hours remainder 1,200 seconds

Step 3: Remaining minutes
1,200 ÷ 60 = 20 minutes

Step 4: Count forward from Jan 1, 1970 by 18,822 days
Result: July 14, 2021 at 10:20:00 UTC
```

**Challenge encountered:** When walking forward through the years to confirm the date, I initially divided the remaining 1,290 days by 365 to find the remaining years. This failed because 2020 is a leap year (366 days). The calendar drifted off by one day. I corrected it by walking through the years one at a time, accounting for the 366-day anomaly explicitly.

**Why Unix time matters in forensics:** When an incident involves a web server, a firewall, and a database, each system may sit in a different time zone. Log entries that appear to contradict each other are often just timezone differences. Since every system records events as Unix timestamps internally, an investigator can sort all log entries numerically to build a single, perfectly synchronized timeline regardless of where each system is located.

#### Date Format Comparison

**MM/DD/YYYY** is the American regional format. When stored as a text string, it breaks database sorting. The string "01/01/2024" comes before "12/31/2023" alphabetically, which puts January 1, 2024 ahead of December 31, 2023. That is a broken timeline. The format also creates regional ambiguity. 04/05/2024 means April 5th to an American and May 4th to a European.

**YYYY-MM-DD** (ISO 8601) solves both problems. Alphabetical sorting and chronological sorting produce identical results because the most significant unit (year) comes first. The format is unambiguous across all regions. Every database standard and forensic tool uses ISO 8601 for stored timestamps.

**When to use each:** Use YYYY-MM-DD in databases, log files, code, and anywhere a machine reads or sorts the date. Use MM/DD/YYYY on user interfaces where the audience is American users who read dates in that format naturally. The database stores ISO 8601. The front end displays it in whatever regional format the user expects.

### Part 3: ASCII Code Understanding

ASCII assigns a unique integer to every printable character and control function. The computer does not store the letter A. It stores the number 65. When displaying that number, the system looks up its ASCII mapping and renders the character. Every text file, every network packet carrying text, every password field operates on this translation layer.

#### Standard ASCII Reference

| Character | ASCII Code |
|-----------|-----------|
| A | 65 |
| B | 66 |
| C | 67 |
| a | 97 |
| b | 98 |
| c | 99 |

The 32-point gap between uppercase and lowercase (A=65, a=97) is not arbitrary. In binary, flipping a single bit (bit 5) converts between cases. This is why case-insensitive string comparison in low-level code is fast: XOR the character with 32 and the case flips.

#### Python ASCII Conversion Script

```python
# Convert characters to ASCII codes
ascii_values = {char: ord(char) for char in ['A', 'B', 'C', 'a', 'b', 'c']}
print(ascii_values)

# Convert ASCII codes back to characters
characters = {value: chr(value) for value in ascii_values.values()}
print(characters)
```

**Output:**
```
{'A': 65, 'B': 66, 'C': 67, 'a': 97, 'b': 98, 'c': 99}
{65: 'A', 66: 'B', 67: 'C', 97: 'a', 98: 'b', 99: 'c'}
```

![Python script output showing ASCII conversion results](screenshots/python_ascii_output.png)



#### Extended ASCII

Standard ASCII (0-127) uses 7 bits and covers only the English alphabet plus basic punctuation and control characters. This was designed in America and reflected American computing needs. A French developer trying to name a file `resumé.txt` would get an error because the character é had no numeric representation in the system.

Extended ASCII uses the 8th bit of a byte to unlock slots 128 through 255.

| Extended Character | ASCII Code |
|-------------------|-----------|
| Ç | 128 |
| Â | 131 |
| é | 130 |
| ¥ | 190 |
| ■ | 254 |

This matters in forensics. File names with extended characters appear in evidence from non-English systems. An investigator who does not understand extended ASCII will misread those file names in a hex dump. The character ¥ stored on disk is the byte 0xBE (190 in decimal). Knowing that mapping is the difference between reading evidence correctly and missing it.

### Part 4: Disk Drives and File Systems

#### Disk Geometry Calculations

Physical hard drives organize storage as cylinders, heads (read/write surfaces), and sectors. Every sector holds 512 bytes. Total capacity is a direct product of these three dimensions.

**Given parameters:**
- Sectors per track: 400
- Heads: 12
- Cylinders: 17,000

**Total sectors:**
```
Total Sectors = Cylinders × Heads × Sectors per Track
= 17,000 × 12 × 400
= 81,600,000 sectors

Total capacity = 81,600,000 × 512 bytes = 41,779,200,000 bytes ≈ 38.9 GB
```

**How geometry affects performance:** A hard disk drive is a mechanical device. The read/write head must physically move to the correct cylinder (seek time) and then wait for the disk to rotate until the correct sector passes underneath (rotational latency). These are measured in milliseconds, which is an eternity compared to electronic operations.

Data stored contiguously on the same track reads at maximum speed because the head stays in place while the disk rotates through the entire file. Fragmented data scattered across multiple cylinders forces the arm to physically move back and forth for every fragment. On a heavily fragmented drive, these mechanical movements add up to significant read delays.

This is also why SSDs are faster. They have no moving parts, so seek time is effectively zero. The cylinder/head/sector geometry concept does not apply to SSDs, which is why forensic tools use logical block addressing (LBA) rather than physical geometry when working with modern drives.

#### File System Comparison

A file system is the organizational layer between raw disk sectors and the user's concept of files and folders. Without it, data is stored as an unstructured block of bytes with no way to locate where one file ends and another begins.

| File System | Best Use | Max File Size | Strengths | Weaknesses |
|-------------|----------|--------------|-----------|------------|
| FAT32 | USB drives, external media | 4 GB | Near-universal compatibility (Windows, Mac, Linux, consoles, TVs) | 4 GB file size limit makes it useless for modern hard drives |
| NTFS | Windows internal drives | 16 Exabytes | File permissions (ACLs), native encryption, journaling prevents corruption | Mac can read but not write without third-party software |
| EXT4 | Linux servers, Android | 16 Terabytes | Fast with large numbers of small files, journaling, Linux permissions | Windows and Mac cannot read it without additional software |

**Forensic relevance of file systems:** The file system type determines what forensic artifacts are available. NTFS keeps a Master File Table (MFT) that records every file ever created, including deleted ones. EXT4 maintains an inode table. FAT32 keeps a File Allocation Table. Each structure preserves different amounts of metadata after deletion. An NTFS MFT entry often retains timestamps, file size, and partial path information even for files deleted months earlier.

### Part 5: Computer Forensics and System Verification

#### System Verification with lscpu

`lscpu` reads processor information directly from the kernel. In a forensic investigation, this command establishes the exact hardware profile of the system under examination.

```bash
lscpu
```



![lscpu output showing processor details and virtualization information](screenshots/lscpu_output.png)



**Forensically relevant findings:**

**Architecture: x86_64** confirms the system runs 64-bit instructions. This affects which malware samples could execute on this machine. A 32-bit malware binary will not run natively on a 64-bit-only system, and vice versa. When analyzing a compromised machine, the architecture narrows the field of possible attack tools.

**Model: Intel Core i5-4300U** identifies the exact physical processor. This matters when examining CPU-level vulnerabilities. Some exploits like Spectre and Meltdown are processor-family-specific. Knowing the exact CPU tells the investigator whether certain hardware-level attacks were possible.

**CPU(s): 1** shows only one logical processor is allocated. A single CPU means no parallel processing advantage for intensive forensic operations like full-disk encryption cracking. This also indicates the VM was not allocated significant resources.

**Hypervisor vendor: KVM** is the most significant finding. KVM (Kernel-based Virtual Machine) confirms the system is running inside a virtual machine. This has direct forensic implications: the VM's disk image is a file on the host machine, the VM's memory can be inspected by the hypervisor, and any actions taken inside this VM are isolated from the host. During an investigation, discovering a suspect used VMs means the real evidence may be on a host machine that was not seized.

#### Challenges in Computer Forensics

**Data Recovery:** When a file is deleted, the operating system removes its entry from the file system index. The raw binary data remains in place on disk until the space is allocated to a new file and overwritten. This is why deletion is recoverable. The challenge is timing. On a heavily used system, the freed sectors may be overwritten within minutes. On a system that was powered off immediately after deletion, the data can survive for years.

File carving addresses this. Tools like Autopsy scan unallocated disk space for file signatures (magic numbers). A JPEG always starts with the bytes `FF D8 FF`. A PDF always starts with `%PDF`. Autopsy identifies these signatures in raw sector data and reconstructs the file even without a file system entry pointing to it.

**Integrity Verification and Chain of Custody:** Plugging a seized drive into a standard computer is enough to corrupt evidence. Windows will immediately update the last-access timestamps on files as it reads the directory. These timestamp changes alter the SHA-256 hash of the drive. If the hash changes, the defense can argue the evidence was tampered with and have it thrown out.

The solution is a write blocker, a hardware device that sits between the drive and the forensic workstation. It allows read operations but physically blocks any write commands. FTK Imager creates a bit-for-bit clone of the drive through the write blocker and generates a hash of the original. Every analysis is performed on the clone. The original drive's hash is verified before and after to prove it was never altered. This hash verification is the mathematical proof that maintains chain of custody.

#### Forensic Tools

**FTK Imager:** Creates forensic disk images in formats like E01 or raw DD. Calculates MD5 and SHA-256 hashes of both the source drive and the resulting image to verify they are identical. Used with a hardware write blocker to ensure the source drive is never modified during acquisition.

**Autopsy:** Open-source forensic analysis platform. Parses NTFS MFT entries to recover deleted file metadata, performs file carving on unallocated space using magic number signatures, reconstructs browser history and email artifacts, and builds timelines from file system timestamps.

## Findings

**Binary and hexadecimal are not abstract math. They are the formats computers actually use to store and process every piece of data.** An investigator reading a hex dump of a disk sector is reading the raw bytes directly. Without number system fluency, that data is unreadable.

**Unix timestamps are the forensic standard for timeline reconstruction.** They are timezone-agnostic integers that sort correctly. Any log entry from any system anywhere in the world can be converted to Unix time and sorted into a single accurate timeline.

**Disk geometry determines both capacity and performance for mechanical drives.** Total sectors = Cylinders × Heads × Sectors per Track. Contiguous data reads fast. Fragmented data forces the mechanical arm to move repeatedly, degrading read speed measurably.

**File system choice determines what forensic artifacts survive deletion.** NTFS MFT retains rich metadata. FAT32's simpler allocation table retains less. EXT4's inode structure works differently again. The investigator needs to know which file system they are working with before starting recovery.

**Standard and extended ASCII are the translation layer between bytes and text.** Forensic tools that display file names, log entries, or message content are all performing ASCII lookups on raw bytes. Extended characters from non-English systems must be recognized correctly to avoid misreading evidence.

**The KVM hypervisor entry in lscpu reveals the machine is a VM.** This is the kind of finding that redirects an investigation. The real evidence is not inside the VM. It is the VM's disk image file sitting on the host machine.

## Challenges Faced

**Unix timestamp leap year drift:** When manually converting Unix timestamp 1626258000, I divided the remaining 1,290 days by 365 to count remaining years. This did not account for 2020 being a leap year (366 days instead of 365). The result drifted off by one calendar day. I corrected this by walking through each year individually, applying 366 days for 2020 and 365 for all others. The lesson here is that calendar arithmetic looks simpler than it is, which is exactly why computers store time as raw integers and apply calendar rules only at display time.

**Extended ASCII value verification:** The extended ASCII codes listed in the lab (Ç=128, é=130, Â=131) are specific to Code Page 437 (the original IBM PC encoding). These values differ in other encodings like Windows-1252 or ISO-8859-1. In forensic work, the encoding matters. The same byte value maps to different characters depending on which code page the original system used.

## Key Takeaways

**Number systems are forensic reading skills.** A memory address like 0x0804842B, a file signature like FF D8 FF, or a hash like 5d41402abc4b2a76b9719d911017c592 are all hex. Reading them fluently is the difference between understanding evidence and staring at noise.

**The Unix Epoch is the forensic standard for time.** All log correlation, all timeline reconstruction, and all timestamp verification in investigations starts with converting to Unix time first. Regional date formats exist for human interfaces, not for evidence handling.

**Deletion is not erasure.** Removing a file from the file system index leaves the binary data intact on disk until the sectors are overwritten. File carving recovers files by scanning for magic number signatures in unallocated space. The window for recovery closes when new data overwrites those sectors.

**Write blockers and hash verification are non-negotiable in forensics.** Without them, the act of examining evidence changes it. With them, every analysis is performed on a verified clone while the original drive's integrity is preserved mathematically for court.

**Knowing which file system you are dealing with determines your recovery strategy.** NTFS, EXT4, and FAT32 each organize data differently and retain different amounts of metadata after deletion. The right tool for NTFS recovery may be useless against an EXT4 partition.

**Hypervisor detection changes the scope of an investigation.** A KVM entry in lscpu means the analyzed machine is a guest VM. The actual storage, memory, and logs exist on the host. Missing the host machine means missing most of the evidence.

## Disclaimer

This lab was performed in a controlled virtual machine environment for educational purposes as part of the ICDFA Basic Computer Skills for Digital Forensics course. No real investigative activity was conducted against live systems.