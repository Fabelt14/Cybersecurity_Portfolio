# Disk Imaging Techniques: Acquisition Methods for USB Drives

## Overview

This lab covered forensic disk acquisition from USB flash drives using two industry-standard approaches: FTK Imager on Windows and the `dd` command on Linux. The goal was to produce bit-for-bit copies of a physical drive while maintaining forensic integrity, and to understand the trade-offs between different acquisition methods including network-based imaging.

A disk image is not a file copy. It captures every sector on the device, including deleted files, file system metadata, and unallocated space that normal file explorers never show. This distinction matters in forensics because the evidence you need is usually in exactly those hidden areas.

## Objectives

- Understand what a forensic disk image is and why investigators use it instead of working on live devices
- Acquire a USB drive image using FTK Imager on Windows
- Acquire a USB drive image using the `dd` command on Linux
- Compare acquisition methods and understand when each is appropriate
- Understand the advantages and limitations of network-based acquisition
- Design a forensic research plan for future recovery exercises

## Lab Environment

- **Windows Machine:** Host machine running Windows with FTK Imager installed
- **Linux Machine:** Kali Linux VM (PRIME_KALI) running inside VirtualBox
- **Evidence Drive:** 2GB virtual USB disk (\\PHYSICALDRIVE1 on Windows, /dev/sdb on Linux)
- **Image Destination (Windows):** C:\Users\user\Desktop\ICDFA\Forensic Space
- **Image Destination (Linux):** ~/usb_image.img
- **Network Type:** Local VirtualBox environment

## Tools Used

- FTK Imager (Exterro, v8.2.0.59 SP1) - Windows acquisition
- dd (Linux built-in) - command-line acquisition
- lsblk - USB drive identification on Linux
- VirtualBox - VM environment for Linux acquisition

## Methodology

### Task 1: Acquisition Using FTK Imager (Windows)

FTK Imager provides a guided GUI workflow for forensic acquisition. It automatically calculates hash values during imaging, which is what makes the resulting image admissible as forensic evidence. If the hash computed during acquisition matches the hash computed later during analysis, you can prove the image was never modified.

#### Step 1: Identify the target drive

After connecting the USB drive to the Windows machine, Windows automatically assigned it drive letter D:, showing 1.97 GB free of 1.99 GB.

![Windows File Explorer showing USB DRIVE (D:) connected alongside C: drive](images/ftk_usb_connected_windows.png)



The drive is visible but not yet touched. A forensic investigator never opens, browses, or writes to the evidence drive before imaging. Any interaction with the live device can update file access timestamps and change the evidence.

#### Step 2: Open FTK Imager and select Create Disk Image

After launching FTK Imager (v8.2.0.59 SP1), I navigated to File > Create Disk Image.



![FTK Imager main interface showing the Create Disk Image option in the File menu](images/ftk_imager_main_menu.png)



#### Step 3: Select the evidence source

FTK Imager presented a list of available physical drives. I selected \\PHYSICALDRIVE1, which corresponds to the 2GB virtual USB disk connected to the machine.

![FTK Imager Select Drive dialog showing PHYSICALDRIVE1 - Microsoft Virtual Disk 2GB SCSI selected](images/ftk_select_drive_physicaldrive1.png)



Selecting the physical drive rather than a logical partition is important. A logical partition (like D:) only sees the file system. The physical drive includes the Master Boot Record, partition table, and any data outside the partition boundaries, which is where investigators sometimes find evidence of tampering or hidden partitions.

#### Step 4: Choose image format and destination

FTK Imager offered four output formats: Raw (dd), SMART, E01, and AFF. I selected Raw (dd).



![FTK Imager image type selection dialog with Raw (dd) selected](images/ftk_select_image_type_raw.png)



**Raw format** produces a flat binary copy of the drive with no compression or encryption. Every bit is written exactly as read. This format is universally compatible with any forensic tool. The downside is that the output file is the same size as the source drive.

