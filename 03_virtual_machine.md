# Chapter 3: Virtual Machines

> **In one sentence:** A virtual machine (VM) is a complete, pretend computer, with its own virtual CPU, memory, disk and its own operating system, running as software inside a real computer.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~25 minutes

**Why this chapter is in a Docker course:** VMs were the standard way to isolate software before containers. Once you see what a VM costs (a whole extra OS for every app), you will see exactly what containers improve.

**Prerequisite:** [Chapter 2 – Kernel](02_kernel.md) (kernel, user space, system calls).

---

## What you will learn

- What a VM is, and what "virtual" means here
- What a **hypervisor** is, and the difference between Type 1 and Type 2
- **Host** vs **guest** operating systems
- How a VM gets its CPU, memory, disk and network
- How hypervisors can hand out more than the machine really has (overcommitment, ballooning)
- What VMs are great at, and where they hurt
- How to check that your own computer supports virtualization, and try one

---

## 1. The problem VMs solve

A physical computer normally runs **one** operating system. That creates problems:

- You use Windows but need a Linux tool. Do you wipe Windows? Dual-boot and restart every time?
- A company has 100 servers, each running one small app and each only 10% busy. That is a lot of wasted hardware and electricity.
- You want to test risky software without harming your real system.

The solution: **run several complete "computers" as software inside one real computer**, each fully separated from the others.

---

## 2. What is a virtual machine?

**Virtual** means "looks and behaves like the real thing, but is created by software."

A VM is a software-created computer made of:

| Virtual part | What it really is on the host |
|---|---|
| Virtual CPU (vCPU) | A share of time on the real CPU cores |
| Virtual RAM | A chunk of the host's real memory |
| Virtual disk | Usually just **one big file** on the host's disk (e.g. `ubuntu.vdi`, `disk.vmdk`, `disk.qcow2`) |
| Virtual network card | A software network interface connected to a virtual switch |

Inside the VM you install a normal operating system (the **guest OS**), with its **own kernel**. It boots like a real PC and works like one.

**Analogy:** your real body is the physical machine. A character in a video game is "virtual". It has its own world with its own rules, and the game software makes it all possible.

---

## 3. The hypervisor

A **hypervisor** (also called a *virtual machine monitor*, VMM) is the software that creates and runs VMs. It:

1. **Creates** virtual hardware for each VM.
2. **Schedules** VMs on the real CPU cores.
3. **Divides** real memory, disk and network between VMs.
4. **Keeps VMs isolated** from each other.

### Two types

```
   Type 1 ("bare metal")              Type 2 ("hosted")
┌────┐ ┌────┐ ┌────┐             ┌────┐ ┌────┐ ┌────┐
│VM 1│ │VM 2│ │VM 3│             │VM 1│ │VM 2│ │VM 3│
└────┘ └────┘ └────┘             └────┘ └────┘ └────┘
  Hypervisor (is the "OS")          Hypervisor (an app)
       Hardware                      Host OS (Windows/macOS/Linux)
                                          Hardware
```

| | Type 1 (bare metal) | Type 2 (hosted) |
|---|---|---|
| Runs on | Directly on hardware | As a program inside a normal OS |
| Typical use | Data centres, cloud providers | Laptops, learning, testing |
| Examples | VMware ESXi, Xen, Microsoft Hyper-V, KVM (built into the Linux kernel) | VirtualBox, VMware Workstation/Fusion, Parallels |

> **Gray areas:** the categories are a teaching aid. KVM turns the Linux kernel itself into a Type 1 hypervisor even though Linux is a "normal" OS. Hyper-V, when enabled on Windows, runs *underneath* Windows and treats it as a special VM.

---

## 4. Host and guest

| Term | Meaning |
|---|---|
| **Host** (machine / OS) | The real computer and the operating system that (in the Type 2 case) runs the hypervisor |
| **Guest** (OS / VM) | An operating system running inside a VM |

You can run a Linux guest on a Windows host, several guests at once, and so on. Each guest has:

- its **own kernel** (this is the key point for the next chapters!)
- its **own file system**, its own processes, its own network settings

```
┌──────────────────────────────── Physical computer ─────────────────────────────────┐
│ Host OS (with its kernel)                                                          │
│   Chrome   Spotify   Hypervisor (VirtualBox)                                       │
│                         ├── VM 1: Linux   → guest kernel → apps                    │
│                         ├── VM 2: Windows → guest kernel → apps                    │
│                         └── VM 3: Linux   → guest kernel → apps                    │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. How a VM actually runs

Beginners often imagine that every instruction passes through many slow translation layers. Modern reality is more efficient.

### CPU: mostly native speed
Modern CPUs have **hardware virtualization support** (Intel **VT-x**, AMD **AMD-V**). Guest code runs *directly on the real CPU* in a special "guest mode". The hypervisor only steps in when the guest does something sensitive, such as touching hardware. So a CPU-heavy program in a VM is usually only a few percent slower than on bare metal.

### Devices: emulated or paravirtualized
When the guest kernel talks to "hardware" (disk, network card), the hypervisor intercepts it:

```
Guest app  ──syscall──►  Guest kernel  ──"write to disk"──►  Hypervisor
                                                                │
                                        turns it into a write on the disk-image file
                                                                ▼
                                                     Host kernel → real disk
