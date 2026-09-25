# Chapter 20: How Docker Builds Images: Layers, Cache and Multi-Stage Builds

> **In one sentence:** An image is a stack of read-only **layers**; `docker build` produces one layer per file-changing instruction, **caches** each layer, and reuses it as long as nothing above or in that step has changed. If you understand that, you can write Dockerfiles that build in seconds and ship small, safe images.

**Level:** 🟡 Intermediate → 🔴 Expert · **Reading time:** ~55 minutes

**Prerequisites:** [Chapter 15](15_towards_the_dockerfile.md) (Dockerfile basics, build context), [Chapters 16–17](16_cmd_deep_dive.md) (`CMD`, `WORKDIR`).

---

## What you will learn

- What a **layer** is, and how instructions map to layers (and which instructions don't create file layers)
- What really happens during a build (legacy builder vs **BuildKit**)
- The **cache**: exactly when a step is reused and when it is invalidated (and the cascade)
- **Ordering** rules that turn 5-minute rebuilds into 5-second ones
- Why deleting files in a later layer doesn't shrink an image
- **`.dockerignore`**, `ADD` vs `COPY`, `ARG` vs `ENV`
- **Multi-stage builds**: the biggest lever for small, secure images (with runnable Go and Node examples)
- BuildKit features: cache mounts, secret mounts, `buildx`, multi-platform images
- Inspecting layers (`docker history`, `dive`) and troubleshooting cache problems

---

## 1. What is a layer?

A **layer** is a set of **file-system changes** (files added, modified or deleted) relative to the layer below, stored as a compressed archive and identified by a SHA-256 hash of its content. An image is an ordered stack of layers plus **metadata** (default command, environment, working directory, exposed ports, ...).

```
┌──────────────────────────────┐  ← top      (COPY server.go .)          1 KB
├──────────────────────────────┤
│ RUN apt-get install golang   │                                        ~500 MB
├──────────────────────────────┤
│ base image layers (ubuntu)   │  ← bottom                              ~78 MB
└──────────────────────────────┘
      + metadata: CMD, WORKDIR, ENV, EXPOSE, USER ...
```

At run time the layers are combined into one view by a **union file system** (usually `overlay2`), with a thin writable layer on top for the container (Chapter 4).

### Which instructions create layers?

| Instruction | Effect |
|---|---|
| `FROM` | Brings in the base image's layers |
| `RUN` | Runs a command; the resulting **file changes** become a layer |
| `COPY`, `ADD` | Adds files → a layer |
| `WORKDIR` | Sets metadata (and creates the directory if missing, a tiny layer) |
| `ENV`, `ARG`, `LABEL`, `EXPOSE`, `USER`, `CMD`, `ENTRYPOINT`, `HEALTHCHECK`, `VOLUME`, `STOPSIGNAL`, `SHELL` | **Metadata only.** They show as `0B` steps in `docker history` |

You can list them: `docker history IMAGE`.

```
IMAGE          CREATED BY                                       SIZE
a1b2c3d4e5f6   CMD ["go" "run" "server.go"]                     0B     ← metadata
b2c3d4e5f6a7   COPY server.go . # buildkit                      1.2kB
c3d4e5f6a7b8   WORKDIR /app                                     0B
d4e5f6a7b8c9   RUN apt-get update && apt-get install ...        450MB
<missing>      ... (ubuntu base layers)                         78MB
```

(`<missing>` for base layers is normal; it only means those layers came from a pulled image.)

---

## 2. What happens during `docker build`

```bash
docker build -t go-server:1.0.0 .
```

1. **The CLI sends the build context** (the `.` directory, minus `.dockerignore`d files) to the builder.
2. **The builder reads the Dockerfile** and turns it into a graph of steps.
3. For each step it first asks: *"do I already have the result of exactly this step in my cache?"* If yes, it reuses it (`CACHED`).
4. If not, it **executes** the step: for a `RUN`, in a temporary sandbox based on the previous result; for a `COPY`, by adding files. The file-system changes are **committed as a new layer**.
5. At the end, layers + metadata are assembled into the **image**, tagged with your `-t` names.

### The builders
- The **classic builder** literally created a temporary container per instruction, committed it as an intermediate *image*, and removed the container. Old tutorials (and the output `---> Running in abc123`, `Removing intermediate container`) describe this.
- **BuildKit** (the default since Docker 23 and in Docker Desktop) builds a dependency graph, runs **independent steps in parallel**, skips stages that aren't needed, has a much smarter cache, and supports secrets, SSH forwarding, cache mounts and multi-platform builds. Its output looks like `[2/4] RUN ...`, `CACHED`, `=> exporting to image`.

The mental model "each instruction starts from the previous result, does its work, and saves the delta" holds for both.

```bash
docker build --progress=plain -t app .   # full, un-collapsed output (best for debugging)
docker build --no-cache -t app .         # ignore the cache
docker build --pull -t app .             # always re-check for a newer base image
```

---

## 3. The layer cache

### The rule
For each instruction, the builder computes a **cache key**. It reuses the cached layer if **all** of these hold:

1. **Everything before it** was also a cache hit (same parent).
2. The **instruction text** is identical.
3. For `COPY`/`ADD`: the **contents (and relevant metadata) of the files** being copied are identical (a checksum, not timestamps).
4. For `RUN`: only the **command string** is compared, **not** what it downloads or whether the outside world changed. (`RUN apt-get update` is "unchanged" forever, even if new packages exist; use `--no-cache` or change something to refresh.)
5. Relevant `ARG`/`ENV` values used by the step are unchanged.

**Once a step misses the cache, every later step misses too** (the cascade), even if the later instructions are unchanged, because their parent changed.

### Worked example

```dockerfile
FROM ubuntu:24.04                                   # (1)
RUN apt-get update && apt-get install -y golang-go  # (2)  slow
WORKDIR /app                                        # (3)
COPY server.go .                                    # (4)
CMD ["go", "run", "server.go"]                      # (5)
```

| Change you make | Result |
|---|---|
| Nothing | All `CACHED`; build takes ~1 s |
| Edit `server.go` | (1)–(3) cached; **(4) rebuilt**; (5) is metadata |
| Change `WORKDIR /app` to `/srv` | (1)–(2) cached; **(3) and (4) rebuilt** |
| Add a package to the `RUN` in (2) | (1) cached; **(2), (3), (4) rebuilt**; the slow step runs again |
| Base image `ubuntu:24.04` is updated upstream | Cached with the old one until you `--pull` (or the tag's digest changes and you pull) |

```
Step:      1       2        3        4        5
        [cached][cached][cached][ MISS ][ meta ]      ← editing server.go
        [cached][ MISS ][ MISS ][ MISS ][ meta ]      ← changing the install line
```

Try it and watch the `CACHED` markers appear and disappear.

---

## 4. Ordering: the single most useful habit

**Put what changes least at the top, what changes most at the bottom.**

**Bad** (a code change reinstalls all dependencies every time):

```dockerfile
FROM node:22
WORKDIR /app
COPY . .                 # source changes on every edit → cache miss here...
RUN npm ci               # ...so this slow step re-runs every time
CMD ["node", "server.js"]
```

**Good** (dependencies only reinstall when the manifests change):

```dockerfile
FROM node:22
WORKDIR /app
COPY package.json package-lock.json ./   # changes rarely
RUN npm ci                               # cached until the manifests change
COPY . .                                 # changes often
CMD ["node", "server.js"]
```

The same pattern works everywhere:

| Language | Copy first | Then |
|---|---|---|
| Node | `package.json`, `package-lock.json` | `npm ci` |
| Python | `requirements.txt` (or `pyproject.toml`, lock file) | `pip install -r requirements.txt` |
| Go | `go.mod`, `go.sum` | `go mod download` |
| Java (Maven) | `pom.xml` | `mvn dependency:go-offline` |
| Ruby | `Gemfile`, `Gemfile.lock` | `bundle install` |

Other ordering tips:

- Don't put **volatile** things early: `RUN echo "built $(date)" > /info` at the top invalidates everything below it every build. Put anything time-stamped (or `ARG GIT_SHA`) near the end.
- Install **system packages** before **application dependencies** before **your code**.

---

## 5. Layers are additive: deleting later doesn't shrink

Each layer records changes; earlier layers are immutable and still ship in the image.

```dockerfile
RUN apt-get update                     # layer: +50 MB of package lists
RUN apt-get install -y build-essential # layer: +200 MB
RUN rm -rf /var/lib/apt/lists/*        # layer: "deletions" (0 MB) – but the 50 MB stays underneath!
```

Clean up in the **same** `RUN` that created the mess:

```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends build-essential \
 && rm -rf /var/lib/apt/lists/*        # nothing left to store from the lists
```

The same applies to: downloading an archive and deleting it later (`RUN curl ... && tar ... && rm archive` in **one** `RUN`), and to secrets: a secret copied in one layer and "deleted" in the next **is still in the image**. Use BuildKit secret mounts (section 9) instead.

**On "fewer layers":** the old advice to squash everything into few layers mainly mattered for the ancient 127-layer limit. Today the goals are (1) correct cleanup in the same layer, (2) cache-friendly ordering, (3) small final content, not an artificially low layer count.

---

## 6. The build context and `.dockerignore`

Everything in the context directory is packaged and sent to the builder before the first step, and any file that a `COPY` touches becomes part of a cache key. So:

- A large context (`.git`, `node_modules`, build output, data files) makes builds slow.
- A `COPY . .` step is invalidated by **any** changed file in the context, including logs and editor swap files.
- Secrets (`.env`, private keys) could be copied into the image by accident.

`.dockerignore` (placed at the root of the context):

```
.git
.gitignore
node_modules
dist
build
*.log
.env
.env.*
.DS_Store
Dockerfile
docker-compose*.yml
.dockerignore
coverage/
```

It uses `.gitignore`-like patterns (`**/` for any depth, `!keep.me` to re-include). Always exclude the **dependency folders you install inside the image** (`node_modules`, virtualenvs) so host copies (possibly for another OS) don't overwrite them.

---

## 7. `COPY` vs `ADD`, `ARG` vs `ENV`

**`COPY`** copies local files/directories from the context (and from other build stages: `COPY --from=`). **`ADD`** does that plus: auto-extracts **local** tar archives, and can fetch **URLs** (and Git repos). Because that "magic" is surprising, and a downloaded URL isn't cache-aware in the way you expect, the guidance is:

- Use **`COPY`** by default.
- Use `ADD` only for auto-extraction of a local `.tar.gz`. To download, `RUN curl -fsSL ... | tar -xz` (with a checksum), or `ADD --checksum=sha256:... URL` in recent versions.

| | `ARG` | `ENV` |
|---|---|---|
| Available | **Build time** only | Build time **and** run time (stored in the image) |
| Set with | `--build-arg NAME=value` | `ENV NAME=value` (override at run time with `-e`) |
| Visible in the final image? | Not as an env var, **but** the value is recorded in `docker history`, so **never** put secrets in it | Yes (`docker inspect` shows it) |
| Example | `ARG GO_VERSION=1.22` | `ENV PORT=8080` |

`ARG` before `FROM` can parameterize the base image (`ARG BASE=ubuntu:24.04` then `FROM ${BASE}`); an `ARG` used in a step becomes part of that step's cache key, so changing it invalidates that step and all later ones.

---

## 8. Multi-stage builds

### The problem
To *build* software you need compilers, headers, package managers and source. To *run* it you need only the result. Shipping the build tools makes images big and gives attackers more to work with.

### The solution
Use **several `FROM` lines** in one Dockerfile. Each `FROM` starts a fresh **stage**. You can copy artifacts from one stage to another with `COPY --from=<stage>`. Only the **last stage** (or the one you `--target`) becomes the final image; the others are thrown away (though their layers remain in the build cache).

### Example: a Go server (dramatic size difference)

`Dockerfile`:

```dockerfile
# ---------- Stage 1: build ----------
FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod ./
# COPY go.sum ./            # if you have dependencies
# RUN go mod download       # cached until go.mod/go.sum change
COPY . .
# static binary: no C library needed at run time
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/server .

# ---------- Stage 2: runtime ----------
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

(You need a `go.mod`: `go mod init example.com/server` in the folder holding `server.go`. Recent Go versions also need the server to bind `:8080` as before.)

```bash
docker build -t go-server:multi .
docker images | grep go-server
```

Typical result: the single-stage Ubuntu + Go image is **hundreds of MB** (well over 500 MB); the multi-stage image is **about 10 MB**. It also contains **no shell, no package manager and no compiler**, and runs as a non-root user (`:nonroot`).

Trade-offs of tiny images: you can't `docker exec ... sh` (no shell). Debug with `docker debug`, an ephemeral debug container sharing the namespaces, or a `:debug` variant of the base (`distroless/static:debug` includes BusyBox).

### Example: Node.js with a build step (correct dependency handling)

```dockerfile
# ---------- build ----------
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci                                # ALL dependencies: dev deps are needed to build
COPY . .
RUN npm run build                         # e.g. TypeScript → dist/

# ---------- production dependencies only ----------
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev                     # runtime dependencies only

# ---------- final image ----------
FROM node:22-alpine
ENV NODE_ENV=production
WORKDIR /app
COPY --from=deps  --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
COPY --chown=node:node package.json ./
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

(Note the trap in many tutorials: running `npm ci --only=production` in the *build* stage and then `npm run build` fails, because build tools live in `devDependencies`.)

### Handy multi-stage features

```bash
docker build --target build -t app:build .    # stop at a named stage (e.g. run tests, or a debug image)
```

```dockerfile
FROM build AS test
RUN go test ./...            # `docker build --target test .` runs the tests in CI

COPY --from=nginx:1.27-alpine /etc/nginx/nginx.conf /tmp/nginx.conf   # copy from ANY image, not just stages
```

BuildKit builds only the stages your target needs, and runs independent stages **in parallel**.

---

## 9. BuildKit features (expert)

### Cache mounts: a persistent cache for package managers

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt          # pip's download cache survives across builds, but stays out of the image
```

Other examples: `--mount=type=cache,target=/var/cache/apt`, `/root/.npm`, `/root/.cache/go-build`.

### Secret mounts: never bake secrets into layers

```dockerfile
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN="$(cat /run/secrets/npm_token)" npm ci
```

```bash
docker build --secret id=npm_token,src=$HOME/.npm_token -t app .
```

The secret exists only while that step runs and is **not stored in any layer**.

### SSH agent forwarding for private repositories
`RUN --mount=type=ssh git clone git@github.com:org/private.git` with `docker build --ssh default .`.

### Bind mounts of the context for a step (no copy, no layer)
`RUN --mount=type=bind,source=.,target=/src go build ...`

### `buildx`, multi-platform and remote cache

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t myorg/app:1.0 --push .
docker buildx build --cache-to type=registry,ref=myorg/app:cache,mode=max \
                    --cache-from type=registry,ref=myorg/app:cache -t myorg/app:1.0 .
```

Multi-platform images (Apple-Silicon laptops vs x86 servers) and shared caches in CI are what `buildx` is for. In CI, layer caches on ephemeral runners are lost unless you export/import them like this.

### Reproducibility and supply chain
Pin base images (tag + digest: `FROM python:3.12-slim@sha256:...`), pin package versions, generate an SBOM and provenance (`--sbom=true --provenance=true`), scan (`docker scout`, Trivy).

---

## 10. Inspecting and understanding your image

```bash
docker history --no-trunc go-server:1.0.0       # each step and its size
docker image inspect go-server:1.0.0 -f '{{json .RootFS.Layers}}'   # layer digests
docker system df -v                             # image sizes, shared vs unique
docker build --progress=plain . 2> build.log    # full log to keep
```

**Third-party tool `dive`** shows each layer's file changes and wasted space interactively (`dive go-server:1.0.0`).

Two images that share base layers store those layers **once** on disk; `docker images` sizes overlap, so they don't add up to disk usage.

---

## 11. Hands-on lab

**Lab 1: see the cache work**

```bash
mkdir layers-lab && cd layers-lab
cat > Dockerfile << 'EOF'
FROM alpine:3.20
RUN echo "step 1" && sleep 3
RUN echo "step 2" && sleep 3
COPY data.txt /data.txt
RUN echo "step 3" && sleep 3
EOF
echo v1 > data.txt

time docker build -t layers .           # ~9 s (BuildKit may run steps in order because each depends on the last)
time docker build -t layers .           # ~0-1 s, all CACHED
echo v2 > data.txt
time docker build -t layers .           # step 1-2 CACHED; COPY and step 3 re-run (~3 s)
```

**Lab 2: a volatile step at the top ruins caching**

```bash
cat > Dockerfile << 'EOF'
FROM alpine:3.20
ARG STAMP=none
RUN echo "$STAMP" > /stamp
RUN sleep 3 && echo expensive
EOF
docker build -t vol --build-arg STAMP=1 .
docker build -t vol --build-arg STAMP=2 .    # ARG changed → step 2 and everything below rebuild
```

**Lab 3: deleting later doesn't shrink**

```bash
cat > Dockerfile << 'EOF'
FROM alpine:3.20
RUN dd if=/dev/zero of=/big bs=1M count=50
RUN rm /big
EOF
docker build -t bloat . && docker history bloat     # a 52.4MB layer, then a 0B "rm" layer
docker images bloat                                  # image still ~55 MB

cat > Dockerfile << 'EOF'
FROM alpine:3.20
RUN dd if=/dev/zero of=/big bs=1M count=50 && rm /big
EOF
docker build -t lean . && docker images lean         # ~8 MB
```

**Lab 4: multi-stage size**: build the Go server both ways (single-stage Ubuntu + `golang-go`, and the multi-stage/distroless version) and compare `docker images`.

**Lab 5: `--target`**: add a `test` stage to your multi-stage file and run `docker build --target test .`.

**Lab 6: `.dockerignore`**: create a 200 MB dummy file (`dd if=/dev/zero of=big.bin bs=1M count=200`), watch "transferring context" with `--progress=plain`, then add `big.bin` to `.dockerignore` and compare.

---

## 12. Troubleshooting builds and cache

| Symptom | Cause | Fix |
|---|---|---|
| Whole build re-runs after a tiny change | The change is high in the Dockerfile (or `COPY . .` too early) | Reorder: manifests → install → source |
| Cache never hits in CI | Fresh runner each time, no cache export | `--cache-from/--cache-to`, or a registry/GitHub Actions cache backend |
| `apt-get install` fails with 404 on old cached layer | Cached `apt-get update` is stale | `update && install` in one `RUN`; rebuild with `--no-cache` |
| Build picks up an old base image | Cached, or local image older than upstream | `docker build --pull` |
| Changing a file doesn't trigger `COPY` | File is in `.dockerignore`, or you edited a different copy | Check `.dockerignore` and paths |
| "Sending build context" is huge / slow | No `.dockerignore` | Add one |
| Secret found in `docker history` | `ARG`/`ENV`/copied file | Rotate the secret. Use `--mount=type=secret` |
| Final image is huge | Build tools kept, layers not cleaned | Multi-stage; cleanup in the same `RUN`; slim/distroless bases |
| `COPY --from=build /x` fails: not found | Wrong path or stage name, or the file isn't produced | Add `RUN ls -R /x` in the stage; check spelling of `AS name` |
| `exec format error` at run time | Built for a different CPU (arm64 vs amd64) | `--platform`, or multi-arch build |
| Different results on each build | Unpinned tags/packages/`latest` | Pin versions and digests |

---

## 13. Best practices checklist

1. Order instructions **least → most frequently changing**; copy dependency manifests before source.
2. `.dockerignore` everything the build doesn't need (and everything it must not leak).
3. Combine `apt-get update`/`install`/cleanup in **one** `RUN`; clean up in the same layer.
4. **Multi-stage** builds: compile in a build stage; ship a minimal runtime stage.
5. Prefer **official, minimal, pinned** base images (tag + digest for production); rebuild regularly to pick up security fixes.
6. **No secrets** in `ARG`, `ENV`, or files; use BuildKit secret mounts.
7. Run as **non-root** (`USER`), one main process, exec-form `CMD`/`ENTRYPOINT`.
8. Use BuildKit **cache mounts** for package managers and `buildx` cache export in CI.
9. Verify: `docker history`, `dive`, and a vulnerability scan (`docker scout`, Trivy).
10. Keep the Dockerfile readable and commented; it is documentation.

---

## 14. Summary

- Layers = file-system deltas identified by hash; `RUN`, `COPY`, `ADD` create them, while `ENV`, `CMD`, `WORKDIR`, etc. are (mostly) metadata.
- The **cache** reuses a step if the parent, the instruction and (for `COPY`) file contents match; **one miss invalidates every step after it**.
- **Order** slow, stable steps first and volatile ones last; copy manifests before source.
- Layers are **additive**: clean up in the same `RUN`; never store secrets in a layer.
- **`.dockerignore`** speeds builds and prevents leaks. Use `COPY` over `ADD`; know `ARG` (build) vs `ENV` (build + run).
- **Multi-stage builds** separate build tools from the runtime image: often 10–50× smaller.
- **BuildKit** adds parallelism, cache/secret/SSH mounts, `buildx` multi-platform builds and remote caches.

---

## 15. Check your understanding

1. Which of these create a filesystem layer: `RUN`, `ENV`, `COPY`, `CMD`, `WORKDIR` (for a new directory)?
2. You edit `server.go` and rebuild; which steps in the example Dockerfile re-run, and why?
3. Why does `RUN apt-get update` stay "cached" for months, and how does that cause 404 errors?
4. Why doesn't `RUN rm bigfile` in a later step make the image smaller?
5. What is the difference between `ARG` and `ENV`? Is `ARG` safe for a password?
6. In a multi-stage build, what ends up in the final image?
7. Why is `COPY package.json ./` followed by `RUN npm ci` and only then `COPY . .` faster?
8. How do you pass a private token to a build step without storing it in the image?

<details>
<summary>Answers</summary>

1. `RUN`, `COPY` and (when it creates a directory) `WORKDIR`; `ENV` and `CMD` are metadata only.
2. `COPY server.go .` and everything after it (metadata steps just get re-recorded). Earlier steps are cached because their inputs are unchanged.
3. The cache key for `RUN` is only the command text, not the outside world. Package lists go stale; a later `install` can request package versions that no longer exist on the mirror. Combine `update` and `install`, or rebuild with `--no-cache`.
4. Layers are immutable and additive; the data stays in the earlier layer. The `rm` only adds a "deleted" marker layer.
5. `ARG` exists only during the build; `ENV` persists into the image and running containers. `ARG` values are visible in `docker history`, so it isn't safe for secrets.
6. Only the last stage (or the `--target` stage) and whatever you `COPY --from` into it; earlier stages are discarded.
7. The `npm ci` layer is reused until the manifests change; editing source only invalidates the last `COPY`.
8. A BuildKit secret mount: `RUN --mount=type=secret,id=...` with `docker build --secret id=...,src=...`.
</details>

**Practice**

1. Take any Dockerfile you have and run `docker history` on its image. Find the biggest layer and reduce it.
2. Rewrite a "copy everything first" Dockerfile for Python or Node to use the manifest-first pattern; measure rebuild time after a one-line code change (before/after).
3. Convert the Chapter 15 Go server to a multi-stage build and compare sizes; then add a `--target test` stage.
4. Add a `.dockerignore` and a `RUN --mount=type=cache` step to a Python build; time three consecutive builds after changing `requirements.txt`.

---

**Next:** Part 2 of the series switches from Docker to **networking**, the knowledge you need to understand container networking, DNS, ports and TLS: [Chapter 21 – The Philosophy of the OSI Model](21_philosophy_of_osi_model.md)
