# Chapter 8: Linux, Distributions, and Why Docker Needs Them

> **In one sentence:** "Linux" is strictly a **kernel**; a **distribution** (Ubuntu, Debian, Alpine, ...) wraps that kernel with tools, libraries and a package manager to make a full operating system. Docker containers use the host's Linux kernel plus the *user-space files of a distribution* that come from the image.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~35 minutes

**Prerequisites:** [Chapter 2 – Kernel](02_kernel.md), [Chapter 5](05_container_vs_vm_docker_engine.md).

---

## What you will learn

- What Linux is (and is not) and where it came from
- What a distribution is, and what is inside one
- The main distribution families (Debian, Red Hat, Arch, Alpine, ...) and how to tell them apart
- **glibc vs musl** and **GNU tools vs BusyBox**, the two differences that trip up Docker beginners
- Which distribution to pick for your Docker base image
- How to identify any Linux system, and how to run several distributions side by side using containers

---

## 1. Linux is a kernel

In 1991, a Finnish student, **Linus Torvalds**, began writing a free, Unix-like **kernel** as a hobby project and announced it to the world. That kernel is **Linux**. It was combined with the tools of the **GNU Project** (started by Richard Stallman in 1983: compiler, shell, `ls`, `cp`, `grep`, ...) to make a complete operating system. That is why some people say "GNU/Linux".

| Term | Meaning |
|---|---|
| **Linux** | The kernel only |
| **GNU** | A large collection of free user-space tools and libraries |
| **Distribution ("distro")** | A complete OS: Linux kernel + GNU (or other) tools + libraries + package manager + configuration, put together and maintained by a project or company |
| **Ubuntu, Debian, Fedora, Alpine...** | Distributions |

A house analogy: the kernel is the foundation, walls and plumbing. A distribution is the finished house with furniture, wiring, appliances and a maintenance service. Different builders (distributions) put different furniture on the same foundation.

You already know from Chapter 2 what the kernel does: process management, memory, devices, file systems, system calls. Linux adds container features to that list: **namespaces**, **cgroups**, seccomp filters, and more, which are exactly what Docker uses.

Check yours:

```bash
uname -r                # kernel version, e.g. 6.8.0-45-generic
uname -a                # more detail
cat /etc/os-release     # which distribution and version
```

---

## 2. What is inside a distribution?

| Part | What it is | Examples |
|---|---|---|
| **Linux kernel** | The core | `/boot/vmlinuz-*` |
| **C library** | Base library nearly every program needs | glibc, musl |
| **Core utilities & shell** | Everyday commands | GNU coreutils (`ls`, `cp`, `cat`), `bash`; or BusyBox (`ash`) |
| **Package manager** | Installs, updates and removes software | `apt`, `dnf`, `pacman`, `apk` |
| **Init system** | The first process (PID 1); starts and supervises services | systemd, OpenRC, SysVinit |
| **Config files** | System-wide settings, in `/etc` | `/etc/passwd`, `/etc/hosts` |
| **Software repositories** | Servers that hold thousands of pre-built packages | Ubuntu archive, Debian mirrors |
| **Desktop environment** (optional) | Graphical interface | GNOME, KDE Plasma, Xfce |

Servers and Docker hosts typically have **no desktop environment**, only a command line.

> **Inside containers:** an image contains the *user-space* parts of a distribution (C library, utilities, package manager, `/etc` files). It normally contains **no init system** (the app itself is PID 1) and **no kernel**.

---

## 3. Distribution families

Most distributions descend from a small number of "parent" projects. Knowing the family tells you the **package manager and commands** to use.

```
Linux kernel
├── Debian (1993) ─────────── apt, .deb
│     ├── Ubuntu (2004, Canonical)
│     │     ├── Linux Mint, Pop!_OS, elementary OS, Zorin OS
│     └── Kali Linux, Raspberry Pi OS ...
├── Red Hat family ────────── dnf/yum, .rpm
│     ├── Fedora (community, fast-moving; upstream of RHEL)
│     ├── Red Hat Enterprise Linux (RHEL, commercial)
│     └── Rocky Linux, AlmaLinux (free RHEL-compatible rebuilds), CentOS Stream
├── SUSE family ───────────── zypper, .rpm   (openSUSE, SLES)
├── Arch (2002) ───────────── pacman         (Manjaro, EndeavourOS)
├── Alpine (2005) ─────────── apk            (tiny; loved by Docker users)
└── Gentoo, Slackware, NixOS ... (independent)
```

