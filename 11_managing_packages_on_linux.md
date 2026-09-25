# Chapter 11: Managing Packages on Linux

> **In one sentence:** On Linux you install software with a **package manager** (like `apt`) that downloads verified **packages** from **repositories**, installs their **dependencies** automatically, and can update or remove everything cleanly.

**Level:** 🟢 Beginner → 🟡 Intermediate (with a Docker-specific expert section) · **Reading time:** ~40 minutes

**Prerequisites:** [Chapter 10 – Running Ubuntu on Docker](10_running_ubuntu_on_docker.md). You will type all commands inside an Ubuntu container.

---

## What you will learn

- What a **package**, a **repository** and a **package manager** are
- Why there are two layers (`apt` on top of `dpkg`)
- The everyday `apt` commands: update, install, remove, purge, search, show, upgrade, autoremove
- How `apt` knows where to download from, and how it verifies downloads
- The equivalent tools on other distributions (`dnf`, `apk`, `pacman`, `zypper`)
- **How to install packages properly in a Dockerfile** (a very common source of bugs and huge images)
- Troubleshooting the most common errors

---

## 1. The problem: how do you install software safely?

Traditionally on some systems you would search the web, download an installer from some site, run it, and hope it is not malware, that it doesn't clash with other software, and that you'll remember to update it.

Linux distributions solve this with **package management**:

```
   ┌────────────────────────┐   1. "apt install git"   ┌────────────────┐
   │ Repository (server)    │ ◄──────────────────────── │ Package manager│
   │ thousands of signed    │                           │ (apt)          │
   │ packages + an index    │ ──── 2. download ───────► │                │
   └────────────────────────┘                           └───────┬────────┘
                                                        3. verify signature,
                                                           resolve dependencies
                                                                │
                                                        ┌───────▼────────┐
                                                        │ Your system    │
                                                        │ /usr/bin/git … │
                                                        └────────────────┘
```

| Piece | Analogy |
|---|---|
| **Repository** | A warehouse of trusted, signed software |
| **Package manager** | The delivery service that fetches, installs and tracks everything |
| **Your system** | Your home, with a record of everything delivered |

---

## 2. What is a package?

A **package** is an archive with the program's files plus **metadata** describing how to install it.

| Contents | Example (`git`) |
|---|---|
| Program files, placed in standard locations | `/usr/bin/git`, `/usr/lib/git-core/...` |
| Documentation and man pages | `/usr/share/doc/git/` |
| Default config files | `/etc/...` |
| **Metadata**: name, version, architecture, description, maintainer, license | `git 1:2.43.0-1ubuntu7`, `amd64` |
| **Dependencies**: other packages this one needs | `libc6`, `libcurl3-gnutls`, `perl`, `zlib1g`, ... |
| **Maintainer scripts** (optional) | `preinst`, `postinst`, `prerm`, `postrm`: small scripts run before/after installing or removing (create a user, start a service, clean up) |

Package file names follow a pattern:

```
git_1%3a2.43.0-1ubuntu7_amd64.deb
 │            │             │
 name      version     architecture (amd64 = 64-bit x86, arm64 = ARM)
```

### Package formats

| Format | Used by | Low-level tool | High-level tool(s) |
|---|---|---|---|
| **`.deb`** | Debian, Ubuntu, Mint, Kali... | `dpkg` | `apt`, `apt-get` |
| **`.rpm`** | Fedora, RHEL, Rocky, Alma, openSUSE | `rpm` | `dnf` (older `yum`), `zypper` on SUSE |
| **`.apk`** | Alpine | `apk` | `apk` (it does both jobs) |
| **`.pkg.tar.zst`** | Arch | `pacman` | `pacman` |

A `.deb` file is an `ar` archive containing a `control` archive (metadata and scripts) and a `data` archive (the actual files). You can peek inside one: `dpkg-deb -I file.deb` (info) and `dpkg-deb -c file.deb` (contents).

---

## 3. Two layers: `dpkg` and `apt`

| Layer | Tool | Job |
|---|---|---|
| **Low level** | `dpkg` | Installs or removes **one local `.deb` file**. Knows what is installed. Does **not** download and does **not** fetch dependencies |
| **High level** | `apt` | Reads repositories, **finds** packages, **downloads** them, works out **dependencies**, verifies signatures, then calls `dpkg` to do the installing |

