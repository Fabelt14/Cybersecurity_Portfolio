# Linux Command Line Fundamentals - File Operations and Permission Management

## Overview

This lab focused on mastering the Linux command line for security work. The goal was to build practical skills in directory management, file permissions, and log analysis using only the terminal - no GUI shortcuts allowed.

## Objectives

- Create complex directory structures efficiently for penetration testing projects
- Manage file permissions using both symbolic and octal notation
- Analyze log files to extract security-relevant information
- Use command piping to combine tools for text processing
- Understand when to use different permission methods

## Lab Environment

- **OS**: Kali Linux
- **Interface**: Terminal (Bash shell)

## Tools Used

- mkdir - Directory creation
- touch - File creation
- chmod - Permission modification
- ls - Directory listing and permission verification
- find - File search
- grep - Pattern matching in text
- wc - Word/line counting
- cat - File viewing
- tr - Text transformation

## Methodology

### Challenge 1: Building a Penetration Testing Directory Structure

#### Step 1: Understanding the Requirement
I needed to create a directory structure for organizing a penetration testing engagement with multiple nested subdirectories.

**Target structure:**
```
Engagement_2024/
├── reconnaissance/
├── exploitation/
├── reporting/
└── evidence/
    ├── screenshots/
    ├── logs/
    └── packets/
```

Plus a readme.txt file in each main subdirectory.

#### Step 2: Evaluating Approaches
I considered two methods:

**Method 1 - Manual creation:**
```bash
mkdir Engagement_2024
cd Engagement_2024
mkdir reconnaissance
mkdir exploitation
mkdir reporting
mkdir evidence
cd evidence
mkdir screenshots
mkdir logs
mkdir packets
```

This works but requires 8 separate commands and multiple directory changes.

**Method 2 - Single command with -p flag:**
```bash
mkdir -p Engagement_2024/{reconnaissance,exploitation,reporting,evidence/{screenshots,logs,packets}}
```

**Why the second approach is better:**
- One command creates the entire structure
- The -p flag creates parent directories as needed
- Brace expansion {} handles multiple subdirectories at the same level
- Nested braces handle the evidence subdirectories
- Faster and less prone to typos

**Command used:**
```bash
mkdir -p Engagement_2024/{reconnaissance,exploitation,reporting,evidence/{screenshots,logs,packets}}
```

![pentest directory](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/pentest%20diretory.jpg)

#### Step 3: Adding README Files
I needed to create readme.txt in each of the four main subdirectories (reconnaissance, exploitation, reporting, evidence).

**Why this required a separate command:**
The mkdir command only creates directories, not files. I used touch with the same brace expansion pattern:

**Command used:**
```bash
touch Engagement_2024/{reconnaissance,exploitation,reporting,evidence}/readme.txt
```

**Verification:**
```bash
ls Engagement_2024/
cd Engagement_2024/evidence
ls
cd ../reporting
ls
```

![Add Readme](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/add%20readme.jpg)

Each directory contained its readme.txt file, confirming the structure was correct.

### Challenge 2: Managing Permissions with Octal Notation

#### Step 1: Making Shell Scripts Executable by Owner Only
I had multiple .sh files that needed execute permission for the owner, but no permissions for group or others.

**Why octal instead of symbolic:**
With symbolic notation (chmod u+x), I'd need to:
1. Add execute for owner: `chmod u+x *.sh`
2. Remove all group permissions: `chmod g-rwx *.sh`
3. Remove all other permissions: `chmod o-rwx *.sh`

With octal notation, I can set all three permission sets at once:
- Owner: execute only = 1
- Group: no permissions = 0
- Others: no permissions = 0

**Command used:**
```bash
cd Lab1
chmod 100 *.sh
ls -l *.sh
```

![executable only by the ownser](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/01%20exec%20by%20owner.jpg)

**Output showed:**
```
---x------ 1 kali kali 0 Dec 12 06:18 admin_tools.sh
```

The wildcard `*.sh` applies the permission to all files ending in .sh in one command.

#### Step 2: Setting Text Files Readable by All, Writable by Owner
Text files needed to be readable by everyone, but only the owner could modify them.

**Permission breakdown:**
- Owner needs: read (4) + write (2) = 6
- Group needs: read (4) = 4
- Others need: read (4) = 4

Combined: 644

**Initial attempt failed:**
```bash
chmod 644 *.txt
```

Got "Operation not permitted" for secret_file.txt.

**Why it failed:**
Running `ls -l *.txt` showed secret_file.txt was owned by root, not kali. Regular users can't change permissions on files they don't own.

![writable and readable by owner](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/01%20writable%20by%20owner.jpg)

**Solution:**
```bash
sudo chmod 644 *.txt
ls -l *.txt
```

**Result:** All .txt files now showed `-rw-r--r--`, confirming owner can read/write while group and others can only read.

#### Step 3: Creating a Completely Inaccessible File
I created a file with zero permissions - nobody can read, write, or execute it.

**Why this is useful:**
In security testing, you might want to verify that access controls are properly enforced. A 000-permission file tests whether applications or users can bypass permission checks.

