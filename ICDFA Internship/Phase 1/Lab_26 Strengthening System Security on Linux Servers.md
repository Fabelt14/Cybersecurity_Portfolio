```markdown
# Linux Server Security Hardening - Multi-Layer Access Control and System Auditing

## Overview

This lab covered comprehensive Linux server security hardening through multiple layers of access control. The goal was to configure system-wide security defaults, implement fine-grained file permissions with ACLs, manage sudo privileges by role, secure network services, implement multi-factor authentication, and enable system auditing for compliance and forensics.

## Objectives

- Modify system-wide user account defaults for consistent security policy
- Implement Access Control Lists (ACLs) for granular file permissions beyond standard Unix permissions
- Configure role-based sudo access to limit administrative privileges
- Harden network services and close unnecessary ports
- Implement two-factor authentication for critical accounts
- Configure SSH key-based authentication and disable password login
- Set password aging policies to enforce regular password changes
- Deploy system auditing to track security-relevant events

## Lab Environment

- **OS**: Kali Linux
- **User Privilege**: Root access via sudo
- **Working Directory**: /ICDFA/INT305, /etc, /projects
- **Network**: Local system with SSH, Apache2 services

## Tools Used

- nano - Configuration file editing
- adduser/useradd - User account creation
- usermod - User modification
- setfacl/getfacl - ACL management
- visudo - Sudo configuration
- ss - Socket statistics (port listing)
- systemctl - Service management
- ssh-keygen - SSH key generation
- ssh-copy-id - Public key deployment
- chage - Password aging configuration
- auditctl - Audit rule management
- ausearch - Audit log searching

## Methodology

### Part 1: Configuring System-Wide User Account Defaults

#### Step 1: Examining adduser.conf
I opened `/etc/adduser.conf` to understand the default settings applied to every new user account created on the system.

**Why this file matters:**
The adduser.conf file controls system-wide defaults like home directory location, UID/GID ranges, default shell, and skeleton files. Changing these settings once affects all future user creation, maintaining consistency across the system instead of manually configuring each user.

**Key finding:**
Default shell was `/bin/bash`, which is standard but can be changed to more restricted shells like `/bin/sh` for security-sensitive accounts.

**Command used:**
```bash
sudo nano /etc/adduser.conf
```

#### Step 2: Analyzing UID and GID Ranges
I checked the FIRST_UID, LAST_UID, FIRST_GID, and LAST_GID parameters to understand how user IDs are assigned.

**Why UID/GID ranges matter:**
Linux systems reserve UIDs 0-999 for system accounts (root is 0, system services are 1-999). Regular users start at 1000 by default. By customizing these ranges, you can segregate different user types (employees, contractors, service accounts) into different ID ranges for easier management and auditing.

**Initial values:**
- FIRST_UID=1000, LAST_UID=59999
- FIRST_GID=1000, LAST_GID=59999

I verified my own account (UID 1001) fell within this range, confirming I was the second regular user created.

#### Step 3: Changing Default Home Directory
I modified DHOME from `/home` to `/mnt/users`.

**Security reasoning:**
Separating user homes onto a different partition or mount point allows you to:
- Apply different filesystem security settings (noexec, nosuid flags)
- Implement disk quotas independently from system directories
- Simplify backups by targeting one location
- Prevent users from filling the root partition

**Note:** This only affects new users. Existing user homes remain in `/home`.

#### Step 4: Customizing UID and GID Ranges
I changed the ranges to:
- FIRST_UID=2000, LAST_UID=2999
- FIRST_GID=3000, LAST_GID=3999

**Use case:**
In a production environment, you might reserve 1000-1999 for permanent employees, 2000-2999 for contractors, and 3000+ for service accounts. This makes it easy to identify account types at a glance.

#### Step 5: Disabling Automatic User-Specific Groups
I set USERGROUPS=no.

**What this does:**
By default, Linux creates a group with the same name as each user. Setting this to "no" means new users won't get their own groups unless you manually create them.

**Security trade-off:**
This forces you to manually assign users to appropriate groups, preventing the sprawl of single-user groups. However, it requires more upfront planning of your group structure.

#### Step 6: Populating /etc/skel with Welcome File
I created a welcome.txt file in `/etc/skel` with a greeting message.

**How /etc/skel works:**
Any file placed in /etc/skel is automatically copied to new user home directories during account creation. This is used for:
- Default .bashrc configurations
- Company policy documents
- Security notices
- README files with instructions

**Commands used:**
```bash
sudo touch /etc/skel/welcome.txt
echo "Welcome to your new account!" | sudo tee /etc/skel/welcome.txt
```

The `tee` command both displays output and writes to the file.

#### Step 7: Changing Default Shell
I modified DSHELL from `/bin/bash` to `/bin/sh`.

**Security consideration:**
`/bin/sh` is a more minimal shell with fewer features than bash. For service accounts or restricted users, this limits what they can do if they manage to get shell access.

#### Step 8: Verifying Changes with Test User
I created a user named "testuser" and verified:
- Home directory was in `/mnt/users/testuser` (not /home)
- UID was 2000 (first ID in new range)
- GID was 2000 (matching new range)
- Shell was `/bin/sh` (not bash)
- welcome.txt existed in home directory

**Verification commands:**
```bash
sudo adduser testuser
sudo ls /mnt/users/testuser
id testuser
grep testuser /etc/passwd
```

The `/etc/passwd` entry showed `testuser:x:2000:2000:Test User,,,:/mnt/users/testuser:/bin/sh`, confirming all modifications worked.

### Part 2: Implementing Access Control Lists (ACLs)

#### Step 1: Creating Project Directory Structure
I created a team_project directory and two test files inside.

**Why ACLs are needed:**
Standard Unix permissions only allow three permission sets: owner, group, others. ACLs let you grant specific permissions to specific users without adding them to groups or changing ownership.

**Commands used:**
```bash
sudo touch /projects/team_project/{file1.txt,file2.txt}
ls /projects/team_project
```

#### Step 2: Setting Ownership and Base Permissions
I changed ownership to `root:developer` and set permissions to 770.

**What 770 means:**
- Owner (root): rwx (7)
- Group (developer): rwx (7)
- Others: no access (0)

This locks down the directory to only root and developer group members before applying ACLs.

**Commands used:**
```bash
sudo chown root:developer /projects/team_project
sudo chmod 770 /projects/team_project
```

#### Step 3: Granting Alice Full Permissions with ACL
I used `setfacl -m u:alice:rwx` to give Alice read, write, and execute permissions.

**Why use ACL instead of adding Alice to developer group:**
ACLs allow per-user granularity. If developer group has certain permissions but Alice needs different ones, ACLs handle that without creating new groups.

**Command used:**
```bash
sudo setfacl -m u:alice:rwx /projects/team_project
```

The `-m` flag means "modify" (add/change ACL entry).

#### Step 4: Granting Bob Read and Execute Only
I gave Bob `r-x` permissions, preventing him from creating or modifying files.

**Use case:**
Bob is a contractor who needs to view project files and run scripts, but shouldn't make changes. This is the digital equivalent of "read-only access" in Windows.

**Command used:**
```bash
sudo setfacl -m u:bob:r-x /projects/team_project
```

#### Step 5: Granting Charlie Read and Write Without Deletion
I gave Charlie `rw-` permissions and applied the sticky bit to prevent file deletion.

**The sticky bit explained:**
On directories, the sticky bit (chmod +t) means users can only delete files they own, even if they have write permission to the directory. This prevents Charlie from deleting Alice or Bob's files.

**Why Charlie can't access the directory:**
The execute (x) permission on directories controls whether you can "enter" them. Without it, even with read/write permissions, Linux blocks directory access entirely. This was an unexpected discovery during testing.

**Commands used:**
```bash
sudo chmod +t /projects/team_project
sudo setfacl -m u:charlie:rw /projects/team_project
```

The capital "T" in `ls -l` output (drwxrwx--T) indicates sticky bit is set but others don't have execute permission.

#### Step 6: Verifying ACL Configuration
I used `getfacl` to view all ACL entries at once.

**Output showed:**
```
user::rwx
user:alice:rwx
user:bob:r-x
user:charlie:rw-
group::rwx
mask::rwx
other::---
```

This confirms each user has their intended permissions.

#### Step 7: Testing Permissions by Switching Users
I used `sudo su - <username>` to test actual access:

**Alice test:** Successfully changed directory, listed files, wrote to file1.txt, and read contents.

**Bob test:** Could read files but got "Permission denied" when trying to create files. The ACL correctly blocked write access.

**Charlie test:** Could not even `cd` into the directory because of missing execute permission. This revealed that directory execute permission is a prerequisite for accessing any files inside, regardless of other permissions.

#### Step 8: Setting Default ACLs for New Files
I applied default ACLs using `setfacl -d -m` so any new file created in the directory automatically inherits the same user permissions.

**Why default ACLs matter:**
Without defaults, new files created in the directory would only have standard Unix permissions (owner, group, others). Default ACLs ensure consistent security even as files are added.

**Commands used:**
```bash
sudo setfacl -d -m u:alice:rwx /projects/team_project
sudo setfacl -d -m u:bob:r-x /projects/team_project
sudo setfacl -d -m u:charlie:rw /projects/team_project
```

### Part 3: Sudo Privilege Management

#### Step 1: Creating User Accounts
I created three users (john, mary, paul) for testing different sudo privilege levels.

**Why separate test users:**
Testing sudo with real accounts is risky. If you misconfigure sudoers, you could lock yourself out of administrative access. Test accounts let you safely verify configurations.

**Commands used:**
```bash
sudo useradd -m john
sudo useradd -m mary
sudo useradd -m paul
sudo passwd john
sudo passwd mary
sudo passwd paul
```

#### Step 2: Granting John Full Administrative Access
I added john to the sudo group, giving him full root privileges.

**What this allows:**
Members of the sudo group can run any command as root by prefixing with `sudo`. This is equivalent to Windows Administrator access.

**Security note:**
Full sudo access should be limited to senior administrators. John can now create users, modify system files, install software, and change security settings.

**Commands used:**
```bash
sudo usermod -aG sudo john
```

Testing: After switching to john and running `sudo whoami`, the output was "root", confirming administrative access.

#### Step 3: Granting Mary Limited Access for System Updates
I edited `/etc/sudoers.tmp` using `visudo` and added:
```
mary ALL=(ALL) NOPASSWD: /usr/bin/apt update, /usr/bin/apt upgrade
```

**Breaking down the syntax:**
- `mary` = username
- `ALL=(ALL)` = can run from any host as any user
- `NOPASSWD:` = doesn't need to enter password
- `/usr/bin/apt update, /usr/bin/apt upgrade` = allowed commands

**Why this matters:**
Mary can keep the system updated but can't install new packages, modify users, or change system configurations. This follows the principle of least privilege.

**Command used:**
```bash
sudo visudo
```

Testing: Mary successfully ran `sudo apt update` but couldn't run `sudo apt install` or other restricted commands.

#### Step 4: Granting Paul Service Management Access
I added:
```
paul ALL=(ALL) NOPASSWD: /bin/systemctl restart apache2, /bin/systemctl restart mysql
```

**Use case:**
Paul is a web developer who needs to restart Apache and MySQL during deployments, but shouldn't have broader system access.

**Testing result:**
Paul could restart apache2 successfully but got "user paul is not allowed to execute '/usr/bin/systemctl restart ssh'" when trying to restart SSH. This proves the restriction worked.

#### Step 5: Restricting Sudo Access
I removed john from the sudo group using `deluser`.

**Why remove access:**
In production, you might revoke sudo when contractors finish their work, employees change roles, or during security incidents.

**Commands used:**
```bash
getent group sudo  # Show current sudo group members
sudo deluser john sudo
getent group sudo  # Verify removal
```

### Part 4: Securing Network Services and Ports

#### Step 1: Listing Open Ports
I used `ss -tulnp` to see what services were listening on the network.

**Why this matters:**
Every open port is a potential attack vector. Attackers scan for open ports and exploit vulnerable services. The first step in hardening is knowing what's exposed.

**Command breakdown:**
- `-t` = TCP
- `-u` = UDP
- `-l` = listening
- `-n` = numeric (show ports as numbers)
- `-p` = process (show which program)

**Findings:**
Ports 80 (HTTP), 22 (SSH), and 3306 (MySQL) were open.

#### Step 2: Closing Unnecessary Services
I stopped and disabled Apache2 because the lab system doesn't need a web server running.

**Why disable, not just stop:**
- `systemctl stop` = stops it now but restarts on reboot
- `systemctl disable` = prevents automatic start on boot

**Command used:**
```bash
sudo systemctl stop apache2
sudo systemctl disable apache2
```

#### Step 3: Securing SSH by Changing Default Port
I edited `/etc/ssh/sshd_config` and changed Port from 22 to 2222.

**Security benefit:**
Changing the SSH port doesn't stop determined attackers, but it eliminates automated bot scans that only target port 22. This reduces log noise and brute force attempts.

**Why port 2222:**
Ports 1-1023 are privileged and require root. Using 2222 keeps it above 1024 while remaining easy to remember.

After changing, I restarted SSH with `sudo systemctl restart ssh`.

#### Step 4: Configuring Automatic Updates
I installed `unattended-upgrades` and configured it to automatically apply security patches.

**Why automation:**
Manual updates require someone to remember to check. Unattended upgrades ensure critical security patches are applied even if admins are busy or unavailable.

**Trade-off:**
Automatic updates can occasionally break things if patches have bugs. Most production environments enable automatic security updates but manually test feature updates.

**Command used:**
```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