Example: try installing a `.deb` with `dpkg` when its dependencies are missing, and it stops with an error about unmet dependencies. With `apt install git`, apt sees that git needs a dozen other packages, downloads them all, and installs everything in the right order.

`dpkg` is still useful for inspection:

```bash
dpkg -l | head            # list installed packages
dpkg -L git               # files a package installed
dpkg -S /usr/bin/git      # which package owns this file?
dpkg -s git               # status of one package
```

### `apt` vs `apt-get`
- **`apt-get`**, **`apt-cache`**: the older tools, with a stable, script-friendly output.
- **`apt`**: newer (2014), friendlier (progress bars, colors, combines common commands).
- **Rule of thumb:** use **`apt` when typing interactively**, use **`apt-get` in scripts and Dockerfiles** (`apt` itself warns that its CLI is not stable for scripts).

---

## 4. Repositories: where packages come from

A **repository** ("repo") is a server holding packages plus an **index** listing them with versions, checksums and dependencies. Ubuntu's main archive is `archive.ubuntu.com` (and `security.ubuntu.com`); Debian's is `deb.debian.org`.

### Ubuntu's "components"

| Component | Contents | Support |
|---|---|---|
| **main** | Free and open-source software supported by Canonical | Yes |
| **restricted** | Proprietary drivers and firmware | Yes (limited) |
| **universe** | Free and open-source, maintained by the community | Community |
| **multiverse** | Software with licensing restrictions or non-free | No |

### "Suites" (per release)
Each Ubuntu release has a codename, e.g. `noble` (24.04), `jammy` (22.04). Each has pockets:

- `noble` – the software as released
- `noble-security` – security fixes
- `noble-updates` – other fixes

### Where apt is configured
On Ubuntu 24.04 (and its Docker image):

```bash
cat /etc/apt/sources.list.d/ubuntu.sources
```

```
Types: deb
URIs: http://archive.ubuntu.com/ubuntu/
Suites: noble noble-updates noble-backports
Components: main universe restricted multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

Older releases and Debian may use `/etc/apt/sources.list` with lines like `deb http://archive.ubuntu.com/ubuntu jammy main universe`. Extra repositories live in `/etc/apt/sources.list.d/`.

### Trust: signatures
Repository indexes are **cryptographically signed** and apt checks them against keys in `Signed-By` keyrings. Every package's checksum is in the signed index, so a tampered download is rejected. When you add a third-party repository, you also add its key. Only do that for sources you trust.

**PPAs** (Personal Package Archives) are third-party Ubuntu repositories hosted on Launchpad; they can offer newer versions, but you are trusting an individual.

---

## 5. The everyday `apt` commands

Run these inside `docker run -it --rm ubuntu:24.04 bash` (you are root, so no `sudo`). On a normal machine, prefix with `sudo`.

### 5.1 `apt update`: refresh the index (not the software!)

```bash
apt update
```

Downloads the latest **package lists** from the repositories. **It installs nothing.** A fresh Ubuntu container has *empty* lists, which is why this error appears first:

```
E: Unable to locate package curl
```

Sample output:

```
Get:1 http://archive.ubuntu.com/ubuntu noble InRelease [256 kB]
Get:2 http://archive.ubuntu.com/ubuntu noble-updates InRelease [126 kB]
...
Fetched 25.5 MB in 3s
Reading package lists... Done
```

`Hit:` = index unchanged; `Get:` = downloading; `Ign:` = ignored.

### 5.2 `apt install`: install packages

```bash
apt install curl
```

Apt shows what it will do and asks `Do you want to continue? [Y/n]`. Add `-y` to say yes automatically (needed in scripts and Dockerfiles).

```
The following additional packages will be installed:
  libcurl4t64 ...
The following NEW packages will be installed:
  curl libcurl4t64 ...
0 upgraded, 4 newly installed, 0 to remove and 0 not upgraded.
Need to get 1.1 MB of archives.
After this operation, 4.7 MB of additional disk space will be used.
```

Handy variants:

```bash
apt install -y curl git nano          # several at once, no prompt
apt install --no-install-recommends -y curl   # skip "recommended" extras (smaller)
apt install nginx=1.24.0-2ubuntu7     # a specific version
```

Dependency types: **Depends** (mandatory), **Recommends** (installed by default, skip with `--no-install-recommends`), **Suggests** (only listed).

