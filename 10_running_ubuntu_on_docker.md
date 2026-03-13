# Chapter 10: Running Ubuntu on Docker - A Complete Beginner's Guide

## Introduction: Why Run Linux Inside Docker?

Welcome back! In this chapter, we're taking an exciting step forward in our Docker journey. Previously, we explored Linux fundamentals and GNU Core Utilities. Now, we're going to combine everything by **running a Linux distribution (Ubuntu) inside Docker containers**.

### The Cross-Platform Challenge

Here's a common problem developers face every day:

- **You** might be working on a Mac
- **Your colleague** might be using Windows
- **Your production servers** are probably running Linux

This creates what we call the **"it works on my machine"** problem. Docker solves this by letting everyone run the exact same Linux environment, regardless of their host operating system.

### What You'll Learn

By the end of this chapter, you will:

1. Understand why containerization bridges the operating system gap
2. Know how to pull Ubuntu images from Docker Hub
3. Be able to start and interact with Ubuntu containers
4. Understand the Docker workflow from image to running container
5. Navigate inside a Linux container using basic shell commands

> **Important Philosophy**: Think of Docker as the foundation of a tree. You can see the fruit at the top (the cool applications), but you must start at the roots (Linux fundamentals) and learn to climb (Docker basics) before you can reach the fruit. Don't skip the fundamentals!

---

## Understanding the Architecture: What Happens When You Run Docker?

Before we dive into commands, let's understand the **complete picture** of what's happening behind the scenes.

### The Docker Desktop Layer

When you install Docker Desktop on Mac or Windows, you're actually installing:

1. **A lightweight virtual machine** running Linux
2. **Docker Engine** (the core Docker server)
3. **Docker CLI** (command-line interface)
4. **A graphical interface** for managing containers

**Why is Docker Desktop "heavy"?** Because it's virtualizing an entire Linux kernel on non-Linux operating systems. This takes time to start up because it needs to:
- Initialize the virtual machine
- Start the Linux kernel
- Launch the Docker daemon (background service)
- Establish networking between your host OS and the VM

### The Terminal Environment

In Linux, there are two primary ways to interact with the system:

1. **Desktop Environment** - Graphical interface (like Windows or macOS)
2. **Terminal/CLI** - Command-line interface (text-based, powerful)

When you open a terminal, you're actually interacting with a **shell**. A shell is a program that:
- Accepts your commands (like `ls`, `cd`, `mkdir`)
- Interprets those commands
- Communicates with the operating system kernel
- Executes programs from GNU Core Utilities
- Returns the output to you

Common shells include:
- **Bash** (Bourne Again Shell) - Most common on Linux
- **Zsh** (Z Shell) - Popular on modern macOS
- **Fish** - User-friendly alternative

> **Key Concept**: The shell is your interpreter. When you type `ls`, the shell finds the `ls` program in the GNU Core Utilities, executes it, and shows you the results.

---

## Step 1: Starting Docker Desktop

Before we can run any containers, Docker Desktop must be running.

### Starting Docker Desktop

**For Windows:**
1. Find Docker Desktop in your Start Menu
2. Click to launch
3. Wait for the whale icon in the system tray to become steady (not animated)

**For macOS:**
1. Find Docker Desktop in Applications
2. Launch the application
3. Wait for the whale icon in the menu bar to show "Docker Desktop is running"

### Why Does It Take So Long?

Docker Desktop is resource-intensive because it needs to:

