# Chapter 5: Containers vs Virtual Machines, and the Docker Engine

> **In one sentence:** Containers and VMs both isolate software, but a VM carries its own OS and kernel while a container shares the host's kernel. **Docker Engine** is the program that makes those containers easy to create, and **Docker Desktop** is how Windows and macOS users get a Linux kernel to run them on.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~30 minutes

**Prerequisites:** Chapters [2 (Kernel)](02_kernel.md), [3 (VMs)](03_virtual_machine.md) and [4 (Containers)](04_container.md).

---

## What you will learn

- A clear side-by-side comparison of VMs and containers, with honest trade-offs
- When to choose a VM, a container, or both
- What **Docker Engine** is and what it does when you run a container
- Why Docker "just works" on Linux but needs a hidden VM on Windows and macOS
- What **Docker Desktop** is (and what it isn't)
- Simple commands to *see* all of this on your own machine

---

## 1. Containers vs VMs, side by side

```
        Virtual machines                          Containers
┌───────────┐  ┌───────────┐              ┌───────────┐  ┌───────────┐
│  App A    │  │  App B    │              │  App A    │  │  App B    │
│  Libs     │  │  Libs     │              │  Libs     │  │  Libs     │
│  Guest OS │  │  Guest OS │              └─────┬─────┘  └─────┬─────┘
│  + kernel │  │  + kernel │                    │   Container   │
└─────┬─────┘  └─────┬─────┘              ┌─────┴───────────────┴─────┐
      │  Hypervisor  │                    │ Docker Engine / runtime   │
┌─────┴──────────────┴─────┐              ├───────────────────────────┤
│ Host OS (or bare metal)  │              │ Host OS + ONE shared kernel│
├──────────────────────────┤              ├───────────────────────────┤
│        Hardware          │              │         Hardware          │
└──────────────────────────┘              └───────────────────────────┘
```

The single most important difference: **VMs each run their own kernel; containers share the host's kernel.** Every other difference follows from this.

| Aspect | Virtual machine | Container |
|---|---|---|
| Kernel | Its own (guest kernel) | The host's (shared) |
| Contains an OS? | A full OS | Only the OS *files* the app needs (libraries, tools), often very small |
| Isolation | Strong (separate kernels, hardware virtualization) | Good, but weaker (one shared kernel, so a kernel bug affects all) |
| Typical size | GBs | Often 10s–100s of MB |
| Start time | Tens of seconds to minutes | Typically a second or less |
| Idle memory cost | Hundreds of MB per VM (an OS is running) | Just the app's own memory |
| Density per host | Handful to dozens | Dozens to hundreds |
| Performance | Near-native CPU, extra cost on I/O | Near-native, very little overhead |
| Run a *different* OS than the host? | Yes (Windows VM on a Linux host) | No: a Linux container needs a Linux kernel |
| Portability | Large disk images | Small layered images, easy to push and pull |
| Best for | Strong isolation, different OSes, legacy systems | Packaging and scaling apps, microservices, CI |

> **Fair comparison note:** the numbers above are typical, not guarantees. A minimal VM (a "microVM" such as Firecracker) can boot in about a tenth of a second, while a heavy container image can take a while to pull. "Containers are always faster" is a myth; they are *usually lighter and quicker to start*.

### A worked example

Ten small web services:

| | 10 VMs | 10 containers |
|---|---|---|
| OS instances running | 10 | 0 extra (1 shared kernel) |
| Idle RAM just for OS | ~5–10 GB | ~0 |
| Disk | ~10 × several GB | Shared layers, a few hundred MB total |
| Time to start all | minutes | seconds |

The containers are not "magic". They simply skip the duplicated OS.

---

## 2. When to use which

**Choose VMs when you need**
- a **different kernel or OS** (Windows software on a Linux server, or old Linux kernels)
- **strong isolation** between untrusted tenants (public cloud, hostile code)
- to run full desktops, or software that needs kernel modules or its own kernel settings
- compliance rules that demand hardware-level separation

**Choose containers when you need**
- to **package an application** and its dependencies reproducibly
- fast start-up, scaling up and down quickly (microservices, batch jobs, CI)
- high **density**: many isolated apps on one machine
- the same artifact from laptop to production

**Use both (very common!)**
- Cloud providers run **your containers inside VMs**, or in lightweight microVMs, so customers are separated by VM boundaries while you still work with containers.
- A Kubernetes cluster's worker nodes are usually VMs running containers.
- Sandboxed runtimes (Kata Containers, gVisor, Firecracker-based services like AWS Fargate) give each container VM-like isolation.

> **Rule of thumb:** *VMs virtualize hardware; containers virtualize the operating system's view.* They solve different problems and complement each other.

---

## 3. What is Docker Engine?

**Docker Engine** is the software that creates, runs and manages containers on a machine. It turns a complicated set of Linux kernel features (namespaces, cgroups, layered file systems, network setup) into simple commands.

Analogy: namespaces and cgroups are raw ingredients; Docker Engine is the kitchen that turns them into a dish on request.

### The pieces (simplified; Chapter 6 opens them up)

```
   you type:  docker run nginx
        │
   ┌────▼─────┐   REST API   ┌───────────────────────────┐
   │ docker   │ ───────────► │ dockerd  (Docker daemon)  │
   │ (CLI)    │  over a      │  images, networks,        │
   └──────────┘  socket      │  volumes, API             │
                             └────────────┬──────────────┘
                                          │ delegates to
                                   containerd  →  runc
                                          │
                                   Linux kernel
                       (namespaces, cgroups, file systems)
```

- **`docker` CLI:** the command you type. It only *asks*.
- **`dockerd` (daemon):** the long-running background service that does the work and owns images, networks and volumes. It listens on a socket (on Linux, `/var/run/docker.sock`).
- **containerd** and **runc:** lower-level components that actually start the container process (Chapter 6).

### What happens on `docker run`

1. If the image is missing locally, Docker **pulls** it from a registry.
2. It creates a **new set of namespaces** (PID, network, mount, UTS, IPC, ...).
3. It creates a **cgroup** and applies any limits you gave (`--memory`, `--cpus`).
4. It assembles the image's layers plus a thin writable layer as the container's root file system.
5. It sets up networking (a virtual network interface, plus port mappings if requested).
6. It starts the image's command as the container's main process.
7. It keeps track of the container (logs, state, exit code) until you stop or remove it.

Docker Engine is also **not** the only such tool. Podman, containerd (with `nerdctl`) and CRI-O do similar jobs; Docker's advantage is its ecosystem and familiarity.

---

## 4. Docker on Linux vs Windows vs macOS

Containers need Linux kernel features. What happens on a machine that isn't running Linux?

### On Linux: direct

```
Linux machine
┌───────────────────────────────┐
│ Docker Engine                 │
│ Container 1   Container 2     │
│ ─────── Linux kernel ──────── │
│            Hardware           │
└───────────────────────────────┘
```

Docker Engine talks to the machine's own kernel. There is no extra layer, so performance is essentially native. This is why **servers running Docker are almost always Linux**.

### On Windows and macOS: with a hidden Linux VM

Neither Windows nor macOS kernels have Linux namespaces or cgroups. So **Docker Desktop** does this for you:

```
Windows / macOS machine
┌────────────────────────────────────────┐
│ Docker Desktop app + docker CLI        │
│ ┌────────────────────────────────────┐ │
│ │ Small Linux VM (managed for you)   │ │
│ │   Docker Engine                    │ │
│ │   Container 1   Container 2        │ │
│ │   ──────── Linux kernel ────────   │ │
│ └────────────────────────────────────┘ │
│ Windows or macOS kernel                │
│ Hardware                               │
└────────────────────────────────────────┘
```

- On **Windows**, the VM typically runs through **WSL 2** (Windows Subsystem for Linux) or Hyper-V.
- On **macOS**, it uses Apple's virtualization framework (or an alternative backend, depending on settings and version).
- Your `docker` commands are forwarded into the VM. Docker Desktop also forwards ports (`localhost:8080` on your Mac reaches a container in the VM) and shares folders between the host and containers.

Important consequences:

| Topic | On Linux | On Windows / macOS |
|---|---|---|
| Where is the kernel? | The host's | Inside the Docker Desktop VM |
| `uname -r` inside a container | Host kernel version | The VM's kernel version (not Windows or macOS) |
| Speed | Native | Slightly slower for CPU; **file sharing between host and container can be noticeably slower** |
| RAM/CPU | Whatever the host has | The VM has a cap you can change in Docker Desktop settings |
| Must you use Docker Desktop? | No, install Docker Engine | It is the usual route; alternatives include Rancher Desktop, Podman Desktop and Colima |

### Good to know
- **Windows containers** also exist (containers that share a *Windows* kernel), used on Windows Server or in Windows-container mode. They are a different family and are not covered in this course. A Linux image cannot run as a Windows container, and vice versa.
- **Docker Desktop also exists for Linux**, where it deliberately runs its own VM for consistency with other platforms. Plain Docker Engine on Linux does not use a VM.
- **Licensing:** Docker Desktop is free for personal use, education and small businesses, but larger companies require a paid subscription (check Docker's current terms). Docker Engine on Linux is open source and free.
- **Different CPU types:** on Apple Silicon (ARM) Macs the VM is ARM Linux. Images built only for `amd64` may run through emulation, which is slower. Multi-architecture images solve this (Chapter 20).

---

## 5. Try it yourself

**1. Is Docker installed and talking to the engine?**

```bash
docker version        # shows Client and Server (Engine) versions
docker info | head -20
```

If you see a `Client` section but an error on `Server`, the engine (or Docker Desktop) is not running.

**2. Which kernel do containers use?**

```bash
uname -r                       # host (or your WSL) kernel
docker run --rm alpine uname -r
```

On Linux the two lines match. On Windows/macOS they differ, because the second one reports the Docker Desktop VM's Linux kernel. That is the VM from section 4.

**3. Compare start-up time and size**

```bash
docker pull alpine
docker images alpine                     # size: only a few MB
time docker run --rm alpine echo hello   # typically well under a second
```

Compare it mentally with booting a VM.

**4. See the container as a host process (Linux only)**

```bash
docker run -d --name web nginx
ps aux | grep '[n]ginx'      # the container's processes appear in the host's process list
docker rm -f web
```

**5. See resource limits in action**

```bash
docker run -d --name limited --memory=256m --cpus=0.5 nginx
docker stats --no-stream limited          # shows MEM USAGE / LIMIT
docker inspect limited --format '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}'
docker rm -f limited
```

---

## 6. Troubleshooting mindset

| Symptom | Likely cause | What to check |
|---|---|---|
| "Cannot connect to the Docker daemon" | Engine or Docker Desktop is not running, or your user can't access the socket | Start Docker Desktop; on Linux `sudo systemctl start docker` and add your user to the `docker` group |
| Container killed, exit code 137 | Out of memory (cgroup limit) or manual kill | `docker inspect <name> --format '{{.State.OOMKilled}}'`, raise `--memory` |
| Container is slow on Mac/Windows when using mounted folders | Host-container file sharing overhead | Keep heavy I/O inside the container or a volume instead of a bind mount |
| Image fails with "exec format error" | Wrong CPU architecture (e.g. `amd64` image on ARM) | Pull the right platform, or use `--platform` |
| Works on Linux server, not on my laptop (or vice versa) | Different kernel or VM configuration | Compare `docker info` and kernel versions |

---

## 7. Common misconceptions

| Misconception | Reality |
|---|---|
| "Containers are lightweight VMs" | No guest OS, no separate kernel: different architecture |
| "Docker invented containers" | Namespaces (2002+) and cgroups (2007+) existed; LXC too. Docker made them easy and standardized |
| "Docker Desktop is just a GUI" | It bundles Docker Engine, a Linux VM, the CLI, Compose, networking and file sharing. The GUI is a small part |
| "Docker runs natively on Windows and macOS" | Linux containers there run inside a Linux VM |
| "Containers are always faster than VMs" | Usually lighter, and quicker to start. Heavy CPU work is comparable, and microVMs narrow the gap |
| "A container has no operating system at all" | It has no *kernel*; it usually has some OS *files* (for example Alpine's or Debian's) |
| "Containers replace VMs" | They are commonly used together |

---

## 8. Summary

- **VM:** own kernel, own OS, strong isolation, heavier. **Container:** shared kernel, light, fast, weaker isolation.
- **Docker Engine** (CLI + `dockerd` + containerd + runc) turns kernel features into a simple workflow: pull image → create namespaces + cgroup → start process.
- On **Linux**, Docker Engine uses the host kernel directly. On **Windows/macOS**, **Docker Desktop** runs a hidden Linux VM and Docker Engine runs inside it.
- Use **VMs** for different OSes and hard isolation, **containers** for packaging and scaling, and **both** when you need each benefit.

---

## 9. Check your understanding

1. Name three differences between a VM and a container.
2. Why can't you run a Linux container directly on Windows, and how does Docker Desktop get around this?
3. On a Mac, `uname -r` in the terminal and inside `docker run alpine uname -r` give different answers. Why?
4. Your container keeps exiting with code 137. What is the most likely reason, and how do you confirm it?
5. A colleague says "we don't need VMs anymore, we have containers". How would you respond?

<details>
<summary>Answers</summary>

1. Any three of: kernel (own vs shared), size, startup time, isolation strength, ability to run another OS, density.
2. The Windows kernel has no Linux namespaces/cgroups. Docker Desktop runs a small Linux VM and runs Docker Engine inside it.
3. The first is macOS's kernel (Darwin); the second is the Linux kernel in Docker Desktop's VM.
4. It was killed for exceeding its memory limit (OOM). Confirm with `docker inspect --format '{{.State.OOMKilled}}'`.
5. VMs are still needed for different operating systems and strong isolation, and clouds usually run containers on top of VMs anyway.
</details>

---

**Next:** [Chapter 6 – Docker Engine Internals](06_docker_engine_internals.md), where we open the box: `dockerd`, containerd, runc and the API.