### Part 5: Implementing Two-Factor Authentication (2FA)

#### Step 1: Installing Google Authenticator
I installed the PAM module for Google Authenticator.

**What Google Authenticator does:**
It generates time-based one-time passwords (TOTP) that change every 30 seconds. Even if someone steals your password, they can't log in without the current code from your phone.

**Command used:**
```bash
sudo apt install libpam-google-authenticator -y
```

#### Step 2: Configuring 2FA for User Account
I ran `google-authenticator` which presented configuration questions:

**Key decisions made:**
- Time-based tokens: Yes (standard TOTP)
- Update ~/.google_authenticator: Yes (saves configuration)
- Disallow multiple uses of same token: Yes (prevents replay attacks)
- Rate limiting: No (avoids lockouts during time sync issues)
- Increase time window: Yes (allows 1 minute window for time skew)

The tool generated a QR code, secret key, and emergency backup codes.

#### Step 3: Configuring PAM to Require 2FA
I edited `/etc/pam.d/common-auth` and added:
```
auth required pam_google_authenticator.so
```

**What PAM is:**
Pluggable Authentication Modules control how Linux authenticates users. Adding this line makes Google Authenticator codes mandatory for all logins.

**CRITICAL MISTAKE I MADE:**
After saving this configuration, I tested 2FA by logging out. However, I hadn't scanned the QR code to my phone yet, so I couldn't generate codes. This completely locked me out of the system.

