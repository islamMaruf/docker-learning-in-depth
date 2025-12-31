# Chapter 19: Linux Basic Commands

## Overview

Welcome to one of the most practical chapters in your Docker and Linux journey! This chapter will teach you the fundamental commands that every Linux user needs to master. Whether you're working inside a Docker container or directly on a Linux system, these commands are your daily tools for navigating, managing, and manipulating files and directories.

In this chapter, we'll cover everything from understanding the terminal prompt structure to mastering file operations. By the end, you'll be comfortable navigating the Linux filesystem, creating and deleting files and directories, viewing and editing file contents, and understanding the difference between absolute and relative paths.

Think of this chapter as learning to walk before you run. These basic commands form the foundation for everything else you'll do in Linux and Docker. They might seem simple at first, but mastering them will make you significantly more efficient and confident when working with containers and Linux systems.

## The Terminal Prompt: Understanding Your Identity

### Anatomy of the Terminal Prompt

When you open a terminal in Linux, you're greeted with a prompt that might look confusing at first. Let's break it down piece by piece:

```
username@hostname ~ %
```

This simple line tells you a lot about your current session:

1. **username**: This is the name of the user currently logged in. In the example, it could be something like `habiburrahman` or `root`.

2. **@**: This is just a separator, like saying "at".

3. **hostname**: This is the name of the machine you're working on. In Docker containers, this is often a random string of characters assigned to the container.

4. **~**: This tilde character represents your home directory. It's a shortcut that always points to your user's home directory (typically `/root` for the root user or `/home/username` for regular users).

5. **%** or **#** or **$**: This final character tells you important information about your shell and permissions:
   - **%**: Z Shell (zsh) with normal user privileges
   - **#**: Root user (superuser) with administrative privileges
   - **$**: Bash shell with normal user privileges

### The Significance of the Prompt Symbol

The symbol at the end of your prompt is more important than it might seem:

```bash
# Root user in bash
root@container:~#

# Normal user in zsh
habiburrahman@container:~%

# Normal user in bash
habiburrahman@container:~$
```

When you see `#`, you should be extra careful. The root user has unlimited power to modify, delete, or break the system. With great power comes great responsibility!

### Switching Between Users and Shells

You can switch to the root user using the `sudo su` command:

```bash
# Normal user prompt
habiburrahman@container:~%

# Switch to root user
$ sudo su
Password: [enter your password]

# Now you're root
root@container:~#
```

The `sudo` command breaks down as:
- **su**: Super User
- **do**: Do (execute)
- So `sudo su` means "execute as superuser to become superuser"

You can also switch between different shells. For example, switching from zsh to bash:

```bash
# Currently in zsh
habiburrahman@container:~%

# Switch to bash
% bash

# Now in bash (notice the $ symbol)
habiburrahman@container:~$
```

To exit and return to your previous shell, simply type `exit`.

## The Linux Filesystem: Your Digital Neighborhood

### Root Directory Structure

In Linux, everything starts from the root directory, represented by a single forward slash `/`. This is the top of the filesystem hierarchy, and all other directories branch out from here.

When you run the `ls /` command from the root directory, you'll see something like this:

```bash
/ $ ls
bin   dev   home  lib    media  opt   root  sbin  sys  usr
boot  etc   lib64 mnt    proc   run   srv   tmp   var
```

Let's understand what each of these directories contains:

### Essential System Directories

**1. /bin (Binaries)**
Contains essential command binaries (executable programs) that all users can run. Commands like `ls`, `pwd`, `mkdir`, `cat`, and many others live here. When you type a command, the shell looks in `/bin` to find the corresponding program.

**2. /boot (Boot Loader)**
Contains files needed to boot the system, including the Linux kernel and bootloader configuration.

**3. /dev (Devices)**
Contains device files that represent hardware components. Linux treats hardware devices as special files. For example, your hard drive might be `/dev/sda`.

**4. /etc (Etcetera/Configuration)**
Contains system-wide configuration files. Think of this as the "settings" folder for your entire Linux system.

**5. /home (User Home Directories)**
Contains personal directories for regular users. Each user gets their own subdirectory here, like `/home/username`. This is where users store their personal files and settings.

**6. /lib and /lib64 (Libraries)**
Contains shared libraries needed by programs in `/bin` and `/sbin`. Think of these as code libraries that multiple programs can use, similar to DLLs in Windows.

