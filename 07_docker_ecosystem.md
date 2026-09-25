# Chapter 7: The Docker Ecosystem

> **In one sentence:** "Docker" is not one program. It is a platform of cooperating parts: the CLI, the Engine, images, registries such as Docker Hub, Docker Desktop, Docker Compose, and more.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~35 minutes

**Prerequisites:** [Chapter 5](05_container_vs_vm_docker_engine.md) and [Chapter 6](06_docker_engine_internals.md).

---

## What you will learn

- Why people say "Docker is a platform", and what its parts are
- **Images**, **layers**, **tags** and **digests**: what they are and how to read an image name
- **Registries** and **Docker Hub**: where images come from
- **Docker Compose**: running several containers as one application
- Other tools around Docker (BuildKit, Swarm, Podman, Kubernetes...)
- Hands-on: pull images, inspect layers, and run a two-service app with Compose

---

## 1. Why "platform" and not "tool"?

A *tool* does one job (`grep` searches text). A *platform* is a set of tools and standards that work together. When someone says "we use Docker", they usually mean several of these:

```
 You ──► docker CLI ──► Docker Engine ──► containers
              │              │
              │              └── pulls/pushes images ──► Registry (Docker Hub)
              │
              └── docker compose ──► many containers as one app

   Docker Desktop = installer + GUI + Linux VM (Windows/macOS) bundling all of it
```

| Part | Job | Chapter |
|---|---|---|
| **Docker CLI** | The `docker` command you type | 6 |
| **Docker Engine** | Daemon + containerd + runc: builds and runs containers | 5, 6 |
| **Images** | Read-only templates for containers | 4, this chapter |
| **Registry / Docker Hub** | Stores and distributes images | this chapter |
| **Docker Desktop** | App for Windows/macOS/Linux bundling the above, plus a GUI | 5 |
| **Docker Compose** | Define and run multi-container apps from one YAML file | this chapter |
| **BuildKit / buildx** | The modern image builder | 20 |
| **Docker Swarm** | Docker's built-in clustering (still exists, but Kubernetes is far more common) | — |

Like any ecosystem, the parts depend on each other, but you can swap many of them: use Podman instead of the Engine, GitHub Container Registry instead of Docker Hub, and so on. That is possible because of **open standards** (the OCI image and runtime specs, the registry API, the Compose Specification).

---

## 2. Images in detail

Recall: an **image** is a read-only template (a root file system plus metadata). Now the details.

### 2.1 Layers

An image is a **stack of layers**. Each layer is a set of file changes (files added, modified, or deleted). Each instruction in a Dockerfile (Chapter 15) creates a layer.

```
┌────────────────────────────┐
│ Layer 4: your app code     │  small, changes often
├────────────────────────────┤
│ Layer 3: pip/npm packages  │
├────────────────────────────┤
│ Layer 2: Python/Node       │
├────────────────────────────┤
│ Layer 1: Debian base files │  big, rarely changes
└────────────────────────────┘
```

Why layers matter:

- **Sharing:** two images built from the same base store that base layer **once** on disk.
- **Faster pulls and pushes:** only layers you don't already have are transferred.
- **Build cache:** unchanged layers are reused when you rebuild.
- **Immutability:** layers are read-only. A running container adds a thin **writable layer** on top.

Every layer is identified by a **SHA-256 hash of its content**, so identical content is stored once and can be verified.

### 2.2 Reading an image name

```
docker.io / library / nginx : 1.27-alpine
    │          │        │         │
 registry  namespace  repository  tag
```

Full form: `[registry/][namespace/]repository[:tag][@digest]`

| You type | Docker understands |
|---|---|
| `nginx` | `docker.io/library/nginx:latest` |
| `nginx:1.27` | `docker.io/library/nginx:1.27` |
| `myuser/myapp:v2` | `docker.io/myuser/myapp:v2` (your own repo on Docker Hub) |
| `ghcr.io/org/app:1.0` | The image on GitHub Container Registry |
| `nginx@sha256:ab12...` | Exactly the image with that content hash |

`library` is the namespace for **Docker Official Images**, curated images such as `nginx`, `postgres`, `python`, `ubuntu`.

### 2.3 Tags are labels, not versions

A **tag** is a movable label pointing at an image. `python:3.12` today may point to a different image tomorrow (a security patch). Key points:

- `latest` is only a **default tag name**. It does **not** automatically mean "newest" or "best"; it is whatever the publisher last tagged as `latest`. If you omit a tag, you get `latest`.
- One image can have several tags (e.g. `16`, `16.4`, `16.4-bookworm` may all point to the same content).
- A **digest** (`@sha256:...`) is a fingerprint and **never changes**. Use it when you need exact reproducibility.