**Commands used:**
```bash
touch private_file.txt
ls -l private_file.txt
chmod 000 private_file.txt
ls -l private_file.txt
```

![Inaccessible file](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/01%20inaccessible%20file.jpg)

**Before:** `-rw-r--r--`
**After:** `----------`

Even the owner (me) couldn't read this file without first using chmod to restore permissions or using sudo to override the restriction.

### Challenge 3: Log Analysis with Command Chaining

#### Step 1: Finding All Log Files
I needed to locate every .log file anywhere in my home directory.

**Why find instead of ls:**
`ls` only searches the current directory. The find command recursively searches subdirectories and can filter by filename patterns.

![Find Log Files](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/01%20find%20log%20files.jpg)

**Command used:**
```bash
find ~ -name "*.log"
```

**Output showed:**
```
/home/kali/Lab1/logs/application.log
/home/kali/Lab1/extracted/home/kali/Lab1/logs/application.log
/home/kali/.cache/mozilla/firefox/m0yjohd6.default-esr/cache2/index.log
/home/kali/.local/share/gvfs-metadata/home-026edf96.log
```

The `~` expands to my home directory path, and `-name "*.log"` matches any file ending in .log.

#### Step 2: Counting Error Lines and Saving Results
I needed to:
1. Find lines containing "ERROR" in logfile.txt
2. Count how many lines matched
3. Save the count to error_report.txt

**Breaking down the command chain:**
```bash
grep "ERROR" logfile.txt | wc -l > error_report.txt
```

![count error and save to file](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/01%20count%20and%20save.jpg)

**How each part works:**
- `grep "ERROR" logfile.txt` - searches logfile.txt for lines containing "ERROR."
- `|` - pipes grep output to the next command
- `wc -l` - counts lines (wc = word count, -l = lines only)
- `> error_report.txt` - redirects output to a new file

**Why piping is powerful:**
Instead of:
1. Running grep and manually counting lines
2. Typing the number into a file

I chained two commands together and automated the entire process.

**Verification:**
```bash
cat error_report.txt
```

Output: `2`

This confirmed logfile.txt contained exactly 2 lines with "ERROR".

**Viewing the actual errors:**
```bash
grep "ERROR" logfile.txt
```

Output:
```
ERROR: Authentication failed
ERROR: Connection timeout
```

## Findings

**Directory Management:**
- Brace expansion with mkdir -p created an 8-directory structure in one command
- Nested braces handled the evidence subdirectories (screenshots, logs, packets)
- Separate touch command required for files since mkdir only creates directories

**Permission Management:**
- Octal notation (chmod 100, 644, 000) sets all three permission groups simultaneously
- Symbolic notation (u+x, g-r) requires multiple commands for the same result
- Files owned by root require sudo for permission changes, even for read-only operations
- Wildcard (*.sh, *.txt) applies permissions to multiple files at once

**Log Analysis:**
- Find command located 4 .log files across multiple subdirectories
- Grep isolated 2 lines containing "ERROR" from logfile.txt
- Command piping (grep | wc) combined search and count in one operation
- Redirection (>) saved output to file without manual copying

**Text Processing:**
- The tr command transformed text case: `echo "hello world" | tr 'a-z' 'A-Z'` produced "HELLO WORLD"
- Head command extracted first 3 lines: `head -3 logfile.txt`
- Piping multiple commands created powerful one-liners for complex tasks

## Challenges Faced

**Permission denied on secret_file.txt:**
Initially couldn't change permissions on one text file because it was owned by root. This taught me that file ownership matters - even with sudo privileges, you can't modify permissions on files you don't own unless you use sudo.

**Understanding octal vs symbolic permissions:**
At first, symbolic notation (u+x, g-r) seemed easier to remember. After working through the challenge, I realized octal is faster when you know exactly what final state you want (100 = owner execute only, no guessing needed).

**Brace expansion syntax:**
The nested braces for `evidence/{screenshots,logs,packets}` were initially confusing. Testing with echo first helped: `echo Engagement_2024/{reconnaissance,evidence/{screenshots,logs}}` showed me the expansion before creating real directories.

## Key Takeaways

- **Efficiency matters in security work:** Creating directory structures for penetration tests manually wastes time. One mkdir -p command does in seconds what would take minutes of manual work.
- **Octal permissions are faster for known states:** When you know exactly what permissions you want (owner execute only = 100), octal is direct. Symbolic is better for modifications (add execute: u+x) when current state is unknown.
- **Wildcards enable batch operations:** Applying permissions to all .sh or .txt files at once prevents inconsistencies and saves time.
- **Piping is fundamental to Linux security work:** Log analysis, text processing, and data extraction all rely on chaining commands. `grep | wc` is simpler and more reliable than manual counting.
- **File ownership affects permissions:** Even root users need sudo to modify files owned by other accounts. This is a security feature, not a bug.
- **Command verification is critical:** Always check your work (ls -l, cat) after making changes. Permissions errors can lock you out of files or create security vulnerabilities.

## Disclaimer

This lab was performed in a controlled Kali Linux environment for educational purposes as part of the ICDFA Linux Operating Systems Fundamentals course. All activities were conducted on a local system with proper authorization.
