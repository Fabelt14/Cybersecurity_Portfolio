# Digital Forensics Investigation of a USB Drive

## Overview

This lab covered the foundational principles of digital forensics and applied them to a practical case using Autopsy. The investigation centered on a USB drive image suspected of containing evidence related to a cybersquatting and extortion scheme. The goal was to recover deleted files, search for relevant keywords, and analyze the recovered evidence to determine whether a crime had occurred.

## Objectives

- Understand the role and purpose of digital forensics in legal and corporate investigations
- Outline the four phases of a digital investigation: identification, preservation, analysis, and presentation
- Load a forensic disk image into Autopsy for examination
- Recover deleted files from unallocated disk space
- Conduct targeted keyword searches across the disk image
- Analyze recovered documents to establish intent and reconstruct the timeline of events

## Lab Environment

- **Forensic Workstation:** Autopsy 4.23.1
- **Evidence Source:** USB drive disk image (Ch01InChap01.dd)
- **Case Name:** ICDFA USB Drive

## Tools Used

- Autopsy (disk image analysis and file recovery)

## Methodology

### Part 1: Foundational Concepts

Before touching the evidence, the lab required outlining the four phases of a digital investigation. This matters because skipping a phase, or doing it out of order, can make evidence inadmissible in court.

- **Identification:** This phase locates every potential source of evidence before anything is touched. It includes finding relevant devices, noting volatile data that disappears when power is cut (RAM, running processes), photographing the scene before disturbing anything, and confirming legal authority to search.

- **Preservation:** This phase locks down the evidence so it cannot be altered. Mobile devices go into Faraday bags to block remote wipe commands. Hardware write-blockers attach to storage media so the investigator's machine cannot accidentally modify the original. A forensic image is taken bit-by-bit, and all analysis happens on the copy, never the original. Every person who touches the evidence gets logged in a chain of custody, and a hash value (SHA-256) is calculated immediately to prove the data has not changed since acquisition.

- **Analysis:** This phase is where the actual investigative work happens. Deleted files get recovered, unallocated space gets carved for fragments, timelines get reconstructed from system events, and keyword searches surface relevant documents. Encrypted files may need to be cracked, and evidence across multiple devices gets cross-referenced for consistency.

- **Presentation:** Technical findings mean nothing if a judge or jury cannot understand them. This phase translates hex dumps and metadata timestamps into plain language, documents every tool and step taken, and presents findings without speculation or emotional language. If the case goes to court, the investigator may need to defend the process under cross-examination.

### Part 2: Loading the Evidence

I loaded the USB drive disk image (Ch01InChap01.dd) into Autopsy as the data source for a new case named "ICDFA USB DRIVE".

![Autopsy interface showing the loaded disk image as a data source](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/45_01%20Autopsy%20interface%20showing%20the%20loaded%20disk%20image%20as%20a%20data%20source.jpg)

Working from a disk image rather than the physical drive is standard practice. The image is an exact bit-for-bit copy, so any analysis I run cannot alter the original evidence. If I make a mistake or need to start over, I load a fresh copy of the same image.

### Part 3: File Recovery

I used Autopsy's File Views and Deleted Files panel to search for recoverable Word documents and images. The scan identified four deleted files total, but only two of them were Word documents worth recovering. No image files were found on the drive.

![Autopsy file system view showing four deleted files](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/45_02%20Autopsy%20file%20system%20view%20showing%20four%20deleted%20files.jpg)

The two recoverable Word files were:

- **Billing Letter.doc**
- **Regrets.doc**

I extracted both files and saved them to my local machine for closer review.

![Recovered Billing Letter.doc and Regrets.doc file icons](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/45_03%20Recovered%20Billing%20Letter.doc%20and%20Regrets.doc%20file%20icons.jpg)

The fact that these files were deleted rather than just sitting in a normal folder is the first signal something is worth investigating. People do not usually delete routine business letters.

### Part 4: Keyword Search

With two suspicious files recovered, I ran a keyword search for "George Montgomery" across the entire disk image to find every file referencing that name, not just the two I had already pulled.


![Autopsy keyword search results for "George Montgomery"](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/45_04%20Autopsy%20keyword%20search%20results%20for%20George%20Montgomery.jpg)

The search returned 5 results, with 4 of them being Word documents. This told me the name appeared in more places than just the two files I had already recovered, including unallocated space fragments (Unalloc_4_121344_1474560) and carved file remnants. This confirmed George Montgomery's name was tied to a broader pattern of activity on the drive, not an isolated incident.

### Part 5: Document Analysis

I opened both recovered files inside Autopsy to read the extracted text and understand what each document actually said.