#### Step 4: Recovery from 2FA Lockout
I had to use GRUB advanced mode during boot to gain single-user access and edit `/etc/pam.d/common-auth` to remove the 2FA requirement.

**Recovery process:**
1. Rebooted and entered GRUB menu
2. Selected "Advanced options"
3. Chose recovery mode
4. Dropped to root shell
5. Remounted filesystem read-write: `mount -o remount,rw /`
6. Edited `/etc/pam.d/common-auth` and commented out the Google Authenticator line
7. Rebooted normally

**Lesson learned:**
ALWAYS test 2FA configuration before logging out completely. Keep a root terminal open or ensure you have physical access to recover.

#### Step 5: Proper 2FA Testing
After recovering, I:
1. Scanned the QR code to my authenticator app first
2. Re-enabled 2FA in PAM
3. Tested login with verification code
4. Confirmed it worked before closing sessions

### Part 6: SSH Key-Based Authentication

#### Step 1: Generating SSH Key Pair
I created an RSA key pair with 4096 bits.

**Why SSH keys are more secure than passwords:**
- Passwords can be guessed or brute-forced
- Keys are mathematically complex and practically impossible to guess
- Private key never leaves your machine
- Can be protected with passphrase for additional security

**Command used:**
```bash
ssh-keygen -t rsa -b 4096
```

