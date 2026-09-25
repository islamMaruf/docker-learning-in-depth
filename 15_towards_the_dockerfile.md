# Chapter 15: Towards the Dockerfile

> **In one sentence:** Instead of typing setup commands by hand inside a container and saving the result, you write them once in a text file (a **Dockerfile**) so that anyone can rebuild the exact same image with a single command.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~45 minutes

**Prerequisites:** [Chapter 14](14_docker_hands_on.md) (images, containers, `exec`, a first `docker build`) and [Chapter 11](11_managing_packages_on_linux.md) (`apt-get`).

---

## What you will learn

- Why the manual "set up a container, then save it" approach is a dead end
- What a Dockerfile is, and the four core instructions: `FROM`, `RUN`, `WORKDIR`, `COPY`
- What the **build context** is and why `.dockerignore` matters
- How **layers** and the **build cache** make rebuilds fast (and how to order instructions to exploit that)
- How to tag and version images
- Containerizing a small Go web server, step by step, then running and reaching it from your browser
- Common mistakes, including one bug that appears in many tutorials (`COPY . ./server.go`)

---

## 1. The manual way, and why it fails

Suppose you want an image with Go installed and your `server.go` inside. By hand:

```bash
docker run -it --name build ubuntu:24.04 bash
  apt-get update && apt-get install -y golang     # inside the container
  mkdir /app
  exit
docker cp server.go build:/app/server.go          # copy the file in
docker commit build go-server                     # save the container as an image
```

`docker commit` really does turn a container's changes into an image, but this approach has serious drawbacks:

| Problem | Why it hurts |
|---|---|
| **Not reproducible** | Which package versions did you get? What did you type exactly? Next month the result may differ |
| **Not documented** | The "recipe" only exists in your shell history and your memory |
| **Not shareable** | Teammates need a 12-step document, and they will miss a step |
| **Not reviewable** | Nothing to keep in Git, no diffs, no code review |
| **Hard to automate** | CI/CD pipelines can't drive an interactive shell |
| **Bloated** | Everything you did (caches, temp files, shell history) ends up in the image |

The fix is to describe the image as **code**.

---

## 2. What is a Dockerfile?

A **Dockerfile** is a plain text file with a list of instructions that `docker build` follows, top to bottom, to produce an image. Think of it as a recipe:

```
Dockerfile  ──docker build──►  Image  ──docker run──►  Container
 (recipe)                    (packaged result)         (running)
```

Benefits: reproducible, versionable in Git, shareable, reviewable, automatable, and self-documenting.

Basics:

- The conventional file name is **`Dockerfile`** (capital D, no extension). Docker looks for that name by default. Use `docker build -f my.Dockerfile .` for another name.
- One **instruction** per line: `INSTRUCTION arguments`. Instructions are conventionally UPPERCASE (not required).
- Lines starting with `#` are comments. A backslash `\` at the end of a line continues it on the next.
- The first real instruction is `FROM` (only `ARG` and comments may precede it).

---

## 3. The four core instructions

### `FROM`: choose the starting point

```dockerfile
FROM ubuntu:24.04
```

Every image starts from another image (or from `scratch`, which is empty). Pick a specific tag; avoid `latest`.

### `RUN`: execute a command **at build time**

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends golang-go
```

Docker starts a temporary container from the current state, runs the command in it, and saves the file-system changes as a new **layer**. `RUN` happens while *building*, not when a container starts.

Two forms:

| Form | Example | Notes |
|---|---|---|
| **Shell form** | `RUN apt-get update && echo done` | Runs via `/bin/sh -c`; allows `&&`, pipes, variables |
| **Exec form** | `RUN ["apt-get", "update"]` | No shell; no `&&` or variable expansion |

### `WORKDIR`: set the working directory

```dockerfile
WORKDIR /app
```

Creates the directory if it doesn't exist, and makes it the current directory for all following instructions **and** for the container when it starts. Prefer it to `RUN cd /app`, because each `RUN` is a separate shell and its `cd` is forgotten when it ends:

```dockerfile
# WRONG: the cd only lasts for that one RUN
RUN mkdir /app
RUN cd /app
COPY server.go .        # this lands in "/" , not /app
```

```dockerfile
# RIGHT
WORKDIR /app
COPY server.go .        # lands in /app/server.go
```