**Billing Letter.doc** was dated October 13, 2005, and addressed to Laura Roper at an address in Seattle, WA. George Montgomery, claiming to represent "IT Connection Servers," wrote that Laura had chosen to register a domain (www.laurasstuff.com) through his company. He demanded a $500 fee be sent as a check or money order to a physical address in Bellevue, WA. The letter included an explicit threat: if Laura failed to pay within 30 days, ownership of the domain would revert to George, who would then sell it "to the highest bidder."



![Extracted text of Billing Letter.doc showing the extortion demand](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/45_06%20Extracted%20text%20of%20Billing%20Letter.doc%20showing%20the%20extortion%20demand.jpg)



**Regrets.doc** was dated November 2, 2005, and addressed to Randall Watson in Des Moines, WA. In this letter, George Montgomery informed Watson that all five domain name variations Watson had requested were "already purchased by someone else."

![Extracted text of Regrets.doc showing the domain purchase notification](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/45_07%20Extracted%20text%20of%20Regrets.doc%20showing%20the%20domain%20purchase%20notification.jpg)



### Analyzing the Significance

- Reading both letters side by side revealed the pattern. George Montgomery was running a domain cybersquatting operation: registering domain names that matched other people's names or businesses, then using the threat of selling those domains to third parties as leverage to extract payment. Billing Letter.doc shows the extortion attempt against Laura Roper directly. Regrets.doc shows the same individual operating the same scheme against a second victim, Randall Watson, where the domains in question had already been claimed.

- The deletion of both files is what elevates this from a business dispute to evidence of intent. A legitimate business invoice gets saved and kept for records. An extortion letter gets written, sent, and then deleted from the drafting machine to remove the paper trail. The act of deletion itself demonstrates George Montgomery understood the letters documented illegal conduct and tried to destroy that evidence. Forensic recovery defeated that attempt.

## Findings

- **Two extortion-related letters were recovered from deleted space on the USB drive.** Billing Letter.doc and Regrets.doc both originate from George Montgomery and both relate to a cybersquatting scheme targeting domain name owners.

- **The keyword search for "George Montgomery" returned five hits across four Word documents,** indicating the name and associated activity appear in more locations on the drive than the two primary letters, including unallocated space fragments.

- **Billing Letter.doc contains a direct extortion demand,** requesting $500 from Laura Roper with an explicit threat to sell her domain if payment was not received within 30 days.

- **Regrets.doc confirms a pattern of repeated conduct,** showing George Montgomery operating the same domain-related scheme against a second target, Randall Watson.

- **Both files were deliberately deleted,** which is itself evidentiary. Deletion of documents tied to a threat or demand for payment supports an inference of guilty knowledge, since routine correspondence is typically retained rather than destroyed.

## Challenges Faced

- **Distinguishing relevant deleted files from noise:** Autopsy flagged four deleted files total, but only two turned out to be substantively relevant Word documents. I had to open and read each one rather than assuming all deleted files were equally important to the case.

- **Interpreting metadata timestamps correctly:** The file system view displayed multiple timestamp fields (Modified, Change, Access, Created), and some showed 0000-00-00 values for Change Time. Reconciling which timestamp actually reflected when each letter was written versus when it was deleted required cross-referencing the dates inside the letter text itself (October 13, 2005 and November 2, 2005) against the file system metadata.

## Key Takeaways

- **Working from a forensic image protects the integrity of the original evidence.** Every recovery and search operation in this lab ran against a copy, never the source drive. This is non-negotiable in real investigations because any modification to original evidence can get it thrown out in court.

- **Deletion is not erasure.** Both Word documents were fully recoverable from a drive where someone had actively tried to remove them. Standard file deletion only removes the file system pointer; the underlying data remains until overwritten. This is why digital forensics exists as a discipline.

- **Keyword searching surfaces connections that file recovery alone misses.** Recovering two named files told me there were two letters. Searching for the name across the entire image told me the activity was broader than those two documents, appearing in carved fragments and unallocated space as well.

- **The act of deletion can itself become evidence.** In this case, the contrast between what gets kept (routine records) and what gets deleted (extortion demands) was the strongest indicator of criminal intent. A forensic investigator's job is not just to recover files, but to interpret what the pattern of deletion reveals about the suspect's state of mind.

- **Technical findings need translation for non-technical audiences.** Reconstructing this case meant explaining metadata, hash verification, and recovery methods in a way a non-technical stakeholder (a judge, a corporate board) could follow, while keeping every claim grounded in what the evidence actually showed.

## Disclaimer

This lab was performed using a forensic training disk image provided for educational purposes within a controlled academic environment. All names, addresses, and case details belong to a fictional training scenario. No real individuals or unauthorized systems were investigated.