Prompts asked where to save (default ~/.ssh/id_rsa) and optional passphrase.

#### Step 2: Copying Public Key to Server
I used `ssh-copy-id` to install the public key in the remote user's authorized_keys file.

**What this does:**
The command connects via SSH (still using password), copies your public key to `~/.ssh/authorized_keys` on the server, and sets proper permissions.

**Command used:**
```bash
ssh-copy-id fatai@10.0.2.15
```

After this, SSH automatically tries key authentication before passwords.

#### Step 3: Disabling Password Authentication
I edited `/etc/ssh/sshd_config` and set:
```
PasswordAuthentication no
PermitEmptyPasswords no
PubkeyAuthentication yes
```

**Security impact:**
With password auth disabled, attackers can't brute force login even if they know the username. They MUST have the private key.

**Warning:**
Make absolutely sure key-based auth works before disabling passwords. Otherwise you'll lock yourself out.

#### Step 4: Testing Key-Based Authentication
I logged in from the remote machine using `ssh fatai@10.0.2.15` and it connected without asking for a password, confirming the key was working.

#### Step 5: Disabling Root Login
I set `PermitRootLogin no` in sshd_config.

**Why disable root SSH:**
- Root account is always targeted because username is known
- If root can SSH, one compromised key = full system access
- Forces use of sudo, creating audit trail