**E01 format** is the Expert Witness Format used by EnCase. It supports compression, encryption, and built-in hash verification inside the file container. It is the standard for professional forensic cases because the image file itself stores integrity verification data.

I set the destination to C:\Users\user\Desktop\ICDFA\Forensic Space and named the image usb_image. Fragment size was set to 0 (do not fragment), compression was set to 0 (none), and AD Encryption was left disabled.



![FTK Imager image destination configuration showing folder path, filename usb_image, and fragment settings](images/ftk_image_destination_config.png)



The case metadata filled in at this stage:
- Case Number: INT313-3
- Evidence Number: 003
- Unique Description: Acquiring a USB Image
- Examiner: Fatai Asekun
- Notes: Image type: dd

This metadata gets embedded in the log file that FTK generates alongside the image, creating a documented chain of custody.

#### Step 5: Run the acquisition and verify hashes

FTK Imager ran the acquisition and reported completion in 2 minutes 16 seconds. The progress bar turned green with "Image created successfully."



![FTK Imager acquisition progress showing Image created successfully status](images/ftk_acquisition_complete.png)



Immediately after imaging, FTK automatically ran hash verification and displayed the results:



![FTK Imager Drive/Image Verify Results showing MD5 and SHA1 hash comparisons](images/ftk_hash_verification_results.png)



**MD5 hash:**
- Computed hash: e93c6881042732af69b93a724b927950
- Report hash: e93c6881042732af69b93a724b927950
- Verify result: Match

**SHA1 hash:**

- Computed hash: 8308d30f455ced130c91fc0dc904d2fbeecf3753
- Report hash: 8308d30f455ced130c91fc0dc904d2fbeecf3753
- Verify result: Match

Both hashes match. This is the most important result in the entire lab. Hash verification proves that not a single bit changed between the source drive and the output image file. If these hashes ever differ, the image is compromised and cannot be used as forensic evidence.

The Forensic Space folder now contains two files: usb_image.001 (the raw image) and usb_image.001.txt (the case log generated by FTK).



![Forensic Space folder showing usb_image.001 and usb_image.001.txt files](images/ftk_output_files_forensic_space.png)



### Task 2: Acquisition Using dd (Linux)

The `dd` command is a Linux built-in that reads and writes data at the block level, making it suitable for forensic acquisition without installing any additional software. It has no GUI, but it is precise, scriptable, and available on any Linux system.

#### Step 1: Add the USB drive to the Linux VM

Before running the acquisition, I attached the virtual USB disk to the Kali Linux VM through VirtualBox settings. Under Storage, the linux_usb.vdi disk appeared as a second device on the SATA controller alongside the main Kali installation disk.



![VirtualBox Storage settings showing linux_usb.vdi attached to PRIME_KALI VM](images/virtualbox_storage_usb_attached.png)



#### Step 2: Identify the drive using lsblk

Before imaging, I needed to confirm exactly which device node corresponds to the USB drive. Running `lsblk` lists all block devices with their sizes and mount points:

```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0 80.1G  0 disk
└─sda1   8:1    0 80.1G  0 part /
sdb      8:16   0    2G  0 disk
sr0     11:0    1 58.5M  0 rom
```

![lsblk output with sdb highlighted as the 2GB USB disk](images/lsblk_output_sdb_identified.png)



`sda` is the 80.1GB main system disk with a single partition (sda1) mounted at /. `sdb` is the 2GB drive with no partitions and no mount point, which confirms it is the USB drive. `sr0` is the optical drive.

Getting this identification wrong is catastrophic. If I had typed `sda` instead of `sdb`, the command would have imaged the operating system disk, not the USB drive. Or if used as an output target, it would have overwritten the entire Kali installation.

#### Step 3: Run the dd acquisition

```bash
sudo dd if=/dev/sdb of=~/usb_image.img bs=4M status=progress
```