```

So the *I/O path* is longer than for a normal program. To reduce that cost, hypervisors provide **paravirtualized drivers** (such as `virtio`, or "guest additions/tools") that let guest and hypervisor cooperate efficiently instead of pretending to be an old real device.

### Memory
The guest thinks it manages real RAM. In fact it manages "guest-physical" memory that the hypervisor maps to real host memory (with hardware help, called nested paging / EPT).

### Does the guest know it is virtual?
Usually a guest OS *can* tell (special CPU flags, device names, installed guest tools). It is designed to work as if on real hardware, but it is not truly "unaware".

---

## 6. Sharing resources and overcommitment

Suppose the host has **6 cores and 10 GB RAM** and you create:

| VM | vCPUs | RAM | Disk |
|---|---|---|---|
| VM 1 | 4 | 4 GB | 50 GB |
| VM 2 | 4 | 8 GB | 50 GB |
| Total | 8 | 12 GB | 100 GB |

You promised more than exists. This is called **overcommitment**, and it usually works because VMs rarely all use their full share at once.

- **CPU:** vCPUs are threads scheduled onto real cores in turns (time-slicing), just as the kernel schedules ordinary processes. Eight vCPUs can share six cores; if both VMs are busy each simply gets less.
- **RAM:** the hypervisor may use several tricks:
  - **Ballooning:** a small driver inside the guest ("balloon") asks the guest OS for memory, "inflating" itself. The hypervisor can then take the pages the balloon holds and give them to another VM. When memory is plentiful again, the balloon "deflates".
  - **Page sharing / compression / swapping:** identical or idle memory pages are merged, compressed, or moved to disk. Swapping is the slowest and can hurt performance badly.
- **Disk:** virtual disks are often **thin-provisioned**: a "50 GB" disk file starts small and grows as the guest writes data. If every VM fills its disk, the host runs out of space, so monitor it.

> **Rule of thumb:** overcommit CPU generously, RAM cautiously, disk carefully.

---

## 7. Isolation

Each VM gets its own kernel and its own virtual hardware, so:

- VM 1 cannot see VM 2's files or memory.
- A crash in VM 1 does not stop VM 2.
- Malware inside a VM is *very* hard to move to the host (though hypervisor bugs, "VM escapes", have existed, they are rare and taken seriously).

This **strong isolation** is why cloud providers use VMs to separate customers from each other.

---

## 8. Try it yourself

### Check whether your CPU supports virtualization (Linux)

```bash
lscpu | grep -i virtualization
# or
grep -E -c '(vmx|svm)' /proc/cpuinfo
```

- `VT-x` (Intel) or `AMD-V` (AMD) in the first command, or a number greater than 0 from the second, means it is supported. It may still need to be enabled in the BIOS/UEFI settings.

### Check whether you are already inside a VM

```bash
systemd-detect-virt        # prints "none" on bare metal, or e.g. "kvm", "oracle", "vmware"
```

### Create a VM with VirtualBox (any OS)

1. Install VirtualBox from virtualbox.org and download an Ubuntu Server or Desktop ISO.
2. **New** → name it, choose the ISO, set 2 CPUs, 4 GB RAM, 25 GB disk.
3. Start it and follow the normal installer. It takes as long as a real install.
4. Once installed, run `uname -r` inside the VM. Compare with the host (`uname -r`). If the host is also Linux you may see different versions: **the guest has its own kernel**.

Notice how long it takes to install and boot, and how much disk the VM file uses (`ls -lh` on the `.vdi`). Keep that in mind for the next chapter.

---

## 9. Strengths and weaknesses of VMs

| Strengths | Weaknesses |
|---|---|
| Run *any* OS on *any* host (Windows on Linux, etc.) | Every VM carries a **full OS**: gigabytes of disk, hundreds of MB of RAM idle |
| Strong isolation, separate kernels | Boot time: tens of seconds to minutes |
| Snapshots and easy rollback | Harder to move around (multi-GB disk images) |
| Server consolidation: many workloads on one machine | Lower density: you can fit only a handful per host, not hundreds |
| Foundation of the public cloud | Some I/O overhead |

**The wasteful example:** to run one 10 MB program in a VM you must first install and boot an entire operating system, perhaps 1–2 GB, just to host it.

That waste is the gap containers fill.

---

## 10. Myths and clarifications

| Myth | Reality |
|---|---|
| "VMs are very slow" | CPU-bound work runs near native speed thanks to VT-x/AMD-V. I/O and start-up cost more |
| "Every instruction goes through the hypervisor" | Only privileged/sensitive operations do |
| "Ballooning means giving 12 GB from 10 GB" | It reclaims *unused* guest memory so it can be reused elsewhere; overcommit beyond real demand still causes swapping |
| "A VM is just a folder of files" | The disk *is* a file, but the VM also needs the hypervisor and virtual hardware definition |
| "Containers are lightweight VMs" | Containers do not virtualize hardware or run a separate kernel. See the next chapters |

---

## 11. Summary

- A **VM** is a software-defined computer with its own virtual hardware and its own OS/kernel.
- A **hypervisor** creates and manages VMs. **Type 1** runs on hardware; **Type 2** runs on a host OS.
- The OS on the real machine is the **host**; OSes inside VMs are **guests**.
- Hardware virtualization (VT-x / AMD-V) makes VMs fast for CPU work; device I/O goes through the hypervisor.
- CPU, RAM and disk can be **overcommitted**; ballooning and thin provisioning help.
- VMs give **strong isolation** but are **heavy**, because every VM includes a full OS.

---

## 12. Check your understanding

1. What is the difference between a hypervisor of Type 1 and Type 2? Name one example of each.
2. How many kernels are running if a Windows laptop hosts three Linux VMs?
3. What does "the virtual disk is a file" mean, and what is thin provisioning?
4. Why can 8 vCPUs run on a 6-core machine?
5. Give two advantages and two disadvantages of VMs.

<details>
<summary>Answers</summary>

1. Type 1 runs directly on the hardware (VMware ESXi, Xen, KVM); Type 2 runs as an application on a host OS (VirtualBox, VMware Workstation).
2. Four: one host Windows kernel plus one guest kernel per VM.
3. The guest's whole disk is stored as one file on the host. Thin provisioning means the file grows only as data is written.
4. vCPUs are time-sliced onto real cores; they take turns.
5. Advantages: strong isolation; can run different OSes; snapshots. Disadvantages: heavy (full OS each), slow boot, large images.
</details>

---

**Next:** [Chapter 4 – Container](04_container.md): isolation without a guest OS.