| Common variants in tags | Meaning |
|---|---|
| `alpine` | Built on Alpine Linux: very small, uses musl libc (rare compatibility surprises) |
| `slim` | Debian with fewer packages: smaller than the default |
| `bookworm`, `jammy`... | The specific OS release used as the base |
| `-windowsservercore` | Windows container variants |

**Recommendation:** in development, a major-version tag is fine (`postgres:16`). In production, pin a specific tag, and ideally a digest.

### 2.4 Turning containers into images
`docker commit` can create an image from a container's file changes, but the recommended and repeatable way is to write a **Dockerfile** and run `docker build` (Chapters 15 and 20).

---

## 3. Registries and Docker Hub

A **registry** is a server that stores and serves images (like GitHub for code, or npm for packages). Pushing and pulling use a standard HTTP API defined by the OCI.

| Registry | Notes |
|---|---|
| **Docker Hub** (`docker.io`) | The default. Public and private repositories, Official Images, verified publishers. Anonymous pulls are rate limited, so log in if you pull a lot |
| GitHub Container Registry (`ghcr.io`) | Tied to GitHub repos and Actions |
| Amazon ECR, Google Artifact Registry, Azure ACR | Cloud-provider registries |
| GitLab Registry, Harbor, Nexus, JFrog | Common self-hosted or corporate options |
| `registry:2` | The open-source registry you can run yourself |

Basic workflow:

```bash
docker search nginx                # look for images (Docker Hub)
docker pull nginx:1.27-alpine      # download
docker images                      # list local images
docker login                       # authenticate (needed to push)
docker tag myapp:1.0 myuser/myapp:1.0   # add a name that includes your namespace
docker push myuser/myapp:1.0       # upload
```

### Trust and safety
Anyone can publish to Docker Hub. Prefer **Official Images**, **Verified Publisher** and **Docker-Sponsored Open Source** badges; scan images for vulnerabilities (`docker scout quickview <image>`, or tools like Trivy); and never put secrets into an image, since layers can be read by anyone who has the image.

---

## 4. Docker Compose

### 4.1 The problem
A real application is usually several containers: web server, API, database, cache. Starting each with a long `docker run` command (network, ports, environment variables, volumes) is slow and easy to get wrong.

### 4.2 The solution
**Docker Compose** lets you describe the whole application in one YAML file, `compose.yaml` (the older name `docker-compose.yml` still works), and start it with one command.

