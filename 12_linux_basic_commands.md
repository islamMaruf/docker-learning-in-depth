# Chapter 12: Linux Basic Commands

> **In one sentence:** Learn to move around the Linux file system and to create, view, copy, move, search and delete files from the command line, the daily toolkit for working inside containers and on servers.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~45 minutes, but the real learning is in typing the commands yourself.

**Prerequisites:** [Chapter 9](09_gnu_coreutils.md) (terminal, shell, pipes) and [Chapter 10](10_running_ubuntu_on_docker.md) (how to start a practice container).

**A safe place to practice.** Start a throw-away container. Anything you break vanishes when you exit:

```bash
docker run -it --rm ubuntu:24.04 bash
```

---

## What you will learn

- How to read the shell **prompt**
- The layout of the Linux **file system** and what each top-level folder is for
- **Paths**: absolute vs relative, and the shortcuts `~`, `.`, `..`, `-`
- Navigate: `pwd`, `ls`, `cd`
- Create: `mkdir`, `touch`, `echo`, redirection
- View: `cat`, `less`, `head`, `tail`, `wc`
- Copy, move, rename, delete: `cp`, `mv`, `rm`, `rmdir`
- **Wildcards**, links, searching (`find`, `grep`), and getting help (`man`, `--help`)
- Safety habits, common mistakes, and a checklist of exercises

---

## 1. Reading the prompt

```
root@3f9c2a1b7d4e:/app#
│    │            │   └ # = you are root   ($ = normal user)
│    │            └ current directory (~ means your home)
│    └ hostname (in a container: the container ID)
└ user name
```

- The **last character** is a hint. `#` traditionally means the user is **root** (the administrator with unlimited power) and `$` a normal user. It is only a convention set by your shell's configuration (`PS1`); zsh often uses `%`. Use `whoami` or `id` to be sure.
- **`~`** in the prompt means "in the home directory" (`/root` for root, `/home/<user>` for others).
- In Docker containers you are root by default. That is convenient for learning, but remember: **root can delete anything**.

```bash
whoami          # root
id              # uid=0(root) gid=0(root) groups=0(root)
hostname
echo $HOME      # /root
```

`sudo` means "**s**uper**u**ser **do**": run *one command* with administrator rights (`sudo apt update`). Regular Ubuntu servers use it; minimal Docker images usually don't include it because you're already root. `su` ("switch user") starts a whole session as another user, which is why you may see `sudo su` in tutorials (though `sudo -i` is the cleaner way).

---

## 2. The Linux file system

Linux has **one tree** with a single top, called **root** and written **`/`**. (Do not confuse it with `/root`, the *home folder of the user root*.) There are no drive letters like `C:`. Other disks and USB drives are **mounted** somewhere inside the same tree.

```
/
├── bin  → usr/bin      essential programs (on modern systems these are links into /usr)
├── boot                kernel and boot loader files (mostly empty in containers)
├── dev                 device files: /dev/null, /dev/sda, ...
├── etc                 system-wide configuration files
├── home                users' personal folders: /home/alice
├── lib, lib64          shared libraries (like DLLs on Windows)
├── media, mnt          mount points for removable / temporary disks
├── opt                 optional third-party software
├── proc                virtual: information about processes and the kernel
├── root                home folder of the root user
├── run                 runtime data (PID files, sockets) since boot
├── sbin → usr/sbin     administration programs
├── srv                 data served by this machine (web, ftp)
├── sys                 virtual: devices and kernel settings
├── tmp                 temporary files (may be cleared at reboot)
├── usr                 the bulk of installed programs, libraries and docs
└── var                 variable data: logs, caches, databases, mail
```

Key facts:

- **`/etc`** is where you find configuration; **`/var/log`** is where you look when something breaks.
- **`/proc` and `/sys`** aren't stored on disk; the kernel generates them on the fly (Chapter 2).
- **Everything is a file**: keyboards, disks and processes appear under paths.
- File names are **case-sensitive**: `Notes.txt`, `notes.txt` and `NOTES.TXT` are three different files.
- A name starting with **`.`** is **hidden** (e.g. `.bashrc`) from a normal `ls`.

---

## 3. Paths

A **path** says where a file or folder is.

| Kind | Starts with | Meaning | Example |
|---|---|---|---|
| **Absolute** | `/` | From the top of the tree; the same from anywhere | `/etc/hostname` |
| **Relative** | anything else | From your **current directory** | `docs/readme.txt` |

Special names available everywhere:

| Symbol | Meaning |
|---|---|
| `/` | The root of the tree |
| `~` | Your home directory (`/root`, `/home/alice`) |
| `.` | The current directory |
| `..` | The parent directory (one level up) |
| `-` | (only with `cd`) the previous directory you were in |

