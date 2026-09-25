# Chapter 17: `WORKDIR` Deep Dive

> **In one sentence:** `WORKDIR` sets the directory in which the following Dockerfile instructions run, and in which the container starts, so you can use short relative paths instead of repeating long absolute ones.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~30 minutes

**Prerequisites:** [Chapter 12](12_linux_basic_commands.md) (paths) and [Chapters 15](15_towards_the_dockerfile.md)–[16](16_cmd_deep_dive.md).

---

## What you will learn

- What the **working directory** is (a general Linux idea) and where containers start by default
- What `WORKDIR` does and why it beats `RUN mkdir` and `RUN cd`
- Absolute vs relative `WORKDIR`, and how a chain of them resolves
- Which instructions are affected (and how `COPY` paths are interpreted)
- How to override at run time (`-w`), how it interacts with `exec`, `USER`, and volumes
- Common patterns, mistakes and troubleshooting

---

## 1. The working directory (a reminder)

Every running process has a **current working directory (cwd)**. **Relative paths** (`server.go`, `./data`, `../x`) are resolved from it (Chapter 12). Check it with `pwd`.

By default the working directory in a container is **`/`**, unless the image sets one:

```bash
docker run --rm ubuntu:24.04 pwd          # /
docker run --rm node:22 pwd               # /  (Node's official image doesn't set one)
docker run --rm python:3.12 pwd           # /
docker run --rm golang:1.22 pwd           # /go   (this image does set WORKDIR)
```

Being in `/` is inconvenient, and it's untidy to put your application into the root of the file system.

---

