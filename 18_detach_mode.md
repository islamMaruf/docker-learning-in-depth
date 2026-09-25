# Chapter 18: Detached Mode: Running Containers in the Background

> **In one sentence:** `docker run -d` starts a container in the background and gives your terminal back; you then use `docker logs`, `docker exec`, `docker stop` and `-p` port publishing to work with it.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~40 minutes

**Prerequisites:** [Chapter 16 – CMD](16_cmd_deep_dive.md) and the Go server image from [Chapter 15](15_towards_the_dockerfile.md). Any image that runs a long-lived server (for example `nginx`) also works.

---

## What you will learn

- **Attached** vs **detached** mode, and what "attached" actually means
- Starting containers with `-d`, and finding them again
- Reading output later with `docker logs` (`-f`, `--tail`, `--since`, `-t`)
- Getting *into* background containers: `docker exec` vs `docker attach`, and how to detach with **Ctrl+P, Ctrl+Q**
- **Publishing ports** with `-p` so you can reach a server (and what `EXPOSE` does *not* do)
- **Restart policies**, resource limits, and health checks
- Troubleshooting, and a multi-container practice exercise

---

## 1. The problem: a blocked terminal

```bash
docker run --rm -p 8080:8080 go-server:1.0.0
# Server listening on port 8080...
# ▊   (nothing else: you can't type)
```

The container is fine. Your terminal is **attached** to it: the container's stdout/stderr are streamed to your screen and **Ctrl+C is forwarded to the container**, which stops it. If you close the terminal, or your SSH session drops, an attached container can be killed too.

For servers, databases, queues and anything long-running, you want them to keep running **in the background**.

---

## 2. Attached vs detached

| | **Attached** (default) | **Detached** (`-d`) |
|---|---|---|
| Terminal | Shows the container's output; blocked until it exits | Returns immediately |
| Output | Streamed live | Saved by Docker; read with `docker logs` |
| Ctrl+C | Sent to the container's main process (usually stops it) | Not connected to anything |
| Closing the terminal | May stop the container | Container keeps running |
| Typical use | Debugging, quick commands, interactive shells | Servers, databases, jobs, production |

```
Attached:   Your terminal ◄──── stdout/stderr ────  Container  (terminal is busy)
Detached:   Your terminal        (free)              Container ──► Docker's log store
```

> **What "detached" changes:** only how your *terminal* relates to the container. The container itself runs the same way. It still lives as long as its main process does (Chapter 16).

---

## 3. Running detached

```bash
docker run -d --name web -p 8080:8080 go-server:1.0.0
```

Output is a long **container ID** (64 hex characters) and your prompt returns:

```
9e7a4c1f0b6d5e3a2c1b8f7e6d5c4b3a2f1e0d9c8b7a6f5e4d3c2b1a0f9e8d7c
```

- `-d` = `--detach`. Everything else works as before.
- You rarely need the full ID. The first 12 characters (or any unique prefix), or the `--name`, are enough.
- Save the ID in a script: `CID=$(docker run -d ...)`.

Confirm it is running:

```bash
docker ps
# CONTAINER ID   IMAGE              COMMAND             STATUS         PORTS                    NAMES
# 9e7a4c1f0b6d   go-server:1.0.0    "go run server.go"  Up 5 seconds   0.0.0.0:8080->8080/tcp   web
curl http://localhost:8080          # Hello World
```

Lost the ID/name? `docker ps` (running), `docker ps -a` (all), `docker ps -l` (latest created), `docker ps -q` (IDs only), `docker ps --filter ancestor=go-server:1.0.0`, `docker ps --filter name=web`.

---

## 4. Publishing ports (`-p`)

A container has its **own network namespace** (Chapter 4), with its own `localhost` and its own ports. A server listening on port 8080 *inside* the container is not automatically reachable from the host or from other machines. You must **publish** (map) the port:

```
-p  [HOST_IP:]HOST_PORT : CONTAINER_PORT [/protocol]
```

```
   Browser ─► host:8080 ──(Docker forwards)──► container:80
```

| Command | Meaning |
|---|---|
| `-p 8080:80` | Host port **8080** → container port **80** (all host interfaces) |
| `-p 127.0.0.1:8080:80` | Only reachable from the host itself (safer for development databases) |
| `-p 80` | Container port 80 → a **random** free host port (see it with `docker port NAME`) |
| `-p 5353:53/udp` | UDP instead of TCP |
| `-P` | Publish **all** ports declared with `EXPOSE` to random host ports |
| `-p 8080:80 -p 8443:443` | Several mappings |

