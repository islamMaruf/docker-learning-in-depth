# Chapter 14: Docker Hands-On (Images, Containers, `exec` and Your First Build)

> **In one sentence:** This is the practical chapter. You install Docker, learn the day-to-day commands for images and containers, open extra shells in running containers, and build your first custom image with a Dockerfile.

**Level:** 🟢 Beginner · **Reading time:** ~45 minutes, plus practice

**Prerequisites:** [Chapter 10](10_running_ubuntu_on_docker.md) (you have already run Ubuntu in a container). If you skipped it, skim it first.

---

## What you will learn

- Install Docker and verify it works
- Manage **images**: `pull`, `images`, `rmi`, `inspect`, `history`
- Manage **containers**: `run`, `ps`, `exec`, `stop`, `rm`, `logs`
- The flags you use constantly: `-i`, `-t`, `--name`, `--rm`
- Why containers are **isolated** from each other (and what `exec` shares)
- Write your first **Dockerfile** and `docker build` it, with names and tags
- The cleanup commands, and how to recover from common errors

---

## 0. Install Docker

| System | What to install | Notes |
|---|---|---|
| **Windows 10/11** | **Docker Desktop** | Uses WSL 2 (recommended). Enable virtualization in BIOS/UEFI if asked |
| **macOS** (Intel or Apple Silicon) | **Docker Desktop** | Runs a hidden Linux VM (Chapter 5) |
| **Linux** | **Docker Engine** (`docker-ce`) | Follow docs.docker.com/engine/install for your distribution; then `sudo systemctl enable --now docker` |

After installing:

```bash
docker --version
docker version            # both "Client" and "Server" sections should appear
docker run --rm hello-world
```

On Linux, to avoid typing `sudo` every time: `sudo usermod -aG docker $USER`, then log out and in. (This makes the user effectively root-equivalent on that machine; see Chapter 13.) Docker Desktop's start-up takes a while because it boots a VM; leave it running while you work.

---

## 1. Working with images

An **image** is a read-only template. Containers are created from images.

```bash
docker images                 # list local images   (same as: docker image ls)
docker pull ubuntu:24.04      # download an image (default registry: Docker Hub)
docker rmi ubuntu:24.04       # remove an image     (same as: docker image rm)
```

Sample output of `docker images`:

```
REPOSITORY   TAG      IMAGE ID       CREATED        SIZE
ubuntu       24.04    9873176a8ff5   2 weeks ago    78.1MB
```

| Column | Meaning |
|---|---|
| REPOSITORY | Image name (`ubuntu`) |
| TAG | Version or variant (`24.04`). If omitted when you pull/run, Docker uses `latest` |
| IMAGE ID | First 12 characters of the image's 64-character SHA-256 ID |
| CREATED | When the image was **built** (not when you downloaded it) |
| SIZE | Disk size of the image's layers (shared layers are stored once) |

You can refer to an image by `name:tag` or by (a prefix of) its ID.

### Removing images
```bash
docker rmi ubuntu:24.04
```
fails with "image is being used by stopped container ..." if any container, even a stopped one, was created from it. Fix it the clean way: remove the containers first (`docker rm <container>`), then the image. `docker rmi -f` forces removal of an image used by *stopped* containers (it cannot remove an image used by a *running* one), but leaves those containers orphaned, so prefer the clean way.

### Look inside an image
```bash
docker inspect ubuntu:24.04                 # full JSON: config, layers, env, default command
docker inspect -f '{{.Config.Cmd}}' ubuntu:24.04   # just the default command → [/bin/bash]
docker history ubuntu:24.04                 # the layers and how each was made
```

---

## 2. Working with containers

A **container** is a running (or stopped) instance of an image.

```bash
docker ps                # running containers only     (docker container ls)
docker ps -a             # ALL containers, including stopped ones
```

```
CONTAINER ID   IMAGE          COMMAND   CREATED          STATUS                    NAMES
a1b2c3d4e5f6   ubuntu:24.04   "bash"    2 minutes ago    Up 2 minutes              my_ubuntu
9f8e7d6c5b4a   ubuntu:24.04   "bash"    5 minutes ago    Exited (0) 4 minutes ago  quirky_hopper
```

- `ps` is borrowed from Linux ("process status"): a container is basically a process.
- **STATUS** `Up ...` = running, `Exited (N)` = stopped with exit code N (0 = success), `Created` = never started, `Paused`, `Restarting`.
- A container lives **exactly as long as its main process** (PID 1). When bash exits, the container stops. This is the number-one Docker principle.

### `docker run`: create and start

