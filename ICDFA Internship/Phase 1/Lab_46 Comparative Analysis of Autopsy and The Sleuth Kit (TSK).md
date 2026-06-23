# Comparative Analysis of Autopsy and The Sleuth Kit (TSK)

## Overview

This lab compared two digital forensics tools that work together but serve different purposes: The Sleuth Kit (TSK), a collection of command-line utilities for low-level disk analysis, and Autopsy, a graphical interface built on top of TSK. The investigation involved a real USB drive image containing deleted files. The goal was to understand how deleted files survive on disk, how forensic tools locate them, and which recovery method works best for a given situation.

## Objectives

- Understand the architectural differences between Autopsy and TSK
- Map the layers of TSK's architecture to how data is physically stored on disk
- Identify deleted files within a disk image using command-line tools
- Recover deleted file content using three distinct recovery approaches
- Compare the speed and precision trade-offs between icat, blkcat, and blkls

## Lab Environment

- **Analysis Machine:** Kali Linux
- **Working Directory:** ~/Forensics
- **Disk Image:** Ch01InChap01.dd (USB drive image)
- **File System:** FAT (identified from $FAT1, $FAT2, $MBR entries in the image)

## Tools Used

- fls (file listing for disk images)
- istat (inode statistics and metadata inspection)
- icat (inode-based file content extraction)
- blkcat (block/sector content extraction, unit by unit)
- blkls (block list extraction across a sector range)
- Autopsy (GUI forensics platform)

## Methodology

### Part 1: Understanding the Tools

#### **Autopsy vs The Sleuth Kit**

- TSK operates entirely through the terminal. An investigator runs individual commands, specifies exact parameters, and sees raw output. This gives precise, scriptable control over every aspect of disk analysis, which is valuable for automation, custom workflows, and situations where a GUI is unavailable (like a headless server or remote forensic environment).

- Autopsy wraps TSK in a point-and-click interface. It automates bulk tasks like keyword indexing, timeline generation, and report writing that would require running dozens of separate TSK commands manually. The GUI makes it faster to get an overview of a disk's contents, but the underlying analysis engine is still TSK.

The relationship matters in practice: if Autopsy produces a finding, an investigator can replicate and verify it using raw TSK commands. This is important for court admissibility, where the methodology must be reproducible.

#### **TSK Architecture Layers**

TSK is organized as a stack of layers, each handling a different level of abstraction between raw physical storage and human-readable file names.

- The **Base Layer** provides the programming foundation. It handles memory management, common data types, and error handling functions that every other layer depends on. Investigators never interact with this layer directly.

- The **Disk Image Layer** sits directly above the physical media. It opens and reads raw disk images regardless of format, whether that is a plain dd image, an Expert Witness Format (EWF/E01) image, or a split image spread across multiple files. This layer hides the complexity of different image formats from everything above it.

- The **Volume System Layer** interprets the partition layout of the disk. It reads the Master Boot Record (MBR) or GUID Partition Table (GPT) to identify where each partition starts and ends. This is how TSK can tell that a 500GB drive contains three separate partitions before it even looks at files.

- The **File System Layer** moves inside a specific partition and identifies the file system type. Whether the partition uses FAT32, NTFS, Ext4, or HFS+, this layer parses the file system metadata to understand the overall structure, including cluster size, total sectors, and journal location.

- The **Content/Data Layer** works with the raw storage blocks inside the file system. It reads the clusters or blocks where actual file data lives. This is the layer blkcat and blkls operate at.

- The **Metadata Layer** handles inodes (Linux/Unix) and MFT entries (NTFS). Each file has a metadata record that stores its size, timestamps (created, modified, accessed), permissions, and pointers to the data blocks containing the file's content. Critically, when a file is deleted, the OS marks the metadata record as unallocated but does not immediately destroy it. TSK reads these unallocated records to find deleted files.

- The **File Name Layer** translates numeric metadata entries into human-readable names and directory paths. This is the layer fls operates at, showing investigators the actual file names (including deleted ones) rather than raw inode numbers.

### Part 2: Examining the USB Disk Image

#### **Listing All Files Including Deleted Ones**

The first task was to understand what the disk image contained. The `-r` flag tells fls to recurse through all directories rather than stopping at the root.

```bash
fls -r Ch01InChap01.dd
```

![fls output showing all files and deleted entries marked with asterisk](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/46_01%20fls%20output%20showing%20all%20files%20and%20deleted%20entries%20marked%20with%20asterisk.jpg)

