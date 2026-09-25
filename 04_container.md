# Chapter 4: Containers (Namespaces + cgroups)

> **In one sentence:** A container is an ordinary Linux process (or group of processes) that the kernel has put in a "bubble": **namespaces** control what it can *see*, and **cgroups** control how much it can *use*.

**Level:** 🟡 Intermediate (explained from zero) · **Reading time:** ~35 minutes

**Prerequisites:** [Chapter 2 – Kernel](02_kernel.md) and [Chapter 3 – Virtual Machines](03_virtual_machine.md).

---

## What you will learn

- The classic problem containers solve (two apps that need conflicting versions of the same software)
- **Namespaces**: the "what can I see?" part of a container
- **cgroups**: the "how much can I use?" part
- How these combine into a container, and why a container is *not* a small VM
- What an **image** really is (and what it is not)
- How to build a tiny container by hand on Linux, with no Docker at all
- What Docker added on top of these kernel features

---

## 1. The problem: version conflicts

Imagine one server that must run two apps:

- **App A** needs Python **3.8** and library `libfoo 1.x`
- **App B** needs Python **3.12** and library `libfoo 2.x`

On a normal Linux system there is **one** file system with **one** `/usr/lib` and typically one system-wide install of each package. Installing the second version replaces or conflicts with the first, and one app breaks.

Ways people dealt with it:

1. **Separate machines**: one server per app. Expensive.
2. **Virtual machines**: one VM per app. Works, but each needs a full OS (Chapter 3).
3. **Language-specific tools** (virtualenv, nvm): only help for one language.

What we really want: *give each app its own private view of the system*, without booting a whole extra OS. Linux can do exactly that.

---

## 2. Namespaces: controlling what a process can see

A **namespace** wraps a global system resource so that processes inside the namespace see their own private copy of it.

> **Analogy: the frog in the well.** A frog at the bottom of a well believes the round patch of sky is the whole sky. Processes in a namespace likewise believe their small view is the entire system. The well (the namespace) doesn't exist as a physical object; it only limits what the frog can perceive.

By default every process on the machine lives in the same set of "initial" namespaces. Linux lets you create new ones. The main types:

| Namespace | Isolates | Effect inside the container |
|---|---|---|
| **PID** | Process IDs | The first process is PID 1; it cannot see the host's other processes |
| **NET** (network) | Network interfaces, IP addresses, routing tables, ports | Its own `eth0`, own `localhost`, own ports |
| **MNT** (mount) | Mount points / file system tree | Its own root directory `/` with its own files |
| **UTS** | Hostname and domain name | Its own hostname |
| **IPC** | Shared memory and message queues | Cannot interfere with other processes' IPC |
| **USER** | User and group IDs | "root" inside can map to an unprivileged user outside |
| **CGROUP** | View of the cgroup tree | Sees only its own cgroup as the root |
| **TIME** (newer kernels) | System clocks offsets | Rarely used |

### How the version conflict is solved

The mount namespace is the key one. Give App A a root file system that contains Python 3.8, and App B a root file system that contains Python 3.12:

```
Host disk
├── /var/lib/appA-root/usr/bin/python  (3.8)      ← App A's "/"
└── /var/lib/appB-root/usr/bin/python  (3.12)     ← App B's "/"
```

When App A opens `/usr/bin/python`, the kernel resolves that path inside **App A's** mount namespace, so it finds 3.8. App B, asking for the same path, gets 3.12. Both live on the same physical disk; they just never see each other's files.

---

## 3. cgroups: controlling how much a process can use

Namespaces do not stop a process from eating all the memory. That is the job of **control groups (cgroups)**.

A cgroup is a group of processes with **resource rules** attached:

| Resource | Example limit |
|---|---|
| Memory | "at most 512 MB" |
| CPU | "at most 1.5 cores worth of time" or a relative weight |
| I/O | "at most 10 MB/s to this disk" |
| Number of processes | "at most 100 processes" (prevents fork bombs) |
| Devices | "may not access `/dev/sda`" |

> **Precision note:** cgroups limit *I/O speed*, not "disk size". Limiting how much disk space a container can fill is done with file-system features (quotas, storage-driver options), not cgroups.

What happens when a limit is hit?

- **CPU:** the group is *throttled* (slowed down). Nothing crashes.
- **Memory:** the kernel first tries to reclaim memory; if it cannot, the **OOM killer** kills a process in that cgroup. In Docker you will see the container exit with code **137**.

### The formula

```
   Namespaces  (what you see)
 +  cgroups    (how much you can use)
 ───────────────────────────────────
 =  a container
```

There is no `container` object in the Linux kernel. "Container" is a *name for the combination*.

---

## 4. Containers are not small VMs