**7. /media (Removable Media)**
Mount point for removable media like USB drives, CD-ROMs, etc.

**8. /mnt (Mount)**
Temporary mount point for filesystems. Administrators can mount filesystems here temporarily.

**9. /opt (Optional)**
Contains optional software packages. Third-party software often gets installed here.

**10. /proc (Process Information)**
A virtual filesystem that provides information about running processes and the kernel. Files here don't actually exist on disk; they're generated on-the-fly by the kernel.

**11. /root (Root User Home)**
The home directory for the root user. Note the difference: `/root` is the root user's home, while `/` is the root of the filesystem.

**12. /run (Runtime Data)**
Contains runtime data for processes that started since the last boot.

**13. /sbin (System Binaries)**
Contains system administration binaries. These are typically used by the root user for system maintenance.

**14. /sys (System)**
A virtual filesystem providing information about the system and kernel, similar to `/proc`.

**15. /tmp (Temporary)**
Temporary files created by programs. This directory is typically cleared on reboot.

**16. /usr (User Programs)**
Contains user programs and data. This is often one of the largest directories, containing most of the programs you'll use.

**17. /var (Variable)**
Contains variable data like logs, caches, and spool files. This directory grows over time as logs accumulate.

### Visualizing the Filesystem Hierarchy

```
/                           (root directory)
├── bin/                    (essential commands)
├── boot/                   (boot files)
├── dev/                    (devices)
├── etc/                    (configuration)
├── home/                   (user homes)
│   └── username/
├── lib/                    (libraries)
├── media/                  (removable media)
├── mnt/                    (mount points)
├── opt/                    (optional software)
├── proc/                   (process info)
├── root/                   (root's home)
├── run/                    (runtime data)
├── sbin/                   (system binaries)
├── sys/                    (system info)
├── tmp/                    (temporary files)
├── usr/                    (user programs)
│   ├── bin/
│   ├── lib/
│   └── share/
└── var/                    (variable data)
    ├── log/
    └── cache/
```

## Core Navigation Commands

### pwd: Present Working Directory

The `pwd` command shows you exactly where you are in the filesystem:

```bash
$ pwd
/root
```

This is incredibly useful when you're navigating deep directory structures and lose track of your location.

### ls: Listing Directory Contents

The `ls` command is one of your most frequently used commands. It lists the contents of a directory.

**Basic usage:**
```bash
$ ls
bin   dev   home  lib    media  opt   root  sbin  sys  usr
boot  etc   lib64 mnt    proc   run   srv   tmp   var
```

**ls -l (long format):**
```bash
$ ls -l
drwxr-xr-x  2 root root 4096 Jan 15 10:30 bin
drwxr-xr-x  3 root root 4096 Jan 15 10:30 boot
drwxr-xr-x  5 root root  360 Jan 20 08:15 dev
```

The long format shows:
- File permissions (drwxr-xr-x)
- Number of links
- Owner name
- Group name
- File size
- Modification date
- Filename

**ls -a (show all, including hidden files):**
```bash
$ ls -a
.   ..   .bashrc   .profile   bin   boot   dev   etc
```

Files starting with a dot (.) are hidden files. The special entries `.` and `..` represent the current directory and parent directory, respectively.

**ls -la (combine long format with all files):**
```bash
$ ls -la
total 64
drwxr-xr-x  20 root root 4096 Jan 20 08:15 .
drwxr-xr-x  20 root root 4096 Jan 20 08:15 ..
-rw-r--r--   1 root root  220 Jan 15 10:30 .bashrc
```

This combines both options, showing all files (including hidden ones) in long format.

### cd: Changing Directories

The `cd` (change directory) command is how you navigate the filesystem. It has several powerful variations:

**Basic navigation:**
```bash
# Go to a specific directory
$ cd /bin
$ pwd
/bin

# Go to home directory
$ cd ~
$ pwd
/root

# Just cd with no arguments also goes home
$ cd
$ pwd
/root
```

**The tilde (~) shortcut:**
The tilde character is a powerful shortcut that always represents your home directory:

```bash
$ cd ~
$ pwd
/root
```

**Parent directory (..):**
Two dots represent the parent directory (one level up):

```bash
$ pwd
/app/habib/rahim
$ cd ..
$ pwd
/app/habib
```