The output showed both active and deleted files. TSK marks deleted files with an asterisk (*) in the output. The disk image contained four deleted files:

- Inode 8: Billing Letter.doc (shown as _ILLIN~1.DOC, a FAT short name artifact)
- Inode 11: confirmation.txt
- Inode 15: letter1.txt
- Inode 17: Regrets.doc

The active files were Client Info.mdb and income.xls. The v/v entries at the bottom ($MBR, $FAT1, $FAT2, $OrphanFiles) are FAT file system structures, not user files.

- **Why the files still exist:** Deleting a file on a FAT file system removes the file name from the directory listing and marks the first character of the file name entry as 0xE5 (a special byte indicating "deleted"). The metadata record pointing to the data blocks is marked as unallocated. The actual data blocks are not wiped. TSK reads the raw metadata layer and finds these records before the OS had a chance to overwrite them with new data.

#### **Recovering letter1.txt Using icat**

With the inode number (15) from the fls output, I ran icat to extract the file content directly:

```bash
icat Ch01InChap01.dd 15 > recovered_letter1.txt
```

![icat command execution and ls showing recovered_letter1.txt in directory](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/46_02%20icat%20command%20execution%20and%20ls%20showing%20recovered_letter1.txt%20in%20directory.jpg)

The `>` operator redirects icat's output from the terminal into a new file. Running `ls` afterward confirmed the file appeared in the working directory. The recovery was immediate because icat handles all the block-level work automatically.

### Part 3: Demonstrating Three Recovery Methods

#### **Method 1: blkcat (Sector-by-Sector Recovery)**

blkcat reads the content of one data block at a time. Before using it, I needed to know which sectors the deleted file occupied. istat provides this by reading the metadata record for a given inode.

```bash
istat Ch01InChap01.dd 8
```

![istat output for inode 8 showing file size, MAC times, and sector list](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/46_03%20istat%20output%20for%20inode%208%20showing%20file%20size%2C%20MAC%20times%2C%20and%20sector%20list.jpg)

The istat output revealed the metadata for inode 8 (Billing Letter.doc):