```
      Virtual machines                        Containers
┌──────────┐ ┌──────────┐             ┌───────────┐ ┌───────────┐
│ App A    │ │ App B    │             │ App A     │ │ App B     │
│ libs     │ │ libs     │             │ + its libs│ │ + its libs│
│ Guest OS │ │ Guest OS │             │ (files)   │ │ (files)   │
│ + kernel │ │ + kernel │             └─────┬─────┘ └─────┬─────┘
└────┬─────┘ └────┬─────┘                   └──────┬──────┘
   Hypervisor                        Host kernel (namespaces + cgroups)
   Hardware                                     Hardware
```

| | Virtual machine | Container |
|---|---|---|
| Kernel | One **per VM** | **One shared** host kernel |
| Boots an OS? | Yes | No, just starts a process |
| Typical size | GBs | Often tens to hundreds of MB |
| Start-up | Tens of seconds to minutes | Usually well under a second to a few seconds |
| Isolation strength | Very strong (hardware-level, separate kernels) | Good but weaker (shared kernel) |
| Runs a different OS kernel? | Yes (Windows VM on Linux host) | No, a Linux container needs a Linux kernel |

Because a container is just a process, from the host you can see it:

```bash
docker run -d --name demo nginx
ps aux | grep nginx        # nginx processes appear in the HOST's process list
docker rm -f demo
```

---

## 5. Container images: a recipe, not a photo of a running program

The original version of this idea is often described as "take a snapshot of a running container". That is a helpful first picture, but it is **not accurate** and can mislead you later. Here is the precise version.

An **image** is a **read-only bundle of files** (a root file system, made of stacked layers) plus some **metadata** (which command to run, environment variables, exposed ports, and so on).

An image does **not** contain:

- running processes
- the memory contents (RAM) of a program
- a kernel

When you `docker run` an image, Docker:

1. Creates namespaces and a cgroup.
2. Presents the image's files as the container's root file system (`/`), adding a thin *writable layer* on top.
3. Starts the image's command as PID 1 inside those namespaces.

```
Image (read-only layers)           Container
┌──────────────────┐        ┌─────────────────────────┐
│ your app files   │        │ writable layer (new     │
├──────────────────┤   +    │  or changed files)      │
│ python 3.12      │        ├─────────────────────────┤
├──────────────────┤        │ image layers (shared,   │
│ Debian base files│        │  read-only)             │
└──────────────────┘        └─────────────────────────┘
```

Consequences:

- **One image → many containers.** They share the read-only layers, so 100 containers do not need 100 copies of the disk data.
- Files a container creates live in *its* writable layer. The image and other containers are untouched.
- When the container is deleted, its writable layer is deleted, unless you saved data in a *volume* (Chapter 14 onwards).
- You *can* create a new image from a container's changed file system (`docker commit`), but you still capture **files**, not running state. (Freezing running processes is a different, rarer technique called checkpoint/restore.)

---

## 6. Hands-on: build a container by hand (Linux only)

Everything below uses only the Linux kernel and standard tools. Use a Linux VM or a machine where you are comfortable with `sudo`. Nothing here modifies your system permanently.

### 6.1 See the namespaces of a normal process

```bash
ls -l /proc/self/ns
```

You will see entries like `pid -> 'pid:[4026531836]'`, `net`, `mnt`, `uts`, ... The number identifies a namespace. Two processes with the same number share it.

### 6.2 Create a new UTS and PID namespace

```bash
sudo unshare --uts --pid --fork --mount-proc bash
```

Now inside that shell:

```bash
hostname                # same as host for now
hostname my-container   # change it: only affects this namespace
hostname                # → my-container
ps aux                  # only bash and ps are visible! You are PID 1's child
echo $$                 # a very small number (1 or 2)
exit
```

On the host, run `hostname` again: it is unchanged. That is namespace isolation, done in three commands.

### 6.3 Give it its own network

```bash
sudo unshare --net bash
ip addr        # only "lo" (loopback), and it is DOWN: a private, empty network
exit
```

### 6.4 Give it its own file system (a mini root)

```bash
# get a tiny Alpine Linux root file system (about 3 MB)
mkdir -p /tmp/myroot && cd /tmp/myroot
curl -L https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz | tar xz

# enter it with new namespaces
sudo unshare --mount --uts --pid --fork chroot /tmp/myroot /bin/sh -c "mount -t proc proc /proc && /bin/sh"
```

Inside, run `cat /etc/os-release`. It says **Alpine Linux**, although your host is something else, and `ls /` shows only Alpine's files. You have just made a (very crude) container. Also run `uname -r`: it prints the **host's** kernel version, proving the kernel is shared.