### 5.3 Finding packages

```bash
apt search "web server"        # search names and descriptions (noisy)
apt show nginx                 # details: version, dependencies, size, description
apt list --installed | head    # what's installed
apt list --upgradable          # what could be upgraded
apt-cache policy nginx         # installed vs candidate version, and where from
apt-file search bin/ping       # which package provides a file (install apt-file first)
```

### 5.4 Removing

```bash
apt remove curl        # remove the program, KEEP config files
apt purge curl         # remove the program AND its system config files
apt autoremove         # remove dependencies nobody needs anymore
apt autoremove --purge # ...and their configs
```

`apt remove` deliberately keeps configuration in `/etc` in case you reinstall. `purge` deletes it. (It does not delete your personal data in home directories.)

### 5.5 Upgrading

```bash
apt update && apt upgrade       # upgrade installed packages; never removes packages
apt full-upgrade                # may also remove or install packages to resolve changes (was dist-upgrade)
apt list --upgradable
```

Always **update first, then upgrade**, since upgrade uses the lists that update downloads. In containers you normally do **not** upgrade in place; you rebuild the image from a newer base instead (see section 8).

### 5.6 Housekeeping

```bash
apt clean            # delete downloaded .deb files in /var/cache/apt/archives
apt autoclean        # delete only obsolete ones
apt-mark hold nginx  # prevent upgrades of this package
apt-mark unhold nginx
apt-mark showhold
```

---

## 6. Where does apt keep things?

| Path | What |
|---|---|
| `/etc/apt/` | Configuration and sources |
| `/var/lib/apt/lists/` | Downloaded package indexes (from `apt update`) |
| `/var/cache/apt/archives/` | Downloaded `.deb` files |
| `/var/lib/dpkg/status` | dpkg's database of installed packages |
| `/var/lib/dpkg/info/` | Per-package file lists and maintainer scripts |
| `/usr/share/keyrings/` | Repository signing keys |

---

## 7. Other package managers (side by side)

| Task | Debian/Ubuntu (`apt`) | Fedora/RHEL/Rocky (`dnf`) | Alpine (`apk`) | Arch (`pacman`) |
|---|---|---|---|---|
| Refresh index | `apt-get update` | (automatic; `dnf makecache`) | `apk update` | `pacman -Sy` |
| Install | `apt-get install -y curl` | `dnf install -y curl` | `apk add --no-cache curl` | `pacman -S curl` |
| Remove | `apt-get remove curl` | `dnf remove curl` | `apk del curl` | `pacman -R curl` |
| Upgrade all | `apt-get upgrade` | `dnf upgrade` | `apk upgrade` | `pacman -Syu` |
| Search | `apt-cache search x` | `dnf search x` | `apk search x` | `pacman -Ss x` |
| Info | `apt-cache show x` | `dnf info x` | `apk info x` | `pacman -Si x` |
| Files of a package | `dpkg -L x` | `rpm -ql x` | `apk info -L x` | `pacman -Ql x` |
| Low-level tool | `dpkg` | `rpm` | (`apk`) | (`pacman`) |