- Status: Not Allocated (confirms it is deleted)
- File Attributes: Archive
- Size: 24064 bytes
- Written: 2005-12-09 06:50:28 EST
- Accessed: 2003-12-09 00:00:00 EST
- Created: 2005-12-09 06:59:05 EST
- Sectors: 237-283 (the full list of sectors containing the file's data)

With the sector list, I used blkcat to read individual sectors:

```bash
blkcat Ch01InChap01.dd 237
blkcat Ch01InChap01.dd 240
blkcat Ch01InChap01.dd 241
```

![blkcat output showing file content fragment from sector 240](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/46_04%20blkcat%20output%20showing%20file%20content%20fragment%20from%20sector%20240.jpg)

- The output from sector 240 showed readable text: a letter about domain registration, mentioning a fee of $500, a contact at www.lauras_stuff.com, and a phone number for George Montgomery. Sector 241 showed an email address. Each blkcat call returns only the content of that one sector.

- **The limitation:** The file spans sectors 237 through 283, which is 47 sectors. To recover the full file using blkcat, I would need to run the command 47 times and concatenate the results. This is precise but slow. blkcat is most useful for inspecting specific sectors of interest rather than recovering complete files.

#### **Method 2: blkls (Range-Based Recovery)**

blkls improves on blkcat by accepting a sector range and returning all content within that range in one operation.

```bash
blkls Ch01InChap01.dd 240-258
```

![blkls output showing full readable content from the sector range](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/46_05%20blkls%20output%20showing%20full%20readable%20content%20from%20the%20sector%20range.jpg)

- The output showed the complete readable portion of the letter in one pass: the domain registration message, the FTP address (ftp.acmeserver.com), login credentials (login id: laura.roper, password: 342rroiu9), and a sign-off from George. This is the same content blkcat would have returned across multiple calls, but blkls retrieved it in a single command.

- **The limitation:** blkls still requires running istat first to get the sector range. If the file is fragmented across non-contiguous sectors, blkls would need multiple calls with different ranges. icat handles fragmentation automatically.

#### **Method 3: icat (Inode-Based Recovery, Fastest)**

icat operates at the Metadata Layer rather than the Data Layer. It reads the inode record, which already contains the complete list of all data blocks belonging to the file, including fragmented blocks scattered across the disk. It then reads all of them and assembles the complete file automatically.

```bash
icat Ch01InChap01.dd 11 > confirmation.txt
cat confirmation.txt
```

![icat recovery of confirmation.txt showing full file content](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/46_06%20icat%20recovery%20of%20confirmation.txt%20showing%20full%20file%20content.jpg)

- The recovered confirmation.txt content showed a letter addressed to Laura, confirming that her domain had been registered and providing FTP login credentials for file upload. The content included a username (laura.roper) and password (342rroiu9), demonstrating that deleted files can contain sensitive information that persists on disk long after deletion.

- Running icat took one command and recovered the complete file without needing istat or sector ranges. This speed advantage matters in real investigations where analysts may need to recover dozens of files.

### **Comparison summary:**

- blkcat reads one sector at a time. Requires istat to find sector addresses. Useful for targeted inspection of specific disk locations. Not practical for full file recovery.

- blkls reads a sector range in one pass. Still requires istat. Better than blkcat for recovery but only works cleanly for contiguous data.

- icat reads the inode, finds all blocks automatically, reassembles fragmented files, and requires only the inode number from fls. Fastest and most reliable method for file recovery.

## Findings

- **Four deleted files existed in the disk image.** TSK's fls command identified them by reading unallocated metadata records that the operating system had not yet overwritten. The asterisk marker and inode numbers (8, 11, 15, 17) provided the entry points for recovery.

- **Deleted files retain their original content until overwritten.** The recovered files contained real readable text including email addresses, domain registration details, FTP server credentials, and personal names. From a forensics perspective, this means that deleting a file does not make it unrecoverable. From a security perspective, it means sensitive data must be actively wiped, not just deleted.

- **MAC timestamps survived deletion.** istat showed the original Written, Accessed, and Created timestamps for inode 8, dating to 2003-2005. These timestamps are stored in the metadata layer and remain intact even after the file is deleted. This is critical for building timelines in forensic investigations.

- **icat is superior to blkcat and blkls for file recovery.** It requires fewer steps, handles file fragmentation automatically, and produces complete output in one command. blkcat and blkls have value for low-level sector inspection and understanding how data is physically laid out on disk, but icat is the practical choice for recovery work.

- **FAT file systems provide minimal deletion protection.** FAT marks deleted entries with a 0xE5 byte and marks data blocks as available for reuse, but it does not zero the data. NTFS with the Secure Delete option or tools like shred are required to make file recovery genuinely difficult.

## Challenges Faced

- **The question selection requirement was ambiguous.** The lab instructed me to use modulus operation (2158 mod 28 = 2) to identify specific questions from a numbered list, but no numbered list of 28 questions was included in the lab materials. Rather than skip this section, I treated the requirement as demonstrating three distinct recovery tools and documented icat, blkcat, and blkls in full. If the question list becomes available, the specific answers can be added.

- **blkcat output required careful tracking across multiple calls.** When reading a file sector by sector, the content appears fragmented and out of context. Sector 237 showed binary-looking characters (the DOC file header), sector 240 showed readable text, and sector 241 showed part of an email address. Without the context from neighboring sectors, each individual blkcat call produces an incomplete picture. This taught me to use blkls or icat for actual recovery and reserve blkcat for targeted inspection of specific disk locations.

## Key Takeaways

- **Deleting a file does not delete the data.** The OS removes the pointer and marks the space as available. TSK reads the raw metadata layer and finds what the OS marked as gone. In a real investigation, this is where evidence of deleted documents, communication records, and credentials lives.

- **The layer you operate at determines what you can see and recover.** blkcat at the data layer shows raw sector content with no file context. istat at the metadata layer shows timestamps, file size, and block addresses. icat bridges metadata and data layers to recover complete files automatically. Understanding which layer each tool targets explains why they produce different outputs.

- **Timestamps are forensic evidence.** The MAC times on inode 8 showed the file was created in December 2005. Even after deletion, these timestamps survive in the unallocated metadata record. In real cases, timestamp analysis places a suspect's actions on a timeline.

- **Recovered credentials confirm why storage hygiene matters.** The confirmation.txt file contained a username and password in plaintext. Any forensic investigator, or an attacker with physical access to a discarded drive, could recover this. Organizations that decommission storage media without proper wiping leave this kind of information accessible.

- **Autopsy adds value for scale, TSK adds value for precision.** For a single disk with a handful of deleted files, TSK commands are fast and direct. For a 2TB drive with thousands of files, Autopsy's automated indexing and search saves hours. Both tools belong in a digital forensics workflow.

## Disclaimer

This lab was performed in a controlled educational environment using a disk image provided specifically for forensics training. No real personal data or production systems were accessed.