## 2. The problem: life without `WORKDIR`

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y --no-install-recommends golang-go && rm -rf /var/lib/apt/lists/*
RUN mkdir /app
COPY server.go /app/server.go
COPY config/app.conf /app/config/app.conf
CMD ["go", "run", "/app/server.go"]
```

Issues:

1. `/app` is spelled out again and again. Moving the app means editing every line.
2. A missed slash sends a file to the wrong place.
3. Anyone who runs `docker run -it IMAGE bash` (or `docker exec`) starts in `/`, far from the code.
4. Relative commands (`go run server.go`) don't work.

---

## 3. What `WORKDIR` does

```dockerfile
WORKDIR /app
```

1. **Creates** the directory (and missing parents) if it doesn't exist. No need for `RUN mkdir`.
2. **Sets it as the current directory** for every *following* `RUN`, `CMD`, `ENTRYPOINT`, `COPY` and `ADD`.
3. **Is stored in the image's metadata**, so at run time the container's main process (and `docker exec`) starts there too.

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y --no-install-recommends golang-go && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY server.go .                # . means "the working directory" → /app/server.go
COPY config/ ./config/          # → /app/config/
CMD ["go", "run", "server.go"]  # runs in /app
```

To change the location you edit **one** line.

### How `COPY`/`ADD` read paths

| Side | Interpreted relative to |
|---|---|
| **Source** (`COPY src ...`) | The **build context** on your machine (not `WORKDIR`) |
| **Destination** (`COPY ... dest`) | The image's **`WORKDIR`** if `dest` is relative; the file system root if it starts with `/` |

```dockerfile
WORKDIR /app
COPY server.go ./           # /app/server.go
COPY server.go .            # same
COPY server.go /app/        # same (absolute)
COPY server.go /server.go   # /server.go: NOT in /app!
```

A common confusion: **build context** (`docker build .`, on the host) vs **WORKDIR** (inside the image). They are unrelated file systems.

---

## 4. `WORKDIR` vs `RUN mkdir` vs `RUN cd`

| | `RUN mkdir /app` | `RUN cd /app` | `WORKDIR /app` |
|---|---|---|---|
| Creates the directory | Yes (fails if it exists, unless `-p`) | No | Yes (no error if it exists) |
| Changes the directory for later instructions | **No** | **No** (only within that one `RUN`'s shell) | **Yes** |
| Affects the container at run time | No | No | **Yes** |

Each `RUN` starts a fresh shell, so:

```dockerfile
RUN cd /app
RUN pwd              # prints "/": the cd was forgotten
```

If you truly need a directory change for one command, chain it: `RUN cd /tmp && make`. Or use `WORKDIR` and change back.

Conclusion: **use `WORKDIR` for the application's directory.**

---

## 5. Absolute and relative `WORKDIR`

```dockerfile
WORKDIR /app        # absolute: always exactly /app
WORKDIR backend     # relative: /app/backend (resolved against the previous WORKDIR)
WORKDIR src         # /app/backend/src
RUN pwd             # /app/backend/src
WORKDIR /opt/other  # absolute again: resets the chain
```

- A relative `WORKDIR` starts from the *previous* `WORKDIR` (or `/` if there was none).
- Prefer **absolute** paths: they are unambiguous and don't break if someone inserts a line above.
- **Variables work**, as long as they are defined by `ENV` or `ARG` earlier: `ENV APP_HOME=/opt/myapp` then `WORKDIR ${APP_HOME}`. (`ENV` also stays available at run time.)
- **Multiple `WORKDIR`s are fine**, but avoid needless chains: `WORKDIR /app/src/main` is clearer than three lines. (Each `WORKDIR` adds a tiny metadata step to the history, and creates the directory if missing; the size impact is negligible.)
- Don't use `WORKDIR` merely for a temporary excursion, or you must remember to switch back. Prefer `RUN cd /tmp && ...` for one-off work, or use absolute paths.

---

## 6. Runtime behavior and overrides

The image's `WORKDIR` becomes the container's default working directory:

```bash
docker run --rm -it go-server:1.0.0 bash      # prompt: root@…:/app#
docker exec -it mycontainer bash              # also starts in /app (the container's working dir)
```

Override when needed:

```bash
docker run --rm -w /tmp ubuntu:24.04 pwd               # /tmp
docker exec -w /etc mycontainer pwd                    # /etc
```

In **Docker Compose**: `working_dir: /app` under the service.

Check what an image uses:

```bash
docker inspect -f '{{.Config.WorkingDir}}' go-server:1.0.0     # /app
```

---

## 7. `WORKDIR` and other features

### With `USER` (permissions) 🔴
`WORKDIR` creates the directory if it doesn't exist. Depending on the builder version, the created directory may be **owned by root**, so a non-root `USER` cannot write into it. Don't rely on the default. Be explicit:

```dockerfile
FROM node:22-alpine
RUN addgroup -g 10001 app && adduser -D -u 10001 -G app app
WORKDIR /app
COPY --chown=app:app package*.json ./     # files owned by app
RUN npm ci --omit=dev
COPY --chown=app:app . .
RUN chown app:app /app                    # make the directory itself writable, if the app must create files there
USER app
CMD ["node", "server.js"]
```

(Chapter 13: permissions and non-root users.)

### With volumes and bind mounts
A mount replaces the contents of its target path, so a mount to `/app` **hides** the files you copied there:

```bash
docker run --rm -v "$PWD":/app myimage      # your host folder now appears at /app; the image's /app content is hidden
```

That is intentional for development ("live code"), but a surprise if you didn't expect it. Mount data into a sub-folder (`-v data:/app/data`) to keep the code.

### With multi-stage builds
Each `FROM` starts a new stage with its own `WORKDIR` (Chapter 20):

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/server .

FROM gcr.io/distroless/static-debian12
WORKDIR /app
COPY --from=build /out/server .
CMD ["./server"]
```

(Distroless has no shell, so exec-form `CMD` is required: Chapter 16.)

### Choosing the directory

| Good | Why |
|---|---|
| `/app` | Short, common convention |
| `/usr/src/app` | Convention used by many Node.js examples |
| `/opt/<name>` | Standard place for optional third-party software |
| `/home/<user>/app` | When running as a non-root user with a home |

Avoid: `/` (messy), `/root` (root's home), `/etc`, `/bin`, `/usr/bin`, `/tmp` (temporary, may be cleared), and other system directories. Follow the base image's convention when one exists (`golang` uses `/go`; `nginx` serves from `/usr/share/nginx/html`).

---

## 8. Common patterns

**Node.js (dependency-cache friendly):**

```dockerfile
FROM node:22-alpine
WORKDIR /usr/src/app
COPY package*.json ./          # copy only the dependency manifests first
RUN npm ci --omit=dev          # this layer is cached until the manifests change
COPY . .                       # your code changes often: keep it last
EXPOSE 3000
CMD ["node", "server.js"]
```

**Python:**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "main.py"]
```

**Tests in the same layout:** `docker run --rm -w /app/tests IMG pytest`.

---

## 9. Hands-on lab

**Lab 1: default working directory and `WORKDIR`**

```bash
mkdir wd-lab && cd wd-lab
cat > Dockerfile << 'EOF'
FROM alpine:3.20
RUN pwd                      # prints / during the build
WORKDIR /test
RUN pwd && echo "hello" > note.txt
WORKDIR sub
RUN pwd                      # /test/sub
CMD ["sh"]
EOF
docker build --progress=plain -t wd-lab . 2>&1 | grep -E '^#[0-9]+ [0-9.]+ /|RUN'
docker run --rm -it wd-lab
```

Inside: `pwd` (→ `/test/sub`), `ls` (nothing), `cat ../note.txt` (`hello`), `cd / && ls` (you'll see `test`).

**Lab 2: prove `RUN cd` doesn't stick**

```bash
printf 'FROM alpine:3.20\nRUN mkdir /work\nRUN cd /work\nRUN pwd\n' > Dockerfile
docker build --no-cache --progress=plain . 2>&1 | grep -A1 'RUN pwd'      # prints /
```

**Lab 3: the Go server with and without `WORKDIR`**: rebuild the Chapter 15 server, then run `docker run --rm -it IMG pwd` and `docker inspect -f '{{.Config.WorkingDir}}' IMG`.

**Lab 4: overrides**

```bash
docker run --rm -w /etc alpine:3.20 pwd            # /etc
docker run -d --name wd alpine:3.20 sleep 300
docker exec wd pwd                                   # /
docker exec -w /tmp wd pwd                           # /tmp
docker rm -f wd
```

**Lab 5: bind-mount hides files**

```bash
mkdir mounted && echo "from host" > mounted/host.txt
docker run --rm -v "$PWD/mounted":/test wd-lab ls /test      # host.txt (image's note.txt is hidden)
docker run --rm wd-lab ls /test                              # note.txt (no mount: the image's files)
```

---

## 10. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `can't open file 'server.go': No such file or directory` at run time | `COPY` destination differs from `WORKDIR`, or CMD uses a wrong path | `docker run --rm -it IMG sh`, then `pwd; ls -R` |
| File copied to `/` instead of `/app` | Absolute destination (`COPY x /x`) or `WORKDIR` set *after* `COPY` | Put `WORKDIR` before `COPY`; use relative destinations |
| `docker exec` starts in `/` | Image has no `WORKDIR` | Add `WORKDIR` (or use `docker exec -w`) |
| `RUN cd dir` then later commands run in the wrong place | `cd` doesn't persist between `RUN`s | Use `WORKDIR` |
| `COPY failed: file not found in build context` | The **source** path is wrong (relative to the context, not `WORKDIR`) | Check the path from where you run `docker build`; check `.dockerignore` |
| Permission denied writing in `/app` at run time | Directory owned by root but the app runs as non-root | `chown` it, or `COPY --chown`, or create it with the right owner |
| My code disappeared when I ran with `-v $PWD:/app` | The mount hides the image's `/app` | Intended: it shows your host files instead. Mount a subfolder for data only |
| `WORKDIR` variable is empty | `ENV/ARG` not defined **before** it | Move the `ENV` above |

---

## 11. Best practices

1. **Always** set a `WORKDIR` for your application. Don't rely on `/`.
2. Use an **absolute** path (`/app`) for the main `WORKDIR`, and set it **early**, right after `FROM` and installation steps.
3. Use **`WORKDIR`**, never `RUN mkdir && cd` to set up the app folder.
4. Copy files with **relative destinations** (`COPY . .`), so the destination follows `WORKDIR`.
5. Order for caching: dependency manifests → install → source (Chapter 15).
6. Don't use system directories; follow conventions of your language and base image.
7. With non-root users, set ownership explicitly.
8. Remember that mounts replace the directory's contents.

---

## 12. Summary

- `WORKDIR /path` creates the directory if needed, then sets the working directory for following `RUN`, `CMD`, `ENTRYPOINT`, `COPY` and `ADD`, and for the container at start-up.
- `RUN cd` and `RUN mkdir` don't do that. Each `RUN` is a separate shell.
- Relative `WORKDIR`s chain from the previous one; prefer absolute paths; `ENV`/`ARG` variables can be used.
- Override at run time with `-w` (`docker run`, `docker exec`).
- `COPY` sources come from the build context; destinations are relative to `WORKDIR`.
- Watch out for ownership with non-root users, and for mounts that hide the directory's contents.

---

## 13. Check your understanding

1. What is the default working directory in a container from an image that doesn't set one?
2. Why does `RUN cd /app` not affect the next `RUN`?
3. After `WORKDIR /app`, where does `COPY config.json ./conf/` put the file? And `COPY config.json /conf/`?
4. What is the working directory after `WORKDIR /a`, `WORKDIR b`, `WORKDIR /c`, `WORKDIR d`?
5. How do you start a container in `/tmp` without editing the Dockerfile?
6. You run `docker run -v "$PWD":/app IMG` and the files you `COPY`d into `/app` seem to vanish. Why?

<details>
<summary>Answers</summary>

1. `/` (the root directory).
2. Each `RUN` runs in a separate shell; the directory change lasts only for that instruction.
3. `/app/conf/config.json`; and `/conf/config.json` (absolute path, outside `/app`).
4. `/c/d`.
5. `docker run -w /tmp IMG ...`
6. The bind mount hides whatever the image had at `/app`; you see your host folder's contents instead.
</details>

**Practice**

1. Rewrite the "without `WORKDIR`" Dockerfile from section 2 to use `WORKDIR` and short paths. Verify with `docker inspect -f '{{.Config.WorkingDir}}'`.
2. Create a small project with `main.py`, `utils/helper.py` and `config/settings.json`; write a Dockerfile that copies the structure into `/usr/src/app` (`COPY . .`) and runs `main.py`. Confirm with `docker run --rm IMG find . -type f`.
3. Add `USER` with a non-root account to that Dockerfile and make the app write a log file into `/usr/src/app/logs`. Make it work (hint: create and `chown` the directory).

---

**Next:** [Chapter 18 – Detached Mode](18_detach_mode.md)