**Current directory (.):**
A single dot represents the current directory:

```bash
$ cd .
# You stay in the same place
```

**Previous directory (-):**
The hyphen takes you back to your previous location:

```bash
$ cd /bin
$ pwd
/bin

$ cd /home
$ pwd
/home

$ cd -
/bin
```

**Navigating to the root:**
```bash
$ cd /
$ pwd
/
```

### Tab Completion: Your Best Friend

When using bash shell, pressing the Tab key will auto-complete commands, filenames, and directory names:

```bash
$ cd /bi[TAB]
# Automatically completes to:
$ cd /bin/

$ cd /ho[TAB]
# Shows suggestions:
home/  

$ cd hom[TAB]
# Completes to:
$ cd home/
```

Tab completion saves time and prevents typos. Press Tab twice to see all available options if there are multiple matches.

## Working with Directories

### mkdir: Creating Directories

The `mkdir` (make directory) command creates new directories:

**Creating a simple directory:**
```bash
$ mkdir app
$ ls
app
```

**Creating nested directories with -p:**
```bash
# This will fail without -p
$ mkdir habib/rahim
mkdir: cannot create directory 'habib/rahim': No such file or directory

# Use -p to create parent directories
$ mkdir -p habib/rahim
$ ls
habib

$ cd habib
$ ls
rahim
```

The `-p` flag stands for "parents" - it creates all necessary parent directories.

**Creating multiple directories:**
```bash
$ mkdir dir1 dir2 dir3
$ ls
dir1  dir2  dir3
```

### Navigating Directory Trees

Let's practice navigating a multi-level directory structure:

```bash
# Create a nested structure
$ mkdir -p /app/habib/rahim

# Navigate into it
$ cd /app/habib/rahim
$ pwd
/app/habib/rahim

# Go up one level
$ cd ..
$ pwd
/app/habib

# Go up another level
$ cd ..
$ pwd
/app

# Jump directly to a nested directory
$ cd habib/rahim
$ pwd
/app/habib/rahim

# Go to home directory
$ cd ~
$ pwd
/root

# Return to previous location
$ cd -
/app/habib/rahim
```

## Working with Files

### touch: Creating Empty Files

The `touch` command creates a new empty file:

```bash
$ touch a.txt
$ ls
a.txt
```

If the file already exists, `touch` updates its modification timestamp without changing the contents.

### cat: Viewing File Contents

The `cat` (concatenate) command displays file contents:

```bash
$ cat a.txt
# Shows the file contents (currently empty)
```

**Viewing multiple files:**
```bash
$ cat file1.txt file2.txt
# Displays both files in sequence
```

### echo: Writing to Files

The `echo` command prints text and can redirect output to files:

**Simple echo:**
```bash
$ echo "Hello World"
Hello World
```

**Writing to a file (overwrite):**
```bash
$ echo "Hello World" > a.txt
$ cat a.txt
Hello World
```

The single `>` operator overwrites the file completely.

**Appending to a file:**
```bash
$ echo "Hello World" >> a.txt
$ echo "How are you?" >> a.txt
$ cat a.txt
Hello World
How are you?
```

The double `>>` operator appends content without erasing existing data.

**Understanding the difference:**
```bash
# Overwrite example
$ echo "First line" > file.txt
$ echo "Second line" > file.txt
$ cat file.txt
Second line    # First line was erased!

# Append example
$ echo "First line" >> file.txt
$ echo "Second line" >> file.txt
$ cat file.txt
First line
Second line    # Both lines preserved!
```

### head: Viewing File Beginnings

The `head` command shows the first few lines of a file:

```bash
# Show first 10 lines (default)
$ head a.txt

# Show first 5 lines
$ head -n 5 a.txt
```

This is incredibly useful for previewing large files without displaying the entire contents.

### tail: Viewing File Endings

The `tail` command shows the last few lines of a file:

```bash
# Show last 10 lines (default)
$ tail a.txt

# Show last 2 lines
$ tail -n 2 a.txt
```

**Real-world example:**
```bash
# Create a file with multiple lines
$ echo "Line 1" >> data.txt
$ echo "Line 2" >> data.txt
$ echo "Line 3" >> data.txt
$ echo "Line 4" >> data.txt
$ echo "Line 5" >> data.txt

# View first 3 lines
$ head -n 3 data.txt
Line 1
Line 2
Line 3

# View last 2 lines
$ tail -n 2 data.txt
Line 4
Line 5
```