A relative `WORKDIR` is resolved against the previous one (`WORKDIR /app` then `WORKDIR data` → `/app/data`), but absolute paths are clearer. (Chapter 17 goes deeper.)

### `COPY`: put files from your machine into the image

```dockerfile
COPY <source in build context> <destination in image>
```

```dockerfile
WORKDIR /app
COPY server.go ./            # → /app/server.go   (relative destination = relative to WORKDIR)
COPY server.go /app/         # same result; trailing slash means "this is a directory"
COPY *.go ./                 # wildcards work
COPY config/ ./config/       # a directory's *contents* are copied into the destination
COPY . .                     # everything in the build context → WORKDIR
```

> ⚠️ **A frequently seen mistake:** `COPY . ./server.go` (seen in many tutorials). It does **not** copy `server.go`. It copies *everything in the context* and tries to put it at `./server.go`, which, with multiple files, means "a directory named `server.go`". You end up with `/app/server.go/server.go`, and `go run server.go` fails. To copy one file: `COPY server.go ./`. To copy everything: `COPY . .`.

Rules of thumb: sources can't reach **outside** the build context (`COPY ../x` fails); `COPY` copies files as **root** by default (use `COPY --chown=user:group ...`). Similar `ADD` also extracts local tar archives and fetches URLs, which makes it surprising, so prefer `COPY` (Chapter 20).

---

## 4. The build context and `.dockerignore`

`docker build -t name:tag .` : the final `.` is the **build context**, the directory whose contents are sent to the builder. Only files inside it can be used by `COPY`.

- A huge context (for example one containing `node_modules`, `.git`, or big data files) slows every build and can leak secrets into the image.
- Exclude files with a **`.dockerignore`** file next to the Dockerfile (same idea as `.gitignore`):

```
.git
node_modules
*.log
tmp/
.env
Dockerfile
.dockerignore
```

Pattern: build context = what the Dockerfile may see; `.dockerignore` = what you *hide* from it.

---

## 5. Worked example: a Go web server

### 5.1 The application

Create a folder `go-server/` with `server.go`:

```go
package main

import (
	"fmt"
	"net/http"
)

func hello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Hello World\n")
}

func main() {
	http.HandleFunc("/", hello)
	fmt.Println("Server listening on port 8080...")
	if err := http.ListenAndServe(":8080", nil); err != nil {
		fmt.Println(err)
	}
}
```

It answers every request on port **8080** with "Hello World". Note `:8080` means "all network interfaces", which matters in a container (if it listened only on `127.0.0.1`, connections from outside the container would fail).

### 5.2 The Dockerfile

`go-server/Dockerfile`:

```dockerfile
FROM ubuntu:24.04

RUN apt-get update \
 && apt-get install -y --no-install-recommends golang-go \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY server.go ./

CMD ["go", "run", "server.go"]
```

Line by line:

| Line | What it does |
|---|---|
| `FROM ubuntu:24.04` | Start from Ubuntu 24.04 |
| `RUN apt-get update && apt-get install ...` | Install Go's compiler, in **one** layer with cleanup (Chapter 11) |
| `WORKDIR /app` | Create and enter `/app` |
| `COPY server.go ./` | Copy the source into `/app/server.go` |
| `CMD [...]` | The **default command** when a container starts: run the server. (`CMD` is Chapter 16.) |

> **Production note:** installing a whole compiler just to run a small program is wasteful. In real projects you build a static binary in a *build stage* and copy only that into a tiny final image (multi-stage builds, Chapter 20). For learning, the simple version keeps focus on the core instructions.

### 5.3 Build

```bash
cd go-server
docker build -t go-server:1.0.0 .
```

```
[+] Building 40.2s (9/9) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load metadata for docker.io/library/ubuntu:24.04
 => [internal] load .dockerignore
 => [1/4] FROM docker.io/library/ubuntu:24.04
 => [2/4] RUN apt-get update && apt-get install ...
 => [3/4] WORKDIR /app
 => [4/4] COPY server.go ./
 => exporting to image
 => => naming to docker.io/library/go-server:1.0.0
```

Reading it: `[n/4]` are your four build steps; `[internal]` steps are BuildKit housekeeping. Check the result: `docker images go-server`.