```
┌─────────────────────────────────┐
│   Your Computer (Mac/Windows)   │
│                                 │
│  ┌───────────────────────────┐  │
│  │  Docker Desktop VM        │  │
│  │  ┌─────────────────────┐  │  │
│  │  │  Linux Kernel       │  │  │
│  │  │  ┌───────────────┐  │  │  │
│  │  │  │ Docker Engine │  │  │  │
│  │  │  │  (daemon)     │  │  │  │
│  │  │  └───────────────┘  │  │  │
│  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

The initialization sequence:
1. **VM Boot** - Start the virtual machine (5-10 seconds)
2. **Kernel Load** - Load the Linux kernel (3-5 seconds)
3. **Docker Daemon** - Start the Docker engine (2-5 seconds)
4. **Network Setup** - Configure networking bridges (1-2 seconds)
5. **Ready** - System is ready to accept commands

**Pro Tip**: Keep Docker Desktop running in the background if you're developing regularly. Closing and reopening wastes time.

---

## Step 2: Understanding Docker Hub - The Image Repository

Before we can run Ubuntu, we need to understand where Docker images come from.

### What is Docker Hub?

**Docker Hub** is the **official public registry** for Docker images. Think of it as:
- **GitHub for code** → **Docker Hub for container images**
- **App Store for apps** → **Docker Hub for containerized software**

### Key Concepts

**Container Image**: A packaged, immutable snapshot containing:
- An operating system (like Ubuntu)
- Pre-installed software
- Configuration files
- Application code
- All dependencies

**Container**: A running instance of an image. The relationship:
```
Image (Blueprint)  →  Container (Running Instance)
    Class          →      Object
    Recipe         →      Cooked Meal
    Program File   →      Running Process
```

### Exploring Docker Hub

Let's find the Ubuntu image:

1. **Open your web browser**
2. **Search for**: `docker hub`
3. **Navigate to**: `https://hub.docker.com`
4. **Search for**: `ubuntu`

You'll see the **official Ubuntu repository** with several important pieces of information:

```
ubuntu
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Official Image
Docker Pulls: 1B+
Stars: 14K+

Tags:
  - latest (22.04 LTS)
  - 24.04
  - 22.04
  - 20.04
  - 18.04
```

### Understanding Tags

**Tags** are like version numbers for images. They let you specify exactly which version you want:

- `ubuntu:latest` - The newest stable release
- `ubuntu:24.04` - Ubuntu 24.04 (Noble Numbat)
- `ubuntu:22.04` - Ubuntu 22.04 LTS (Jammy Jellyfish)
- `ubuntu:20.04` - Ubuntu 20.04 LTS (Focal Fossa)

**LTS** means "Long Term Support" - these versions receive updates for 5 years.

### The Pull Command

Docker Hub provides a command to download (pull) images:

```bash
docker pull ubuntu:24.04
```

This command means:
- `docker` - Use the Docker CLI
- `pull` - Download an image from a registry
- `ubuntu` - The image name
- `:24.04` - The specific tag/version

---

## Step 3: Pulling the Ubuntu Image

Now let's actually download Ubuntu to our local system.

### Opening Your Terminal

**macOS:**
- Press `Cmd + Space`
- Type "Terminal"
- Press Enter

**Windows (with WSL2):**
- Press `Win + R`
- Type `cmd` or `powershell`
- Press Enter

**Linux:**
- Press `Ctrl + Alt + T`

### Running the Pull Command

Type this command exactly:

```bash
docker pull ubuntu:24.04
```

**What Happens Behind the Scenes:**

```
┌────────────────┐     1. Request      ┌─────────────┐
│  Docker CLI    │ ───────────────────>│ Docker      │
│  (your input)  │                     │ Daemon      │
└────────────────┘                     └─────────────┘
                                              │
                                              │ 2. Check local cache
                                              ├──> Not found locally
                                              │
                                              │ 3. Request from Hub
                                              ▼
                                       ┌─────────────┐
                                       │ Docker Hub  │
                                       │ (Registry)  │
                                       └─────────────┘
                                              │
                                              │ 4. Download layers
                                              ▼
                                       ┌─────────────┐
                                       │ Local Cache │
                                       │ (containerd)│
                                       └─────────────┘
```

### Understanding the Output

You'll see output like this:

```
24.04: Pulling from library/ubuntu
b237fe92c9bc: Pull complete
Digest: sha256:aabed3296a3d45cede1dc866a24476c4d7e093aa806263c27ddaadbdce3c1054
Status: Downloaded newer image for ubuntu:24.04
docker.io/library/ubuntu:24.04
```

**Line-by-line explanation:**

1. **`24.04: Pulling from library/ubuntu`**
   - Confirms we're downloading Ubuntu version 24.04
   - `library/ubuntu` is the official repository path

2. **`b237fe92c9bc: Pull complete`**
   - This is a **layer hash** (unique identifier)
   - Docker images are composed of layers (we'll learn more later)
   - Each layer is downloaded and verified

3. **`Digest: sha256:aab...`**
   - A cryptographic hash of the entire image
   - Ensures the image hasn't been tampered with
   - Used for verification and security

4. **`Status: Downloaded newer image`**
   - Confirms successful download
   - "newer" means this version wasn't cached locally

5. **`docker.io/library/ubuntu:24.04`**
   - The full canonical name
   - `docker.io` - The registry (Docker Hub)
   - `library` - Official images namespace
   - `ubuntu:24.04` - Image name and tag

---

## Step 4: Verifying the Downloaded Image

After pulling, let's verify the image is stored locally.

### The Images Command

```bash
docker images
```

**Expected Output:**

```
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
ubuntu       24.04     e4c58958181a   2 weeks ago   77.9MB
```

**Understanding Each Column:**

| Column | Meaning | Example Value |
|--------|---------|---------------|
| `REPOSITORY` | Image name | `ubuntu` |
| `TAG` | Version identifier | `24.04` |
| `IMAGE ID` | Unique hash (shortened) | `e4c58958181a` |
| `CREATED` | When image was built | `2 weeks ago` |
| `SIZE` | Disk space required | `77.9MB` |

### Why is Ubuntu Only 77MB?

A full Ubuntu desktop installation is usually 2-4 GB. This Docker image is minimal because it includes:

✅ **Included:**
- Linux kernel interface (uses host kernel)
- Core system libraries
- Package manager (apt)
- Essential command-line utilities
- Bash shell

❌ **Not Included:**
- Graphical desktop environment
- Office applications
- Browsers
- Media players
- Development tools (installed separately if needed)

This is the **beauty of containers** - they contain only what's necessary for your application.

---

## Step 5: Running the Ubuntu Container

Now for the exciting part - let's actually run Ubuntu!

### The Run Command - First Attempt

```bash
docker run ubuntu:24.04
```

**What happens?**

You'll see... nothing? The command returns immediately. Let's check if anything is running:

```bash
docker ps
```

**Output:**
```
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
(empty)
```

**Why did nothing happen?** Because:
1. Docker created a container from the Ubuntu image
2. The container started
3. The container had nothing to do (no command specified)
4. The container **immediately exited**

**Key Principle**: Containers run as long as their main process is running. When that process ends, the container stops.

### The Run Command - Correct Way

To interact with Ubuntu, we need to start a **shell** and keep it running:

```bash
docker run -it ubuntu:24.04 bash
```

**Breaking Down the Command:**

```
docker run -it ubuntu:24.04 bash
│      │   │  │           └─── Command to run inside container
│      │   │  └───────────────── Image name and tag
│      │   └──────────────────────── Flags (options)
│      └──────────────────────────────── Docker action
└─────────────────────────────────────────── Docker CLI
```

### Understanding the Flags

**`-i` (interactive):**
- Keeps STDIN (standard input) open
- Allows you to type commands
- Without this, you couldn't send input to the container

**`-t` (tty):**
- Allocates a pseudo-TTY (terminal)
- Provides a proper terminal interface
- Makes output formatted and interactive

**Combined `-it`:**
- Creates a fully interactive terminal session
- You can type commands and see formatted output
- Essential for working with shells

**`bash`:**
- The command to run inside the container
- Starts the Bash shell
- This becomes the container's main process
- The container runs as long as bash is running

### What Happens When You Run It

```
┌─────────────────────────────────────────────────┐
│ Your Host Machine (Mac/Windows/Linux)          │
│                                                 │
│  Terminal running Docker CLI                   │
│  ↓                                              │
│  Docker Engine creates:                        │
│  ┌─────────────────────────────────────────┐   │
│  │ Ubuntu Container                        │   │
│  │                                         │   │
│  │  - Isolated filesystem                  │   │
│  │  - Isolated process space               │   │
│  │  - Bash shell running                   │   │
│  │                                         │   │
│  │  root@a3d5f7890b:/# ← You are here!    │   │
│  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

---

## Step 6: Inside the Ubuntu Container

Congratulations! If you see a prompt like this, you're inside Ubuntu:

```bash
root@a3d5f7890b12:/#
```

### Understanding the Prompt

Let's break down what each part means:

```
root@a3d5f7890b12:/#
│    │           │ └─ You're in the root (/) directory
│    │           └─── Separator
│    └───────────────── Hostname (container ID prefix)
└────────────────────────── Current user (root)
```

- **`root`** - You're logged in as the root user (superuser with all permissions)
- **`@`** - Separator between username and hostname
- **`a3d5f7890b12`** - The container's unique ID (first 12 characters)
- **`/`** - Your current directory (root of filesystem)
- **`#`** - Prompt character (# for root, $ for regular users)

### Your First Commands

Let's explore! Type:

```bash
ls
```

**Output:**
```
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

**What are these directories?** These are the **standard Linux filesystem directories**:

| Directory | Purpose |
|-----------|---------|
| `/bin` | Essential command binaries (ls, cp, mv, etc.) |
| `/boot` | Boot loader files (usually minimal in containers) |
| `/dev` | Device files (hardware interfaces) |
| `/etc` | System configuration files |
| `/home` | User home directories |
| `/lib` | Shared libraries needed by binaries |
| `/root` | Home directory for root user |
| `/usr` | User programs and utilities |
| `/var` | Variable data (logs, caches, etc.) |
| `/tmp` | Temporary files |

### Navigating the Filesystem

Try these commands:

**1. Change to the /bin directory:**
```bash
cd /bin
```

**2. List what's inside:**
```bash
ls
```

You'll see hundreds of programs! These are the GNU Core Utilities we discussed in previous chapters.

**3. Go back to the root directory:**
```bash
cd /
```

Or simply:
```bash
cd
```

**4. Check your current directory:**
```bash
pwd
```

Output: `/root` (the root user's home directory)

---

## Understanding What Just Happened: The Full Docker Workflow

Let's trace the complete journey from Docker Hub to running container:

### The Complete Flow

```
1. DOCKER HUB (Registry)
   ┌─────────────────────────┐
   │ Ubuntu Images           │
   │ - 24.04 (77.9 MB)      │
   │ - 22.04 (77.8 MB)      │
   │ - 20.04 (72.8 MB)      │
   └─────────────────────────┘
              │
              │ docker pull ubuntu:24.04
              ↓
2. LOCAL IMAGE STORAGE (Cached)
   ┌─────────────────────────┐
   │ Docker Engine           │
   │ /var/lib/docker/        │
   │   └─ ubuntu:24.04       │
   └─────────────────────────┘
              │
              │ docker run -it ubuntu:24.04 bash
              ↓
3. RUNNING CONTAINER
   ┌─────────────────────────┐
   │ Container ID: a3d5f789  │
   │ Image: ubuntu:24.04     │
   │ Command: bash           │
   │ Status: Running         │
   │ PID: 12345             │
   └─────────────────────────┘
```

### The Docker Engine Internals

Remember from earlier chapters, the Docker Engine has several components:

1. **Docker CLI (`docker` command)**
   - Your interface to Docker
   - Sends REST API requests to Docker daemon

2. **Docker Daemon (`dockerd`)**
   - Background service
   - Manages containers, images, networks, volumes
   - Communicates with containerd

3. **containerd**
   - Container runtime
   - Manages container lifecycle
   - Pulls and stores images

4. **runC**
   - Low-level container runtime
   - Interacts with Linux kernel
   - Creates namespaces and cgroups

### The Execution Flow

When you run `docker run -it ubuntu:24.04 bash`:

```
┌─────────────┐
│ docker CLI  │
└──────┬──────┘
       │ 1. Parse command
       │ 2. Send REST request
       ↓
┌─────────────┐
│  dockerd    │  3. Check if image exists locally
└──────┬──────┘
       │ 4. Image found in cache
       │ 5. Request container creation
       ↓
┌─────────────┐
│ containerd  │  6. Prepare container filesystem
└──────┬──────┘  7. Set up container configuration
       │
       │ 8. Request kernel container
       ↓
┌─────────────┐
│    runC     │  9. Create namespaces:
└──────┬──────┘     - PID namespace (process isolation)
       │            - Network namespace (network isolation)
       │            - Mount namespace (filesystem isolation)
       │            - UTS namespace (hostname isolation)
       │         10. Create cgroups (resource limits)
       │         11. Execute bash inside namespaces
       ↓
┌─────────────┐
│ bash shell  │  12. Running inside container
│ (PID 1)     │  13. Waiting for your commands
└─────────────┘
```

### Why Does This Matter?

Understanding this flow helps you:
- **Debug issues** - Know where problems occur
- **Optimize performance** - Understand caching behavior
- **Secure containers** - Understand isolation mechanisms
- **Troubleshoot networking** - Know how containers communicate

---

## Important Concepts: Images vs Containers

Let's cement this crucial distinction:

### The Blueprint Analogy

```
┌─────────────────────┐
│   IMAGE (Class)     │
│   ubuntu:24.04      │
│   - Read-only       │
│   - Stored on disk  │
│   - Reusable        │
└─────────────────────┘
          │
          │ docker run (instantiate)
          │
          ├──────────────┬──────────────┬──────────────┐
          ↓              ↓              ↓              ↓
    ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
    │Container1│   │Container2│   │Container3│   │Container4│
    │ Running  │   │ Stopped  │   │ Running  │   │ Exited   │
    │ ID: a3d5 │   │ ID: b7f2 │   │ ID: c9e1 │   │ ID: d4a8 │
    └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

### Key Differences

| Aspect | Image | Container |
|--------|-------|-----------|
| **Nature** | Template/Blueprint | Running instance |
| **Mutability** | Immutable (read-only) | Mutable (can change) |
| **Storage** | Stored in Docker's image store | Created from image + writable layer |
| **Lifetime** | Permanent until deleted | Temporary (deleted when removed) |
| **Resource Usage** | Disk space only | CPU, RAM, disk, network |
| **Command** | `docker images` | `docker ps` |

### The Layered Filesystem

Docker images use a **layered filesystem**:

```
Container (Read-Write Layer)
────────────────────────────────
│ Your changes, new files     │
│ Temporary data              │
────────────────────────────────
        ↓ based on
────────────────────────────────
Image (Read-Only Layers)
────────────────────────────────
│ Layer 4: Application code   │
│ Layer 3: Dependencies       │
│ Layer 2: Package manager    │
│ Layer 1: Base OS (Ubuntu)   │
────────────────────────────────
```

**Why layers matter:**
- **Efficiency** - Shared layers save disk space
- **Speed** - Only changed layers need downloading
- **Caching** - Unchanged layers are reused

---

## Exiting and Managing Containers

### Exiting the Container

You're inside the Ubuntu container. To exit:

```bash
exit
```

Or press: `Ctrl + D`

**What happens?**
1. The `bash` process terminates
2. Since bash was the main process (PID 1), the container stops
3. You return to your host machine's terminal

### Checking Container Status

After exiting, check if the container still exists:

```bash
docker ps
```

**Output:**
```
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
(empty - no running containers)
```

**Check all containers (including stopped):**

```bash
docker ps -a
```

**Output:**
```
CONTAINER ID   IMAGE          COMMAND   CREATED          STATUS                      
a3d5f7890b12   ubuntu:24.04   "bash"    2 minutes ago    Exited (0) 1 minute ago
```

**Understanding the status:**
- `Exited (0)` - Container stopped, exit code 0 (success)
- `Exited (1)` - Container stopped with error
- `Up 5 minutes` - Container currently running

### The Container Lifecycle

```
                docker run
    Image ────────────────────> Container (Running)
                                     │
                                     │ Main process exits
                                     ↓
                               Container (Stopped)
                                     │
                                     │ docker start
                                     ↓
                               Container (Running)
                                     │
                                     │ docker rm
                                     ↓
                                  Deleted
```

---

## Common Beginner Mistakes and Troubleshooting

### Mistake 1: Docker Desktop Not Running

**Symptom:**
```bash
docker: Cannot connect to the Docker daemon
```

**Solution:** Start Docker Desktop and wait until it's fully loaded.

### Mistake 2: Running Without `-it` Flags

**Command:**
```bash
docker run ubuntu:24.04 bash
```

**Result:** Container starts and immediately exits.

**Why?** Without `-it`, there's no interactive terminal. Bash sees no input source and exits immediately.

**Correct command:**
```bash
docker run -it ubuntu:24.04 bash
```

### Mistake 3: Forgetting the Shell Command

**Command:**
```bash
docker run -it ubuntu:24.04
```

**Result:** You'll see errors or unexpected behavior.

**Why?** The Ubuntu image has a default command (usually `/bin/bash`), but it's better to be explicit.

### Mistake 4: Not Understanding Persistence

**Problem:** Made changes inside container, exited, and changes are gone.

**Why?** Each `docker run` creates a **new container**. To reuse a container:

```bash
# First run - creates container
docker run -it --name my-ubuntu ubuntu:24.04 bash

# After exiting, start the same container
docker start -i my-ubuntu
```

---

## Why This Matters: The Bigger Picture

### The Development Workflow Problem

**Before Docker:**
```
Developer's Mac     QA's Windows     Production Linux
     ↓                   ↓                  ↓
Different OS → Different behaviors → Bugs in production!
"Works on my machine" syndrome
```

**With Docker:**
```
Everyone runs:
docker run -it ubuntu:24.04 bash

↓
Identical environment for everyone!
```

### Real-World Use Cases

**1. Development Environment Consistency**
- Team uses different OS (Mac, Windows, Linux)
- Docker ensures everyone has identical Ubuntu environment
- No more "works on my machine" excuses

**2. Testing Different Linux Distributions**
- Test on Ubuntu 24.04, 22.04, 20.04
- Test on Debian, Alpine, CentOS
- No need to install each OS separately

**3. Safe Experimentation**
- Try commands without affecting your computer
- Break things without consequences
- Delete and recreate in seconds

**4. CI/CD Pipelines**
- Automated testing in clean environments
- Reproducible builds
- Version-controlled infrastructure

---

## The Philosophy: Understanding vs. Memorizing

### The Tree Climbing Metaphor

Imagine learning Docker is like climbing a tree to get fruit:

```
                    🍎 Fruit (Advanced Docker Skills)
                    │
            ┌───────┴───────┐
        🌿 Branches (Docker Commands)
            │
    ───────┴─────── (Docker Basics)
            │
    ═══════╬═══════ (Linux Fundamentals)
         Root
```

**The Journey:**
1. **Intention** - You see the fruit and want it (this course)
2. **Foundation** - You must come to the tree base (Linux basics)
3. **Climbing** - Learn Docker fundamentals (this chapter)
4. **Reaching** - Practice until you reach the fruit (mastery)

### Why Linux Fundamentals Come First

Without Linux knowledge, you'll face:
- **Mystery commands** - Why do these work? Where do they come from?
- **Debugging nightmares** - Can't troubleshoot without understanding
- **Impostor syndrome** - Feel like everyone knows magic you don't
- **Career ceiling** - Can't advance to Kubernetes, cloud platforms

With Linux knowledge:
- **Confidence** - Understand why things work
- **Problem-solving** - Debug issues independently
- **Career advancement** - Stand out from other developers
- **Future-proof** - Foundation for DevOps, cloud, microservices

---

## Practice Exercises

### Exercise 1: Pull Different Ubuntu Versions

Pull and explore multiple Ubuntu versions:

```bash
# Pull Ubuntu 22.04
docker pull ubuntu:22.04

# Pull Ubuntu 20.04
docker pull ubuntu:20.04

# Verify all images
docker images

# Run 22.04
docker run -it ubuntu:22.04 bash

# Inside container, check version
cat /etc/os-release

# Exit and try 20.04
exit
docker run -it ubuntu:20.04 bash
cat /etc/os-release
```

**Question:** What differences do you notice between versions?

### Exercise 2: Named Containers

Create containers with specific names:

```bash
# Create named container
docker run -it --name dev-environment ubuntu:24.04 bash

# Exit container
exit

# Start the same container again
docker start -i dev-environment

# Remove the container
docker rm dev-environment
```

### Exercise 3: Multiple Containers from One Image

Run multiple containers from the same image:

```bash
# Terminal 1
docker run -it --name container1 ubuntu:24.04 bash

# Open Terminal 2
docker run -it --name container2 ubuntu:24.04 bash

# In Terminal 3, list running containers
docker ps
```

**Question:** Can multiple containers run from one image simultaneously? (Answer: Yes!)

---

## Key Takeaways

1. **Docker Hub** is the public registry where images are stored
2. **Images** are immutable templates; **containers** are running instances
3. **`docker pull`** downloads images to your local system
4. **`docker images`** shows cached images on your computer
5. **`docker run -it ubuntu:24.04 bash`** creates an interactive Ubuntu container
6. **Flags matter**: `-i` for interactive, `-t` for terminal, `bash` for shell
7. **Containers are isolated** - they have their own filesystem and processes
8. **Exiting bash** stops the container because bash is PID 1
9. **`docker ps`** shows running containers; **`docker ps -a`** shows all containers
10. **Understanding the flow** (Hub → Image → Container) is crucial

---

## Coming Up Next

In the next chapters, we'll dive deeper into:
- **Package Management** - Installing software inside containers
- **Linux Commands** - Navigating and manipulating the filesystem
- **User and Permissions** - Understanding security and access control
- **Advanced Docker** - Building custom images with Dockerfile

---

## Conclusion

Congratulations! You've taken your first real step into the containerized world. You now understand:
- Why we run Linux inside Docker
- How to pull images from Docker Hub
- How to create and interact with containers
- The distinction between images and containers
- The internal Docker workflow from CLI to running container

This foundational knowledge is critical. Don't rush past it. Practice until these concepts feel natural. The journey to mastering Docker, Kubernetes, microservices, and cloud platforms **starts here**, with these fundamentals.

**Remember:** You're not just learning commands to memorize. You're building a mental model of how containerization works. This understanding will serve you throughout your entire career as a software engineer.

**Next time**, we'll explore package management in Linux - how to install, update, and remove software inside our Ubuntu containers. This will unlock the ability to create truly custom environments for any application.

Keep practicing, stay curious, and remember: the strongest developers are those who master the fundamentals!

---

**Chapter Progress**: ✅ Chapter 17 Complete

**Next Chapter**: Chapter 18 - Managing Packages on Linux