**Testing:**
Attempting `ssh root@10.0.2.15` resulted in "Permission denied (publickey)", confirming the block worked.

### Part 7: Password Aging Policies

#### Step 1: Examining login.defs
I reviewed `/etc/login.defs` to see default password policies.

**Key parameters:**
- PASS_MAX_DAYS: Maximum password age before forced change
- PASS_MIN_DAYS: Minimum days before user can change password again
- PASS_WARN_AGE: Days warning before expiration

**Why these matter:**
Password aging prevents users from keeping the same password forever, reducing damage if credentials are leaked.

#### Step 2: Setting Password Policies
I modified:
```
PASS_MAX_DAYS 90
PASS_MIN_DAYS 7
PASS_WARN_AGE 14
```

**Reasoning:**
- 90 days max: Passwords must change quarterly
- 7 days min: Prevents users from immediately changing back to old password
- 14 days warning: Gives users two weeks notice before expiration

#### Step 3: Setting UID and GID Ranges
I configured:
```
UID_MIN 1000
UID_MAX 60000
GID_MIN 1000
GID_MAX 60000
```

This defines the range for regular user accounts, separate from system accounts (0-999).

#### Step 4: Testing with Test User
I created a user "Testuser" and checked password aging with `chage -l Testuser`.

**Output showed:**
- Last password change: Mar 14, 2026
- Password expires: Jun 12, 2026 (90 days later)
- Password inactive: never
- Account expires: never
- Minimum days: 7
- Maximum days: 90
- Warning days: 14

#### Step 5: Testing Early Password Change Prevention
I attempted to immediately change the Testuser password, and got:
```
You must wait longer to change your password.
passwd: Authentication token manipulation error
```

This confirmed the PASS_MIN_DAYS=7 policy was enforced.

### Part 8: System Auditing with auditd