| Family | Style | Package tool | Typical use |
|---|---|---|---|
| **Debian** | Stable, huge package archive | `apt` / `dpkg` (`.deb`) | Servers, Docker bases (`debian`, `ubuntu`) |
| **Ubuntu** (Debian-derived) | Friendly, regular releases: every 6 months, plus an **LTS** every 2 years with 5 years of support | `apt` | Desktops, cloud servers, most tutorials |
| **Red Hat family** | Enterprise support, SELinux by default | `dnf` (`yum` older) (`.rpm`) | Enterprise servers (RHEL, Rocky, Alma) |
| **Arch** | Rolling release: always the latest, minimal, do-it-yourself | `pacman` | Enthusiasts |
| **Alpine** | Tiny, security-focused | `apk` | **Containers**, embedded |

> **CentOS note:** classic CentOS Linux (a free RHEL rebuild) was discontinued: CentOS 8 ended in 2021 and CentOS 7 in 2024. Rocky Linux and AlmaLinux are its usual replacements; **CentOS Stream** now sits just *ahead of* RHEL.

### Base vs derived distributions
A **derived** distribution takes a base, then changes the defaults (desktop, branding, extra drivers, extra repositories) while inheriting the package format and most of the layout. Ubuntu is derived from Debian; Linux Mint is derived from Ubuntu. The tell-tale file:

```bash
cat /etc/os-release
# ID=ubuntu
# ID_LIKE=debian       ← "behaves like Debian": use apt and .deb
```

Because of that inheritance, commands you learn on Ubuntu carry over to Debian, Mint, Kali and more.

---

## 4. Two differences that matter for containers

### 4.1 glibc vs musl (the C library)
Almost every program calls a C library for basic functions (memory, files, DNS lookups, threads).

| | **glibc** | **musl** |
|---|---|---|
| Used by | Debian, Ubuntu, Fedora, RHEL... | **Alpine** |
| Size | Larger, feature-rich | Small, simple |
| Compatibility | The default target for most pre-built binaries | Sometimes different behavior |

A binary compiled against glibc **may not run** on Alpine (typical error: `not found` for a file that clearly exists, because the loader `/lib64/ld-linux-x86-64.so.2` is missing). Some language packages that ship pre-built native binaries (Python wheels, some Node modules) may need to be compiled from source on Alpine, which makes builds slower.

### 4.2 GNU coreutils vs BusyBox
- Debian/Ubuntu images ship full **GNU** versions of `ls`, `cp`, `grep`, `bash`.
- Alpine ships **BusyBox**, one small program that acts as ~300 commands (each name is a link to it), with a simpler shell (`ash`, not `bash`). Some options differ or are missing.

```bash
docker run --rm alpine ls -l /bin/ls          # → /bin/ls -> /bin/busybox
docker run --rm alpine sh -c 'bash --version' # error: bash not found
```

---

## 5. Which distribution should my container use?

| Base image | Rough size* | Pros | Cons |
|---|---|---|---|
| `ubuntu` | ~30–80 MB | Familiar, big package selection, glibc | Bigger |
| `debian` / `debian:*-slim` | ~30–120 MB | Stable, slim variants are compact, glibc | |
| `alpine` | ~4–8 MB | Tiny, fast to pull, small attack surface | musl and BusyBox surprises |
| `fedora`, `rockylinux`, `almalinux` | ~60–200 MB | Match RHEL-based production servers | Larger |
| **distroless** (`gcr.io/distroless/*`) | ~2–30 MB | No shell or package manager: minimal attack surface | Hard to debug; needs a multi-stage build |
| `scratch` | 0 MB | Empty image, for a static binary (Go, Rust) | You have nothing but your binary |

*Sizes change with versions and architecture; run `docker images` to see real numbers.

**Practical advice**
1. Learning or unsure? Use **Debian slim** or **Ubuntu**. Fewer surprises.
2. Want small images and your language supports it well? **Alpine**, but test.
3. Production hardening? Consider **distroless** or a hardened base, plus vulnerability scanning.
4. Prefer the **official language images** (e.g. `python:3.12-slim`, `node:22-alpine`) instead of installing the language yourself.

---

## 6. Why Docker requires Linux (a reminder)

Docker containers rely on Linux kernel features: namespaces, cgroups, capabilities, seccomp, overlay file systems, netfilter. The Windows (NT) and macOS (XNU/Darwin) kernels do not have them, so:

- On **Linux**: Docker Engine uses the host kernel directly.
- On **Windows/macOS**: Docker Desktop runs a small Linux VM and puts the Engine inside (Chapter 5). Inside your containers, `uname -r` shows that VM's kernel (often containing text like `linuxkit` or `microsoft-standard-WSL2`), not Windows or macOS.

```
Container image = user-space files of a distro   (ubuntu, alpine, debian ...)
         +
Host or VM Linux kernel                          (shared by all containers)
```

That is why you can run Ubuntu, Alpine and Debian containers at the same time on one machine, all using the same kernel.

