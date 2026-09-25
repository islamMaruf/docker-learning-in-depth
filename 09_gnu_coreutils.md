# Chapter 9: GNU Coreutils, Shells and Terminals

> **In one sentence:** When you type `ls` in a terminal, a **terminal** shows your keystrokes, a **shell** interprets them and starts a **program** from **coreutils**, and that program asks the **kernel** to do the work.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~35 minutes

**Prerequisites:** [Chapter 2 – Kernel](02_kernel.md), [Chapter 8 – Linux](08_linux.md).

---

## What you will learn

- Where the GNU project and the classic Unix tools come from
- What **coreutils** are (and which everyday tools are *not* part of them)
- The difference between a **terminal**, a **shell**, and a **command**
- Shell types: `sh`, `bash`, `zsh`, `dash`, BusyBox `ash`, and why it matters in Docker
- How a shell finds and runs a command (`PATH`, builtins, aliases)
- The three building blocks of the command line: **pipes**, **redirection** and **exit codes**
- Desktop environments (briefly) and how they relate
- How all this appears inside containers (`docker exec`, `RUN`, `CMD`)

---

## 1. A little history

In the early 1980s, Unix was powerful but proprietary. In 1983 **Richard Stallman** launched the **GNU Project** ("GNU's Not Unix", a recursive acronym) to build a completely *free* Unix-like system: free meaning users may run, study, share and modify the software. The GNU project produced many core pieces:

| GNU component | Job |
|---|---|
| **GCC** | Compiler collection (C, C++, ...) |
| **glibc** | The standard C library |
| **Bash** | The most common shell (first released 1989) |
| **Coreutils** | The basic file, text and shell utilities |
| **GNU Make, GDB, grep, sed, tar ...** | Build tool, debugger, text search, stream editor, archiver |

GNU was missing one big piece: a working kernel. In 1991 Linus Torvalds' **Linux** kernel filled that gap. Together they made a full free operating system, which is why the tools on a typical Linux system come from GNU.

---

## 2. What are coreutils?

**GNU Coreutils** is a single package of about a hundred small programs for the most basic tasks: working with files, text and the system. Each one does one job, and you combine them.

| Category | Commands (all in coreutils) |
|---|---|
| **Files and directories** | `ls`, `cp`, `mv`, `rm`, `mkdir`, `rmdir`, `touch`, `ln`, `cat`, `head`, `tail`, `stat`, `du`, `df`, `chmod`, `chown`, `chgrp`, `pwd` |
| **Text** | `sort`, `uniq`, `wc`, `cut`, `tr`, `tac`, `paste`, `fold`, `nl` |
| **Output & scripting** | `echo`, `printf`, `yes`, `true`, `false`, `sleep`, `test`, `env`, `expr` |
| **System info** | `uname`, `hostname` (in some versions), `whoami`, `id`, `date`, `nproc`, `uptime` (from procps on many systems) |

### Common tools that are *not* coreutils
People often assume these are, but they come from other packages:

| Tool | Actual package |
|---|---|
| `grep`, `sed`, `awk`, `find`, `tar`, `less` | Their own GNU (or similar) packages |
| `ps`, `top`, `kill` (the program), `free`, `uptime` | **procps** |
| `bash`, `zsh` | The shells themselves |
| `apt`, `dnf`, `apk` | Package managers |
| `cd`, `export`, `alias` | **Shell builtins** (see section 4) |

**Where are they?** Normally `/usr/bin` (and `/bin`, which on modern systems is a link to `/usr/bin`):

```bash
which ls          # /usr/bin/ls
file /usr/bin/ls  # ELF 64-bit executable
ls --version      # first line says "ls (GNU coreutils) 9.x"
```

The commands are ordinary compiled programs, not magic.

---

## 3. Terminal vs shell vs command

People use "terminal", "console", "shell" and "command line" as if they were the same. They are three separate things:

| Layer | What it is | Examples |
|---|---|---|
| **Terminal (emulator)** | A window application that shows text and sends your keystrokes. It does **not** understand commands | GNOME Terminal, Konsole, Windows Terminal, iTerm2, macOS Terminal, VS Code's terminal |
| **Shell** | A program that reads what you type, interprets it, and starts other programs | `bash`, `zsh`, `sh`, `fish`, `ash` |
| **Command / program** | The thing the shell runs | `ls`, `cat`, `docker`, `python` |