#### Step 1: Installing and Enabling auditd
I installed the audit daemon to track security-relevant system events.

**Why auditing matters:**
- Compliance requirements (PCI-DSS, HIPAA, SOC 2)
- Forensic investigation after incidents
- Detecting unauthorized access attempts
- Proving who did what and when

**Commands used:**
```bash
sudo apt install auditd
sudo systemctl start auditd
sudo systemctl enable auditd
sudo systemctl status auditd
```

#### Step 2: Checking Current Audit Rules
I ran `sudo auditctl -l` which showed "No rules", meaning no monitoring was active yet.

**Default state:**
Auditd installs with no rules to avoid performance impact. You must explicitly define what to monitor.

#### Step 3: Creating Audit Rule for User Logins
I added a rule to monitor `/var/log/auth.log` for authentication events:
```
-w /var/log/auth.log -p wa -k user-logins
```

**Breaking down the rule:**
- `-w` = watch this file
- `-p wa` = permissions to monitor (write, attribute change)
- `-k user-logins` = label for searching logs later

**Why monitor auth.log:**
This file records all login attempts, sudo usage, and authentication failures. It's the primary source for detecting unauthorized access.

#### Step 4: Creating Audit Rule for Sensitive File Changes
I added:
```
-w /etc/passwd -p wa -k passwd-modifications
```

**Why monitor /etc/passwd:**
Changes to this file mean user accounts were added, modified, or deleted. Attackers often create backdoor accounts, so monitoring this catches that activity.

#### Step 5: Saving and Applying Rules
I restarted auditd to load the new rules.

**Persistence:**
To make rules permanent across reboots, they should be added to `/etc/audit/rules.d/audit.rules`.

**Command used:**
```bash
sudo service auditd restart
```

#### Step 6: Generating Test Audit Logs
I simulated a login attempt by switching to the Testuser account.

**What gets logged:**
- Who attempted login
- When it occurred
- Whether it succeeded or failed
- Source IP if remote

#### Step 7: Searching Audit Logs
I used `sudo ausearch -k user-logins` to find all events tagged with "user-logins".

**Output showed:**
Multiple entries recording:
- Session opened for Testuser
- TTY assignment
- UID/GID information
- Timestamp

This proves the audit rule is working and capturing login events.

#### Step 8: Testing File Modification Auditing
I opened `/etc/passwd` in nano, changed a comment field, and saved.

Then ran `sudo ausearch -k passwd-modifications`.

**Log entries showed:**
- Type: PROCTITLE (process that made change)
- Type: PATH (file path affected)
- Type: SYSCALL (system call used)
- Process: nano
- User: fatai (uid=1001)
- Result: success

This detailed information would be critical during incident investigation.

#### Step 9: Verifying Audit Persistence After Reboot
I rebooted the system and checked `sudo systemctl status auditd`.

**Status showed:**
Service was active and running, proving the enable command made it persistent.

I also ran `sudo ausearch -i` to verify previous logs were intact, confirming audit log integrity survived the reboot.

## Findings

**System-Wide Configuration:**
- Custom UID/GID ranges (2000-2999, 3000-3999) successfully segregated test users from production accounts
- Home directory relocation to /mnt/users worked for new users but did not affect existing accounts
- Default shell change to /bin/sh applied to testuser account
- /etc/skel welcome.txt file automatically copied to testuser home directory

**ACL Implementation:**
- Alice (rwx) could fully access and modify team_project directory and files
- Bob (r-x) could read files and navigate directory but write operations were blocked
- Charlie (rw-) couldn't access directory at all due to missing execute permission on parent directory
- Sticky bit successfully prevented users from deleting each other's files
- Default ACLs ensured new files inherited user-specific permissions

**Sudo Privilege Separation:**
- John had full administrative access via sudo group membership
- Mary could only run `apt update` and `apt upgrade`, all other commands denied
- Paul could restart apache2 and mysql but not other services like SSH
- Sudo logging captured all privileged command execution in auth.log