---

## 7. Hands-on

You need Docker installed (Chapter 14 shows how). If you can't yet, read along.

### 7.1 Identify yourself

```bash
cat /etc/os-release
uname -r
ps -p 1 -o comm=                      # what is PID 1? (systemd on most desktops/servers)
which apt dnf pacman apk zypper 2>/dev/null   # which package manager exists
```

### 7.2 Three distributions, one kernel

```bash
docker run --rm ubuntu:24.04 sh -c 'cat /etc/os-release | head -2; uname -r'
docker run --rm debian:12   sh -c 'cat /etc/os-release | head -2; uname -r'
docker run --rm alpine:3.20 sh -c 'cat /etc/os-release | head -2; uname -r'
uname -r      # the host
```

The `NAME=` lines differ; the kernel version is identical on all four (on Windows/macOS it will be the Docker Desktop VM's kernel, but still identical for the three containers).

### 7.3 Different package managers

```bash
docker run --rm ubuntu:24.04 sh -c 'apt-get update -qq && apt-get install -y -qq curl >/dev/null && curl --version | head -1'
docker run --rm alpine:3.20  sh -c 'apk add --no-cache curl >/dev/null && curl --version | head -1'
```

(In a Dockerfile you would use `apt-get`, not `apt`, because `apt` warns about unstable command-line output.)

### 7.4 See glibc vs musl

```bash
docker run --rm ubuntu:24.04 ldd --version | head -1     # GNU libc
docker run --rm alpine:3.20  sh -c 'ldd 2>&1 | head -1'  # musl libc
```

### 7.5 Compare sizes

```bash
docker pull ubuntu:24.04 && docker pull debian:12-slim && docker pull alpine:3.20
docker images | grep -E 'ubuntu|debian|alpine'
```

### 7.6 GNU vs BusyBox

```bash
docker run --rm alpine:3.20 sh -c 'ls -l /bin | head -5'   # links to busybox
docker run --rm ubuntu:24.04 ls -l /bin/ls                  # a real GNU binary (or a link into /usr/bin)
```

---

## 8. Common misconceptions

| Misconception | Reality |
|---|---|
| "Linux is an operating system" | Linux is the kernel; the OS is a distribution (often called Linux for short) |
| "A Docker image contains the whole OS including the kernel" | Only user-space files. The kernel is always the host's |
| "Alpine is just a smaller Ubuntu" | Different family: `apk`, musl libc, BusyBox, its own release cycle |
| "`latest` Ubuntu image = the Ubuntu I have installed" | Only if the versions match; and it uses *your* kernel, not Ubuntu's |
| "Debian-based, so the same commands work everywhere" | True within a family (`apt`); other families differ (`dnf`, `apk`, `pacman`) |
| "Windows can't run Linux containers" | It can, via a Linux VM (WSL 2 or Hyper-V) |
| "Smaller image is always better" | Smaller can mean less compatibility and fewer debugging tools. Balance it |

---

## 9. Summary

- **Linux = kernel.** A **distribution** adds tools, libraries, a package manager and defaults.
- Families: **Debian** (apt), **Red Hat** (dnf), **Arch** (pacman), **Alpine** (apk), and **SUSE** (zypper). Derived distros inherit their parent's tooling.
- In containers, the important differences are **glibc vs musl** and **GNU coreutils vs BusyBox**.
- A container image = a distribution's user-space files; the **kernel comes from the host** (or Docker Desktop's VM).
- Pick a base image for compatibility first (Debian/Ubuntu), then optimize size (slim, Alpine, distroless).

---

## 10. Check your understanding

1. What is the difference between Linux and Ubuntu?
2. `/etc/os-release` shows `ID_LIKE=debian`. Which package manager will you use?
3. Why might a binary that runs on Ubuntu fail on Alpine?
4. You run `docker run alpine uname -r` and `docker run ubuntu uname -r`. Will the results differ? Why or why not?
5. When would you choose a distroless image?

<details>
<summary>Answers</summary>

1. Linux is the kernel. Ubuntu is a distribution that includes the Linux kernel plus many other components.
2. `apt` (and `dpkg` for `.deb` files).
3. It was likely built against glibc, while Alpine uses musl (and may lack expected libraries or tools).
4. They are identical, because both containers use the host's kernel.
5. For hardened production images that need a minimal attack surface and no shell, typically with a compiled or self-contained app built via a multi-stage build.
</details>

**Practice:** find out which distribution and version the `python:3.12` and `python:3.12-alpine` images are based on, using `docker run --rm <image> cat /etc/os-release`.

---

**Next:** [Chapter 9 – GNU Coreutils](09_gnu_coreutils.md): the everyday commands you will use inside every container.