Analogy: the terminal is the picture frame, the shell is the assistant who understands your requests, and the commands are the workers who do the jobs.

```
Terminal window  ← draws the text, captures keys
   └── Shell (bash)  ← interprets the line, starts programs
          └── Program (ls)  ← does the work through system calls
                 └── Kernel
```

When you open a terminal, it starts a shell for you. You can start a different shell inside it and leave again:

```bash
echo $0       # bash
zsh           # start zsh (if installed)
echo $0       # zsh
exit          # back to bash; the terminal window never changed
```

(Historically, terminals were physical screen-and-keyboard devices connected to a big computer. The modern "terminal emulator" imitates them, using a kernel feature called a **pseudo-terminal (pty)**.)

---

## 4. How a shell runs a command

Type `ls -l /etc` and press Enter. The shell:

1. **Splits** the line into words: the command `ls` and arguments `-l`, `/etc`.
2. **Expands** things like `~`, `$VARIABLES` and wildcards (`*.txt`).
3. **Decides what `ls` is**, in this order:
   1. an **alias** (a nickname you defined),
   2. a **function**,
   3. a **shell builtin** (a command built into the shell itself, like `cd`, `export`, `echo`),
   4. an **executable file** found by searching the folders listed in **`PATH`**.
4. **Starts a new process** for the program (the `fork` + `execve` system calls; Chapter 2), waits for it to end, and stores its **exit code**.

```bash
type cd        # cd is a shell builtin
type ls        # ls is /usr/bin/ls   (or "aliased to ls --color=auto")
type -a echo   # shows all: a builtin AND /usr/bin/echo
echo $PATH     # /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

**Why `cd` must be a builtin:** each program runs in its own process, and a child process cannot change its parent's current directory. Only the shell itself can.

> **Docker hint:** in a Dockerfile, `RUN cd /app` changes directory only for that one `RUN` line's shell. The next `RUN` starts fresh. Use `WORKDIR` instead (Chapter 17).

### Shells you will meet

| Shell | Notes |
|---|---|
| **sh** | The POSIX standard shell: minimal, portable. On Debian/Ubuntu `/bin/sh` is actually **dash** (very fast, minimal); on Alpine it is BusyBox **ash** |
| **bash** | "Bourne Again SHell". Default on most Linux distros; has history, tab completion, arrays, and more |
| **zsh** | Powerful and customizable; default on macOS since Catalina |
| **fish** | Friendly, with helpful defaults; not POSIX compatible |
| **ksh, tcsh** | Older alternatives |

Scripts should start with a *shebang* line naming their interpreter, e.g. `#!/bin/sh` or `#!/usr/bin/env bash`. If you write `bash`-only features (like `[[ ... ]]`) but run under `sh`/`ash`, you get errors. This is a very common Docker surprise on Alpine.

```bash
echo $SHELL                 # your login shell
cat /etc/shells             # shells installed on the system
```

---

## 5. Essential commands with examples

Try these in a scratch folder: `mkdir -p ~/playground && cd ~/playground`.

```bash
pwd                        # where am I?
mkdir -p project/src       # create nested directories
cd project                 # go in
touch a.txt b.txt          # create empty files
echo "hello" > a.txt       # write text into a file (overwrites)
echo "world" >> a.txt      # append
cat a.txt                  # show file
ls -l                      # long listing: permissions, owner, size, date
ls -la                     # include hidden files (names starting with .)
cp a.txt c.txt             # copy
mv c.txt src/              # move (or rename: mv old new)
rm b.txt                   # delete a file (no recycle bin!)
rm -r src                  # delete a directory and contents
head -n 3 file             # first 3 lines
tail -f /var/log/syslog    # follow a file as it grows (Ctrl+C to stop)
wc -l a.txt                # count lines
sort names.txt | uniq -c   # count duplicates
du -sh .                   # size of the current folder
df -h                      # free disk space
```

> ⚠️ **Be careful with `rm -rf`**. It deletes without asking. Never run it with a path you have not double-checked, especially with `sudo` or variables that may be empty (`rm -rf "$DIR/"` with `$DIR` empty means `/`).

`ls -l` output explained:

```
-rw-r--r-- 1 alice staff 6 Jan 10 12:00 a.txt
│└┬┘└┬┘└┬┘   │    │      │
│ │  │  │    │    │      └ size in bytes
│ │  │  │    │    └ group
│ │  │  │    └ owner
│ │  │  └ permissions for others (r--)
│ │  └ permissions for group (r--)
│ └ permissions for owner (rw-)
└ file type: - file, d directory, l symlink
```

