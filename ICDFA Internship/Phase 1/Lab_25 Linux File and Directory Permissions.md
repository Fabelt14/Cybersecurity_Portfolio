# Linux File and Directory Permissions

## Overview

This lab focused on managing file and directory permissions in Linux. The goal was to understand how permission settings control who can read, write, or execute files, and how proper ownership assignment strengthens system security.

## Objectives

- Create directories with proper permission settings
- Understand numeric permission notation (755, 644, etc.)
- Change file and directory ownership for access control
- Apply the principle of least privilege through permission management
- Prevent unauthorized access to sensitive data

## Lab Environment

- **OS**: Kali Linux
- **User Privilege**: Root access via sudo

## Commands Used

- mkdir - Directory creation
- chmod - Permission modification
- chown - Ownership modification
- ls -l - Permission verification

## Methodology

### Step 1: Creating the Project Directory

I created a directory named "my_project" using `mkdir`. This would serve as a workspace where I could demonstrate permission and ownership controls.

**Why directories need permissions:**

Directories in Linux have their own permission rules separate from files. Without proper directory permissions, even if files inside have restricted access, attackers could still list or access them.

**Command used:**
```bash
mkdir my_project
```

Verification with `ls` showed the directory was created successfully.

![Create Directory](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/create%20directory.jpg)

### Step 2: Understanding and Setting 755 Permissions

I changed the directory permissions to 755 using `chmod`. Before running the command, I checked the default permissions with `ls -l`, which showed `drwxrwxr-x` (775).

**What 755 means:**
- First digit (7) = Owner permissions: read(4) + write(2) + execute(1) = 7 (rwx)
- Second digit (5) = Group permissions: read(4) + execute(1) = 5 (r-x)
- Third digit (5) = Others permissions: read(4) + execute(1) = 5 (r-x)

**Why 755 for directories:**

This permission set allows the owner full control while giving group members and others the ability to view contents and navigate into the directory, but not modify it. This is the standard for shared project directories where one person owns it, but others need read access.

**Commands used:**
```bash
ls -l
chmod 755 my_project
ls -l
```

After the change, permissions showed `drwxr-xr-x`, confirming the 755 setting was applied correctly.

![Set permission](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/set%20permission.jpg)

### Step 3: Changing Ownership
I transferred ownership of my_project to student1 with the students' group using `chown`. This demonstrates access delegation - giving another user control while maintaining group-level access.

**Why ownership matters:**

Ownership determines who has ultimate control over a file or directory. Even with restrictive permissions, the owner can always modify their own files. By changing ownership to student1, I'm simulating handing off a project to another user while keeping it accessible to the students group.

**Syntax breakdown:**
- `student1:students` means "owner:group"
- If only I wanted to change the owner, I would use just `student1`
- If only I wanted to change the group, I would use `:students`

**Commands used:**
```bash
ls -l
sudo chown student1:students my_project
ls -l
```

Final verification showed `drwxr-xr-x 2 student1 students 4096 Mar 6 10:30 my_project`, confirming student1 now owns the directory with students as the group.

![Change Ownership](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/change%20ownership.jpg)

## Findings

**Permission Analysis:**
- Default directory permissions were 775 (drwxrwxr-x)
- Changed to 755 (drwxr-xr-x) to remove group write access
- This prevents group members from accidentally or maliciously modifying directory contents
- Execute permission (x) on directories allows users to access files inside

**Ownership Changes:**
- Original owner: fatai
- Original group: fatai
- New owner: student1
- New group: students
- Transfer required sudo privileges, demonstrating separation of administrative control

**Security Implications:**
- 755 permissions enforce read-only access for non-owners
- Prevents privilege escalation where group members could plant malicious files
- Maintains transparency (anyone can see contents) while protecting integrity

## Challenges Faced

- Initially needed to verify that student1 user and students group existed from the previous lab
- Had to understand the difference between file permissions and directory permissions - execute (x) on directories means "can enter/traverse"
- Required sudo for ownership change because changing ownership is an administrative action that affects other users

## Key Takeaways

- **Principle of least privilege**: Users get only the minimum access needed - group members can read but not write
- **Permission numbers are additive**: 755 = 7(rwx) + 5(r-x) + 5(r-x), making it easy to calculate and set precise access levels
- **Ownership and permissions work together**: Even with open permissions, the owner has ultimate control and can restrict access at any time
- **Directory execute permission is critical**: Without (x), users can't access files inside even if they have read permission on those files
- **Proper access control prevents unauthorized modifications**: In cybersecurity, preventing unauthorized writes is as important as preventing unauthorized reads - attackers could inject malicious code if write access isn't restricted

## Disclaimer

This lab was performed in a controlled Kali Linux environment for educational purposes as part of the ICDFA Secure User Access Management in Linux course. All activities were conducted on a local system with proper authorization.