## File and Directory Operations

### rm: Removing Files

The `rm` (remove) command deletes files:

```bash
$ touch test.txt
$ ls
test.txt

$ rm test.txt
$ ls
# test.txt is gone
```

**Removing directories requires the -r flag:**
```bash
# This will fail
$ rm mydirectory
rm: cannot remove 'mydirectory': Is a directory

# Use -r for recursive removal
$ rm -r mydirectory
# Directory and all its contents removed
```

**Force removal with -f:**
```bash
# Remove without confirmation, even if write-protected
$ rm -rf mydirectory
```

The `-rf` combination is powerful but dangerous:
- `-r`: Recursive (remove directories and contents)
- `-f`: Force (no confirmation prompts)

**⚠️ WARNING:** The command `rm -rf /` will delete your entire system! Never run this command. Always double-check before using `rm -rf`.

### cp: Copying Files

The `cp` (copy) command duplicates files:

**Basic file copy:**
```bash
$ touch file1.txt
$ echo "Hello World" > file1.txt
$ cp file1.txt file2.txt

$ cat file2.txt
Hello World
```

**Copying directories requires -r:**
```bash
$ mkdir -p rahim
$ cp -r rahim ataur

$ ls
rahim  ataur
```

**Copying to a different location:**
```bash
# Copy file2.txt to parent directory as file3.txt
$ cp file2.txt ../rahim/file3.txt

# Verify
$ ls ../rahim/
file3.txt
```

### mv: Moving and Renaming

The `mv` (move) command serves two purposes: moving files and renaming them.

**Renaming a file:**
```bash
$ touch file3.txt
$ ls
file3.txt

$ mv file3.txt file4.txt
$ ls
file4.txt
```

**Moving files to different directories:**
```bash
$ mkdir documents
$ mv file4.txt documents/
$ ls documents/
file4.txt
```

**Moving and renaming simultaneously:**
```bash
$ mv file1.txt documents/renamed_file.txt
$ ls documents/
renamed_file.txt
```

## Understanding Paths: Absolute vs. Relative

### Absolute Paths

An absolute path always starts from the root directory (`/`) and specifies the complete path to a file or directory:

```bash
$ cd /app/habib/rahim
$ pwd
/app/habib/rahim

# Absolute path example
$ cd /bin
$ pwd
/bin
```

Absolute paths always begin with `/` and specify the exact location regardless of where you currently are.

### Relative Paths

A relative path specifies a location relative to your current directory:

```bash
# Currently in /app/ataur
$ pwd
/app/ataur

# Relative path using ..
$ cd ../habib/rahim
$ pwd
/app/habib/rahim
```

**Relative path components:**
- `.` = current directory
- `..` = parent directory
- `../..` = two levels up
- `dirname` = subdirectory in current location

**Practical example:**
```bash
# Currently in /app/habib/rahim
$ pwd
/app/habib/rahim

# Go to parent's parent
$ cd ../..
$ pwd
/app

# Navigate using relative path
$ cd habib/rahim
$ pwd
/app/habib/rahim
```

### When to Use Each

**Use absolute paths when:**
- Writing scripts that must work from any location
- Specifying system directories like `/etc`, `/var/log`
- You want to be explicit and avoid ambiguity

**Use relative paths when:**
- Working within a project directory structure
- The exact absolute location might change
- You want shorter, more readable commands

## Practical Command Examples

Let's put everything together with a practical workflow:

```bash
# 1. Start in home directory
$ cd ~
$ pwd
/root

# 2. Create a project structure
$ mkdir -p projects/myapp/src
$ mkdir -p projects/myapp/docs

# 3. Navigate to the project
$ cd projects/myapp

# 4. Create some files
$ touch src/main.py
$ touch docs/README.md

# 5. Add content to files
$ echo "print('Hello, World!')" > src/main.py
$ echo "# My Application" > docs/README.md
$ echo "This is a sample project." >> docs/README.md

# 6. View the files
$ cat src/main.py
print('Hello, World!')

$ cat docs/README.md
# My Application
This is a sample project.

# 7. List the project structure
$ ls -R
.:
docs  src

./docs:
README.md

./src:
main.py

# 8. Copy a file
$ cp docs/README.md docs/README_backup.md

# 9. Remove a file
$ rm docs/README_backup.md

# 10. Return to home
$ cd ~
```