(If the URL or version has changed, browse https://dl-cdn.alpinelinux.org/alpine/ for a current "minirootfs" tarball. `chroot` is used here for simplicity; real runtimes use `pivot_root`, which is more robust.)

### 6.5 Add a memory limit with a cgroup (cgroups v2)

```bash
cat /sys/fs/cgroup/cgroup.controllers            # available controllers (cpu memory io pids ...)
sudo mkdir /sys/fs/cgroup/demo
echo 100M | sudo tee /sys/fs/cgroup/demo/memory.max
echo $$    | sudo tee /sys/fs/cgroup/demo/cgroup.procs   # move THIS shell into the cgroup
```

Any program you start from this shell can now use at most 100 MB before the OOM killer acts. Clean up: open a new terminal, then `sudo rmdir /sys/fs/cgroup/demo`. (On older systems with cgroups v1 the paths differ, and on some systems you need to enable controllers first. This exercise is meant as a peek.)

### 6.6 The same thing in Docker (for comparison)

```bash
docker run --rm -it --memory=100m --cpus=1 --hostname my-container alpine sh
```

Inside: `hostname`, `ps`, `cat /etc/os-release`. Docker did all of the above (namespaces, root file system, cgroup limits) with one command, which is exactly why Docker became popular.

---

## 7. What Docker added

Docker did **not** invent containers. Namespaces (2002 onward) and cgroups (2007) are kernel features, and LXC and others already used them. Docker's contribution was **usability and a standard workflow**:

| Piece | What it gives you |
|---|---|
| **Image format with layers** | Portable, cache-friendly, share only the differences |
| **Dockerfile** | Repeatable recipe to build an image |
| **Registry (Docker Hub)** | `docker pull` / `docker push` to share images |
| **Simple CLI and API** | One command instead of the manual steps in section 6 |
| **Networking, volumes, Compose** | Batteries included for real applications |

---

## 8. Security note (intermediate → expert)

Because the kernel is shared:

- A kernel vulnerability can allow a **container escape** (rare but real).
- **root inside a container is dangerous** unless user namespaces map it to an unprivileged user outside. Prefer running as a non-root user (Chapter 13 and 15).
- Docker applies extra hardening by default: dropped Linux **capabilities**, a **seccomp** syscall filter, and AppArmor/SELinux profiles.
- For stronger isolation, sandboxed runtimes such as gVisor or Kata Containers put a thin VM or user-space kernel between container and host.

---

## 9. Common misconceptions

| Misconception | Reality |
|---|---|
| "Containers = Docker" | Containers are a Linux kernel feature; Docker is one tool that makes them easy. Others: Podman, containerd, CRI-O, LXC |
| "A container is a lightweight VM" | No guest OS or kernel: it is an isolated process on the host kernel |
| "Containers each get their own disk space" | They get a layered file system view; only their *writable layer* is separate. Disk-size limits are not a cgroup feature |
| "An image is a snapshot of a running program" | An image is a set of files plus metadata. No memory or processes are captured |
| "Containers are insecure" / "containers are perfectly secure" | Both extremes are wrong. They are isolated, not sandboxed like VMs; harden them |
| "You can only run Linux apps in containers" | Docker on Linux runs Linux containers. Windows also has native Windows containers, and Docker Desktop on Mac/Windows runs Linux containers inside a hidden VM |

---

## 10. Summary

- A **container** = a process (group) + **namespaces** (what it sees) + **cgroups** (how much it can use) + an **image-provided root file system**.
- All containers on a host **share one kernel**. That is why they are small and fast, and also why isolation is weaker than a VM.
- An **image** is a read-only, layered set of files plus metadata; a container adds a writable layer and a running process.
- Docker made these kernel features convenient; it did not invent them.

---

## 11. Check your understanding

1. Which mechanism decides what files a container can see, and which limits its memory?
2. What signal that a container was OOM-killed might you see in Docker?
3. You start 3 containers from the same image. How many kernels are running? How many copies of the image's read-only layers are on disk?
4. Is "an image is a snapshot of a running container, including memory" correct?
5. Why does `uname -r` inside a container show the same value as on the host?

<details>
<summary>Answers</summary>

1. Mount namespace (files) and cgroup memory controller (memory).
2. Exit code 137 (SIGKILL); `docker inspect` shows `OOMKilled: true`.
3. One kernel (the host's); one copy of the layers, shared by all three.
4. No. An image holds files and metadata only, not running processes or RAM.
5. Because containers share the host's kernel.
</details>

**Practice:** run the commands in 6.2 and 6.3. Then, in 6.6, try `--memory=10m` with a program that allocates memory (`python3 -c "x=' '*50_000_000"` in a `python` image) and watch it die.

---

**Next:** [Chapter 5 – Container vs VM and the Docker Engine](05_container_vs_vm_docker_engine.md)
