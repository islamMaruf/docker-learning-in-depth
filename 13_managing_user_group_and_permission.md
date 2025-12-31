# Chapter 20: Managing Users, Groups, and Permissions

## Overview

Welcome to one of the most critical chapters in your Linux journey! Understanding users, groups, and permissions is fundamental to Linux system administration and security. These concepts form the backbone of Linux's multi-user design and its robust security model.

In this chapter, we'll explore the Linux permission system from the ground up. You'll learn how to create and manage users, organize them into groups, and control exactly who can read, write, or execute files and directories. These skills are essential not just for Docker and containers, but for any work you'll do with Linux systems.

Think of Linux permissions as a sophisticated access control system for your files. Just as you wouldn't want anyone walking into your home and rearranging your belongings, Linux ensures that users can only access and modify files they're authorized to work with. By the end of this chapter, you'll be the master of your Linux domain, deciding precisely who gets to do what.

## Understanding Users in Linux

### The Two Types of Users

Linux fundamentally has two types of users:

1. **Root User (Superuser)**: The all-powerful administrator with unrestricted access to everything
2. **Normal Users**: Regular users with limited permissions

**Why this distinction matters:**
- **Root user** can do absolutely anything: install software, modify system files, access anyone's data, even accidentally delete the entire system
- **Normal users** work in a protected environment where they can't accidentally (or maliciously) harm the system

### The Root User Philosophy

When you set up a typical Ubuntu system, you get two user accounts:
1. An administrator account (root or similar)
2. Your personal account (like "habiburrahman")

**Best Practice**: Always work as a normal user, not root. Why?
- Security: If your account is compromised, damage is limited
- Safety: You can't accidentally delete critical system files
- Audit trail: Actions are traceable to specific users

Think of it like driving: even though you have a driver's license (root access), you don't drive recklessly just because you can. You follow rules to keep everyone safe.

## Creating and Managing Users

### Creating a New User

The `useradd` command creates new user accounts. Let's start with the basics:

**Basic user creation:**
```bash
# Run inside a Docker container or as root
$ docker run -it ubuntu:22.04 bash
root@container:/#

# Create a new user named "habib"
root@container:/# useradd -m habib
```

The `-m` flag is crucial:
- **With `-m`**: Creates a home directory for the user at `/home/habib`
- **Without `-m`**: User is created but has no home directory

**Why the home directory matters:**
Every user needs a personal space to store their files, configurations, and data. Without it, the user has nowhere to call "home."

### Viewing User Information

The `id` command displays comprehensive user information:

```bash
root@container:/# id habib
uid=1001(habib) gid=1001(habib) groups=1001(habib)
```

This output tells you:
- **uid=1001(habib)**: User ID is 1001, username is "habib"
- **gid=1001(habib)**: Primary group ID is 1001, group name is "habib"
- **groups=1001(habib)**: User belongs to group 1001 (habib)

**What's a UID and GID?**
- **UID** (User ID): A unique number identifying each user
- **GID** (Group ID): A unique number identifying each group
- Linux uses these numbers internally; names are for human convenience

### Listing All Users

All user information is stored in the `/etc/passwd` file:

```bash
root@container:/# cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
habib:x:1001:1001::/home/habib:/bin/sh
```

**Understanding the passwd file format:**
Each line represents one user with seven fields separated by colons:

```
username:password:UID:GID:comment:home_directory:shell
```

Example breakdown for habib:
- **habib**: Username
- **x**: Password is stored in `/etc/shadow` (encrypted)
- **1001**: User ID
- **1001**: Primary group ID
- **(empty)**: Comment/description field
- **/home/habib**: Home directory
- **/bin/sh**: Default shell

### Home Directories

When you create a user with `-m`, Linux automatically creates their home directory:

```bash
root@container:/# cd /home
root@container:/home# ls
habib
```

You can verify which home directory a user has:

```bash
# Currently as root user (in /root)
root@container:/# pwd
/root

# The tilde (~) always represents the current user's home
root@container:/# cd ~
root@container:/# pwd
/root
```

### Switching Between Users

The `su` (switch user) command lets you change to another user:

```bash
# Switch to habib user
root@container:/# su - habib
habib@container:~$

# Check current location
habib@container:~$ pwd
/home/habib

# Notice the prompt changed:
# $ instead of # (indicating normal user, not root)
```

