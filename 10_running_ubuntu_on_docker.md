# Chapter 10: Running Ubuntu in Docker (Your First Real Container)

> **In one sentence:** In this chapter you download an Ubuntu image, start an interactive container from it, look around inside, exit, and learn what happens to the container afterward.

**Level:** 🟢 Beginner · **Reading time:** ~30 minutes (plus hands-on time)

**Prerequisites:** Docker installed and running (see the install notes below), and the basics from [Chapter 8](08_linux.md) and [Chapter 9](09_gnu_coreutils.md).

---

## What you will learn

- How to check that Docker works
- How to **pull** an image and read the output
- How to **run** a container, and why `docker run ubuntu` "does nothing"
- What `-i`, `-t`, `--name` and `--rm` mean
- What you see inside a fresh Ubuntu container, and what is missing on purpose
- The container life cycle: created → running → exited → removed
- Why your changes disappear, and three ways to deal with it
- Common errors and how to fix them

---

## 0. Setup: is Docker ready?

**Install Docker** (one-time), from the official docs at docs.docker.com/get-started/get-docker:

- **Windows / macOS:** install **Docker Desktop**, start it, and wait until it says it is running (it boots the Linux VM from Chapter 5; this can take a minute the first time).
- **Linux:** install **Docker Engine** using your distribution's instructions, then start it (`sudo systemctl enable --now docker`). Optionally allow your user to run Docker without `sudo`: `sudo usermod -aG docker $USER`, then log out and back in. (Remember: this group is root-equivalent; see Chapter 6.)

**Verify:**

```bash
docker version                 # should show both Client and Server sections
docker run --rm hello-world    # downloads a tiny image and prints "Hello from Docker!"
```

If you see `Cannot connect to the Docker daemon`, the engine isn't running yet: start Docker Desktop, or `sudo systemctl start docker` on Linux.

---

## 1. Why run Ubuntu in a container?

Ubuntu is the most common Linux for tutorials, servers and CI. A container gives you a **throw-away Ubuntu** in about a second, on any host OS, without a VM, without touching your real system:

- Learn Linux commands safely (break things freely, then delete and start again).
- Test how an install script behaves on a clean system.
- Reproduce a bug seen on an Ubuntu server.
- Try several Ubuntu versions side by side.

> Remember Chapter 4: the container has Ubuntu's **files** (programs, libraries), but it runs on **your machine's kernel**.

---

## 2. Pull the image

An **image** must be present locally before a container can start from it. Docker downloads it from a registry (Docker Hub by default):

```bash
docker pull ubuntu:24.04
```

Typical output:

```
24.04: Pulling from library/ubuntu
<layer-id>: Pull complete
Digest: sha256:<long hash>
Status: Downloaded newer image for ubuntu:24.04
docker.io/library/ubuntu:24.04
```