```
Current directory: /app/habib/rahim

../          → /app/habib
../..        → /app
../../ataur  → /app/ataur
./run.sh     → /app/habib/rahim/run.sh
```

Use **absolute paths** in scripts and Dockerfiles for clarity, and **relative paths** for quick work inside a project.

---

## 4. Navigating: `pwd`, `ls`, `cd`

```bash
pwd                 # print working directory → where am I?
ls                  # list files here
ls /etc             # list another place without going there
ls -l               # long format: permissions, owner, size, date
ls -a               # include hidden files (names starting with .)
ls -la              # both
ls -lh              # human-readable sizes (K, M, G)
ls -lt              # newest first (sort by time); add -r to reverse
ls -R               # recursive: list sub-folders too
ls -d */            # only directories
```

Reading `ls -l`:

```
drwxr-xr-x 2 root root 4096 Jan 15 10:30 docs
-rw-r--r-- 1 root root  220 Jan 15 10:30 notes.txt
lrwxrwxrwx 1 root root    7 Jan 15 10:30 bin -> usr/bin
```

| Field | Meaning |
|---|---|
| First character | Type: `-` file, `d` directory, `l` symbolic link |
| Next nine | Permissions: owner / group / others, each `rwx` (Chapter 13) |
| Number | Hard links count |
| `root root` | Owner and group |
| Number | Size in bytes |
| Date | Last modification |
| Last | Name (`-> target` for links) |

**Changing directory:**

```bash
cd /etc          # absolute path
cd ..            # up one level
cd ../..         # up two levels
cd ~             # home    (same as just: cd)
cd -             # back to where I was before (toggles between two places)
cd /             # the top
cd "My Folder"   # quote names with spaces
```

**Tab completion**: type the first letters and press **Tab** to auto-complete a name; press Tab twice to list candidates. It saves typing and prevents typos. (Ubuntu's Docker image has bash's completion available, but a few very minimal images may not.) **Up/Down arrows** recall earlier commands; **Ctrl+R** searches history; **Ctrl+C** stops a running command; **Ctrl+L** clears the screen.

---

## 5. Creating things

### Directories

```bash
mkdir project                 # one directory
mkdir a b c                   # several
mkdir -p project/src/utils    # -p = create missing parent directories (and no error if it exists)
```

Without `-p`, `mkdir x/y` fails with `No such file or directory` if `x` doesn't exist.

### Files

```bash
touch a.txt               # create an empty file (or update its timestamp if it exists)
touch a.txt b.txt c.txt
echo "Hello" > a.txt      # write text into a file, OVERWRITING what was there
echo "World" >> a.txt     # APPEND to the end
```

> **Redirection** (`>` and `>>`) is done by the **shell**, not by `echo`. It works with any command's output: `ls > list.txt`.

| Operator | Effect |
|---|---|
| `>` | Send output to a file; **replaces** the file's content |
| `>>` | Send output to a file; **adds** to the end |
| `<` | Read input from a file |
| `2>` | Send error messages to a file |
| `\|` | Pipe: send output to another command |

**A common mistake:** `echo Hello World > file.txt` works fine. It writes `Hello World` into the file. (Quotes are only *required* when the text contains special characters like `*`, `$`, `;` or `>`.)

### Editing text
Small images have no editor by default. Install one (`apt-get update && apt-get install -y nano`) or write with `echo`/`cat`:

```bash
cat > hello.sh << 'EOF'
#!/bin/sh
echo "Hello from a script"
EOF
```

