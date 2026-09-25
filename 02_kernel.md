# Chapter 2: The Kernel

> **In one sentence:** The kernel is the core program of an operating system. It is the only software allowed to touch the hardware directly, and every other program must ask it for help.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~25 minutes

**Why this chapter is in a Docker course:** containers do not have their own kernel. They all share the kernel of the host machine, and the isolation between them is a kernel feature. If you understand the kernel, the rest of Docker stops feeling like magic.

---

## What you will learn

- The three hardware parts that matter: CPU, RAM, disk
- What an operating system (OS) is, and how it differs from the kernel
- User space vs kernel space, and user mode vs kernel mode
- What a system call is, and how to watch one happen on your own machine
- What the kernel does for you (processes, memory, devices, files, security)
- Why containers share the host kernel

---

## 1. Quick refresher: the hardware

| Part | What it does | Keeps data when powered off? |
|---|---|---|
| **CPU** | Runs instructions, one tiny step at a time. Has several **cores**, each with **registers** (very fast, tiny storage) | No |
| **RAM** (memory) | Holds the programs and data that are running *right now*. Fast, but limited (GBs) | No (volatile) |
| **Disk** (SSD/HDD) | Stores files and programs long-term. Slower, but large (TBs) | Yes |

A program is just a file on disk. To run it, the OS copies it into RAM and the CPU executes its instructions.

---

## 2. What is an operating system?

When you press the power button:

1. A tiny program stored on the motherboard (firmware, such as UEFI) starts.
2. It finds a **bootloader** on the disk and runs it.
3. The bootloader loads the **kernel** from disk into RAM and starts it.
4. The kernel starts the first process, which starts everything else (login screen, desktop, background services).

An **operating system** is the kernel **plus** the programs that make the computer usable:

```
┌───────────────────────────────────────────────┐
│ Operating system (e.g. Ubuntu)                │
│                                               │
│   Programs: shell, file tools, package        │
│   manager, desktop, libraries ...             │
│  ┌─────────────────────────────────────────┐  │
│  │ KERNEL (Linux)                          │  │
│  └─────────────────────────────────────────┘  │
└───────────────────────────────────────────────┘
                   Hardware
```

> **Terminology tip:** strictly speaking, **Linux is only the kernel**. Ubuntu, Debian, Fedora and Alpine are *distributions*: Linux kernel + a collection of other software. People often say "Linux" to mean the whole thing, which is fine in casual talk but matters in this course. Docker images contain a *distribution's files* (like Ubuntu's programs) but never a kernel.

---

## 3. What does the kernel do?

The kernel is the middle layer between programs and hardware. Its main jobs:

| Job | In plain words |
|---|---|
| **Process management** | Creates and ends processes, and decides which one gets the CPU next (*scheduling*). This is how one CPU seems to run hundreds of programs at once |
| **Memory management** | Gives each process its own private memory and reclaims it when the process ends. One program cannot read another's memory |
| **Device management** | Talks to the disk, network card, keyboard, screen, etc. through **drivers** |
| **File system management** | Turns "the file `notes.txt` in `/home/me`" into actual blocks on disk; enforces file permissions |
| **System call interface** | The official "front desk" through which programs request all of the above |
| **Security and isolation** | Checks who is allowed to do what; keeps processes apart |

(The exact list differs between textbooks. What matters is the idea: **the kernel controls the hardware and shares it fairly and safely.**)

### The "everything is a file" idea (Linux and Unix)

Linux exposes many things through file-like paths and file descriptors:

- regular files and directories
- devices: `/dev/sda` (a disk), `/dev/null` (a black hole)
- kernel information: `/proc/<pid>/` (details of a process)
- network connections (sockets), pipes

That is why the same `read` and `write` operations work on all of them. (Note: programs such as `ls` are *executable files*, but they are not special device files.)

---

## 4. User space and kernel space

Memory and CPU privileges are split into two worlds.

```
┌──────────────────────────────────────────────┐
│ USER SPACE  (limited privileges)             │
│  Chrome   VS Code   Your Python app   bash   │
└──────────────────────┬───────────────────────┘
                       │  system calls
┌──────────────────────▼───────────────────────┐
│ KERNEL SPACE  (full privileges)              │
│  scheduler · memory manager · drivers · FS   │
└──────────────────────┬───────────────────────┘
                       │
┌──────────────────────▼───────────────────────┐
│ HARDWARE:   CPU     RAM     Disk     NIC     │
└──────────────────────────────────────────────┘
```

| | User space | Kernel space |
|---|---|---|
| Who runs there | Applications (yours, the browser, `bash`, `docker` CLI...) | The kernel and its drivers |
| Hardware access | **None directly** | Full |
| If it crashes | Only that program dies | The whole machine can crash (a "kernel panic" on Linux) |

### CPU modes

The CPU itself enforces the split. It has (at least) two modes:

- **User mode**: some instructions are forbidden (for example, talking to hardware ports). Attempting them triggers an error.
- **Kernel mode**: everything is allowed.

Your program always runs in user mode. Only while the CPU is executing kernel code is it in kernel mode. (On x86 processors these are called *rings*: ring 3 for user, ring 0 for kernel.)

---

## 5. System calls: how programs ask the kernel for help

A **system call** (*syscall*) is a request from a user-space program to the kernel. It is the *only* door into kernel space.

Analogy: the kernel is a bank vault; you cannot walk in. You hand a request slip to the teller (system call). The teller checks your ID (permissions), fetches what you asked for, and hands it back.

### What happens when a program reads a file