Details worth knowing:

- Left side = **host**, right side = **container**. It's easy to reverse them.
- Two containers can't publish the **same host port**; you'll get `port is already allocated`. But containers *can* each listen on the same container port (`-p 8081:8080`, `-p 8082:8080`).
- **Inside the container, the app must listen on all interfaces** (`0.0.0.0` or `:8080`), not `127.0.0.1`, or the forwarded traffic can't reach it.
- **`EXPOSE 8080` in a Dockerfile is documentation.** It does *not* publish anything by itself. Only `-p`/`-P` (or `ports:` in Compose) does.
- Containers talking **to each other** don't need published ports: put them on the same Docker network and use the container name as hostname (Chapter 19/Compose).
- **Security:** published ports are reachable from other machines and, on some Linux setups, can bypass host firewall rules (Docker manages its own iptables rules). Bind to `127.0.0.1` unless the service must be public.

```bash
docker port web              # 8080/tcp -> 0.0.0.0:8080
```

---

## 5. Looking at what a detached container is doing

### `docker logs`

Everything the container's main process writes to **stdout and stderr** is captured (by the default `json-file` logging driver):

```bash
docker logs web                  # everything so far
docker logs -f web               # FOLLOW live (Ctrl+C stops following, NOT the container)
docker logs --tail 50 web        # last 50 lines
docker logs --since 10m web      # last 10 minutes (also: --since 2025-01-01T10:00:00)
docker logs -t web               # add timestamps
docker logs -f --tail 20 web     # a common combination
```

Notes:

- Apps must log to **stdout/stderr** for this to work. If an app writes only to a log file inside the container, `docker logs` is empty (send the file to stdout, e.g. symlink to `/dev/stdout` as the official nginx image does).
- **Default logs are unbounded.** With the default driver, log files in `/var/lib/docker/containers/<id>/` grow forever unless you configure rotation:
  `docker run -d --log-opt max-size=10m --log-opt max-file=3 ...` or set defaults in `/etc/docker/daemon.json`.
- Logs are deleted with the container (`docker rm`).

### Other inspection commands

```bash
docker stats --no-stream         # CPU, memory, network I/O per container
docker top web                   # processes inside
docker inspect web               # everything (JSON); use -f for a field:
docker inspect -f '{{.State.Status}} {{.State.ExitCode}}' web
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web
docker events                    # live stream of daemon events (start, die, ...)
```

---

## 6. Getting into a background container

### `docker exec` (the everyday tool)

Starts an **additional** process inside the running container. Exiting it does **not** stop the container.

```bash
docker exec -it web bash            # a shell (use sh on Alpine/minimal images)
docker exec web ls /app             # one-off command, no terminal needed
docker exec -it -u root -w /etc web sh
```

### `docker attach` (connect to the *main* process)

Connects your terminal to the container's **PID 1**: its stdin/stdout/stderr.

```bash
docker run -dit --name box ubuntu:24.04 bash    # -d background, -i keep stdin, -t terminal
docker attach box                                # you are now in that bash
```

Key behaviors:

- If you `exit` (or press Ctrl+C in a server), you **stop the main process, so the container stops**.
- To **detach without stopping**: press **Ctrl+P, then Ctrl+Q** (only works when the container has a TTY, `-t`). You can change the sequence with `--detach-keys`.
- Several `attach` sessions **mirror** the same stream, which is confusing.

| | `docker exec` | `docker attach` |
|---|---|---|
| Runs | A **new** process | Connects to the **existing main** process |
| Exiting stops the container? | No | Yes (unless you detach with Ctrl+P, Q) |
| Typical use | Debug, admin | Rare: interact with a main process that reads stdin |

**Use `exec` almost always.**

