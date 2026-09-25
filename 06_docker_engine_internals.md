# Chapter 6: Docker Engine Internals

> **In one sentence:** When you type `docker run`, the `docker` CLI sends an HTTP request to a background service (`dockerd`), which hands the work down through **containerd** and **runc**, and finally the Linux kernel creates the container.

**Level:** 🟡 Intermediate → 🔴 Expert (explained from the ground up) · **Reading time:** ~40 minutes

**Prerequisites:** [Chapter 4 – Containers](04_container.md) and [Chapter 5 – Container vs VM & Docker Engine](05_container_vs_vm_docker_engine.md).

---

## What you will learn

- What a **daemon** is (and why the word appears everywhere in Linux)
- The chain of components: **docker CLI → dockerd → containerd → shim → runc → kernel**
- Why Docker was split into layers, and what the **OCI** standard is
- How the CLI talks to the daemon (a REST API over a socket), and how you can talk to it yourself with `curl`
- The full journey of `docker run hello-world`, with commands that let you *observe* each step
- Where images and containers live on disk
- Why Kubernetes can run your Docker images without Docker

---

## 1. Two words first: "daemon" and "client-server"

### Daemon
A **daemon** (say "DEE-mun") is a program that runs **in the background**, without a terminal, and waits for requests. Examples: `sshd` (accepts SSH logins), `nginx` (serves web pages), `systemd` (starts services), `dockerd` (Docker's daemon). By convention the names often end in "d".

A normal command like `ls` runs, prints, and exits. A daemon starts (usually at boot), keeps running, and serves other programs. If you close your terminal, the daemon keeps going.

### Client and server
Docker uses a **client-server** design:

- The **server** is the daemon, `dockerd`. It does the work.
- The **client** is the `docker` command. It only *asks*. It runs, sends a request, prints the reply, and exits.

That is why closing a terminal after `docker ps` changes nothing: the client was just a messenger. And if the daemon is not running, `docker` prints "Cannot connect to the Docker daemon".

---

## 2. The big picture

```
┌──────────────┐   HTTP over a Unix socket    ┌────────────────────────────────┐
│ docker CLI   │ ───────────────────────────► │ dockerd  (the Docker daemon)   │
│ (client)     │  /var/run/docker.sock        │ API, images, networks, volumes │
└──────────────┘                              └───────────────┬────────────────┘
                                                              │ gRPC
                                              ┌───────────────▼────────────────┐
                                              │ containerd                     │
                                              │ container lifecycle, snapshots │
                                              └───────────────┬────────────────┘
                                                              │ starts one per container
                                              ┌───────────────▼────────────────┐
                                              │ containerd-shim-runc-v2        │
                                              │ keeps the container's I/O and  │
                                              │ exit status alive              │
                                              └───────────────┬────────────────┘
                                                              │ calls
                                              ┌───────────────▼────────────────┐
                                              │ runc  (OCI runtime)            │
                                              │ namespaces, cgroups, chroot,   │
                                              │ then exec your program         │
                                              └───────────────┬────────────────┘
                                                              │ system calls
                                              ┌───────────────▼────────────────┐
                                              │ Linux kernel                   │
                                              └────────────────────────────────┘
```

Each layer has one job. Docker calls the `dockerd + containerd + runc` group the **Docker Engine**.

---

## 3. Meet the components

### 3.1 `docker` CLI (the client)
- Parses your command line (`docker run -p 8080:80 nginx`).
- Turns it into HTTP requests to the daemon's **REST API**.
- Prints the result. It never creates a container itself.
- It is a separate program from the daemon. They can even be on different machines (`docker -H ssh://user@server ps`).

### 3.2 `dockerd` (the Docker daemon)
The "manager" and the friendly front door. It:
- Serves the Docker API (by default on the Unix socket `/var/run/docker.sock`).
- Builds images (through BuildKit) and stores them.
- Manages **networks** (bridges, port mappings, DNS between containers) and **volumes**.
- Handles registry logins and pulls/pushes images.
- Asks containerd to create and run containers.

It runs as **root**, which is why access to the Docker socket is effectively root access to the machine (see the security note in section 9).

### 3.3 `containerd` (the "high-level runtime")
A separate open-source project (donated to the CNCF; it "graduated", meaning mature). It:
- Manages the **lifecycle** of containers (create, start, stop, delete).
- Manages **snapshots** (the layered file systems) and, in newer setups, the image content store.
- Exposes a gRPC API. Docker uses it, and so does Kubernetes.

"High-level" means it does not create containers by itself; it decides *what* must happen and delegates *how* to a low-level runtime.

### 3.4 The shim (`containerd-shim-runc-v2`)
One small process **per container**. It:
- Stays alive as the container's parent process, so the container is not tied to containerd's or dockerd's life. This is why you can restart the daemon and (with `live-restore` enabled) containers keep running.
- Holds the container's stdin/stdout/stderr and reports its exit code.

### 3.5 `runc` (the "low-level runtime")
A small command-line tool that implements the **OCI Runtime Specification**. Given a *bundle* (a root file system folder plus a `config.json`), it:
1. Creates the **namespaces** (PID, NET, MNT, UTS, IPC, ...).
2. Puts the process in a **cgroup** and applies limits.
3. Sets up the root file system (`pivot_root`), mounts, capabilities, seccomp filter.
4. `exec`s your program as the container's PID 1.

Then **runc exits**. It does not stay around to supervise (that's the shim's job).

### Who does what? (a common source of confusion)

| Task | Who |
|---|---|
| Parse `docker run ...` | CLI |
| Pull and store images (classic Docker) | dockerd |
| Networking (bridge, port publishing) | dockerd (using kernel features such as iptables/nftables) |
| Container lifecycle | containerd |
| Actually creating namespaces/cgroups | runc |
| Keep the container running and relay output | shim |

> **Version note:** newer Docker Engine releases can use containerd's *image store* instead of Docker's own storage (this is the default on fresh installs of recent versions). In that mode containerd also pulls and stores images. The overall chain stays the same, but the answer to "who stores images?" depends on your version. Run `docker info | grep -i 'storage driver\|driver-type'` to see which one you have.

---

## 4. Why split Docker into layers?

Early Docker (2013) was one big program that called kernel features itself. Over the years it was split up:

| Reason | Explanation |
|---|---|
| **Stability** | The daemon can be restarted or upgraded without killing every container (thanks to shims) |
| **Reuse** | Kubernetes only needs "run this container", so it talks to containerd (or CRI-O) directly and skips dockerd |
| **Standards** | The **Open Container Initiative (OCI)** defines an *image spec* and a *runtime spec*. Any OCI runtime (runc, crun, gVisor's runsc, Kata) can run any OCI image |
| **Replaceable parts** | Need stronger isolation? Swap runc for gVisor or Kata without changing your images or `docker` commands (`docker run --runtime=...`) |
| **Security** | Small components have small attack surfaces |

---

## 5. How the CLI talks to the daemon

The Docker API is plain HTTP + JSON. On Linux it is served on a **Unix domain socket** (a special file that works like a local network port). You can call it without the `docker` command:

```bash
curl --unix-socket /var/run/docker.sock http://localhost/version
```

(You might need `sudo`, or to be in the `docker` group.) You will see JSON such as `{"Platform":{"Name":"Docker Engine - Community"},"Version":"27.x.x","ApiVersion":"1.4x", ...}`.

More examples:

```bash
curl --unix-socket /var/run/docker.sock http://localhost/containers/json       # like `docker ps`
curl --unix-socket /var/run/docker.sock http://localhost/images/json           # like `docker images`
```

So `docker ps` is essentially `GET /containers/json`, and `docker run` uses several calls:

| Step | API call |
|---|---|
| Pull image if missing | `POST /images/create?fromImage=hello-world&tag=latest` |
| Create the container | `POST /containers/create` |
| Start it | `POST /containers/{id}/start` |
| Attach to output (when you don't use `-d`) | `POST /containers/{id}/attach` |
| Wait for exit | `POST /containers/{id}/wait` |

(A single `docker run` is a CLI convenience that chains these.)

The client picks where to connect from the `DOCKER_HOST` environment variable, the current *context* (`docker context ls`), or the default socket.

---

## 6. The journey of `docker run hello-world`

```bash
docker run hello-world
```

1. **CLI** reads the command and calls the API on `/var/run/docker.sock`.
2. **dockerd** checks its local images. `hello-world:latest` is missing, so it **pulls** it: it contacts Docker Hub (a *registry*), downloads the image manifest, then each layer, and verifies the hashes. You see `Unable to find image 'hello-world:latest' locally` and `Pull complete`.
3. **dockerd** creates the container's metadata: an ID, a writable layer, network settings, its config.
4. **dockerd** asks **containerd** (over gRPC) to create and start a container with that configuration.
5. **containerd** prepares the root file system from the image's layers (an *overlay* mount) and starts a **shim**.
6. The shim runs **runc** with an OCI bundle.
7. **runc** asks the **kernel** to create namespaces and cgroups, mounts the file system, drops privileges, and `exec`s `/hello`.
8. `/hello` prints its message to stdout. The shim captures it and streams it back up to dockerd and the CLI, which shows it on your terminal.
9. `/hello` ends, the shim reports the exit code, and the container becomes **Exited**. The CLI exits with that code.

The second time you run it, step 2 is skipped because the image is cached. That is why the second run feels much faster.

### Watch it happen

**a) Docker events (a live log of what dockerd does):** open a second terminal and run

```bash
docker events
```

then in the first terminal run `docker run --rm hello-world`. You will see events such as `image pull`, `container create`, `container start`, `container die`, `container destroy`.

**b) The process tree (Linux):**

```bash
docker run -d --name web nginx
pstree -sp $(pgrep -o -f 'nginx: master')
```

You should see a chain like `systemd → containerd-shim → nginx → nginx`. (`dockerd` and `containerd` are siblings under systemd, not ancestors: the shim is the parent of the container's process.) Clean up: `docker rm -f web`.

**c) Which parts are running?**

```bash
ps -eo pid,ppid,user,cmd | grep -E 'dockerd|containerd|shim' | grep -v grep
systemctl status docker containerd --no-pager
```

**d) Versions of each component:**