![Terminal showing dd command running with progress output](images/dd_command_progress.png)



**Breaking down the command:**

`if=/dev/sdb` - Input file. This points dd at the raw block device for the USB drive, reading every sector from start to finish regardless of what the file system contains.

`of=~/usb_image.img` - Output file. The resulting image goes to the home directory. This must be on a different drive from the source, never image a drive to itself.

`bs=4M` - Block size of 4 megabytes. dd reads and writes in chunks. Larger block sizes reduce the number of read/write operations and improve speed significantly. The default block size is 512 bytes, which is slow for large drives.

`status=progress` - Shows live transfer statistics so you can monitor the operation instead of staring at a blank screen.

**dd output:**
```
2092957696 bytes (2.1 GB, 1.9 GiB) copied, 103 s, 20.3 MB/s
512+0 records in
512+0 records out
2147483648 bytes (2.1 GB, 2.0 GiB) copied, 103.763 s, 20.7 MB/s
```

The acquisition completed in 103 seconds at an average of 20.7 MB/s, copying all 2,147,483,648 bytes (exactly 2GB) from the USB drive to the image file. `512+0 records in` and `512+0 records out` means dd processed 512 full blocks with no partial blocks and no errors.

**Critical difference from FTK Imager:** dd does not automatically calculate and verify hashes. To maintain forensic integrity with dd, you must run hash verification manually after acquisition:

```bash
md5sum /dev/sdb
md5sum ~/usb_image.img
```

If both outputs match, the image is an exact copy. This step is not optional in a real investigation.

### Task 3: Network-Based Acquisition

Network imaging acquires a disk image from a remote machine without physically transporting the drive. The forensic analyst runs imaging software on the remote suspect machine, and the image file is transmitted over the network to the analyst's storage.

**The primary advantage** is speed of response. If a bank in Lagos is breached and the suspect server is in Abuja, an investigator can begin acquisition immediately from Lagos instead of driving four to six hours. For volatile evidence like RAM contents or active network connections, every minute of delay means evidence loss.

**The practical problems:**

A 500GB hard drive transmitted over a 100 Mbps corporate network takes approximately 11 hours to transfer. During that time, the network link is saturated, which degrades performance for every other user and system on that network. Most corporate networks are not provisioned for this kind of sustained throughput.

Network instability compounds this problem. An interrupted acquisition over an unstable WAN link produces a partial image. Unlike a file download that can resume, dd-based acquisitions have no built-in resume capability. An interrupted acquisition means starting over from the beginning.

Tools like `netcat` combined with `dd` can pipe acquisition data across a network, but they require careful setup and ideally a dedicated out-of-band network link so investigation traffic does not compete with production traffic.

## Forensic Research Plan: Future Recovery Exercise

**Goal:** Demonstrate that deleted files remain recoverable from a formatted drive using forensic imaging and file carving.

**Phase 1 - Evidence creation:** Connect a physical 16GB flash drive and write three distinct text files simulating a harassment or insider threat scenario. Record exact file names, sizes, and creation timestamps before any modification.

**Phase 2 - Suspect behavior simulation:** Permanently delete all three files using Shift+Delete (bypassing the Recycle Bin), then perform a Quick Format on the drive. A Quick Format wipes the file system index (the table of contents) but does not overwrite the actual file data on disk. The data remains physically present in the unallocated space.

**Phase 3 - Forensic acquisition:** Use FTK Imager on Windows or `dd` on Linux to create a raw bit-for-bit image of the physical drive immediately after formatting. Compute and record MD5 and SHA1 hashes for the image.

**Phase 4 - File carving and recovery:** Load the image into a forensic tool and analyze the unallocated space. File carving works by searching for known file headers and footers (for example, a text file starts with recognizable byte patterns) and reconstructing the file from the raw sectors, even without a valid file system index pointing to it.