### 5.4 Run it and reach it

```bash
docker run --rm -p 8080:8080 go-server:1.0.0
```

- The default `CMD` starts the server, so **no** `bash` and no manual steps are needed.
- **`-p 8080:8080`** *publishes* a port: `host_port:container_port`. Without it, the server is running but reachable **only inside the container's own network**. (Full detail in Chapter 18; here it is just the switch that makes it visible.)

In another terminal (or a browser at `http://localhost:8080`):

```bash
curl http://localhost:8080
# Hello World
```

Stop with **Ctrl+C**.

**Explore the image you built:**

```bash
docker run --rm -it go-server:1.0.0 bash    # override the CMD: an interactive shell instead
pwd                                         # /app     ← from WORKDIR
ls                                          # server.go
go version                                  # go version go1.22.x linux/amd64
exit
```

---

## 6. Layers and the build cache

Each `RUN`, `COPY` (and `ADD`) instruction creates a **layer**:

```
Layer 5:  COPY server.go ./         ← changes often
Layer 4:  WORKDIR /app
Layer 3:  RUN apt-get ... golang    ← slow, changes rarely
Layer 2:  (Ubuntu base layer)
```

**The cache rule:** Docker reuses a layer from a previous build if the instruction is identical **and all earlier layers are unchanged** (for `COPY`, it also compares the copied files' contents). The first changed instruction and *everything after it* are rebuilt.

Try it:

```bash
docker build -t go-server:1.0.1 .          # everything shows CACHED: seconds
# edit server.go (change "Hello World" to "Hello Docker")
docker build -t go-server:1.0.2 .          # only the COPY step onward is re-run
```

### Order instructions from *least* to *most* frequently changing

Good (source last):

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y --no-install-recommends golang-go && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY server.go ./
```

Bad (source first): every code change invalidates the cache and reinstalls Go:

```dockerfile
FROM ubuntu:24.04
COPY server.go /app/
RUN apt-get update && apt-get install -y golang-go
```

Other cache facts:

- `docker build --no-cache .` ignores the cache (use it when you want fresh package versions).
- `RUN apt-get update` alone in its own layer can go **stale**: keep `update` and `install` in the same `RUN`.
- Fewer layers is not the goal; **cache-friendly ordering and small layers** are.

---

## 7. Tags and versions

```bash
docker build -t myapp:1.0.0 .        # name:tag
docker build -t myapp .              # tag defaults to "latest"
docker tag myapp:1.0.0 myapp:stable  # add a second name to the same image (no copy)
docker build -t myapp:1.0.0 -t myapp:latest .
```

- **`latest` is only a default label**, not "the newest": it moves only when *you* tag something as `latest`. Building `2.0.0` does not update `latest` by itself.
- Common schemes: **semantic versions** (`1.4.2`), **git commit** (`abc123d`), **environment** (`dev`, `staging`), or a combination.
- Treat a published tag as **immutable**: don't reuse `1.0.0` for different content.
- Several tags can point to one image ID (`docker images` shows the same ID twice).

---

## 8. `docker commit`: when is it acceptable?

`docker commit <container> <image>` snapshots a container's file changes into an image. It is fine for **quick experiments and forensic debugging** ("save the state of this broken container"). Do **not** use it to produce images you ship: nobody can see how they were made. Keep the recipe in a Dockerfile.

---

## 9. Manual vs Dockerfile

| | Manual + `commit` | Dockerfile |
|---|---|---|
| Reproducible | No | Yes |
| Documented | No | The file *is* the documentation |
| Version-controlled | No | Yes (Git) |
| Shareable | A long document | One file |
| Automated (CI/CD) | Not really | Yes |
| Rebuild after a change | Repeat all steps | `docker build` (cache makes it fast) |
| Cache/layer reuse | No | Yes |

---

## 10. Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `failed to compute cache key: "/server.go": not found` (or `COPY failed: file not found in build context`) | File isn't inside the build context, is excluded by `.dockerignore`, or the path is wrong | Put the file next to the Dockerfile; check `.dockerignore`; run from the right folder |
| `go run server.go` → `no required module provides package` / `server.go: no such file` | Files landed in the wrong place (see the `COPY . ./server.go` warning) | `docker run --rm -it IMG bash`, then `ls -R` to see what is where |
| Build is slow at "transferring context" | Big context | Add `.dockerignore` |
| Changes not reflected | Cached layer, or you're running an old image | `docker build --no-cache`; check `docker images`; rebuild and re-run with the new tag |
| `E: Unable to locate package golang-go` | Missing `apt-get update` in the same `RUN` | Combine them |
| `curl: (7) Failed to connect` / `connection reset` from the host | Port not published (`-p`), or the app listens on `127.0.0.1` inside the container | `docker run -p 8080:8080 ...`; bind to `0.0.0.0` / `:8080` |
| `port is already allocated` | Something on the host already uses 8080 | Use another host port: `-p 9090:8080` |
| `pull access denied` on `FROM` | Image name typo, or private image | Check the name; `docker login` |

---

## 11. Best practices so far

1. **Pin the base image** (`ubuntu:24.04`), not `latest`.
2. **Order for the cache**: rarely-changing steps first, your source last.
3. **`apt-get update && apt-get install -y --no-install-recommends ... && rm -rf /var/lib/apt/lists/*`** in one `RUN`.
4. Use **`WORKDIR`**, not `RUN cd`.
5. Be precise with **`COPY`** (`COPY server.go ./`), and use a **`.dockerignore`**.
6. **No secrets** in the Dockerfile or the context; anything you `COPY` or `RUN` is in the image forever (in a layer).
7. Give images **meaningful names and immutable version tags**.
8. Start from an **official language image** where one exists (for example `golang:1.22`), since it is smaller, maintained, and you skip the install step. (Compare: `FROM golang:1.22` replaces the whole `RUN apt-get` block.)
9. Later chapters add: `CMD`/`ENTRYPOINT`, `ENV`, `EXPOSE`, `USER` (non-root), volumes, health checks, multi-stage builds.

---

## 12. Summary

- Manual container setup is slow, unrepeatable and unshareable. A **Dockerfile** turns it into versioned code.
- Core instructions: **`FROM`** (base), **`RUN`** (build-time command → new layer), **`WORKDIR`** (directory), **`COPY`** (files from the build context).
- The **build context** limits what `COPY` can see; **`.dockerignore`** trims it.
- **Layers are cached**; order instructions from stable to volatile.
- Tags label images; `latest` is just a name.
- `-p host:container` publishes a port so you can reach a containerized server.

---

## 13. Check your understanding

1. Give three problems with creating an image via `docker commit`.
2. Why does `RUN cd /app` not affect the next instruction?
3. What does `COPY . ./server.go` really do, and how would you copy only `server.go` to `/app`?
4. You change one line of `server.go` and rebuild. Which layers are rebuilt in the example Dockerfile, and why?
5. What is the build context and how do you keep it small?
6. Does building `myapp:2.0.0` update `myapp:latest`?

<details>
<summary>Answers</summary>

1. Not reproducible, not documented/reviewable, can't be automated, includes leftover junk. (Any three.)
2. Each `RUN` runs in a separate shell; the working directory change lasts only within that instruction. Use `WORKDIR`.
3. It copies the whole build context to a path named `server.go` (a directory when several files are involved), which is wrong. Use `COPY server.go /app/` or `COPY server.go ./` after `WORKDIR /app`.
4. Only `COPY server.go ./` (and any later steps), since earlier layers are unchanged and cached.
5. The directory sent to the builder (`.` in `docker build .`). Keep it small with `.dockerignore` and by building in a focused folder.
6. No. `latest` only changes if you tag it explicitly.
</details>

**Practice**

1. Build the Go server image and run it with `-p 8080:8080`; confirm `curl localhost:8080` prints `Hello World`.
2. Change the message, rebuild as `1.0.1`, and watch which steps say `CACHED`.
3. Swap the instruction order so `COPY` comes before the `apt-get` step, change the source, rebuild, and notice what got slower.
4. Add a `.dockerignore` that excludes `*.md` and a `notes.md` file; prove with `docker run --rm IMG ls /app` that it is absent even with `COPY . .`.
5. Rewrite the Dockerfile using `FROM golang:1.22` (no `apt-get`) and compare the two images' sizes with `docker images`.

---

**Next:** [Chapter 16 – `CMD` Deep Dive](16_cmd_deep_dive.md): how a container decides what to run.