## Running Docker Ubuntu Container

To practice these commands, you can run an Ubuntu container:

```bash
# Pull and run Ubuntu 22.04 with bash
$ docker run -it ubuntu:22.04 bash

# You'll see a root prompt
root@container-id:/#

# Now you can practice all the commands we learned
root@container-id:/# pwd
/

root@container-id:/# ls
bin  boot  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

root@container-id:/# cd /bin
root@container-id:/bin# ls
# You'll see all the binary commands
```

The `-it` flags mean:
- `-i`: Interactive (keep STDIN open)
- `-t`: TTY (allocate a pseudo-terminal)

Together, they give you an interactive terminal session inside the container.

## Command Summary Cheat Sheet

Here's a quick reference for all the commands we covered:

### Navigation
```bash
pwd                    # Print working directory
ls                     # List directory contents
ls -l                  # Long format listing
ls -a                  # Show hidden files
ls -la                 # Long format with hidden files
cd directory           # Change directory
cd ~                   # Go to home directory
cd ..                  # Go up one level
cd -                   # Go to previous directory
cd /                   # Go to root directory
```

### Directory Operations
```bash
mkdir dirname          # Create directory
mkdir -p path/to/dir   # Create nested directories
```

### File Operations
```bash
touch filename         # Create empty file
cat filename           # View file contents
echo "text"            # Print text
echo "text" > file     # Overwrite file
echo "text" >> file    # Append to file
head -n 5 file         # Show first 5 lines
tail -n 5 file         # Show last 5 lines
```

### File Management
```bash
rm filename            # Remove file
rm -r dirname          # Remove directory recursively
rm -rf dirname         # Force remove (dangerous!)
cp source dest         # Copy file
cp -r source dest      # Copy directory
mv source dest         # Move or rename
```

### Path Types
```bash
/absolute/path         # Absolute path from root
relative/path          # Relative to current directory
../parent              # Parent directory
./current              # Current directory
```

## Best Practices and Tips

### 1. Always Know Where You Are
Before running commands, especially destructive ones like `rm -rf`, always check your current location with `pwd`.

### 2. Use Tab Completion
Tab completion is your friend. It saves time and prevents typos. Get in the habit of pressing Tab after typing a few characters.

### 3. Be Careful with rm -rf
The `rm -rf` command is powerful and permanent. There's no "undo" or "trash" in Linux command line. Double-check before executing.

### 4. Use ls to Verify
After operations like copy or move, use `ls` to verify that everything worked as expected.

### 5. Practice in a Safe Environment
Docker containers are perfect for practice. If you break something, just delete the container and start fresh:

```bash
# Exit the container
$ exit

# The container is gone - no harm done!
```

### 6. Learn the Difference Between Shells
Zsh (%) and Bash ($) have different features. Bash is more universal, zsh has better tab completion and suggestions. Know which one you're using.

### 7. Understand File Permissions
The long listing (`ls -l`) shows file permissions. While we didn't cover this in detail, understanding `drwxr-xr-x` will be important as you advance.

## Common Mistakes and How to Avoid Them

### Mistake 1: Forgetting Spaces in Echo Commands
```bash
# Wrong - creates multiple files
$ echo Hello World > file.txt
# Creates: file.txt, Hello, World (three separate files!)

# Correct - use quotes
$ echo "Hello World" > file.txt
```

### Mistake 2: Confusing > and >>
```bash
# This overwrites the file each time
$ echo "Line 1" > file.txt
$ echo "Line 2" > file.txt
# Result: Only "Line 2" remains

# This appends
$ echo "Line 1" >> file.txt
$ echo "Line 2" >> file.txt
# Result: Both lines preserved
```

### Mistake 3: Forgetting -r with Directories
```bash
# This fails
$ rm directory
rm: cannot remove 'directory': Is a directory

# This works
$ rm -r directory
```

### Mistake 4: Using Relative Paths Without Understanding Current Location
```bash
$ pwd
/home/user

$ cd ../etc
# Where are you now? Not in /etc!
# You're in /etc because you went up from /home/user
```