Compose gives you:
- one command to build, start, stop and remove the whole stack
- an automatic **private network** in which services reach each other by **service name** (Docker's built-in DNS)
- named **volumes** for persistent data
- the file is version-controlled, so every teammate gets the same setup

### 4.3 Hands-on: a web server plus a database

Create a folder `mystack` with two files.

`compose.yaml`:

```yaml
services:
  web:
    image: nginx:1.27-alpine
    ports:
      - "8080:80"                 # host:container
    volumes:
      - ./html:/usr/share/nginx/html:ro
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: example  # demo only! use secrets for real projects
      POSTGRES_DB: appdb
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  db-data:
```

`html/index.html`:

```html
<h1>Hello from Docker Compose!</h1>
```

Run it:

```bash
docker compose up -d          # create network + volume, start db then web
docker compose ps             # see services and ports
curl http://localhost:8080    # → Hello from Docker Compose!
docker compose logs -f db     # follow one service's logs (Ctrl+C to stop following)
docker compose exec db psql -U postgres -d appdb -c "SELECT version();"
docker compose down           # remove containers and network (keeps the volume)
docker compose down -v        # ...and also delete named volumes (data is lost!)
```

What to notice:

- The default network is created for you. Inside it, `web` could reach the database at hostname **`db`**, port 5432. No IP addresses are needed.
- **Ports:** `"8080:80"` publishes container port 80 on host port 8080. The database has no `ports:`, so it is reachable only from other containers, which is a good security default.
- **Volumes:** `db-data` survives `docker compose down`. It is deleted only with `-v`.
- **`depends_on` with `condition: service_healthy`** waits until the database is *ready*. Plain `depends_on` only controls start *order*, not readiness.

### 4.4 Everyday Compose commands

| Command | Purpose |
|---|---|
| `docker compose up -d` | Create and start in the background |
| `docker compose up -d --build` | Rebuild images defined with `build:` first |
| `docker compose ps` | List services |
| `docker compose logs -f [service]` | Follow logs |
| `docker compose exec <service> sh` | Shell into a running service |
| `docker compose run --rm <service> <cmd>` | One-off command in a new container |
| `docker compose stop` / `start` / `restart` | Control without removing |
| `docker compose pull` | Update images |
| `docker compose down [-v]` | Tear down (optionally with volumes) |
| `docker compose config` | Validate and print the final merged config |

> **Version notes:** `docker compose` (with a space) is Compose v2, a plugin included with modern Docker. The old standalone `docker-compose` (hyphen, Python) reached end of life. The top-level `version:` key seen in old tutorials is obsolete and ignored. `links:` is legacy: use networks and service names instead.

---

## 5. Hands-on: explore images and layers

```bash
docker pull python:3.12-slim
docker images python                 # size and IDs
docker history python:3.12-slim      # the layers and the instruction that made each
docker inspect python:3.12-slim --format '{{.Config.Cmd}} {{.Config.Env}}'
docker image ls --digests            # show sha256 digests
docker system df                     # disk used by images, containers, volumes, cache
```

Try the effect of layer sharing:

```bash
docker pull python:3.12-slim
docker pull python:3.12              # a bigger sibling; compare their sizes and layers
docker system df -v | head -20
```

Cleaning up (be careful, these delete things):

```bash
docker container prune       # remove stopped containers
docker image prune           # remove dangling (untagged) images
docker system prune          # containers + networks + dangling images + build cache
```

---

## 6. The wider ecosystem

| Tool | What it is |
|---|---|
| **Kubernetes** | Orchestrator for running containers across many machines (scheduling, self-healing, scaling). Runs OCI images |
| **Docker Swarm** | Docker's simpler built-in orchestrator; less used today |
| **Podman / Buildah / Skopeo** | Daemonless, rootless-friendly alternatives for running, building and copying images. Mostly command-compatible with Docker |
| **containerd, CRI-O** | Container runtimes used by Kubernetes |
| **BuildKit / buildx** | Modern, fast, cache-aware image builder with multi-platform support |
| **Docker Scout / Trivy / Grype** | Image vulnerability scanners |
| **Portainer, Lazydocker** | UIs for managing containers |
| **Dev Containers** | Use containers as full development environments in VS Code |
| **Testcontainers** | Spin up throwaway containers (databases, queues) inside automated tests |
| **CI/CD (GitHub Actions, GitLab CI, Jenkins)** | Build, test and push images on every commit |

The common thread is the **OCI standard**: an image built with Docker can run under Podman, containerd, Kubernetes, and cloud container services.

---

## 7. Common misconceptions

| Misconception | Reality |
|---|---|
| "Docker Hub *is* Docker" | It is one registry, the default one. You can use others |
| "`latest` is the newest version" | It is just a tag name chosen by the publisher |
| "Images are snapshots of running containers" | They are layered file systems plus metadata, normally built from a Dockerfile |
| "Compose is for production" | Fine for single hosts, demos and development. Large-scale production usually uses Kubernetes or a managed service |
| "`depends_on` waits for the database to be ready" | Only with `condition: service_healthy` and a healthcheck |
| "`docker compose down` deletes my data" | Named volumes survive unless you add `-v` |
| "Docker Desktop and Docker Engine are the same" | Desktop is a bundle (GUI + VM + Engine + Compose); Engine is the core daemon |

---

## 8. Summary

- Docker is a **platform**: CLI, Engine, images, registries, Desktop, Compose, BuildKit, and integrations.
- Images are **stacks of read-only layers** identified by hashes; **tags** are movable labels, **digests** are exact.
- An image name is `registry/namespace/repository:tag`, defaulting to Docker Hub and `latest`.
- **Compose** runs multi-container apps from a single YAML file, with automatic networking by service name and named volumes.
- Standards (OCI) let you mix and swap components.

---

## 9. Check your understanding

1. What does `nginx` expand to as a full image reference?
2. Two images share the same base layer. How many times is that layer stored on disk?
3. Why is pinning an image by digest more reproducible than using a tag?
4. In the Compose example, how does `web` reach the database, and why does the database have no `ports:`?
5. What is the difference between `docker compose down` and `docker compose down -v`?
6. Name two registries other than Docker Hub.

<details>
<summary>Answers</summary>

1. `docker.io/library/nginx:latest`
2. Once. Layers are content-addressed and shared.
3. Tags can be moved to new content; a digest identifies exact content and cannot change.
4. By the hostname `db` on the Compose network. No `ports:` means it isn't published to the host, which reduces exposure.
5. `-v` also removes named volumes, deleting stored data.
6. Any of GHCR, Amazon ECR, Google Artifact Registry, Azure ACR, GitLab, Harbor.
</details>

**Practice:** extend the Compose example with a third service (for example `redis:7-alpine`), run `docker compose up -d`, and use `docker compose exec web ping -c 1 redis` (install ping if missing, or use `getent hosts redis`) to prove that service names resolve.

---

**Next:** [Chapter 8 – Linux](08_linux.md). Containers are Linux, so we now learn enough Linux to feel at home.
