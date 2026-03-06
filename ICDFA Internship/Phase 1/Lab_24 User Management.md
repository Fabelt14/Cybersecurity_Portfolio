# Linux User Management

## Overview

This lab covered basic user account management in Linux. The goal was to practice creating users, managing group memberships, locking/unlocking accounts for security purposes, and properly removing users from the system.

## Objectives

- Create new user accounts and set secure passwords
- Add users to groups for access control
- Lock and unlock user accounts when needed
- Delete user accounts with and without preserving their data
- Verify account status changes using system commands

## Lab Environment

- **OS**: Kali Linux
- **User Privilege**: Root access via sudo

## Commands Used

- adduser - User creation
- usermod - User modification
- groupadd - Group creation
- passwd - Password management
- deluser - User deletion
- ps - Process checking
- id - User information verification
- ls - Directory listing
- rm - File/directory removal

## Methodology

### Step 1: Creating the User Account

I started by creating a new user named student1 using the `adduser` command. This command not only creates the user but also sets up their home directory and prompts for additional information, such as full name and contact details.

**Why adduser instead of useradd:**
The `adduser` command is more user-friendly because it handles home directory creation and initial setup automatically, while `useradd` requires manual configuration.

**Command used:**
```bash
sudo adduser student1
```

After creation, I verified the user's existence by running `id student1`, which displayed the user ID (UID), group ID (GID), and group memberships.

![create user](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/create%20user.jpg)


### Step 2: Adding the User to the Group

I needed to add student1 to a group called "students" for organized access control. First, I created the group, then added the user to it.

**Why groups matter:**
Groups allow managing permissions for multiple users at once instead of configuring each user individually. This is important in real environments where there might be dozens of students or employees needing similar access.

**Commands used:**
```bash
sudo groupadd students
sudo usermod -aG students student1
```

The `-aG` flag means "append to group" - it adds the user to the new group without removing them from existing groups.

![Add to group](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/add%20user%20to%20group.jpg)


### Step 3: Setting a Secure Password for the User
I changed the password for student1 using the `passwd` command. The system prompted me to enter and confirm the new password.

**Security consideration:**
In production, I would enforce password complexity requirements (minimum length, special characters, etc.). For this lab, I followed basic password security practices.

**Command used:**
```bash
sudo passwd student1
```

![Change User Password](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/change%20user%20passwd.jpg)


### Step 4: Checking for Active Processes
Before locking the account, I checked if student1 had any running processes that might cause issues.

**Why this matters:**
If a user has active sessions or processes, locking their account might not immediately disconnect them. Checking first lets me decide whether to kill processes or wait for a clean logout.

**Command used:**
```bash
ps -u student1
```
The result showed no active processes (PID TTY TIME CMD header only), so it was safe to proceed.

![Check User Proccesses](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/check%20user%20proccesses.jpg)


### Step 5: Locking the User's Account
I locked student1's account using `usermod -L`, which prevents the user from logging in by disabling their password.

**Use case for locking:**
Locking is useful when there is a need to temporarily suspend access (employee on leave, security investigation) without deleting the account. The user's files and settings remain intact.

**Commands used:**
```bash
sudo usermod -L student1
sudo passwd -S student1
```

The status check showed `L 2026-03-05 0 99999 7 -1`, where the 'L' confirms the account is locked.

![Lock User](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/Lock%20user%20account.jpg)


### Step 6: Unlocking the User's Account
I reversed the lock with `usermod -U` to restore student1's login ability.

**Verification method:**
Running `passwd -S` again showed `P 2026-03-05 0 99999 7 -1`, where 'P' means a usable password is set (account unlocked).

**Commands used:**
```bash
sudo usermod -U student1
sudo passwd -S student1
```

![Unlock User](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/Unlock%20user%20account.jpg)