```
docker run [OPTIONS] IMAGE [COMMAND] [ARGS...]
```

Things after the image are the command *inside* the container; if you omit it, the image's default command is used (`bash` for Ubuntu).

**Why `docker run ubuntu:24.04` seems to do nothing:** bash starts with no terminal attached, reads end-of-input, and exits immediately. You need to attach a terminal:

| Flags | What you get |
|---|---|
| *(none)* | Non-interactive: stdin closed. bash exits at once |
| `-i` | Keeps **stdin** open; you can type, but there is no prompt or line-editing |
| `-t` | Allocates a **pseudo-TTY**: a proper prompt/formatting, but stdin isn't attached, so you can't type |
| `-it` | Both: **use this for interactive shells** |

```bash
docker run -it ubuntu:24.04 bash
# root@a1b2c3d4e5f6:/#      ← now inside; try: ls, cat /etc/os-release, exit
```

`exit` (or **Ctrl+D**) ends bash, which ends the container.

### Give containers names
Without `--name`, Docker picks a random name (`quirky_hopper`). Choose your own:

```bash
docker run --name my_ubuntu -it ubuntu:24.04 bash
```

- Names must be **unique** among existing containers (running or stopped). Reusing one gives *"Conflict. The container name is already in use"*: remove the old one (`docker rm my_ubuntu`) or pick another.
- Allowed characters: letters, digits, `_`, `.` and `-` (not starting with a symbol).
- Anywhere Docker asks for a container, you can use the **name** or the (prefix of the) **ID**.

### `--rm`: clean up automatically
```bash
docker run --rm -it ubuntu:24.04 bash    # container is deleted when it exits
```
Use `--rm` for experiments so stopped containers don't pile up.

### `docker exec`: run a command inside a *running* container

```bash
# terminal 1
docker run --name my_ubuntu -it ubuntu:24.04 bash

# terminal 2  (the container must still be running)
docker exec -it my_ubuntu bash
```

You now have **two shells in the same container**. Prove it:

```bash
# terminal 1
echo "hello from 1" > /test.txt
# terminal 2
cat /test.txt                  # hello from 1
echo "hello from 2" >> /test.txt
# terminal 1
cat /test.txt                  # both lines
```

| | `docker run` | `docker exec` |
|---|---|---|
| Creates a new container? | **Yes** | **No** |
| Needs the container running? | n/a | **Yes** |
| Typical use | Start something new | Debug or administer something already running |

`exec` also works for one-off commands without a terminal: `docker exec my_ubuntu cat /etc/os-release`. Exiting an `exec`'d shell does **not** stop the container, only the process you started. (Stopping happens when the *main* process ends.) Useful `exec` options: `-u root` (as a specific user), `-w /app` (working directory), `-e VAR=value` (environment variable).

### Stopping, starting, removing

```bash
docker stop my_ubuntu       # polite: SIGTERM, then SIGKILL after 10 s
docker start -ai my_ubuntu  # restart a stopped container, attach output (-a) and input (-i)
docker restart my_ubuntu
docker kill my_ubuntu       # immediate SIGKILL
docker rm my_ubuntu         # delete a STOPPED container
docker rm -f my_ubuntu      # stop and delete
docker logs my_ubuntu       # what the container's main process printed (-f to follow)
docker top my_ubuntu        # processes inside
docker stats --no-stream    # CPU/memory usage
```

(Chapter 19 covers container management in depth.)

---

## 3. Isolation: every container has its own file system

```bash
docker run --name c1 -it ubuntu:24.04 bash
echo "I live in c1" > /myfile.txt
ls /
exit

docker run --name c2 -it ubuntu:24.04 bash
ls /                          # no myfile.txt: a different container
exit

docker start -ai c1
cat /myfile.txt               # still here! c1's own writable layer survived
exit
```

Lessons:

- Two containers from the **same image** start identical but **diverge independently** (each has its own thin *writable layer*; the image itself never changes).
- A **stopped** container keeps its files. Deleting it (`docker rm`) deletes them.
- Their process tables, networks, and hostnames are isolated too (Chapter 4). What they *share* is the host kernel (and any volumes you mount).

```bash
docker rm c1 c2      # clean up
```

---

## 4. Building your first image

So far we have only used ready-made images. To make your own, write a **Dockerfile**: a text file of instructions Docker follows to build an image.

### 4.1 The Dockerfile

Make a folder and a file named exactly **`Dockerfile`** (capital D, no extension):

```bash
mkdir -p ~/docker-tutorial && cd ~/docker-tutorial
```

`Dockerfile`:

```dockerfile
FROM ubuntu:24.04
RUN echo "Hello World" > /hello.txt
```

| Instruction | Meaning |
|---|---|
| `FROM ubuntu:24.04` | The **base image** to start from. Every Dockerfile starts with `FROM` (optionally preceded by `ARG`) |
| `RUN echo ... > /hello.txt` | Run a command **at build time** inside a temporary container; the file changes it makes become a new **layer** in the image |

Important: `RUN` executes while *building*, not when you later run a container.

### 4.2 Build it

```bash
docker build .
```

The `.` is the **build context**: the folder whose files Docker can send to the builder (for `COPY`). By default Docker looks for a file named `Dockerfile` in it. (Use `-f path/to/File` for another name.)

Typical output:

```
[+] Building 2.3s (6/6) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load metadata for docker.io/library/ubuntu:24.04
 => [internal] load .dockerignore
 => [1/2] FROM docker.io/library/ubuntu:24.04
 => [2/2] RUN echo "Hello World" > /hello.txt
 => exporting to image
 => => writing image sha256:7ea6a91e298f...
```

Docker (using **BuildKit**) read the Dockerfile, fetched the base image if needed, ran your `RUN` in a temporary container, saved the resulting layer, and wrote a new image with a unique ID.

```bash
docker images
# REPOSITORY   TAG      IMAGE ID       ...
# <none>       <none>   7ea6a91e298f   ...     ← built without a name
```

`<none>` is inconvenient. Name (tag) it with **`-t`**:

```bash
docker build -t custom_ubuntu .             # name custom_ubuntu, tag latest (default)
docker build -t custom_ubuntu:1.0 .         # explicit tag
docker build -t custom_ubuntu:1.0 -t custom_ubuntu:latest .   # several tags at once
```