On other operating systems the same idea exists: **Homebrew** on macOS, **winget**/**Chocolatey** on Windows, and language-level managers such as `pip`, `npm` or `cargo`. Do not confuse them with OS packages: `pip install` installs *Python libraries*, `apt install python3-requests` installs a *Debian-packaged* one.

---

## 8. Packages in Docker: doing it right (intermediate → expert)

Almost every Dockerfile installs packages. Because each instruction creates an image **layer** (Chapter 7), *how* you write it matters for image size, correctness and reproducibility.

### 8.1 The standard Debian/Ubuntu pattern

```dockerfile
FROM ubuntu:24.04

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      curl \
      ca-certificates \
 && rm -rf /var/lib/apt/lists/*
```

Why each part:

| Part | Reason |
|---|---|
| `apt-get`, not `apt` | Stable interface for scripts |
| `update && install` in the **same `RUN`** | If they are separate layers, Docker may cache the `update` layer and later install from a *stale* index, producing 404 errors |
| `-y` | No interactive prompt (there is nobody to press Y) |
| `--no-install-recommends` | Skips optional extras: much smaller image |
| `rm -rf /var/lib/apt/lists/*` (in the **same** layer) | Deletes the downloaded indexes. Deleting them in a *later* layer would not shrink the image, because earlier layers are immutable |
| One package per line, alphabetical | Cleaner diffs, easy to review |
| `ca-certificates` | Needed for HTTPS downloads. Slim images often lack it |

> Ubuntu and Debian *base images* already include a small config that deletes downloaded `.deb` files after installation. But the package **lists** stay unless you remove them.

### 8.2 Avoiding prompts
Some packages ask questions (for example `tzdata` asks for a time zone), and the build hangs. Prevent it:

```dockerfile
ENV DEBIAN_FRONTEND=noninteractive
```

Better yet, set it only for the command: `RUN DEBIAN_FRONTEND=noninteractive apt-get install -y tzdata`. (Avoid setting it globally in the final image with `ENV`, because it also affects later interactive use.)

### 8.3 Pin versions for reproducibility (production)

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
      curl=8.5.0-2ubuntu10.6 \
 && rm -rf /var/lib/apt/lists/*
```

Look up exact versions with `apt-cache policy curl`. Pinning stops silent changes, but you must update it deliberately for security fixes. Also pin the base image (`ubuntu:24.04`, or by digest), not `latest`.

### 8.4 Don't do these

| Bad practice | Why |
|---|---|
| `apt-get upgrade` in a Dockerfile | Makes builds non-reproducible. Start from an updated base image instead |
| `RUN apt-get update` alone in its own layer | Stale cache problem, plus wasted layer |
| Installing editors, `ssh`, `sudo`, debugging tools in production images | Larger attack surface, larger image |
| Adding third-party repos without verifying keys | Supply-chain risk |
| Using `apt` (not `apt-get`) | Warning: "apt does not have a stable CLI interface" |

### 8.5 Other distributions in Dockerfiles

```dockerfile
# Alpine
RUN apk add --no-cache curl ca-certificates

# Fedora / Rocky / Alma
RUN dnf install -y curl && dnf clean all

# Debian slim is the same pattern as Ubuntu:
FROM debian:12-slim
```

`apk add --no-cache` avoids storing the index, so no cleanup step is needed.

### 8.6 Multi-stage builds: keep build tools out of the final image
Compilers and `-dev` packages are needed only to *build*. Install them in a builder stage and copy just the result into a slim final stage (Chapter 20). This is the most effective way to shrink images and reduce attack surface.

---

## 9. Hands-on lab

Run inside `docker run -it --rm ubuntu:24.04 bash` (it is disposable).

**Lab 1: Empty cache, then fill it**

```bash
apt install -y tree          # E: Unable to locate package tree  (no lists yet)
ls /var/lib/apt/lists | head # almost empty
apt update
ls /var/lib/apt/lists | head # now filled
apt install -y tree
tree --version
tree -L 1 /etc | head
```

**Lab 2: Dependencies**

```bash
apt show git | grep -E '^(Depends|Recommends)'
apt install -s git | head -20        # -s = simulate; shows what WOULD be installed
```

**Lab 3: Inspect an installed package**

```bash
dpkg -L tree                 # files it installed
dpkg -S "$(which tree)"      # which package owns /usr/bin/tree
apt-cache policy tree        # version and origin
```

**Lab 4: remove vs purge**

```bash
apt install -y nginx
ls /etc/nginx | head -3
apt remove -y nginx
ls /etc/nginx | head -3          # config still there
apt purge -y nginx
ls /etc/nginx                    # No such file or directory
apt autoremove -y
```

**Lab 5: Measure the effect of `--no-install-recommends`**

```bash
apt install -s git | grep -E 'newly installed'
apt install -s --no-install-recommends git | grep -E 'newly installed'
```

You will see fewer packages with the second form.

**Lab 6: Build a small image (needs a text editor on the host)**

`Dockerfile`:

```dockerfile
FROM ubuntu:24.04
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl ca-certificates \
 && rm -rf /var/lib/apt/lists/*
CMD ["curl", "--version"]
```

```bash
docker build -t curl-demo .
docker run --rm curl-demo
docker images curl-demo
```

Now change the `RUN` line by removing the `rm -rf /var/lib/apt/lists/*` part, rebuild under another tag, and compare sizes with `docker images`.

---

## 10. Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `E: Unable to locate package X` | Index empty or stale; misspelled name; package in another component/release | `apt update`; check with `apt search X`; make sure `universe` is enabled if needed |
| `E: Could not get lock /var/lib/dpkg/lock-frontend` | Another apt/dpkg is running (or crashed) | Wait for it. Check `ps aux \| grep -E 'apt\|dpkg'`. Only if you are sure it is dead, kill it and run `dpkg --configure -a`. Avoid deleting lock files by hand |
| `E: Failed to fetch ... 404 Not Found` | Stale index (very common in Docker caches), or the release is end-of-life and moved to `old-releases` | `apt-get update` in the same `RUN`; use a supported release |
| `dpkg: dependency problems prevent configuration of X` | Broken or partial install | `apt --fix-broken install` |
| `E: Unmet dependencies. Try 'apt --fix-broken install'` | Conflicting versions | Same; or remove the conflicting package |
| `debconf: unable to initialize frontend: Dialog` / build hangs at a question | Interactive package in a non-interactive shell | `DEBIAN_FRONTEND=noninteractive` |
| `Temporary failure resolving 'archive.ubuntu.com'` | No network or DNS (proxy, VPN, firewall) | Check connectivity; configure a proxy or DNS for Docker |
| `W: GPG error: ... NO_PUBKEY` | Missing signing key for a third-party repo | Add the vendor's key as documented (to a `Signed-By` keyring) |
| `sudo: command not found` in a container | You are already root and sudo isn't installed | Just drop `sudo` |
| `command not found` after `apt install` | Package installs the tool under a different name | `apt-file search bin/<tool>` or `dpkg -L <package>` |

---

## 11. Common misconceptions

| Misconception | Reality |
|---|---|
| "`apt update` updates my software" | It only refreshes the index. `apt upgrade` updates software |
| "`apt` and `dpkg` are alternatives" | `apt` calls `dpkg`. dpkg alone doesn't resolve dependencies |
| "Removing a package removes everything" | `remove` keeps config; deps stay until `autoremove` |
| "`git` depends on vim, nano and curl" | It does not. Always check with `apt show` |
| "I can `rm` the apt lists in a later layer to make the image smaller" | Earlier layers are immutable; the data still ships. Clean in the same `RUN` |
| "`pip install` and `apt install` are the same thing" | Different ecosystems and tools |
| "More installed packages = better container" | Each package adds size and attack surface |

---

## 12. Summary

- A **package** = files + metadata + dependencies (+ scripts). **Repositories** serve signed packages. A **package manager** automates it all.
- **`dpkg`** handles single `.deb` files; **`apt`** adds repositories, downloading and dependency resolution.
- The core commands: `apt update`, `apt install`, `apt remove` / `purge`, `apt upgrade`, `apt search`, `apt show`, `apt autoremove`.
- Fresh Docker images have **empty package lists**: run `apt-get update` first.
- In Dockerfiles: `update` + `install -y --no-install-recommends` + cleanup, all in **one `RUN`**; pin versions when reproducibility matters; prefer multi-stage builds.
- Other families: `dnf` (Red Hat), `apk` (Alpine), `pacman` (Arch), `zypper` (SUSE).

---

## 13. Check your understanding

1. What is the difference between `apt update` and `apt upgrade`?
2. Why does `apt install curl` fail in a brand-new `ubuntu:24.04` container?
3. Why should `apt-get update` and `apt-get install` be in the same `RUN` instruction?
4. What does `apt purge` do that `apt remove` does not?
5. Why doesn't deleting `/var/lib/apt/lists` in a separate `RUN` reduce the image size?
6. Which command shows which package installed `/usr/bin/curl`?

<details>
<summary>Answers</summary>

1. `update` refreshes the list of available packages; `upgrade` installs newer versions of installed packages.
2. The image ships with empty package lists; `apt update` must fetch them first.
3. Docker caches layers; a cached `update` layer can go stale, so a later `install` may look for package versions that no longer exist.
4. It also deletes the package's system configuration files.
5. Layers are additive and immutable: the files still exist in the earlier layer. Cleanup must happen in the same layer that created them.
6. `dpkg -S /usr/bin/curl`
</details>

**Practice:** write a Dockerfile that installs `git` and `curl` on `debian:12-slim` with a minimal layer, build it, and compare the image size with an `ubuntu:24.04` version. Then write the same for `alpine:3.20` using `apk`.

---

**Next:** [Chapter 12 – Linux Basic Commands](12_linux_basic_commands.md)