```
1. Program (user mode):       f = open("data.txt")
2. The library issues the `openat` system call
3. CPU switches to kernel mode
4. Kernel: does the file exist? does this user have permission?
5a. Yes → kernel reads from disk (or its cache) and returns a file handle / data
5b. No  → kernel returns an error, e.g. "Permission denied"
6. CPU switches back to user mode
7. Program continues with the result
```

This is also the answer to "why can't a malicious program just edit my bank file?": it has no direct access to the disk. Everything goes through the kernel, and the kernel checks permissions each time.

### Common system calls

| Area | Examples |
|---|---|
| Files | `open`, `read`, `write`, `close` |
| Processes | `fork` / `clone` (create), `execve` (run a program), `wait`, `exit` |
| Memory | `mmap`, `brk` |
| Network | `socket`, `connect`, `bind`, `send`, `recv` |

> **Are syscalls just function calls?** No. A normal function call stays in user mode. A syscall crosses the boundary into kernel mode, which is slower, so programs try not to make more than needed.

---

## 6. Try it yourself (Linux)

You do not need Docker for these. Use any Linux machine, a WSL2 terminal, or a Linux VM.

**See the kernel version:**

```bash
uname -r
```

Example output: `6.8.0-45-generic`. That is the running kernel.

**Watch system calls with `strace`:**

```bash
# install if missing (Debian/Ubuntu)
sudo apt install strace

# trace the system calls made by `cat`
strace cat /etc/hostname 2> trace.txt
grep -E 'openat|read|write' trace.txt | tail -5
```

You should see lines such as `openat(AT_FDCWD, "/etc/hostname", O_RDONLY) = 3` followed by `read(3, ...)` and `write(1, ...)`. That is your `cat` command asking the kernel to open, read and print a file.

**See a permission check in action:**

```bash
cat /etc/shadow
```

Output: `cat: /etc/shadow: Permission denied`. The kernel refused the `open` system call.

**Look at kernel-provided information:**

```bash
ls /proc | head          # one directory per running process (numbers) + system info
cat /proc/cpuinfo | head # CPU details, generated live by the kernel
cat /proc/meminfo | head # memory details
```

`/proc` is not on your disk. The kernel creates it on demand.

---

## 7. Why this matters for Docker

Compare two ways of running programs side by side:

```
    Virtual machines                    Containers
┌────────┐ ┌────────┐            ┌────────┐ ┌────────┐
│  App   │ │  App   │            │  App   │ │  App   │
│ Guest  │ │ Guest  │            │ (files │ │ (files │
│ kernel │ │ kernel │            │ only)  │ │ only)  │
└────────┘ └────────┘            └────────┘ └────────┘
    Hypervisor                    ONE shared host kernel
     Hardware                          Hardware
```

- A **virtual machine** boots its own full kernel. Heavy, slow to start.
- A **container** is just a process on the host, running in user space, with the kernel *restricting what it can see*. Two Linux kernel features do the work (details in Chapter 4):
  - **namespaces**: give the process its own view of processes, network, file system, hostname
  - **cgroups**: limit how much CPU, memory and I/O it can use

Consequences you will meet again and again:

- Containers start in about a second (no OS to boot).
- A Linux container needs a **Linux kernel**. Docker Desktop on Windows/macOS therefore runs a small hidden Linux VM to provide one.
- Because the kernel is shared, a kernel-level bug or misconfiguration can affect every container. This is the main security trade-off compared to VMs.
- `docker run ubuntu` does **not** install Ubuntu's kernel. It only gives the process Ubuntu's *files* (programs and libraries). You can check this:

```bash
uname -r                                # on your host
docker run --rm ubuntu uname -r         # inside a container: the same kernel version
```

Both print the same version. Try it after Chapter 14.

---

## 8. Common mistakes and myths

| Myth | Reality |
|---|---|
| "Kernel and OS are the same thing" | The kernel is the core; the OS includes many more programs |
| "Programs can read the disk directly" | They must use system calls; the kernel decides |
| "User space is a separate chip" | It is a logical split of privileges and memory, not physical |
| "A container has its own kernel" | It shares the host's kernel |
| "Each Docker image contains Ubuntu, so it must contain the Ubuntu kernel" | It contains Ubuntu's *user-space files* only |

---

## 9. Summary

- **Kernel** = core of the OS, the only code with full hardware access.
- **User space** = where applications run, with limited privileges; **kernel space** = where the kernel runs.
- The CPU switches between **user mode** and **kernel mode** to enforce this.
- **System calls** are the controlled entry points to the kernel.
- The kernel manages processes, memory, devices, files and security.
- **Containers share the host kernel**, which is why they are light, and why they can use kernel features (namespaces, cgroups) for isolation.

---

## 10. Check your understanding

1. Is Chrome running in user space or kernel space? What about a Wi-Fi driver?
2. What does the CPU do when a program makes a system call?
3. Why can't one program simply read another program's memory?
4. Your teammate says "Docker containers each contain their own Linux kernel". Correct them in one sentence.
5. Which command shows the running kernel's version?

<details>
<summary>Answers</summary>

1. Chrome: user space. The Wi-Fi driver: kernel space (drivers usually run inside the kernel).
2. It switches from user mode to kernel mode, runs the kernel's handler, then switches back and returns the result.
3. The kernel gives each process its own virtual memory and the CPU enforces it; the only way to interact is through kernel-approved mechanisms.
4. Containers share the host machine's kernel; only virtual machines have their own.
5. `uname -r`.
</details>

**Practice:** run `strace -c ls` and look at the summary table. Which system call was used the most?

---

**Next:** [Chapter 3 – Virtual Machine](03_virtual_machine.md), the older way to isolate software, which will make the container idea clearer by contrast.