(Permissions are the subject of [Chapter 13](13_managing_user_group_and_permission.md).)

---

## 6. Pipes, redirection and exit codes

These three ideas turn small tools into powerful ones.

### Standard streams
Every program starts with three open "files":

| Name | Number | Default |
|---|---|---|
| **stdin** | 0 | keyboard |
| **stdout** | 1 | the terminal screen |
| **stderr** | 2 | the terminal screen (for error messages) |

### Redirection

```bash
ls > files.txt            # stdout → file (overwrite)
ls >> files.txt           # append
sort < files.txt          # file → stdin
ls /nonexistent 2> err.txt   # stderr → file
cmd > out.txt 2>&1        # both stdout and stderr → file
cmd > /dev/null 2>&1      # throw away all output
```

### Pipes
`|` connects one program's **stdout** to the next program's **stdin**:

```bash
ls /usr/bin | wc -l               # how many programs?
cat access.log | grep 404 | sort | uniq -c | sort -nr | head
```

Each stage does one small job. This is the "Unix philosophy": small tools that do one thing well, combined with pipes.

### Exit codes
Every program returns a number when it ends: **0 means success, anything else means failure**.

```bash
ls /etc; echo $?             # 0
ls /nonexistent; echo $?     # 2 (error)
cmd1 && cmd2                 # run cmd2 only if cmd1 succeeded
cmd1 || cmd2                 # run cmd2 only if cmd1 failed
cmd1 ; cmd2                  # run both regardless
```

Docker uses exit codes heavily: `docker run` returns the container's exit code; `docker ps -a` shows `Exited (0)` or `Exited (1)`. **137** typically means the process was killed (128 + signal 9), for example by the out-of-memory killer.

### Quoting and variables

```bash
NAME="Ada Lovelace"
echo "Hello, $NAME"     # double quotes: variables are expanded
echo 'Hello, $NAME'     # single quotes: literal text
export APP_ENV=prod     # visible to child processes (like the programs you start)
env | grep APP_ENV
```

Always quote variables that may contain spaces: `rm "$file"`, not `rm $file`.

---

## 7. Desktop environments (very briefly)

A **desktop environment** provides the graphical interface: windows, panels, file manager, settings, and a bundled terminal. Servers and containers usually don't have one.

| Desktop | Found on | Feel |
|---|---|---|
| **GNOME** | Ubuntu, Fedora, Debian (default) | Modern, minimal |
| **KDE Plasma** | Kubuntu, openSUSE, many others | Highly customizable |
| **Xfce / LXQt** | Xubuntu, Mint Xfce | Lightweight |
| **Aqua (macOS)** / **Windows shell** | macOS / Windows | Proprietary equivalents |

They matter for Docker only in the sense that the *terminal* you use lives in one. Everything in this chapter works over SSH on a server with no desktop at all.

---

## 8. Coreutils and shells inside containers

An image includes the user-space tools of its base distribution (Chapter 8):

| Base image | Core tools | Default shell |
|---|---|---|
| `ubuntu`, `debian` | GNU coreutils | `bash` (interactive), `dash` as `/bin/sh` |
| `alpine` | **BusyBox** (one binary, many commands) | `ash` via `/bin/sh`; **no bash** unless installed |
| distroless, `scratch` | Almost nothing, no shell | none |

```bash
docker run --rm ubuntu:24.04 ls --version | head -1     # GNU coreutils
docker run --rm alpine:3.20 ls --version 2>&1 | head -1 # BusyBox ... (error text mentions BusyBox)
docker run --rm alpine:3.20 sh -c 'readlink -f /bin/ls' # /bin/busybox
```

Practical consequences:

- **`docker exec -it <container> bash`** works only if `bash` exists. On Alpine use `sh`.
- **Options differ.** Some GNU-only flags (for example `ls --time-style`, `sed -i` behaviors, `grep -P`, `date -d`) may not work in BusyBox.
- **Scripts:** if a script uses `bash` features, either install bash (`apk add bash`) or write POSIX `sh`.
- **`RUN`, `CMD` and `ENTRYPOINT` forms** (Chapters 15–16): *shell form* (`CMD echo hi`) runs through `/bin/sh -c`, so it needs a shell and does variable expansion; *exec form* (`CMD ["echo","hi"]`) runs the program directly with no shell, so `$VAR` and pipes don't work unless you call a shell explicitly.
- **Minimal images have no tools.** In a distroless container, `docker exec ... sh` fails. Use `docker debug`, an ephemeral debug container, or `docker cp` instead.