**Network Hardening:**
- Apache2 service stopped and disabled, closing port 80
- SSH port changed from 22 to 2222 to reduce automated attacks
- Unattended upgrades configured to automatically apply security patches
- Port scan showed only essential services (SSH on 2222, MySQL on 3306) remained open

**Authentication Hardening:**
- 2FA with Google Authenticator successfully implemented after recovery from lockout
- SSH key-based authentication working, password authentication disabled
- Root SSH login blocked, forcing use of regular user + sudo
- Emergency recovery via GRUB single-user mode demonstrated feasibility of lockout recovery

**Password Policy Enforcement:**
- 90-day password expiration enforced via login.defs
- 7-day minimum between password changes prevented immediate reset
- 14-day warning period gave users advance notice
- Test showed early password change was correctly blocked

**System Auditing:**
- Auditd successfully tracked user login events
- File modification to /etc/passwd captured with full context (who, when, what)
- Audit logs survived system reboot
- Search functionality allowed quick filtering by event type

## Challenges Faced

**2FA Lockout Incident:**
After enabling Google Authenticator in PAM, I logged out without first setting up the authenticator app on my phone. This created a catch-22: I needed a verification code to log in, but I hadn't scanned the QR code to generate codes. I was completely locked out of the system.

**Recovery process:**
I used GRUB advanced mode to boot into single-user recovery mode, remounted the root filesystem as read-write, and edited `/etc/pam.d/common-auth` to comment out the Google Authenticator requirement. This allowed normal login so I could properly configure 2FA.

**Prevention going forward:**
Always test 2FA with the app configured before closing all sessions. Keep one root terminal open until verification is complete, or ensure physical access for recovery mode.

**Charlie's Directory Access Issue:**
Initially confused why Charlie with `rw-` permissions couldn't access the directory at all. Research revealed that directory execute permission is required to "enter" the directory, regardless of read/write permissions. This is different from file permissions where execute controls whether a file can be run as a program.

**Understanding sticky bit behavior:**
The sticky bit showing as capital "T" instead of lowercase "t" was initially unclear. Learned that capital T means sticky bit is set but others don't have execute permission, while lowercase t means both are present.

**Auditd rule persistence:**
First audit rules were lost after reboot because I added them with `auditctl` command instead of writing them to `/etc/audit/rules.d/audit.rules`. Learned that auditctl rules are temporary unless saved to the rules file.

## Key Takeaways

- **Defense in depth:** Multiple security layers (ACLs, sudo restrictions, 2FA, audit logging) protect the system even if one control fails
- **Principle of least privilege:** Mary can update packages but not install new ones; Paul can restart specific services but not all; Charlie can read/write but not execute
- **Configuration files control system behavior:** Modifying `/etc/adduser.conf` once affects all future user creation, maintaining consistency without manual work
- **ACLs extend beyond traditional Unix permissions:** You can grant specific users specific access without group membership or ownership changes
- **Directory execute permission is critical:** Without it, users cannot access directory contents regardless of read/write permissions
- **2FA lockouts are real:** Always configure authenticator apps before enforcing 2FA in PAM, and maintain emergency access paths
- **SSH key authentication eliminates password brute force:** Attackers cannot guess cryptographic keys the way they can guess passwords
- **Disabling root SSH prevents targeted attacks:** Forces use of regular accounts plus sudo, creating an audit trail
- **Password aging policies enforce regular changes:** Helps limit damage from credential leaks, but requires user education to prevent weak patterns
- **System auditing provides forensic evidence:** Audit logs show who did what and when, critical for incident investigation and compliance
- **Automation reduces human error:** Unattended upgrades ensure security patches are applied even when admins forget

## Disclaimer

This lab was performed in a controlled Kali Linux environment for educational purposes as part of the ICDFA Strengthening System Security on Linux Servers course. All activities were conducted on a local system with proper authorization. The 2FA lockout incident occurred during testing and was resolved using legitimate recovery procedures.
```