```bash
docker version                      # client + engine versions
docker info | grep -iE 'runtime|containerd|runc'
containerd --version
runc --version
```

**e) Talk to containerd directly (expert):**

```bash
sudo ctr namespaces ls              # Docker uses the "moby" namespace
sudo ctr -n moby containers ls
```

---

## 7. Where things live on disk (Linux)

| Path | Contents |
|---|---|
| `/var/run/docker.sock` | The API socket |
| `/var/lib/docker/` | Docker's data root: images, layers, volumes, container metadata |
| `/var/lib/docker/overlay2/` | Image and container layers (the default `overlay2` storage driver) |
| `/var/lib/docker/volumes/` | Named volumes |
| `/var/lib/docker/containers/<id>/` | A container's config and log file |
| `/etc/docker/daemon.json` | Optional daemon settings (log driver, registry mirrors, `live-restore`...) |

On Docker Desktop these live *inside the hidden Linux VM*, not on your Mac/Windows disk.

Useful checks:

```bash
docker system df          # how much space images, containers, volumes use
docker info | grep -i 'root dir\|storage driver'
```

---

## 8. Kubernetes and Docker

Kubernetes talks to a **CRI** (Container Runtime Interface) implementation such as **containerd** or **CRI-O**. It used to support Docker Engine through a compatibility layer (*dockershim*), which was removed in Kubernetes 1.24. Effects:

- Your **images still work**, because they follow the OCI standard.
- On the cluster, `docker ps` shows nothing (there is no dockerd); use `crictl ps` instead.
- You still use Docker on your laptop and CI to *build* images.

---

## 9. Security and operations notes (expert)

- **The Docker socket is powerful.** Anyone who can talk to `/var/run/docker.sock` can start a privileged container that mounts the host's `/`, which is effectively root. Adding a user to the `docker` group grants that power. Never mount the socket into an untrusted container.
- **Do not expose the API on plain TCP.** If you must, use TLS with client certificates (`dockerd --tlsverify`), or use SSH (`docker -H ssh://...`).
- **Rootless mode:** dockerd (and containers) can run as a non-root user, using user namespaces, for a much smaller blast radius.
- **`live-restore`:** set `{"live-restore": true}` in `/etc/docker/daemon.json` to keep containers running while the daemon restarts.
- **Logs:** daemon logs: `journalctl -u docker`. Container logs: `docker logs <name>`.

---

## 10. Common misconceptions

| Misconception | Reality |
|---|---|
| "`docker` creates containers" | The CLI only sends requests. The daemon, containerd and runc do the work |
| "dockerd talks to the kernel to make containers" | It asks containerd, which uses runc. (dockerd does set up parts like network rules itself.) |
| "containerd belongs to Docker" | It is an independent CNCF project also used by Kubernetes |
| "runc is a long-running daemon" | It runs briefly to set up the container, then exits |
| "A daemon is any server" | A daemon runs in the background and has no controlling terminal |
| "If dockerd crashes, all containers die" | Containers are kept alive by shims; they usually survive (especially with `live-restore`) |
| "Docker needs Docker Hub" | The registry is configurable; Hub is just the default |

---

## 11. Summary

- `docker` (client) → **dockerd** (API, images, networks, volumes) → **containerd** (lifecycle, snapshots) → **shim** → **runc** (namespaces, cgroups) → **kernel**.
- The client and daemon talk with HTTP + JSON over `/var/run/docker.sock`.
- Splitting the Engine into layers gave stability, reuse (Kubernetes), and standards (OCI).
- First `docker run` pulls the image; later runs use the local cache.
- You can *observe* every layer with `docker events`, `pstree`, `curl --unix-socket` and `ctr`.

---

## 12. Check your understanding

1. What is a daemon? Give two examples besides `dockerd`.
2. Which component actually asks the kernel for namespaces and cgroups?
3. What does the shim do that runc doesn't?
4. Write the `curl` command that lists running containers without using the `docker` CLI.
5. Why can Kubernetes run an image you built with Docker even though it doesn't use dockerd?
6. Why is adding a user to the `docker` group a security decision?

<details>
<summary>Answers</summary>

1. A background program that waits for requests, e.g. `sshd`, `nginx`, `systemd`.
2. runc (invoked by containerd's shim).
3. It stays alive as the container's parent, holds its I/O, and reports its exit status, so the container outlives runc and the daemon restarts.
4. `curl --unix-socket /var/run/docker.sock http://localhost/containers/json`
5. Images follow the OCI standard, and Kubernetes uses containerd/CRI-O, which can run any OCI image.
6. Access to the Docker socket is effectively root on the host.
</details>

**Practice:** run `docker events` in one terminal, then run `docker run --rm alpine echo hi` in another. Write down the order of events you see, and match each to a step in section 6.

---

**Next:** [Chapter 7 – The Docker Ecosystem](07_docker_ecosystem.md)