---

## 9. Hands-on lab

**Lab 1 – What am I running?**
```bash
echo $SHELL; echo $0; ps -p $$ -o comm=
ps -o comm= -p $PPID          # the parent (your terminal, or sshd over SSH)
type cd ls echo
```

**Lab 2 – Watch a command run**
```bash
strace -f -e trace=execve,openat,getdents64,write ls 2>&1 | tail -15
```
You will see the shell-launched `ls` open the current directory, read entries (`getdents64`), and `write` the names to file descriptor 1 (stdout).

**Lab 3 – Build a pipeline**
```bash
printf 'b\na\nb\nc\nb\na\n' | sort | uniq -c | sort -nr
```
Expected: `3 b`, `2 a`, `1 c` (counts descending).

**Lab 4 – Exit codes**
```bash
true;  echo $?     # 0
false; echo $?     # 1
grep -q root /etc/passwd && echo found || echo missing
```

**Lab 5 – Same tool, two distributions**
```bash
docker run --rm ubuntu:24.04 sh -c 'echo "shell: $(readlink -f /bin/sh)"'   # /usr/bin/dash
docker run --rm alpine:3.20  sh -c 'echo "shell: $(readlink -f /bin/sh)"'   # /bin/busybox
docker run --rm alpine:3.20 bash -c 'echo hi'                                # fails: bash not found
```

---

## 10. Common mistakes and myths

| Mistake or myth | Correction |
|---|---|
| "Terminal and shell are the same" | Terminal displays; shell interprets |
| "`grep`, `ps`, `cd` are coreutils" | Mostly not. They belong to other packages, or the shell |
| "Every Linux has bash" | Alpine and minimal images don't |
| "`sh` is bash" | On many systems `sh` is dash or BusyBox ash, with fewer features |
| "`rm` moves files to trash" | It deletes permanently |
| "Spaces don't matter in filenames" | They do; always quote |
| "Output from `ls` goes through the shell" | The program writes to the terminal (stdout) directly; the shell just started it |
| "Exit code 0 means output was correct" | It only means the program said it succeeded |

---

## 11. Summary

- GNU supplied most of the classic user-space tools; **coreutils** is the set of ~100 basic ones (`ls`, `cp`, `cat`, `sort`, ...).
- **Terminal** (window) → **shell** (interpreter) → **program** → **kernel**.
- The shell resolves a command as alias → function → builtin → executable on `PATH`.
- **Pipes**, **redirection** and **exit codes** combine small tools into powerful workflows.
- Containers carry their base image's tools: GNU on Debian/Ubuntu, BusyBox on Alpine, often nothing on distroless.

---

## 12. Check your understanding

1. What is the difference between a terminal and a shell?
2. Why is `cd` a shell builtin and not a separate program?
3. What does `cmd1 | cmd2` do? And `cmd > file 2>&1`?
4. `docker exec -it web bash` says `executable file not found`. What is the likely cause and fix?
5. What does exit code `137` usually indicate?
6. Which of these are not part of GNU coreutils: `ls`, `grep`, `sort`, `ps`, `cp`?

<details>
<summary>Answers</summary>

1. The terminal is a window that displays text and captures keys. The shell is a program that interprets your commands and starts other programs.
2. A child process cannot change its parent's working directory; only the shell itself can.
3. The first sends cmd1's stdout into cmd2's stdin. The second writes both stdout and stderr of `cmd` into `file`.
4. The image (probably Alpine or minimal) has no `bash`. Use `sh` instead, or install bash.
5. The process was killed with SIGKILL (128 + 9), often by the out-of-memory killer or `docker kill`.
6. `grep` and `ps` (`grep` has its own package; `ps` is from procps).
</details>

**Practice:** write a one-line pipeline that lists the five largest files in `/usr/bin` (hint: `ls -S`, `head`, or `du -a | sort -nr | head`).

---

**Next:** [Chapter 10 – Running Ubuntu on Docker](10_running_ubuntu_on_docker.md), where we use all of this inside a real container.