### Mistake 5: Not Checking Before rm -rf
```bash
# Dangerous - always verify first!
$ rm -rf *

# Better approach
$ ls              # Check what's here first
$ rm -rf specific_directory  # Be specific
```

## ASCII Diagram: Directory Navigation

```
Root (/)
    │
    ├── bin/
    ├── home/
    │   └── user/           ← cd ~/user takes you here
    ├── app/
    │   ├── habib/
    │   │   └── rahim/      ← You are here (pwd: /app/habib/rahim)
    │   │       │
    │   │       ├── cd ..   → Takes you to /app/habib
    │   │       ├── cd ../.. → Takes you to /app
    │   │       └── cd /    → Takes you to /
    │   │
    │   └── ataur/
    │
    └── usr/
        └── bin/

Navigation Examples:
• cd ~         → Go to /home/user (or /root for root user)
• cd /         → Go to root directory
• cd ..        → Go up one level
• cd -         → Go to previous directory
• cd /app      → Absolute path to /app
• cd habib     → Relative path (only works if habib is in current dir)
```

## Exercises

### Exercise 1: Directory Navigation
1. Start in your home directory (`cd ~`)
2. Check your location (`pwd`)
3. List all contents including hidden files (`ls -la`)
4. Go to the root directory (`cd /`)
5. List contents (`ls`)
6. Return to home (`cd ~`)

### Exercise 2: Creating a Project Structure
Create this directory structure:
```
projects/
├── backend/
│   ├── src/
│   └── tests/
└── frontend/
    ├── components/
    └── styles/
```

Commands:
```bash
mkdir -p projects/backend/src
mkdir -p projects/backend/tests
mkdir -p projects/frontend/components
mkdir -p projects/frontend/styles
```

### Exercise 3: File Operations
1. Create a file called `notes.txt`
2. Add "Day 1: Started learning Linux" to it
3. Display the contents
4. Add "Day 2: Learned basic commands" to it
5. Display the contents again
6. Copy the file to `notes_backup.txt`
7. Rename `notes.txt` to `diary.txt`
8. Delete `notes_backup.txt`

### Exercise 4: Path Practice
Starting from `/app/habib/rahim`:
1. Navigate to `/app` using relative path
2. Navigate to `/bin` using absolute path
3. Navigate back to `/app/habib/rahim` using absolute path
4. Navigate to `/app/ataur` using relative path from rahim
5. Return to home directory

### Exercise 5: Advanced File Operations
1. Create a file with 10 lines (use echo and >> 10 times)
2. View only the first 3 lines
3. View only the last 3 lines
4. Copy the file to a subdirectory using relative path
5. Verify the copy was successful

## Key Takeaways

1. **The prompt tells a story**: Username, hostname, current directory, and shell type are all visible in your prompt.

2. **/ is the root**: Everything in Linux starts from the root directory, and all paths branch from there.

3. **Essential commands**: `pwd`, `ls`, `cd`, `mkdir`, `touch`, `cat`, `echo`, `head`, `tail`, `rm`, `cp`, and `mv` are your daily tools.

4. **Two path types**: Absolute paths start with `/` and relative paths are relative to your current location.

5. **Special directory shortcuts**:
   - `~` = home directory
   - `.` = current directory
   - `..` = parent directory
   - `-` = previous directory

6. **Redirection operators**:
   - `>` = overwrite file
   - `>>` = append to file

7. **Flags modify behavior**:
   - `-r` = recursive (for directories)
   - `-f` = force (skip confirmations)
   - `-p` = parents (create parent directories)
   - `-l` = long format listing
   - `-a` = show all (including hidden files)

8. **Tab completion saves time**: Use it liberally to avoid typos and speed up your workflow.

9. **Practice in containers**: Docker containers are perfect for learning because mistakes don't matter - just delete and start fresh.

10. **Be careful with rm -rf**: This command is permanent and powerful. Always verify your location and target before executing.

## Looking Ahead

You now have a solid foundation in Linux basic commands. In the next chapter, we'll build on this knowledge by exploring:
- User and group management
- File permissions and ownership
- More advanced file manipulation techniques
- Process management
- Environment variables

These basic commands might seem simple, but they're the building blocks for everything else. Practice them daily, and they'll become second nature. Remember: every Linux expert started exactly where you are now - learning `ls`, `cd`, and `pwd` for the first time.

Happy commanding! 🐧