**Understanding the prompt symbols:**
- `#`: Root user (danger zone!)
- `$`: Normal user (safe zone)
- `%`: Zsh shell with normal user

**The power of root:**
- Root → any user: No password needed (root has all permissions)
- Normal user → another user: Password required (security!)

```bash
# Root switching to habib (no password needed)
root@container:/# su - habib
habib@container:~$

# Habib trying to switch to habiburhman (password required)
habib@container:~$ su - habiburrahman
Password: [password required]
```

### Setting User Passwords

The `passwd` command manages user passwords:

**As root (can set anyone's password):**
```bash
root@container:/# passwd habib
New password: 12345
Retype new password: 12345
passwd: password updated successfully
```

**As normal user (can only change own password after providing current one):**
```bash
habib@container:~$ passwd
Current password: [required!]
New password: 
Retype new password:
```

**Why the difference?**
- Root bypasses security checks (administrator privilege)
- Normal users must prove they own the account (security requirement)

### Deleting Users

The `userdel` command removes user accounts:

**Delete user only (keep home directory):**
```bash
root@container:/# userdel habib
```

**Delete user and home directory:**
```bash
root@container:/# userdel -r habib
```

The `-r` flag removes:
- User's home directory
- User's mail spool
- All files owned by the user in the home directory

**⚠️ Warning**: Use `-r` carefully! Once deleted, the user's files are gone forever.

### Locking and Unlocking Users

Sometimes you want to temporarily disable a user account without deleting it:

**Lock a user (prevent login):**
```bash
root@container:/# usermod -L habiburrahman
```

**Unlock a user:**
```bash
root@container:/# usermod -U habiburrahman
```

When locked, the user cannot log in, even with the correct password:
```bash
# Try to switch to locked user
habib@container:~$ su - habiburrahman
Password: [correct password]
su: Authentication failure
```

**Use cases for locking:**
- Employee on vacation (temporary)
- Investigating security incident
- Suspended account pending review

## Understanding Groups in Linux

### What Are Groups?

Groups are collections of users that share common permissions. Instead of setting permissions for each user individually, you assign permissions to a group, and all members inherit those permissions.

**Real-world analogy:**
Think of groups like departments in a company:
- "Engineering" group: access to code repositories
- "HR" group: access to employee records  
- "Finance" group: access to financial systems

### Two Types of Groups

Every Linux user belongs to two types of groups:

1. **Primary Group**: Automatically created when the user is created
2. **Secondary (Supplementary) Groups**: Additional groups the user joins

**Identifying group types:**
```bash
root@container:/# id habib
uid=1001(habib) gid=1001(habib) groups=1001(habib),1003(admin)
```

Breaking this down:
- **gid=1001(habib)**: Primary group (shows in GID field)
- **groups=1001(habib),1003(admin)**: All groups (primary first, then secondary)

So habib's primary group is "habib" (1001), and he also belongs to secondary group "admin" (1003).

### Creating Groups

The `groupadd` command creates new groups:

```bash
root@container:/# groupadd admin
```

Verify it was created:

```bash
root@container:/# cat /etc/group
root:x:0:
daemon:x:1:
...
admin:x:1003:
```

**Understanding the group file format:**
Each line in `/etc/group` has four fields:

```
groupname:password:GID:member_list
```

Example:
```
admin:x:1003:habib,rahman
```
- **admin**: Group name
- **x**: Password (rarely used, stored in `/etc/gshadow` if set)
- **1003**: Group ID
- **habib,rahman**: Comma-separated list of members

### Adding Users to Groups

The `usermod` command modifies user accounts:

**Add user to a group:**
```bash
root@container:/# usermod -aG admin habib
```

Flags explained:
- **-a**: Append (add to group without removing from others)
- **-G**: Supplementary groups

**⚠️ Critical**: Always use `-aG` together!
```bash
# WRONG - removes user from all other groups
$ usermod -G admin habib

# CORRECT - adds to admin while keeping other groups
$ usermod -aG admin habib
```

**Verify the user was added:**
```bash
root@container:/# id habib
uid=1001(habib) gid=1001(habib) groups=1001(habib),1003(admin)
```

**Check from the user's perspective:**
```bash
root@container:/# su - habib
habib@container:~$ groups
habib admin
```

The `groups` command shows all groups the current user belongs to.

### Removing Users from Groups

The `gpasswd` command manages group membership:

**Remove user from a group:**
```bash
root@container:/# gpasswd -d habib faltu
Removing user habib from group faltu
```

**Verify removal:**
```bash
root@container:/# id habib
uid=1001(habib) gid=1001(habib) groups=1001(habib),1003(admin)
# Notice "faltu" group is gone
```

### Deleting Groups

The `groupdel` command removes groups:

```bash
root@container:/# groupadd faltu
root@container:/# groupdel faltu
```

**Note**: You cannot delete a group if it's the primary group of any user. You must first delete the user or change their primary group.

## Understanding File Permissions

### The Permission Model

Every file and directory in Linux has permissions that control:
- **Who** can access it (user, group, others)
- **What** they can do (read, write, execute)

This is the foundation of Linux security.

### The Permission Display

When you run `ls -l`, you see detailed file information:

```bash
habib@container:~/documents$ ls -l
drwxr-xr-x  2 habib  habib  4096 Dec 31 10:30 files
-rw-r--r--  1 habib  habib    12 Dec 31 10:25 a.txt
```

Let's break down each component:

```
-rw-r--r--  1  habib  habib    12  Dec 31 10:25  a.txt
│  │ │  │   │    │      │      │      │          │
│  │ │  │   │    │      │      │      │          └─ Filename
│  │ │  │   │    │      │      │      └─ Date modified
│  │ │  │   │    │      │      └─ Size (bytes)
│  │ │  │   │    │      └─ Group owner
│  │ │  │   │    └─ User owner
│  │ │  │   └─ Number of hard links
│  │ │  └─ Others permissions (read only)
│  │ └─ Group permissions (read only)
│  └─ User permissions (read, write)
└─ File type (- = file, d = directory)
```

### The Permission String Breakdown

The permission string has 10 characters:

```
-rw-r--r--
│└┬┘└┬┘└┬┘
│ │  │  └─ Others permissions (3 chars)
│ │  └─ Group permissions (3 chars)
│ └─ User/Owner permissions (3 chars)
└─ File type (1 char)
```

**First character - File type:**
- `-`: Regular file
- `d`: Directory
- `l`: Symbolic link
- `b`: Block device
- `c`: Character device

**Next 9 characters - Permissions (in sets of 3):**

```
rwx  rwx  rwx
│││  │││  │││
│││  │││  └┴┴─ Others: read, write, execute
│││  └┴┴─ Group: read, write, execute
└┴┴─ User/Owner: read, write, execute
```

Each position can be:
- **r**: Read permission (4)
- **w**: Write permission (2)
- **x**: Execute permission (1)
- **-**: Permission not granted (0)

### Permission Meanings

**For files:**
- **r (read)**: View file contents
- **w (write)**: Modify file contents
- **x (execute)**: Run file as a program/script

**For directories:**
- **r (read)**: List directory contents (`ls`)
- **w (write)**: Create/delete files in directory
- **x (execute)**: Enter directory (`cd`)

**Practical examples:**

```bash
# File with read and write for owner, read-only for others
-rw-r--r--  a.txt
# Owner: read + write
# Group: read only
# Others: read only

# Directory with full permissions for owner, read+execute for others
drwxr-xr-x  files/
# Owner: read + write + execute (full access)
# Group: read + execute (can list and enter, but not create files)
# Others: read + execute (can list and enter, but not create files)

# Script file with execute permission
-rwxr-xr-x  script.sh
# Owner: read + write + execute (can run the script)
# Group: read + execute (can run, but not modify)
# Others: read + execute (can run, but not modify)
```

### The Three Permission Categories

**1. User (u)**: The file owner
```bash
# Create a file as habib user
habib@container:~/documents$ touch myfile.txt
habib@container:~/documents$ ls -l myfile.txt
-rw-r--r--  1 habib  habib  0 Dec 31 10:30 myfile.txt
              ^^^^^^
              User owner
```

**2. Group (g)**: Users in the file's group
```bash
# Same example - group is also "habib"
-rw-r--r--  1 habib  habib  0 Dec 31 10:30 myfile.txt
                     ^^^^^^
                     Group owner
```

**3. Others (o)**: Everyone else on the system
```bash
# Users who are neither the owner nor in the group
-rw-r--r--  1 habib  habib  0 Dec 31 10:30 myfile.txt
        ^^^
        Others permissions (last 3 characters)
```

## Changing File Permissions

### The chmod Command

The `chmod` (change mode) command modifies file permissions. There are two ways to use it:

1. **Symbolic method**: Use letters (u, g, o, a) and symbols (+, -, =)
2. **Numeric method**: Use numbers (octal notation)

### Symbolic Method

**Syntax:**
```bash
chmod [who][operation][permission] filename
```

**Who:**
- `u`: User (owner)
- `g`: Group
- `o`: Others
- `a`: All (user + group + others)

**Operations:**
- `+`: Add permission
- `-`: Remove permission
- `=`: Set exact permissions

**Permissions:**
- `r`: Read
- `w`: Write
- `x`: Execute

**Practical examples:**

**1. Add execute permission for user:**
```bash
habib@container:~/documents$ ls -l a.txt
-rw-r--r--  1 habib habib  12 Dec 31 10:25 a.txt

habib@container:~/documents$ chmod u+x a.txt

habib@container:~/documents$ ls -l a.txt
-rwxr--r--  1 habib habib  12 Dec 31 10:25 a.txt
   ^
   Added execute permission for user
```

**2. Remove write permission from user:**
```bash
habib@container:~/documents$ chmod u-w a.txt
habib@container:~/documents$ ls -l a.txt
-r-xr--r--  1 habib habib  12 Dec 31 10:25 a.txt
  ^
  Write permission removed
```

**3. Add read permission for group:**
```bash
habib@container:~/documents$ chmod g+r a.txt
```

**4. Remove read permission from group:**
```bash
habib@container:~/documents$ chmod g-r a.txt
habib@container:~/documents$ ls -l a.txt
-r-x---r--  1 habib habib  12 Dec 31 10:25 a.txt
     ^^
     Group has no permissions now
```

**5. Add execute permission for others:**
```bash
habib@container:~/documents$ chmod o+x a.txt
```

**6. Remove all permissions from others:**
```bash
habib@container:~/documents$ chmod o-rwx a.txt
habib@container:~/documents$ ls -l a.txt
-r-x------  1 habib habib  12 Dec 31 10:25 a.txt
       ^^^
       Others have no permissions
```

**7. Set exact permissions:**
```bash
# Set user to read+write, group to read, others to nothing
habib@container:~/documents$ chmod u=rw,g=r,o= a.txt
habib@container:~/documents$ ls -l a.txt
-rw-r-----  1 habib habib  12 Dec 31 10:25 a.txt
```

**8. Add permissions to multiple categories:**
```bash
# Add execute to user and group
habib@container:~/documents$ chmod ug+x a.txt

# Add read to all (user, group, others)
habib@container:~/documents$ chmod a+r a.txt
```

### Common Permission Patterns

**Read-only for everyone:**
```bash
$ chmod a-w file.txt   # Remove write from all
$ chmod 444 file.txt   # Numeric method (explained below)
```

**Full access for owner, read-only for others:**
```bash
$ chmod u=rwx,go=r file.txt
$ chmod 744 file.txt
```

**Private file (owner access only):**
```bash
$ chmod u=rw,go= private.txt
$ chmod 600 private.txt
```

**Executable script:**
```bash
$ chmod u+x script.sh
$ chmod 755 script.sh   # Standard for scripts
```

### Understanding the Effects

Let's see how permissions affect file operations:

```bash
# Create a file and remove write permission
habib@container:~/documents$ echo "Hello World" > a.txt
habib@container:~/documents$ chmod u-w a.txt

# Try to write to it
habib@container:~/documents$ echo "Goodbye" > a.txt
bash: a.txt: Permission denied

# Restore write permission
habib@container:~/documents$ chmod u+w a.txt
habib@container:~/documents$ echo "Goodbye" > a.txt
# Success!
```

## Changing File Ownership

### The chown Command

The `chown` (change owner) command changes file ownership. However, only the root user can change ownership (security feature).

**Syntax:**
```bash
chown [user]:[group] filename
```

**Examples:**

**Change user owner:**
```bash
root@container:/home/habib/documents# chown habiburrahman a.txt
```

**Change group owner:**
```bash
root@container:/home/habib/documents# chown :admin a.txt
```

**Change both user and group:**
```bash
root@container:/home/habib/documents# chown habiburrahman:admin a.txt
```

**Why only root can do this:**
```bash
# As normal user habib
habib@container:~/documents$ chown habiburrahman a.txt
chown: changing ownership of 'a.txt': Operation not permitted
```

**Security reasoning**: Imagine if any user could give their files to someone else:
1. User Alice creates a malicious file
2. Alice gives it to Bob (by changing ownership)
3. When Bob runs it, the virus activates
4. Bob gets blamed, not Alice!

By restricting `chown` to root, Linux prevents this social engineering attack.

## Advanced Permission Concepts

### The Number in ls -l Output

When you see `ls -l`, notice the number after permissions:

```bash
drwxr-xr-x  3  habib  habib  4096  Dec 31  files/
-rw-r--r--  1  habib  habib    12  Dec 31  a.txt
            ^
            This number
```

**For files**: Always `1` (number of hard links)

**For directories**: Number of subdirectories + 2

Why +2?
- `.` (current directory)
- `..` (parent directory)

**Example:**
```bash
habib@container:~/documents/files$ ls -la
total 12
drwxr-xr-x  3  habib  habib  4096  Dec 31  .
drwxr-xr-x  3  habib  habib  4096  Dec 31  ..
drwxr-xr-x  2  habib  habib  4096  Dec 31  newdir/

# Directory has 3 links:
# 1. . (itself)
# 2. .. (parent reference)
# 3. newdir/ (subdirectory)
```

### File Size Display

```bash
-rw-r--r--  1  habib  habib    12  Dec 31  a.txt
drwxr-xr-x  3  habib  habib  4096  Dec 31  files/
                              ^^^^
                              Size in bytes
```

**For files**: Actual content size
```bash
habib@container:~/documents$ echo "Hello World" > a.txt
habib@container:~/documents$ ls -l a.txt
-rw-r--r--  1  habib  habib  12  Dec 31  a.txt
                             ^^
# "Hello World" + newline = 12 bytes
```

**For directories**: Always 4096 bytes (or multiples)
- This is the size of the directory structure itself
- Not the total size of files inside
- Standard block size on most filesystems

**To see total size of directory contents:**
```bash
$ du -sh files/
128K    files/
```

### Permission Inheritance

**Important**: Permissions don't inherit automatically!

When you create a new file, it gets default permissions:
```bash
habib@container:~/documents$ touch newfile.txt
habib@container:~/documents$ ls -l newfile.txt
-rw-r--r--  1  habib  habib  0  Dec 31  newfile.txt
```

Default permissions are controlled by `umask` (covered in advanced topics).

## Practical Permission Scenarios

### Scenario 1: Shared Project Directory

**Goal**: Create a directory where team members can collaborate

```bash
# As root
root@container:/# groupadd developers
root@container:/# useradd -m -G developers alice
root@container:/# useradd -m -G developers bob

# Create shared directory
root@container:/# mkdir /projects
root@container:/# chown :developers /projects
root@container:/# chmod 770 /projects

# Result
root@container:/# ls -ld /projects
drwxrwx---  2  root  developers  4096  Dec 31  /projects
```

**What this achieves:**
- Owner (root): Full access
- Group (developers): Full access
- Others: No access
- Alice and Bob can both create/modify files in /projects
- Non-developers can't even see what's inside

### Scenario 2: Read-Only Configuration File

**Goal**: Prevent accidental modification of important config

```bash
# Create config file
root@container:/etc# echo "SERVER=production" > app.conf
root@container:/etc# chmod 444 app.conf

# Result
root@container:/etc# ls -l app.conf
-r--r--r--  1  root  root  18  Dec 31  app.conf

# Even root can't accidentally overwrite it
root@container:/etc# echo "SERVER=dev" > app.conf
bash: app.conf: Permission denied

# Must explicitly change permissions first
root@container:/etc# chmod 644 app.conf
root@container:/etc# echo "SERVER=dev" > app.conf
# Success!
```

### Scenario 3: Private User Files

**Goal**: Keep sensitive files private

```bash
# As habib
habib@container:~$ mkdir private
habib@container:~$ chmod 700 private

habib@container:~$ ls -ld private
drwx------  2  habib  habib  4096  Dec 31  private
```

**What this achieves:**
- Only habib can access this directory
- Other users can't even list contents
- Perfect for sensitive documents, SSH keys, etc.

### Scenario 4: Executable Script

**Goal**: Make a script runnable

```bash
# Create script
habib@container:~$ echo '#!/bin/bash' > script.sh
habib@container:~$ echo 'echo "Hello, World!"' >> script.sh

# Try to run it
habib@container:~$ ./script.sh
bash: ./script.sh: Permission denied

# Add execute permission
habib@container:~$ chmod u+x script.sh
habib@container:~$ ls -l script.sh
-rwxr--r--  1  habib  habib  33  Dec 31  script.sh
   ^
   Execute bit set

# Now it works
habib@container:~$ ./script.sh
Hello, World!
```

## Permission Best Practices

### 1. Principle of Least Privilege

Give users/groups only the minimum permissions they need:

```bash
# BAD - unnecessarily permissive
chmod 777 file.txt   # Everyone can do everything

# GOOD - restrictive but functional
chmod 644 file.txt   # Owner writes, others read
chmod 755 script.sh  # Owner writes, everyone executes
chmod 600 private.txt # Owner only
```

### 2. Never Use 777 Permissions

```bash
# DANGEROUS - security nightmare
chmod 777 important_file.txt
```

This allows:
- Any user to read your files
- Any user to modify your files
- Any user to execute your files

Only use in temporary testing scenarios, never in production!

### 3. Protect Sensitive Files

```bash
# SSH keys
chmod 600 ~/.ssh/id_rsa

# Configuration files with passwords
chmod 600 /etc/app/database.conf

# User directories
chmod 700 ~/private/
```

### 4. Use Groups for Shared Access

Instead of opening permissions to "others," create appropriate groups:

```bash
# BAD
chmod 666 shared_file.txt   # Anyone can write

# GOOD
groupadd team
usermod -aG team alice
usermod -aG team bob
chown :team shared_file.txt
chmod 660 shared_file.txt   # Only team members can write
```

### 5. Regular Permission Audits

Periodically check for overly permissive files:

```bash
# Find world-writable files (potential security risk)
find / -type f -perm -002 2>/dev/null

# Find files with SUID bit (potential privilege escalation)
find / -perm -4000 2>/dev/null
```

## Common Permission Errors and Solutions

### Error 1: Permission Denied When Reading

**Symptom:**
```bash
habib@container:~$ cat /home/habiburrahman/secret.txt
cat: /home/habiburrahman/secret.txt: Permission denied
```

**Solution:**
1. Check file permissions: `ls -l /home/habiburrahman/secret.txt`
2. File needs read permission for your user/group/others
3. Directory needs execute permission to traverse
4. Ask owner to grant access or use root

### Error 2: Permission Denied When Writing

**Symptom:**
```bash
habib@container:~$ echo "test" > file.txt
bash: file.txt: Permission denied
```

**Solution:**
1. Check permissions: `ls -l file.txt`
2. Need write permission: `chmod u+w file.txt`
3. Check if directory is writable
4. Ensure you're the owner or in the group

### Error 3: Cannot Execute Script

**Symptom:**
```bash
habib@container:~$ ./script.sh
bash: ./script.sh: Permission denied
```

**Solution:**
```bash
# Add execute permission
chmod u+x script.sh
./script.sh  # Now works
```

### Error 4: Operation Not Permitted (chown)

**Symptom:**
```bash
habib@container:~$ chown bob file.txt
chown: changing ownership of 'file.txt': Operation not permitted
```

**Solution:**
- Only root can change ownership
- Switch to root: `su -` or `sudo chown bob file.txt`

### Error 5: Cannot Access Directory

**Symptom:**
```bash
habib@container:~$ ls /home/habiburrahman
ls: cannot access '/home/habiburrahman': Permission denied
```

**Solution:**
- Directory needs execute permission to enter
- Check: `ls -ld /home/habiburrahman`
- Fix: `chmod +x /home/habiburrahman` (as root or owner)

## ASCII Diagrams for Permissions

### Permission Structure Visualization

```
-rwxr-xr--
│││││││││└─ Others: no write
│││││││└┴─ Others: read, execute
││││││└──── Group: no write
││││└┴───── Group: read, execute
│││└──────── User: execute
││└───────── User: write
│└────────── User: read
└─────────── File type (regular file)

Permission Breakdown:
┌──────────┬──────────┬──────────┬──────────┐
│ Position │ Category │ Symbol   │ Meaning  │
├──────────┼──────────┼──────────┼──────────┤
│    1     │   Type   │    -     │   File   │
│  2-4     │   User   │   rwx    │  Owner   │
│  5-7     │  Group   │   r-x    │  Group   │
│  8-10    │  Others  │   r--    │ Everyone │
└──────────┴──────────┴──────────┴──────────┘
```

### User-Group-Others Hierarchy

```
                    ┌─────────────────────┐
                    │    Linux File       │
                    │   "project.txt"     │
                    └──────────┬──────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
    ┌────▼────┐           ┌────▼────┐          ┌────▼────┐
    │  USER   │           │  GROUP  │          │ OTHERS  │
    │ (owner) │           │ (team)  │          │ (world) │
    │  habib  │           │  admin  │          │everyone │
    └────┬────┘           └────┬────┘          └────┬────┘
         │                     │                     │
    ┌────▼────┐           ┌────▼────┐          ┌────▼────┐
    │  rwx    │           │  r-x    │          │  r--    │
    │ Read ✓  │           │ Read ✓  │          │ Read ✓  │
    │ Write ✓ │           │ Write ✗ │          │ Write ✗ │
    │ Exec ✓  │           │ Exec ✓  │          │ Exec ✗  │
    └─────────┘           └─────────┘          └─────────┘
```

## Command Reference Summary

### User Management Commands

```bash
# Create user with home directory
useradd -m username

# Set user password
passwd username

# View user information
id username

# Switch to another user
su - username

# Delete user (keep home)
userdel username

# Delete user and home directory
userdel -r username

# Lock user account
usermod -L username

# Unlock user account
usermod -U username

# List all users
cat /etc/passwd
```

### Group Management Commands

```bash
# Create group
groupadd groupname

# Add user to group (supplementary)
usermod -aG groupname username

# Remove user from group
gpasswd -d username groupname

# Delete group
groupdel groupname

# List all groups
cat /etc/group

# Show current user's groups
groups

# Show specific user's groups
groups username
```

### Permission Management Commands

```bash
# Change permissions (symbolic)
chmod u+rwx file     # Add all permissions for user
chmod g-w file       # Remove write from group
chmod o=r file       # Set others to read-only
chmod a+x file       # Add execute for all

# Change permissions (multiple)
chmod ug+x file      # Add execute for user and group
chmod a-w file       # Remove write from all

# Change ownership (root only)
chown user file
chown user:group file
chown :group file

# View permissions
ls -l file
ls -ld directory
```

## Key Takeaways

1. **Two user types**: Root (all-powerful) and normal users (restricted)

2. **Two group types**: Primary (automatically created) and secondary (added manually)

3. **Three permission categories**:
   - User (u): File owner
   - Group (g): Users in file's group
   - Others (o): Everyone else

4. **Three permission types**:
   - Read (r): View contents
   - Write (w): Modify contents
   - Execute (x): Run file / enter directory

5. **Permission format**: `drwxrwxrwx`
   - 1st char: File type (d=directory, -=file)
   - Next 3: User permissions
   - Next 3: Group permissions
   - Last 3: Others permissions

6. **Key commands**:
   - `useradd`: Create users
   - `groupadd`: Create groups
   - `usermod`: Modify users (add to groups, lock/unlock)
   - `chmod`: Change permissions
   - `chown`: Change ownership (root only)
   - `id`: View user/group info

7. **Best practices**:
   - Work as normal user, not root
   - Use groups for shared access
   - Follow principle of least privilege
   - Never use 777 permissions
   - Protect sensitive files (600 or 700)

8. **Root power**: Can do anything, including switching users without passwords

9. **Security model**: Permissions prevent users from accessing each other's files

10. **Home directories**: Every user gets `/home/username` (except root gets `/root`)

## Looking Ahead

You now understand the Linux security model! This knowledge is crucial for:
- Setting up secure Docker containers
- Managing multi-user systems
- Troubleshooting access issues
- Writing secure scripts and applications

In the next chapter, we'll put your Linux knowledge to practical use with hands-on Docker exercises. You'll see how these user and permission concepts apply in containerized environments.

Remember: Linux permissions might seem complex at first, but they're actually quite logical once you understand the pattern. Practice with different scenarios, and soon checking and modifying permissions will become second nature! 🔒