(That's a *here-document*: everything until `EOF` goes into the file.) Editors: **nano** (easy: Ctrl+O save, Ctrl+X exit), **vi/vim** (powerful: `i` to insert, `Esc`, then `:wq` to save and quit, `:q!` to quit without saving).

---

## 6. Viewing files

```bash
cat a.txt             # print the whole file
cat -n a.txt          # with line numbers
cat a.txt b.txt       # several files, one after another
less bigfile.log      # scroll: Space/PageDown, b = back, /word = search, q = quit
head a.txt            # first 10 lines
head -n 3 a.txt       # first 3
tail a.txt            # last 10 lines
tail -n 2 a.txt       # last 2
tail -f app.log       # FOLLOW the end of a growing file (Ctrl+C to stop); great for logs
wc -l a.txt           # count lines (-w words, -c bytes)
file photo.jpg        # what kind of file is this?
stat a.txt            # detailed info (size, times, permissions)
```

---

## 7. Copy, move, rename, delete

| Command | Does | Notes |
|---|---|---|
| `cp src dst` | Copy a file | `cp -r dir1 dir2` for directories; `cp -a` preserves permissions and times; `cp -i` asks before overwriting |
| `mv src dst` | **Move** *or* **rename** | Same command for both; overwrites silently unless `-i` |
| `rm file` | Delete a file | **Permanent. No trash bin** |
| `rm -r dir` | Delete a directory and everything inside | |
| `rmdir dir` | Delete an **empty** directory | Safer, fails if not empty |
| `rm -i file` | Ask before deleting | Good habit while learning |
| `rm -f file` | "Force": no prompts, ignore missing files | |

```bash
cp a.txt backup.txt            # copy to a new name
cp a.txt docs/                 # copy into a directory (keeps the name)
cp -r docs docs_backup         # copy a directory
mv backup.txt old.txt          # RENAME
mv old.txt docs/               # MOVE into docs/
mv old.txt docs/renamed.txt    # move AND rename
rm docs/renamed.txt
rm -r docs_backup
```

The rule for `cp` and `mv`: if the destination is an **existing directory**, the file goes *inside* it; otherwise the destination is the *new name*.

### ⚠️ Deleting safely
- **There is no undo.** Deleted files are not in a recycle bin.
- Run `pwd` and `ls` **before** any `rm -r`.
- Be specific: `rm -r project/build`, not `rm -rf *`.
- Never combine `rm -rf` with an unchecked variable: `rm -rf "$DIR/"` deletes `/` if `$DIR` is empty. In scripts, use `set -u`, or `${DIR:?}`.
- GNU `rm` refuses to remove `/` itself unless you add `--no-preserve-root`, but it will happily delete everything *inside* your current directory tree.

---

## 8. Wildcards (globbing)

The shell expands these patterns **before** the command runs:

| Pattern | Matches |
|---|---|
| `*` | Any number of characters (including none) |
| `?` | Exactly one character |
| `[abc]` | One character from the set |
| `[0-9]` | One character in the range |
| `{a,b,c}` | (brace expansion) each listed alternative |

```bash
ls *.txt               # all .txt files
ls file?.txt           # file1.txt, fileA.txt ...
ls report_[0-9][0-9].csv
touch note_{1..5}.txt  # creates note_1.txt ... note_5.txt
rm *.tmp               # delete all .tmp files (check with ls *.tmp first!)
```

`*` does **not** match hidden files (starting with `.`) by default.

---

## 9. Links

```bash
ln -s /etc/hostname myhost     # symbolic (soft) link: a shortcut that points to a path
ls -l myhost                   # myhost -> /etc/hostname
ln a.txt a_hard.txt            # hard link: a second NAME for the same file content
readlink -f myhost             # resolve the real path
```

Symbolic links can point to directories and across disks, and break if the target is removed. Hard links can't. You met symlinks already: `/bin -> usr/bin`, and in Alpine `/bin/ls -> /bin/busybox`.

---

## 10. Finding things

### `find`: search by name, type, size, age

```bash
find /etc -name "*.conf"             # by name pattern
find . -type d                       # directories only  (-type f = files)
find . -name "*.log" -size +1M       # bigger than 1 MB
find . -mtime -1                     # modified in the last day
find /tmp -name "*.tmp" -delete      # find AND delete (careful!)
find . -name "*.txt" -exec wc -l {} \;   # run a command on each match
```

### `grep`: search inside files

```bash
grep "error" app.log              # lines containing "error"
grep -i "error" app.log           # case-insensitive
grep -n "error" app.log           # show line numbers
grep -r "TODO" .                  # search all files under here (recursive)
grep -v "debug" app.log           # lines NOT containing "debug"
grep -c "error" app.log           # just count matching lines
ps aux | grep nginx               # filter another command's output
```

### Locating programs

```bash
which ls           # where is the program that runs when I type ls?
type cd            # builtin or program?
whereis ls         # binary, source and man page locations
command -v curl    # portable way to check something is installed (good in scripts)
```

---

## 11. Getting help

```bash
ls --help                # short usage summary for most commands
man ls                   # the manual page (q to quit); may be missing in minimal Docker images
type -a echo             # what will actually run
history | tail           # my recent commands
```

**Minimal Docker images often exclude man pages** ("This system has been minimized"). Use `--help`, or read the manual online at man7.org, or `man` on your own computer.

---

## 12. A complete practice session

```bash
# 1. Start at home and look around
cd ~ && pwd && ls -la

# 2. Build a project
mkdir -p projects/myapp/src projects/myapp/docs
cd projects/myapp

# 3. Create and fill files
echo "print('Hello, World!')" > src/main.py
echo "# My App" > docs/README.md
echo "A sample project." >> docs/README.md

# 4. Inspect
ls -R
cat docs/README.md
wc -l docs/README.md

# 5. Copy, rename, delete
cp docs/README.md docs/README.bak
mv docs/README.bak docs/OLD.md
rm docs/OLD.md

# 6. Search
grep -rn "Hello" .

# 7. Navigate using different path styles
cd /                      # absolute
cd ~/projects/myapp/src   # absolute with ~
cd ../docs                # relative
cd -                      # back to src
pwd
```

---

## 13. Cheat sheet

| Goal | Command |
|---|---|
| Where am I? | `pwd` |
| List (details, hidden) | `ls -la` |
| Go somewhere / home / back / up | `cd path` · `cd` · `cd -` · `cd ..` |
| Make folder (with parents) | `mkdir -p a/b/c` |
| Create empty file | `touch f` |
| Write / append | `echo "x" > f` · `echo "x" >> f` |
| Show file / start / end / follow | `cat f` · `head -n 5 f` · `tail -n 5 f` · `tail -f f` |
| Page through a big file | `less f` |
| Copy file / folder | `cp a b` · `cp -r a b` |
| Move or rename | `mv a b` |
| Delete file / folder / empty folder | `rm f` · `rm -r d` · `rmdir d` |
| Search by name / by content | `find . -name "*.py"` · `grep -rn "text" .` |
| Size of a folder / free disk | `du -sh dir` · `df -h` |
| Help | `cmd --help` · `man cmd` |

---

## 14. Common mistakes and myths

| Mistake or myth | Correction |
|---|---|
| "`echo Hello World > f` creates three files" | It writes `Hello World` into `f`. Redirection takes exactly one target |
| "`>` and `>>` are the same" | `>` **overwrites**, `>>` **appends** |
| "`rm` has a trash can" | No. Deleted means gone |
| "`cd` with no argument does nothing" | It goes to your home directory |
| "`/root` and `/` are the same" | `/` is the top of the tree, `/root` is root's home |
| "`ls` shows all files" | Hidden files need `-a` |
| "Exiting a container deletes it" | Only with `--rm` (or `docker rm`); otherwise it's just stopped |
| "Case doesn't matter" | Linux is case-sensitive |
| "A file's extension decides its type" | Linux mostly ignores extensions; use `file` to check |
| "`$` means I'm safe" | Prompt symbols are only conventions; check with `whoami` |
| "Spaces in names are fine unquoted" | `rm my file` tries to delete `my` and `file`. Quote: `rm "my file"` |

---

## 15. Summary

- Prompt: `user@host:directory#`; `#` usually means root.
- One tree rooted at `/`; know `/etc`, `/var`, `/home`, `/tmp`, `/usr`, `/proc`.
- Absolute paths start with `/`; relative paths start from where you are; `.`, `..`, `~`, `-` are shortcuts.
- Core commands: `pwd`, `ls`, `cd`, `mkdir`, `touch`, `cat`, `less`, `head`, `tail`, `cp`, `mv`, `rm`.
- `>` overwrites, `>>` appends, `|` pipes; wildcards `* ? []`; search with `find` and `grep`.
- No undo. Look before you delete.

---

## 16. Check your understanding

1. Your prompt is `alice@web:/var/log$`. Who are you, where are you, and are you root?
2. What is the difference between `cd /app` and `cd app`?
3. From `/app/habib/rahim`, what does `cd ../..` do?
4. What does `mkdir -p a/b/c` do that `mkdir a/b/c` may not?
5. What is the difference between `mv a b` when `b` is a directory and when it is not?
6. How would you find every `.log` file under `/var` that is bigger than 10 MB?
7. What will `echo "one" > f; echo "two" > f; cat f` print?

<details>
<summary>Answers</summary>

1. User `alice`, on host `web`, in `/var/log`; `$` suggests a normal user (verify with `id`).
2. `/app` is absolute (always from the root). `app` is relative to the current directory.
3. It moves up two levels: to `/app`.
4. It creates missing parent directories, and doesn't complain if they exist.
5. If `b` is an existing directory, `a` is moved into it. Otherwise `a` is renamed to `b`.
6. `find /var -name "*.log" -size +10M`
7. `two`. The second `>` overwrote the first.
</details>

**Exercises**

1. Create `projects/backend/{src,tests}` and `projects/frontend/{components,styles}` with as few commands as possible (hint: brace expansion with `mkdir -p`).
2. Create `notes.txt` with two lines using `echo`, copy it to `notes.bak`, rename `notes.txt` to `diary.txt`, delete `notes.bak`.
3. Create a file with 10 lines (`seq 1 10 > numbers.txt`), show only the first 3 and last 3 lines, then count them with `wc -l`.
4. From `/usr/share`, go to `/etc` with an absolute path, then back with `cd -`, then to `/var/log` using a relative path.
5. Use `grep -rn root /etc/passwd /etc/group`, then `find /etc -name "*.conf" | wc -l`.

---

**Next:** [Chapter 13 – Users, Groups and Permissions](13_managing_user_group_and_permission.md)