| Line | Meaning |
|---|---|
| `Pulling from library/ubuntu` | `library` = Docker Official Images namespace |
| `<layer-id>: Pull complete` | One image **layer** downloaded and verified (Ubuntu's base image is typically a single layer) |
| `Digest: sha256:...` | The image's immutable content fingerprint |
| `Status: Downloaded newer image` | It wasn't cached before. (`Image is up to date` if you already had it) |

### Tags: which Ubuntu?
- `ubuntu:24.04` is Ubuntu 24.04 LTS ("Noble Numbat"), `ubuntu:22.04` is 22.04 LTS ("Jammy Jellyfish"), and so on. **LTS** = Long Term Support (5 years of security updates).
- `ubuntu:latest` (or just `ubuntu`) points to the newest LTS *as chosen by the image maintainers*, not "the newest thing that exists". For repeatable work, **name the version explicitly**.
- Browse tags at hub.docker.com/_/ubuntu.

### List and inspect what you have

```bash
docker images                     # (same as `docker image ls`)
```

```
REPOSITORY   TAG      IMAGE ID       CREATED       SIZE
ubuntu       24.04    <12 chars>     2 weeks ago   ~78MB
```

The image is only tens of MB (versus several GB for a desktop install), because it holds only a minimal set of user-space files: no desktop, no kernel, no documentation, few tools. That is deliberate: you add what you need.

---

## 3. Run it: the "nothing happens" surprise

```bash
docker run ubuntu:24.04
```

It returns instantly and prints nothing. Why?

1. Docker created a container from the image.
2. Ubuntu's default command is `bash`.
3. That `bash` had **no terminal and no input** (stdin closed), so it exited immediately.
4. **A container lives exactly as long as its main process (PID 1).** Process ended → container stopped.

```bash
docker ps        # running containers: none
docker ps -a     # ALL containers: shows one with STATUS "Exited (0) ..."
```

### Keep it alive: interactive mode

```bash
docker run -it ubuntu:24.04 bash
```

Your prompt changes to something like:

```
root@3f9c2a1b7d4e:/#
```

| Piece | Meaning |
|---|---|
| `root` | The current user, the all-powerful root user *inside the container* |
| `3f9c2a1b7d4e` | The hostname, which is the first 12 characters of the container ID |
| `/` | The current directory (the container's root; on a normal Linux desktop the prompt would start in your home) |
| `#` | Prompt for root (`$` for normal users) |

You are now inside an Ubuntu container.

### The flags

| Flag | Long form | Meaning |
|---|---|---|
| `-i` | `--interactive` | Keep **stdin** open so you can type |
| `-t` | `--tty` | Allocate a **pseudo-terminal** so you get a proper prompt, colors, line editing |
| `-it` | | Both. Use these together for any interactive shell |
| `--name web1` | | Give the container a friendly name instead of a random one like `quirky_hopper` |
| `--rm` | | **Automatically delete** the container when it exits |
| `bash` (last) | | The command to run instead of the image's default |

Anatomy:

```
docker run   -it   --name lab   ubuntu:24.04   bash
   │          │        │           │             └ command inside the container
   │          │        │           └ image (repository:tag)
   │          │        └ option with a value
   │          └ flags
   └ subcommand
```

Everything after the image name is the command and its arguments; everything before is an option for Docker itself.

---

## 4. Look around inside

Try these one at a time:

```bash
cat /etc/os-release       # Ubuntu 24.04 LTS
whoami                    # root
hostname                  # the container ID
pwd                       # /
ls /                      # the standard Linux directory tree
ps aux                    # only a couple of processes: bash and ps! (PID namespace)
uname -r                  # the HOST's kernel version (shared kernel!)
echo $$                   # 1  (bash is PID 1)
```

### The directory tree (Linux "FHS")

| Directory | Purpose |
|---|---|
| `/bin`, `/usr/bin` | Programs (`ls`, `cat`, ...) |
| `/sbin`, `/usr/sbin` | System administration programs |
| `/lib`, `/usr/lib` | Shared libraries |
| `/etc` | System configuration files |
| `/home` | Normal users' home folders |
| `/root` | The root user's home folder |
| `/var` | Data that changes: logs, caches, package lists |
| `/tmp` | Temporary files |
| `/proc`, `/sys` | Virtual files served by the kernel (process and system info) |
| `/dev` | Device files (`/dev/null`, ...) |
| `/opt`, `/usr/local` | Optionally installed software |

### What is missing on purpose

A fresh Ubuntu container is very bare:

```bash
ping -c1 8.8.8.8     # bash: ping: command not found
curl --version       # bash: curl: command not found
nano file            # not found
man ls               # "This system has been minimized..."
sudo ls              # not found (you are root; you don't need it)
```

To install things, first update the package list (the image ships without one):

```bash
apt-get update
apt-get install -y curl iputils-ping nano
curl --version
```

(`apt-get` is the script-friendly form of `apt`; Chapter 11 covers package management in detail.)

**Important:** what you install lives only in **this container**. Continue reading to see what that means.

---

## 5. Leaving, and what stays behind

Exit with `exit` or **Ctrl+D**. Because `bash` was PID 1, the container **stops**, but it is **not deleted**:

```bash
docker ps -a
```

```
CONTAINER ID   IMAGE          COMMAND   CREATED         STATUS                     NAMES
3f9c2a1b7d4e   ubuntu:24.04   "bash"    2 minutes ago   Exited (0) 10 seconds ago  quirky_hopper
```

`Exited (0)` = ended successfully. `Exited (1)`, `(127)`, etc. = error codes. (Recall Chapter 9: 137 usually means killed.)

### The life cycle

```
docker create ──► Created ──docker start──► Running ──exit / docker stop──► Exited
                                              ▲                                │
                                              └──────── docker start ──────────┘
                                                                               │
                                                            docker rm ─────────▼
                                                                            Removed
```

`docker run` = `docker create` + `docker start` (+ attach if `-it`).

### Three ways to get your work back

**A. Restart the same container.** Its writable layer is intact, including the packages you installed:

```bash
docker start -ai quirky_hopper       # -a attach output, -i interactive
# or, if it is still running in another terminal:
docker exec -it quirky_hopper bash
```

**B. Start a new container.** It starts fresh from the image, so your `apt-get install` is **gone**:

```bash
docker run -it ubuntu:24.04 bash
curl --version                        # command not found again
```

This surprises beginners. Each `docker run` makes a **new** container. Containers are meant to be **disposable**.

**C. Keep important data outside the container** (volumes or bind mounts, covered in later chapters), and build a proper image with a **Dockerfile** for anything you need to repeat (Chapters 15–20).

---

## 6. Useful everyday commands

```bash
docker run -it --rm ubuntu:24.04 bash   # throw-away shell: auto-removed on exit  ← your default for experiments
docker run -it --name lab ubuntu:24.04 bash   # named, persists after exit
docker ps                               # running containers
docker ps -a                            # all containers
docker start -ai lab                    # restart a stopped container and attach
docker exec -it lab bash                # open ANOTHER shell in a running container
docker stop lab                         # ask it to stop (SIGTERM, then SIGKILL after 10 s)
docker rm lab                           # delete a stopped container
docker rm -f lab                        # force delete a running one
docker cp lab:/etc/os-release .         # copy a file OUT of a container (works both ways)
docker rmi ubuntu:24.04                 # delete the image (no containers may be using it)
docker container prune                  # delete ALL stopped containers
```

**Detach without stopping** a container you started with `-it`: press **Ctrl+P then Ctrl+Q**. The container keeps running; re-enter with `docker attach <name>` or `docker exec -it <name> bash`.

---

## 7. Experiments to try

### Experiment 1: Disposable environments
```bash
docker run -it --rm ubuntu:24.04 bash
touch /important.txt && ls /
exit
docker run -it --rm ubuntu:24.04 bash
ls /                                     # no important.txt: a brand-new container
```

### Experiment 2: Several containers from one image
Open two terminals:

```bash
# terminal 1
docker run -it --name c1 ubuntu:24.04 bash -c 'echo I am c1; sleep 300'
# terminal 2
docker run -it --name c2 ubuntu:24.04 bash -c 'echo I am c2; sleep 300'
# terminal 3
docker ps        # two containers, ONE image
docker rm -f c1 c2
```

### Experiment 3: Different versions
```bash
docker run --rm ubuntu:22.04 cat /etc/os-release | head -2
docker run --rm ubuntu:24.04 cat /etc/os-release | head -2
```

### Experiment 4: Run one command, no shell
You don't always need an interactive shell:

```bash
docker run --rm ubuntu:24.04 echo "hello from a container"
docker run --rm ubuntu:24.04 cat /etc/os-release
docker run --rm ubuntu:24.04 ls /
```

### Experiment 5: The shared kernel
```bash
uname -r
docker run --rm ubuntu:24.04 uname -r    # identical (on Windows/macOS: the Docker Desktop VM's kernel)
```

### Experiment 6: See it from the host (Linux)
```bash
docker run -d --name sleeper ubuntu:24.04 sleep 600
ps aux | grep '[s]leep 600'              # the container's process appears in the HOST's list
docker rm -f sleeper
```

---

## 8. Troubleshooting

| Message or symptom | Cause | Fix |
|---|---|---|
| `Cannot connect to the Docker daemon at unix:///var/run/docker.sock` | Engine/Docker Desktop not running | Start it (`systemctl start docker` or launch Docker Desktop) |
| `permission denied while trying to connect to the Docker daemon socket` | Your Linux user isn't in the `docker` group | `sudo usermod -aG docker $USER`, re-login, or use `sudo` |
| `docker run ubuntu` returns immediately | No terminal/stdin attached | Use `-it` (and a command like `bash`) |
| `bash: <tool>: command not found` | Minimal image | `apt-get update && apt-get install -y <package>` |
| `E: Unable to locate package` | Package lists empty | Run `apt-get update` first |
| `Conflict. The container name "/lab" is already in use` | Name taken by an old (stopped) container | `docker rm lab`, or choose another name, or use `--rm` |
| `Unable to find image ... locally` then a long pause | First-time download | Normal; the second run is fast |
| `pull access denied` / `repository does not exist` | Typo in image name, or private image | Check the name at hub.docker.com; `docker login` |
| `toomanyrequests: You have reached your pull rate limit` | Anonymous Docker Hub limit | `docker login` or wait |
| Disk filling up | Old containers and images | `docker system df`, then `docker container prune` / `docker image prune` |

---

## 9. Common misconceptions

| Misconception | Reality |
|---|---|
| "The container is an Ubuntu VM" | It's an isolated process with Ubuntu's files, running on your kernel |
| "When I `exit`, my container is deleted" | It's stopped. Only `--rm` or `docker rm` deletes it |
| "Running `docker run` again continues my session" | It creates a **new** container. Use `docker start -ai <name>` to continue |
| "`ubuntu:latest` is always the newest Ubuntu" | It's whatever tag the maintainers point to (currently the newest LTS) |
| "Being `root` in the container means being root on my machine" | Root inside is still confined by namespaces and capabilities, but it is not harmless. Chapter 13 discusses non-root users |
| "The image is small because it lacks Linux" | It lacks the kernel (shared), docs, and most tools, but has the core Ubuntu user space |

---

## 10. Summary

- `docker pull ubuntu:24.04` downloads an image; `docker images` lists them.
- `docker run -it ubuntu:24.04 bash` starts an **interactive** container; `-i` keeps stdin open, `-t` gives a terminal.
- A container runs **only as long as its main process**. Without a terminal, `bash` exits at once.
- After `exit` the container is **stopped, not removed** (`docker ps -a`). Restart with `docker start -ai`; delete with `docker rm`; use `--rm` for throw-away containers.
- Each `docker run` = a fresh container from the unchanged image, so installed packages disappear. Use Dockerfiles and volumes for anything that must last.
- The image has minimal tools by design; install what you need with `apt-get`.

---

## 11. Check your understanding

1. Why does `docker run ubuntu:24.04` print nothing and return immediately?
2. What do `-i` and `-t` do, and why are they usually used together?
3. You installed `curl` in a container, exited, and ran `docker run -it ubuntu:24.04 bash` again. Why is `curl` missing?
4. What is the difference between `docker exec` and `docker run`?
5. How do you run one command in Ubuntu and clean up automatically afterward?
6. Which command shows stopped containers?

<details>
<summary>Answers</summary>

1. The default command (`bash`) had no terminal or input, so it exited immediately, and the container stops when its main process ends.
2. `-i` keeps stdin open; `-t` allocates a pseudo-terminal. Together they give you a usable interactive shell.
3. `docker run` creates a new container from the unchanged image; the previous container's changes live only in its own writable layer.
4. `docker run` creates and starts a *new* container. `docker exec` runs an extra command inside an *already running* container.
5. `docker run --rm ubuntu:24.04 <command>`.
6. `docker ps -a`.
</details>

**Practice:** start a named container, install `curl`, exit, restart it with `docker start -ai`, and confirm `curl` is still there. Then delete the container and confirm it's gone from `docker ps -a`.

---

**Next:** [Chapter 11 – Managing Packages on Linux](11_managing_packages_on_linux.md)