Rebuilding the same content is **instant**: Docker reuses cached layers (you'll see `CACHED`), and only the new tag is added.

Image names must be **lowercase**; tags may contain letters, digits, `_`, `.`, `-`.

### 4.3 Run it

```bash
docker run -it --rm custom_ubuntu:1.0 bash
cat /hello.txt        # Hello World
```

The file was created once at **build** time, and is present in every container created from the image.

### 4.4 A slightly more useful example

`Dockerfile`:

```dockerfile
FROM ubuntu:24.04
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl ca-certificates \
 && rm -rf /var/lib/apt/lists/*
RUN echo "This is my custom image" > /readme.txt
CMD ["curl", "--version"]
```

```bash
docker build -t my-first-image .
docker run --rm my-first-image                      # runs the CMD → prints curl's version
docker run --rm my-first-image cat /readme.txt      # override the CMD with your own command
docker history my-first-image                       # see each layer and its size
```

(Why the `apt-get` lines look this way: Chapter 11. `CMD`: Chapter 16. More instructions: Chapters 15–20.)

### 4.5 What happens during a build (mental model)

```
Dockerfile ──docker build──► Image ──docker run──► Container
 (recipe)                  (frozen result)        (running instance)
```

For each instruction Docker: starts a temporary container from the previous layer's result → runs the instruction → saves the file changes as a **new read-only layer** → moves on. Layers are **cached**: if an instruction and everything before it are unchanged, Docker reuses the cached layer, which makes rebuilds fast. So put things that change **rarely** (installing packages) **early** and things that change **often** (your code) **late**.

### `docker run` vs `docker build`

| | `docker build` | `docker run` |
|---|---|---|
| Input | A Dockerfile (+ build context) | An image |
| Output | An **image** | A **container** |
| Analogy | Writing and printing the blueprint | Building a house from it |

---

## 5. Everyday housekeeping

```bash
docker ps -a                          # see all containers
docker rm $(docker ps -aq)            # remove ALL stopped containers (also: docker container prune)
docker image prune                    # remove dangling images (untagged leftovers)
docker image prune -a                 # remove all images not used by a container (careful)
docker system df                      # what's using disk space
docker system prune                   # stopped containers + unused networks + dangling images + build cache
docker system prune -a --volumes      # everything unused, INCLUDING volumes (data loss risk!)
```

Command names have two forms. `docker ps`, `docker images`, `docker rm`, `docker rmi` are shortcuts for `docker container ls`, `docker image ls`, `docker container rm`, `docker image rm`. Both work, and the long forms are the modern grouped syntax.

---

## 6. Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `Cannot connect to the Docker daemon` | Docker Desktop/Engine isn't running | Start it; on Linux `sudo systemctl start docker` |
| `permission denied ... docker.sock` | User isn't in the `docker` group | Add the user or use `sudo` |
| Container exits immediately | Main process finished | Use `-it` with a shell, or check `docker logs <name>` |
| `name "/x" is already in use` | Old container with that name | `docker rm x` or new name |
| `image is being used by stopped container` | Containers still reference it | `docker rm` them first (or `docker rmi -f`) |
| `docker build` says `failed to read dockerfile: open Dockerfile: no such file` | Wrong folder, or file name/case wrong | `ls` to check; `cd` into the folder or use `-f` |
| Build fails at `RUN apt-get install` with `Unable to locate package` | No `apt-get update` in the same `RUN` | Chapter 11 |
| `pull access denied` / `manifest unknown` | Typo in image name/tag, or private image | Check spelling and tag on Docker Hub; `docker login` |
| Changes I made in a container vanished | You started a **new** container; changes live in the old one | `docker start -ai <old>`, or bake changes into an image with a Dockerfile |
| Disk full | Accumulated images, containers, build cache | Section 5 |

---

## 7. Good habits

1. **Name** important containers (`--name`) and images (`-t name:tag`).
2. Use **`--rm`** for experiments.
3. **Pin versions** (`ubuntu:24.04`, not `latest`) in Dockerfiles.
4. Put the **rarely changing** steps first in a Dockerfile to benefit from the build cache.
5. Don't treat containers as pets: make changes in the **Dockerfile**, then rebuild.
6. Check `docker ps -a` and clean up regularly.

---

## 8. Cheat sheet

| Goal | Command |
|---|---|
| Download / list / remove image | `docker pull IMG` · `docker images` · `docker rmi IMG` |
| List running / all containers | `docker ps` · `docker ps -a` |
| Interactive throw-away shell | `docker run -it --rm IMG bash` |
| Named container | `docker run --name NAME -it IMG bash` |
| Second shell in running container | `docker exec -it NAME bash` |
| Stop / start / delete | `docker stop NAME` · `docker start -ai NAME` · `docker rm NAME` |
| Output of the container | `docker logs -f NAME` |
| Build (untagged / tagged) | `docker build .` · `docker build -t NAME:TAG .` |
| Cleanup | `docker container prune` · `docker image prune` · `docker system prune` |

---

## 9. Summary

- Images are templates; containers are instances. `docker images` / `docker ps -a` show them.
- A container runs while its main process runs; `-it` gives an interactive shell; `--name` and `--rm` keep things tidy.
- `docker exec` opens additional processes in a **running** container; `docker run` always creates a **new** one.
- Each container has its own file system layer, isolated from siblings and preserved while the container exists.
- A **Dockerfile** + `docker build -t name:tag .` creates your own image; `RUN` executes at build time; layers are cached.

---

## 10. Check your understanding

1. What is the difference between `docker run` and `docker exec`?
2. What happens if you run `docker run -t ubuntu:24.04 bash` without `-i`?
3. You created a file in container `c1`, exited, and started a new container `c2` from the same image. Is the file there? Why or why not?
4. What does the `.` at the end of `docker build -t app .` mean?
5. Why is `<none>` shown as the name of an image, and how do you avoid it?
6. When does a `RUN` instruction execute: at build time or at container start?
7. `docker rmi ubuntu:24.04` fails. What are the likely reasons and the clean fix?

<details>
<summary>Answers</summary>

1. `run` creates and starts a new container from an image; `exec` runs an extra command inside an existing running container.
2. You get a terminal prompt, but stdin isn't attached, so you can't type anything into it.
3. No. Each container has its own writable layer; `c2` starts from the unchanged image.
4. The build context: the current directory (where the Dockerfile and files for `COPY` live).
5. The image was built without a name/tag. Use `docker build -t name:tag .`.
6. At **build** time; its result is stored in the image.
7. A container (even stopped) was created from that image. Remove it with `docker rm`, then remove the image (or use `-f`).
</details>

**Practice**

1. Pull `alpine`, compare its size to Ubuntu's with `docker images`, then remove it.
2. Start a container, create `/mydata/test.txt`, exit, restart it with `docker start -ai`, and confirm the file exists. Then `docker rm` it and confirm it is gone.
3. Reproduce the two-terminal `exec` experiment with two shells sharing a file.
4. Write a Dockerfile that installs `curl` and creates `/readme.txt`; build it as `my-first-image:1.0`; run `curl --version` and `cat /readme.txt` in a container from it.
5. Build the same image twice and observe `CACHED` in the output. Then add a `RUN` line at the *top* (after `FROM`) and rebuild, noticing which steps are re-run.

---

**Next:** [Chapter 15 – Towards the Dockerfile](15_towards_the_dockerfile.md)
