# Chapter 19: Managing Containers: The Whole Life Cycle

> **In one sentence:** Containers are created, started, stopped, restarted and removed. This chapter is the practical toolkit for doing each of those things, for finding and inspecting containers, and for cleaning up, with the important rule that *most configuration changes require re-creating a container, not restarting it*.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~40 minutes

**Prerequisites:** Chapters [14](14_docker_hands_on.md), [16](16_cmd_deep_dive.md) and [18](18_detach_mode.md). Any long-running image works for practice; examples use `nginx` so they run anywhere:

```bash
docker pull nginx:1.27-alpine
```

---

## What you will learn

- The container **states** and which command moves a container between them
- Listing and **filtering** containers (`ps`, `--filter`, `--format`)
- `create`, `start`, `stop`, `restart`, `kill`, `pause`, `rm`, `rename`, `update`
- What signals and exit codes tell you
- Looking inside: `logs`, `inspect`, `stats`, `top`, `diff`, `cp`, `events`
- **What survives** a stop, a restart and a removal (and what doesn't)
- Cleaning up safely; scripting with `-q` and `xargs`
- Update/upgrade patterns and troubleshooting

---

## 1. States and transitions

A container is always in one **state**:

```
                    docker create
                         │
                         ▼
                    ┌─────────┐   docker start     ┌─────────┐   docker pause    ┌────────┐
  docker run ─────► │ Created │ ─────────────────► │ Running │ ────────────────► │ Paused │
  (= create+start)  └────┬────┘                    └────┬────┘ ◄──────────────── └────────┘
                         │                              │        docker unpause
                         │            main process exits, docker stop / kill,
                         │            or crash (may auto-restart via --restart)
                         │                              ▼
                         │                         ┌─────────┐   docker start
                         │                         │ Exited  │ ─────────────────► Running
                         │                         └────┬────┘
                         └──────── docker rm ───────────┘
                                                        ▼
                                                    (removed)
```

| State | Meaning |
|---|---|
| **created** | Exists, configured, never started |
| **running** | Main process is executing |
| **paused** | All processes frozen in memory (cgroup freezer). No CPU is used, but memory is still held |
| **restarting** | A restart policy is bringing it back (visible in crash loops) |
| **exited** | Main process ended. The container still exists with its file system and logs |
| **dead / removing** | Rare: removal failed or is in progress |

| Command | Transition |
|---|---|
| `docker create IMG` | (nothing) → created |
| `docker start C` | created / exited → running |
| `docker run IMG` | (nothing) → created → running (**a new container each time**) |
| `docker stop C` | running → exited (graceful) |
| `docker kill C` | running → exited (immediate) |
| `docker restart C` | running → exited → running |
| `docker pause / unpause C` | running ⇄ paused |
| `docker rm C` | created/exited → **removed** |
| `docker rm -f C` | any → removed |

---

## 2. Listing containers: `docker ps`

```bash
docker run -d --name web nginx:1.27-alpine

docker ps                # running only
docker ps -a             # all (including exited and created)
docker ps -l             # the latest created container
docker ps -n 3           # the last 3 created
docker ps -q             # only IDs (great for scripts)
docker ps -s             # add each container's writable-layer SIZE
docker ps --no-trunc     # show full IDs and commands
```

`docker container ls` is the same command with the modern name (`docker ps` is an alias).

### Filtering

```bash
docker ps -a --filter status=exited
docker ps -a --filter status=running --filter name=web
docker ps --filter ancestor=nginx:1.27-alpine      # containers created from this image
docker ps --filter publish=8080                    # containers publishing port 8080
docker ps -a --filter exited=137                   # by exit code
docker ps --filter health=unhealthy
docker ps -a --filter label=env=prod               # by label (--label env=prod at run time)
```

Filters of the same kind are OR-ed; different kinds are AND-ed. `name=` matches **substrings** (use `name=^web$` for an exact match).

### Formatting

```bash
docker ps --format 'table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}'
docker ps -a --format '{{.Names}}: {{.State}}'
docker ps --format json                 # one JSON object per line (newer versions)
```

### Reading `STATUS`

```
Up 2 hours                  running for 2 hours
Up 5 seconds (healthy)      has a health check, currently passing
Exited (0) 3 minutes ago    ended normally
Exited (137) 1 minute ago   killed (128 + 9): OOM, docker kill, or stop timeout
Restarting (1) 4 seconds    crash loop with a restart policy
Created                     never started
Paused                      frozen
```

---

## 3. Names

Docker auto-generates names (`vigilant_tesla`). Use `--name` for anything you'll touch again: descriptive and short (`postgres-dev`, `api`, `redis-cache`).

```bash
docker run -d --name web nginx:1.27-alpine
docker rename web frontend               # you can rename later
docker stop frontend
```

Names must be **unique** among all containers, *including stopped ones*. That's why re-running `docker run --name web ...` fails with a conflict until you `docker rm web`. Anywhere a command accepts a container you can give the **name**, the **full ID**, or a **unique ID prefix**.

Labels add searchable metadata: `docker run -d --label team=backend --label env=dev ...`, then `docker ps --filter label=team=backend`.

---

## 4. `create` and `start`

`docker run` is shorthand. Splitting it is occasionally useful (create everything first, start later, or copy files in before the first start):

```bash
docker create --name web -p 8080:80 nginx:1.27-alpine    # prints the ID; state: created
docker cp ./index.html web:/usr/share/nginx/html/index.html   # works even before the first start
docker start web
curl -s localhost:8080
```

- `docker start` re-runs the **same container** with the **same configuration** it was created with (same ports, env, mounts, and its own writable layer).
- `start` runs detached by default; `-a` attaches output, `-i` attaches stdin.

---

## 5. Stopping

```bash
docker stop web           # SIGTERM → wait 10 s → SIGKILL
docker stop -t 30 web     # wait up to 30 s before killing
docker stop -s SIGINT web # send a different first signal (or use STOPSIGNAL in the Dockerfile)
docker kill web           # SIGKILL immediately
docker kill -s HUP web    # send any signal (e.g. ask nginx to reload)
docker stop web1 web2     # several at once
docker stop $(docker ps -q)   # stop everything running
```

What you should know:

- `stop` is **graceful**: the app gets SIGTERM and time to finish requests, flush data, and close connections. If it doesn't exit in time it's SIGKILLed. Data-loss risks come from apps that ignore SIGTERM.
- **Exit code**: `0` = clean exit; `143` = ended on SIGTERM (128+15); `137` = SIGKILLed (128+9; `docker kill`, OOM killer, or the stop timeout ran out); `130` = SIGINT (Ctrl+C). If your `docker stop` *always* takes 10 s and ends in 137, the app isn't handling SIGTERM as PID 1 (Chapter 16).
- Stopping does **not** delete anything: the container's file system, logs and configuration stay.

Pause instead of stop when you just want to freeze a container (for example to take a consistent snapshot):

```bash
docker pause web && docker ps      # STATUS: Up ... (Paused)
docker unpause web
```

---

## 6. Restarting, and what a restart does not do

```bash
docker restart web        # stop (with timeout) then start
docker restart -t 5 web
```

Good for: recovering a hung process, reloading an app that reads its config **from files inside the container or from a mounted volume** at start-up.

> ⚠️ **A restart does NOT apply changes to the container's configuration.** The port mappings, environment variables (`-e`), mounts (`-v`), image, command and network are fixed when the container is **created**. To change them you must **remove and re-create** the container with `docker run` (or `docker compose up -d`, which recreates when the definition changes).
>
> Exceptions that you *can* change live with **`docker update`**: restart policy, memory/CPU limits, `--pids-limit`:
>
> ```bash
> docker update --restart unless-stopped --memory 512m --memory-swap 512m web
> ```

Similarly, **restarting a container does not pick up a newly built image.** A container keeps using the image it was created from, even if you rebuild the tag. You must create a new container from the new image.

---

## 7. Removing

```bash
docker rm web                 # remove a STOPPED container
docker rm -f web              # kill (SIGKILL) if running, then remove
docker rm -v web              # also remove its ANONYMOUS volumes
docker rm web1 web2 web3
```

Removing deletes the container's **writable layer** (files it created or changed), its logs, and its metadata. It does **not** delete:

- the **image** (`docker rmi`),
- **named volumes** (`docker volume rm`) or **bind-mounted** host folders,
- other containers.

### What survives what

| Action | Container's own files (writable layer) | Volumes / bind mounts | Logs | Running processes / memory |
|---|---|---|---|---|
| `stop` → `start` | **Kept** | Kept | Kept | Lost (process restarts fresh) |
| `restart` | **Kept** | Kept | Kept | Lost |
| `pause` → `unpause` | Kept | Kept | Kept | **Kept** (frozen) |
| `rm` | **Deleted** | Kept (named) / Kept (bind) | Deleted | n/a |
| Re-create with `docker run` | Starts empty from the image | Reattached if you mount again | New | n/a |

The lesson: treat containers as **disposable**. Anything that must survive goes in a **volume** (`-v data:/var/lib/...`) or an external service. Then you can delete and recreate containers freely.

---

## 8. Looking inside

### Logs (Chapter 18)
```bash
docker logs -f --tail 50 web
```

### Inspect: the full configuration and state

```bash
docker inspect web                                     # big JSON
docker inspect -f '{{.State.Status}}' web              # running
docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}' web
docker inspect -f '{{.State.StartedAt}}' web
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web
docker inspect -f '{{json .HostConfig.PortBindings}}' web
docker inspect -f '{{json .Mounts}}' web
docker inspect -f '{{json .Config.Env}}' web           # environment (may show secrets!)
docker inspect --type image nginx:1.27-alpine          # inspect an image instead
```

### Resource usage
```bash
docker stats                    # live table for all running containers (Ctrl+C to leave)
docker stats --no-stream web    # single snapshot
docker top web                  # processes inside (as seen from the host)
```

### File-system changes
```bash
docker diff web                 # A added, C changed, D deleted files vs the image
docker cp web:/etc/nginx/nginx.conf ./nginx.conf     # copy OUT
docker cp ./nginx.conf web:/etc/nginx/nginx.conf     # copy IN
docker export web -o web-fs.tar                      # tar of the container's file system (no metadata)
```

### Events
```bash
docker events --filter container=web        # live stream: create, start, die, oom, ...
docker wait web                             # blocks until it exits, prints the exit code
```

### Run something inside (Chapter 14)
```bash
docker exec -it web sh
docker exec web nginx -t
```

---

## 9. Cleaning up

Stopped containers, unused images, and old volumes/networks accumulate. See what is using disk first:

```bash
docker system df            # summary: images, containers, volumes, build cache
docker system df -v         # per-item detail
docker ps -a -s             # each container's writable-layer size
```

### Remove containers

```bash
docker container prune                     # all STOPPED containers (asks to confirm; -f to skip)
docker container prune --filter until=24h  # only those stopped more than 24 h ago
docker rm $(docker ps -aq --filter status=exited)      # same idea, manual
```

Scripting notes:

- If `docker ps -aq ...` returns nothing, `docker rm $(...)` complains about missing arguments. Use `xargs -r` (GNU) to skip empty input: `docker ps -aq --filter status=exited | xargs -r docker rm`.
- `docker rm -f $(docker ps -aq)` deletes **everything**, running included. Be deliberate.
- `--filter before=<container>`/`since=<container>` filter by *another container's creation*, not by time.

### Auto-remove: `--rm`
```bash
docker run --rm -it alpine:3.20 sh          # gone as soon as it exits
```
Best for one-off commands, experiments and CI jobs. (Don't combine `--rm` with `--restart`.)

### Other cleanup

```bash
docker image prune            # dangling images (untagged leftovers)
docker image prune -a         # all images not used by any container
docker volume prune           # unused volumes (DATA LOSS if it held something you wanted)
docker network prune          # unused networks
docker builder prune          # build cache
docker system prune           # stopped containers + unused networks + dangling images + build cache
docker system prune -a --volumes    # everything unused; be very careful
```

---

## 10. Common patterns

### Development: replace the container after rebuilding

```bash
docker build -t myapp:dev .
docker rm -f dev-server 2>/dev/null
docker run -d --name dev-server -p 8080:8080 myapp:dev
```

(Or, with Compose: `docker compose up -d --build`.)

### Batch operations

```bash
for i in 1 2 3; do docker run -d --name worker-$i myworker:1.0; done
docker ps --filter name=worker
docker stop $(docker ps -q --filter name=worker)
docker rm   $(docker ps -aq --filter name=worker)
```

### Manual "blue-green" upgrade of a service (single host)

```bash
docker run -d --name api-v2 -p 8081:8080 api:2.0     # new version next to the old
curl -fsS localhost:8081/health                       # verify
# switch your reverse proxy / load balancer to :8081, then
docker stop api-v1 && docker rm api-v1
```

Plain Docker has no built-in rolling updates or auto-healing; that is what Compose, Swarm and Kubernetes add.

### Production-oriented run

```bash
docker run -d --name prod-api \
  --restart unless-stopped \
  --memory 512m --cpus 1 \
  --log-opt max-size=10m --log-opt max-file=3 \
  --read-only --tmpfs /tmp \
  -p 127.0.0.1:8080:8080 \
  myorg/api:2.3.1
```

(Immutable version tags, restart policy, limits, log rotation, read-only file system, bound to localhost behind a reverse proxy. Chapters 13 and 18 explain the individual pieces.)

---

## 11. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Container disappeared from `docker ps` | It exited. `docker ps -a` | `docker logs`, `inspect` for exit code |
| `Cannot remove container ... is running` | Running | `docker stop` then `rm`, or `rm -f` |
| `Conflict. The container name "/x" is already in use` | Existing container (maybe stopped) has the name | `docker rm x`, choose another name, or use `--rm` |
| `docker rm $(docker ps -aq)` → "requires at least 1 argument" | Empty list | `xargs -r`, or check the list first |
| I changed `-e`/`-p`/`-v` but nothing changed after `restart` | Config is fixed at creation | Remove and re-run with the new options |
| I rebuilt the image but the container runs old code | Container still uses the old image | Remove the container, run again from the new image |
| `docker stop` always takes 10 s | App ignores SIGTERM (PID 1) | Handle SIGTERM, use exec form, or `--init` |
| Exit 137 with `OOMKilled: true` | Memory limit exceeded | Raise `--memory` or fix the leak |
| Status `Restarting` forever | Crash loop + restart policy | `docker logs`; fix the crash; `docker update --restart no` |
| Disk full | Old containers, images, logs, volumes | `docker system df`, then prune |
| Lost data after removing a container | It was in the writable layer | Use volumes next time |
| `docker exec` says "container is not running" | It exited | Check `docker ps -a`, logs |

---

## 12. Command reference

| Task | Command |
|---|---|
| List (running / all / IDs / filtered) | `docker ps` · `-a` · `-q` · `--filter k=v` |
| Create without starting | `docker create --name N IMG` |
| Start / stop / restart / kill | `docker start N` · `stop N` · `restart N` · `kill N` |
| Freeze / unfreeze | `docker pause N` · `docker unpause N` |
| Rename | `docker rename OLD NEW` |
| Change limits/restart policy live | `docker update --restart ... --memory ... N` |
| Remove (force / with anonymous volumes) | `docker rm N` · `rm -f N` · `rm -v N` |
| Output / follow | `docker logs -f --tail 100 N` |
| Details / one field | `docker inspect N` · `-f '{{...}}'` |
| CPU/memory | `docker stats [--no-stream]` |
| Processes | `docker top N` |
| Changed files | `docker diff N` |
| Copy files | `docker cp N:/path ./` · `docker cp ./f N:/path` |
| Events / wait for exit | `docker events` · `docker wait N` |
| Run inside | `docker exec -it N sh` |
| Cleanup | `docker container prune` · `docker system prune` · `docker system df` |

---

## 13. Summary

- Containers move through **created → running → (paused) → exited → removed**; `docker run` creates *and* starts a **new** container each time.
- `ps -a`, `--filter`, `--format`, `-q` help you find and script; **name** your containers.
- `stop` (SIGTERM, then SIGKILL after 10 s) is graceful; `kill` is immediate; **exit codes** (0, 130, 137, 143) tell you why a container ended.
- A **restart keeps configuration and files** but doesn't apply new options or images; **re-create** for that (`docker update` covers only limits and restart policy).
- A **removal deletes** the writable layer and logs, but not images or volumes. Keep valuable data in volumes.
- Inspect with `logs`, `inspect`, `stats`, `top`, `diff`, `cp`, `events`; clean up with `prune` and `--rm`.

---

## 14. Check your understanding

1. What is the difference between `docker start` and `docker run`?
2. You changed a container's environment variable by editing your script and ran `docker restart web`. The app still sees the old value. Why?
3. What happens to files a container wrote when you (a) stop and start it, (b) remove it?
4. What does exit code 137 mean, and how can you tell whether it was caused by the OOM killer?
5. Write a command that removes all containers created from the image `myapp:dev`, whether running or not.
6. Which command shows what files a container changed relative to its image?
7. Why can't you create a second container named `web` when the first one has exited?

<details>
<summary>Answers</summary>

1. `start` restarts an *existing* stopped container; `run` creates a *new* container from an image and starts it.
2. A container's environment is fixed when it is created. `restart` reuses it. You must remove and re-create the container.
3. (a) They're kept. (b) They're deleted (volumes and bind mounts are not).
4. The process was killed by SIGKILL (128+9). `docker inspect -f '{{.State.OOMKilled}}' NAME` prints `true` if it was the out-of-memory killer.
5. `docker ps -aq --filter ancestor=myapp:dev | xargs -r docker rm -f`
6. `docker diff NAME`
7. Names are unique across *all* containers, including stopped ones; remove the old one first (or use `--rm`).
</details>

**Practice**

1. Create three named containers from `nginx:1.27-alpine` publishing ports 8081–8083. List them with a custom `--format`. Stop one, restart another, kill the third, and check each one's exit code with `inspect`.
2. Prove what a restart keeps: `docker exec web sh -c 'echo hi > /tmp/marker'`, `docker restart web`, then `docker exec web cat /tmp/marker`; then `docker rm -f web`, re-run, and check again.
3. Change a container's memory limit and restart policy live with `docker update`; confirm with `docker inspect`.
4. Use `docker diff` and `docker cp` to fetch nginx's config out of a running container, edit it, and copy it back; reload with `docker exec web nginx -s reload`.
5. Run `docker system df` before and after `docker container prune` and `docker image prune`.

---

**Next:** [Chapter 20 – The Magic Behind the Dockerfile (how builds work)](20_building_magic_behind_dockerfile.md)