### Combining `-d` with `-it`
`docker run -d -it ubuntu:24.04` keeps a shell alive in the background (it has a terminal and open stdin, so it doesn't exit). Handy for a long-lived scratch environment you `exec` into. (Without `-it`, bash would exit and so would the container.)

---

## 7. Stopping, starting, restarting

```bash
docker stop web          # SIGTERM, wait 10 s, then SIGKILL   (-t 30 to wait longer)
docker start web         # start it again: a started container runs in the BACKGROUND by default
docker start -a web      # start and attach to its output
docker restart web       # stop + start
docker kill web          # immediate SIGKILL
docker pause web / docker unpause web      # freeze/unfreeze its processes
docker rm web            # delete (must be stopped)
docker rm -f web         # stop and delete
docker wait web          # block until it exits, then print the exit code
```

`docker stop` on several at once: `docker stop web1 web2 web3`; all: `docker stop $(docker ps -q)`.

---

## 8. Restart policies (making background containers resilient)

By default a container that exits, or that dies because the machine reboots, stays down. Tell Docker to bring it back:

```bash
docker run -d --restart unless-stopped --name web -p 8080:8080 go-server:1.0.0
```

| Policy | Behavior |
|---|---|
| `no` (default) | Never restart |
| `on-failure[:N]` | Restart only if it exits with a **non-zero** code (at most N times if given) |
| `always` | Restart whenever it stops (also at daemon start). Restarts even if *you* stopped it, after the daemon restarts |
| `unless-stopped` | Like `always`, except it stays stopped if you stopped it manually |

Change later without recreating: `docker update --restart unless-stopped web`. Docker waits a growing delay between restarts (100 ms, doubling), so a crash-looping container shows `Restarting` in `docker ps`.

> A restart policy handles **crashes**, not **hangs**. A container that is alive but unresponsive needs a **health check** and something that acts on it (an orchestrator like Kubernetes, or Swarm; plain Docker only *reports* health).

---

## 9. Resource limits

A runaway background container can starve the host. Cap it (these use cgroups; Chapter 4):

```bash
docker run -d --name limited --memory 512m --memory-swap 512m --cpus 0.5 --pids-limit 200 myimage
docker stats --no-stream limited
```

If a container exceeds `--memory` it is **OOM-killed** (exit code **137**; `docker inspect -f '{{.State.OOMKilled}}'` shows `true`).

---

## 10. Health checks

A **health check** is a command Docker runs periodically inside the container; its result is shown as `healthy`/`unhealthy` in `docker ps`.

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -fsS http://localhost:8080/ || exit 1
```

- The command **must exist in the image** (so `curl` must be installed; alternatives are `wget`, or a small built-in check in your app).
- Exit code 0 = healthy, 1 = unhealthy.
- Or at run time: `docker run -d --health-cmd 'curl -fsS localhost:8080 || exit 1' --health-interval 30s myimage`.
- `docker ps` shows `Up 2 minutes (healthy)`; `docker inspect -f '{{json .State.Health}}' web` has details.
- In Compose, `depends_on: condition: service_healthy` uses it (Chapter 7).

---

## 11. Real-world examples

```bash
# A web server (public)
docker run -d --name proxy -p 80:80 nginx:1.27

# A database for local development (only reachable from this machine; data survives in a named volume)
docker run -d --name pg \
  -e POSTGRES_PASSWORD=devpassword \
  -p 127.0.0.1:5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16-alpine

# A cache
docker run -d --name cache -p 127.0.0.1:6379:6379 redis:7-alpine

# Wait for postgres to accept connections, then use it
docker exec pg pg_isready -U postgres
docker exec -it pg psql -U postgres
```

Cautions: `-e POSTGRES_PASSWORD=...` on the command line is visible in `docker inspect` and shell history. Use secrets or env files (`--env-file`) for real credentials. Data written into a container's file system disappears with the container; use **volumes** (`-v`) for anything valuable.

---

## 12. Choosing the mode

| Scenario | Mode | Why |
|---|---|---|
| Web server, API, database, queue, cache | **Detached** | Long-running; you don't need the output on screen |
| Production | **Detached** + restart policy + limits + health check | Runs unattended |
| Several services at once | **Detached** (or Compose) | One terminal |
| Debugging a start-up failure | **Attached** (or `logs`) | See errors immediately |
| Interactive shell / REPL | **Attached** `-it` | You type into it |
| Quick one-off command (`--version`, a test) | **Attached** `--rm` | Finishes right away |

---

## 13. Troubleshooting

| Symptom | Diagnosis | Fix |
|---|---|---|
| `docker run -d` prints an ID but `docker ps` is empty | The container exited. `docker ps -a` shows `Exited (N)` | `docker logs <id>` for the error; make sure the `CMD` is a long-running foreground process |
| `curl: (7) Failed to connect` | No published port (`docker ps` PORTS shows only `8080/tcp`), or app bound to 127.0.0.1, or still starting | Recreate with `-p 8080:8080`; bind to `0.0.0.0`; check `docker logs` |
| `curl: (52) Empty reply` / `(56) Connection reset` | Port published but the app in the container isn't listening on that port | Check the app's actual port; `docker exec c ss -tlnp` (or `netstat`) |
| `Bind for 0.0.0.0:8080 failed: port is already allocated` | Another container or process uses host port 8080 | Choose another host port; find the user with `docker ps` or `ss -tlnp` / `lsof -i :8080` |
| `docker logs` is empty | App writes to files, or buffers output (Python: use `python -u` or `PYTHONUNBUFFERED=1`) | Log to stdout/stderr; disable buffering |
| Container keeps restarting | Crash loop with a restart policy | `docker logs`, `docker inspect -f '{{.State.ExitCode}}'` |
| Disk filling up from logs | Unrotated `json-file` logs | `--log-opt max-size`/`max-file` or daemon defaults |
| `docker attach` then Ctrl+C stopped my container | Ctrl+C reached the main process | Detach with Ctrl+P, Ctrl+Q instead (needs `-t`); prefer `docker exec` |
| Container is slow / using too much CPU | Unbounded background container | `docker stats`; recreate with `--cpus`/`--memory` |
| Name conflict on re-run | Old stopped container with the same name | `docker rm name` (or use `--rm`, or `docker run` with a new name) |

---

## 14. Practice: three services

```bash
# 1. Start three instances of the Go server on different host ports
docker run -d --name svc1 -p 8081:8080 go-server:1.0.0
docker run -d --name svc2 -p 8082:8080 go-server:1.0.0
docker run -d --name svc3 -p 8083:8080 go-server:1.0.0

# 2. Verify
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
for p in 8081 8082 8083; do curl -s localhost:$p; done

# 3. Logs and stats
docker logs svc2
docker stats --no-stream

# 4. Stop and start one
docker stop svc2 && docker ps
docker start svc2 && curl -s localhost:8082

# 5. Add resilience to a fourth one
docker run -d --name svc4 -p 8084:8080 --restart unless-stopped --memory 256m go-server:1.0.0
docker kill svc4 ; sleep 3 ; docker ps       # kill is a stop *you* asked for... compare with `docker exec svc4 kill 1`

# 6. Clean up
docker rm -f svc1 svc2 svc3 svc4
```

For step 5, try both `docker kill svc4` and `docker exec svc4 sh -c 'kill 1'` (if the app doesn't handle it, PID 1 ignores the signal, remember Chapter 16) and observe when the restart policy kicks in.

---

## 15. Summary

- **Attached:** terminal streams the container's output and is blocked. **Detached (`-d`):** background; get output with `docker logs`.
- `docker run -d` prints the container ID; always give containers a **name**.
- **`-p host:container`** publishes ports; `EXPOSE` is only documentation; apps must listen on `0.0.0.0`.
- `docker logs [-f --tail N --since T]` reads stdout/stderr; configure log rotation.
- Use **`docker exec`** to get in; `attach` connects to PID 1 (**Ctrl+P, Ctrl+Q** detaches without stopping).
- For unattended containers add a **restart policy**, **resource limits** and a **health check**.

---

## 16. Check your understanding

1. What is the practical difference between `docker run image` and `docker run -d image`?
2. You ran `docker run -d --name web myserver` and `curl localhost:8080` fails, though `docker ps` shows the container `Up`. Give two possible causes.
3. What is the difference between `docker exec -it web sh` and `docker attach web`? Which can accidentally stop the container?
4. What does `--restart unless-stopped` do that `--restart always` doesn't?
5. Why might `docker logs` show nothing for a container that is clearly running?
6. What does `EXPOSE 8080` in a Dockerfile do?
7. How do you follow the last 20 log lines live?

<details>
<summary>Answers</summary>

1. Without `-d` your terminal is attached and blocked (Ctrl+C stops it); with `-d` it runs in the background and the terminal is returned.
2. The port was not published (`-p 8080:8080` missing), or the app listens on 127.0.0.1 or a different port, or it hasn't finished starting.
3. `exec` runs a new process; `attach` connects to the main process. Exiting/Ctrl+C in an attached session can stop the container.
4. A container you stopped manually with `docker stop` stays stopped after a daemon restart, whereas `always` would start it again.
5. The app logs to files or has buffered output instead of writing to stdout/stderr.
6. Documents which port the app uses. It does not publish it; only `-p`/`-P` does.
7. `docker logs -f --tail 20 NAME`
</details>

---

**Next:** [Chapter 19 – Managing Containers](19_managing_containers.md)