**Expected result:** All three deleted text files are recoverable from the unallocated space despite the Quick Format. This demonstrates why forensic imaging matters: the suspect believed the files were gone, but the data survived.

## Findings

**FTK Imager and dd both produced verifiable forensic images.** FTK's MD5 hash (e93c6881042732af69b93a724b927950) and SHA1 hash (8308d30f455ced130c91fc0dc904d2fbeecf3753) both showed Match between computed and report values, confirming the image is an exact copy of the source drive.

**dd transferred 2,147,483,648 bytes in 103 seconds at 20.7 MB/s** with 512+0 clean records in and out. No read errors or partial blocks occurred, indicating the source drive had no bad sectors during acquisition.

**Raw format captures everything.** Unlike a file copy that only transfers existing files, the raw image includes unallocated space (where deleted file data lives), slack space (unused bytes within allocated sectors), and file system metadata (directory entries, timestamps, inode tables). This is what makes forensic recovery possible.

**Network acquisition is viable but bandwidth-intensive.** The method is best suited for incident response scenarios where time matters more than network performance, and ideally over dedicated out-of-band links rather than shared corporate networks.

**Quick Format does not destroy data.** The formatting process overwrites the file system index but leaves the underlying sectors untouched. Forensic imaging captures those sectors, and file carving tools can reconstruct the deleted files from raw sector data.

## Challenges Faced

**Selecting the correct block device on Linux:** Running `lsblk` before imaging confirmed that `sdb` was the USB drive and `sda` was the system disk. The two drives differ in size (2GB vs 80.1GB), which made identification straightforward in this lab. In a real investigation with multiple similar-sized drives attached, a more careful approach using drive serial numbers (from `udevadm info /dev/sdb`) would be needed to avoid imaging the wrong device.

**dd provides no automatic hash verification:** FTK Imager built hash calculation into the acquisition workflow. With dd, the hash step is separate and easy to skip when working quickly. In a forensic investigation, skipping hash verification means the image cannot be authenticated in court. The discipline to always run `md5sum` on both source and image after dd must become habit.

**Understanding fragment size in FTK Imager:** The fragment size option (set to 0 for "do not fragment") controls whether FTK splits the output image into multiple smaller files. A 0 value produces a single image file. Setting it to 650 would split the image into 650MB chunks, useful for writing to optical discs. Leaving it at 0 was the correct choice for this lab since the destination had sufficient space.

## Key Takeaways

**Hash verification is the foundation of forensic integrity.** Without matching hashes before and after acquisition, no court will accept the image as evidence. Both MD5 and SHA1 matching confirmed that not a single bit was altered during the FTK acquisition. dd acquisitions require manual hashing, which makes the investigator's discipline the weak link.

**Physical drive selection versus logical partition selection determines what evidence you capture.** Selecting \\PHYSICALDRIVE1 in FTK gives you everything on the disk. Selecting drive D: gives you only what the file system exposes. Forensic evidence often lives outside what the file system shows, which is why physical drive acquisition is the standard.

**Block size significantly affects dd performance.** The default 512-byte block size would have made the 2GB acquisition take far longer. Using `bs=4M` cut acquisition time by reading and writing in 4MB chunks. For large drives in real investigations, block size tuning is part of the workflow.

**A Quick Format is not secure deletion.** This lab design proves the point that suspects who format a drive to hide evidence are not hiding anything from a trained investigator. The data survives in unallocated space and is recoverable through file carving. Secure deletion requires multiple passes of overwriting, not just a format.

**Tool choice depends on the situation.** FTK Imager is better for formal investigations because hash verification is automatic and chain of custody documentation is built in. dd is better for rapid field acquisition on Linux systems where FTK is not installed, and for scripting automated imaging across many machines.

## Disclaimer

This lab was performed in a controlled virtual environment using VirtualBox for educational purposes only. The USB drive imaged was a simulated virtual disk created specifically for this exercise. No unauthorized devices or systems were accessed.