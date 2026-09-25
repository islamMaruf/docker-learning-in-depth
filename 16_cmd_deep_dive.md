# Chapter 16: `CMD` (and `ENTRYPOINT`): What Runs When a Container Starts

> **In one sentence:** `CMD` stores the **default command** in an image so that `docker run IMAGE` starts your application by itself; you can replace it at run time, and `ENTRYPOINT` is its stricter sibling that fixes *what program* runs.

**Level:** 🟢 Beginner → 🟡 Intermediate → 🔴 Expert (signals, PID 1, forms) · **Reading time:** ~45 minutes

**Prerequisites:** [Chapter 15](15_towards_the_dockerfile.md) (Dockerfile basics: `FROM`, `RUN`, `WORKDIR`, `COPY`).

---

## What you will learn

- Why an image needs a default command
- **Build time vs run time**: `RUN` vs `CMD`
- The two syntaxes, **exec form** (`["a","b"]`) and **shell form** (`a b`), and the practical differences
- How to **override** `CMD` from the command line
- What `ENTRYPOINT` does and how it combines with `CMD`
- Why containers must run in the **foreground**, and what **PID 1** and signals mean (`docker stop`)
- How to inspect what an image will run, and how to debug "container exits immediately"

---

## 1. The problem

Our Go image from Chapter 15 contains Go and `server.go`, but starting the server still required someone to know the right command. If the Dockerfile has no default command, running it just gives the base image's default (for Ubuntu: `bash`, which exits immediately when no terminal is attached).

We want:

```bash
docker run go-server:1.0.0      # the server just starts
```

---

## 2. What `CMD` does

`CMD` records **the default command (and arguments) that a container will run when it starts**. It does **not** run during the build.

```
Dockerfile ──docker build──► Image (CMD saved as metadata) ──docker run──► Container (CMD executes now)
```

|  | `RUN` | `CMD` |
|---|---|---|
| Runs during | **Build** (`docker build`) | **Container start** (`docker run`) |
| Purpose | Prepare the image (install, create files) | Say what to run by default |
| Effect on image | Adds a **layer** with file changes | Adds **metadata** (a zero-size history entry) |
| How many take effect | All of them, in order | **Only the last one** |

Example:

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y --no-install-recommends golang-go && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY server.go ./
CMD ["go", "run", "server.go"]
```

```bash
docker build -t go-server:1.0.1 .
docker run --rm -p 8080:8080 go-server:1.0.1
# Server listening on port 8080...
```

While it runs, in another terminal:

```bash
docker ps
# COMMAND column shows: "go run server.go"      ← your CMD
curl http://localhost:8080          # Hello World
```

**Note:** the terminal is "held" by the server. That is normal: the container's main process is running in the foreground. (Detached mode, Chapter 18, runs it in the background.)

---

## 3. Exec form vs shell form

`CMD` (and `RUN` and `ENTRYPOINT`) can be written two ways:

| Form | Syntax | How Docker runs it |
|---|---|---|
| **Exec form** (recommended for `CMD`) | `CMD ["go", "run", "server.go"]` | Runs the program **directly**, with no shell. It is a JSON array: double quotes only |
| **Shell form** | `CMD go run server.go` | Runs `/bin/sh -c "go run server.go"` |

### Consequences

| | Exec form | Shell form |
|---|---|---|
| Needs `/bin/sh` in the image? | No (works in distroless/`scratch`) | **Yes** |
| Environment variable expansion (`$PORT`) | **No** (no shell to expand) | Yes |
| Pipes, `&&`, redirects (`a \| b`, `a && b`) | No | Yes |
| PID 1 is | **Your program** | The shell (`sh`), with your program as a child |
| Receives `docker stop` signal (SIGTERM) | **Yes**, directly | Often **no**: `sh` doesn't forward it, so the container is killed after the timeout |

Rules of thumb:

- Use **exec form** for the main process, so your app gets shutdown signals.
- If you *need* variable expansion or shell features, either use shell form deliberately, or call a shell explicitly in exec form: `CMD ["sh", "-c", "exec myapp --port $PORT"]` (the `exec` replaces the shell with your program so signals work).
- **Quote each argument separately**. This is a very common mistake:

```dockerfile
CMD ["go run server.go"]           # WRONG: looks for a program literally named "go run server.go"
CMD ["go", "run", "server.go"]     # RIGHT
```

  The wrong form fails with `exec: "go run server.go": executable file not found in $PATH`.

- In JSON form, use straight double quotes (`"`), not single quotes or fancy quotes. If the JSON is invalid, Docker silently treats the line as *shell form* and things behave differently.

A third variant, `CMD ["param1", "param2"]` (arguments only), supplies default arguments to `ENTRYPOINT` (section 6).

---

## 4. Overriding `CMD`

Anything you write **after the image name** in `docker run` replaces the `CMD`:

```bash
docker run go-server:1.0.1                      # runs the CMD: go run server.go
docker run --rm -it go-server:1.0.1 bash        # runs bash instead
docker run --rm go-server:1.0.1 go version      # one-off command
docker run --rm go-server:1.0.1 ls -la /app
docker run --rm go-server:1.0.1 env
```

**When to override**

- **Debugging:** `docker run -it --rm IMAGE bash` (or `sh`) to look around, check files and run the app by hand to see the error.
- **One-off tasks:** database migrations, running tests, printing a version.
- **Development vs production:** the same image with different commands.

If you find yourself *always* overriding the default, the default is wrong.

To see or override at build time what an image will do:

```bash
docker inspect -f '{{json .Config.Cmd}}' go-server:1.0.1        # ["go","run","server.go"]
docker inspect -f '{{json .Config.Entrypoint}}' go-server:1.0.1 # null (none set)
docker history go-server:1.0.1                                  # the CMD shows as a 0B step
```

---

## 5. Containers need a **foreground** process

A container lives as long as its **main process** (the command from `CMD`/`ENTRYPOINT`). When that process exits, the container stops. Consequences:

- Programs that **daemonize** (fork into the background and exit) make the container exit right away. Run them in the foreground:

```dockerfile
# WRONG: nginx daemonizes; the container exits immediately   (if you were writing your own image)
CMD ["nginx"]
# RIGHT
CMD ["nginx", "-g", "daemon off;"]
```

  (The official `nginx` image already does this for you.)

- **Never** run `service foo start` or `foo &` as the CMD. It starts something in the background and exits.
- For a script that does setup and then runs the app, end the script with **`exec app`** so the app replaces the script as the main process.
- **Logs** should go to **stdout/stderr**, not files, so `docker logs` can show them.

### "It exits immediately": how to debug

```bash
docker ps -a                    # STATUS: Exited (0) or Exited (1) ...
docker logs <container>         # what the process printed
docker inspect -f '{{.State.ExitCode}} {{.State.Error}}' <container>
docker run -it --rm --entrypoint sh IMAGE      # get a shell instead of the configured command
```

| Exit code | Typical meaning |
|---|---|
| **0** | The command finished successfully (it just wasn't a long-running one) |
| **1** | The application failed |
| **126** | Command found but not executable (missing `x` permission) |
| **127** | Command not found (typo, not installed, or wrong exec-form quoting) |
| **137** | Killed (128+9): out of memory, or `docker stop` timed out |
| **143** | Terminated by SIGTERM (128+15), which is a normal graceful stop |

---

## 6. `ENTRYPOINT`: the fixed part of the command

`ENTRYPOINT` also defines what runs at start-up, but it is **harder to override**. Together, the two are combined like this:

```
   final command  =  ENTRYPOINT  +  CMD (or the arguments you type after the image name)
```

Think of **ENTRYPOINT as "the program"** and **CMD as "its default arguments"**.

```dockerfile
FROM alpine:3.20
ENTRYPOINT ["ping", "-c", "3"]
CMD ["localhost"]
```

```bash
docker build -t pinger .
docker run --rm pinger                 # ping -c 3 localhost        (uses default CMD)
docker run --rm pinger example.com     # ping -c 3 example.com      (your args replace CMD only)
docker run --rm --entrypoint sh -it pinger    # replace the ENTRYPOINT itself (needs --entrypoint)
```

### How they combine (exec forms)

| | No ENTRYPOINT | `ENTRYPOINT ["e"]` |
|---|---|---|
| **No CMD** | Nothing (error: no command) | `e` |
| **`CMD ["c"]`** | `c` | `e c` |
| `docker run IMG x y` | `x y` (replaces CMD) | `e x y` (appended) |

(If you use *shell form* for `ENTRYPOINT`, CMD and run-time arguments are **ignored**, another reason to prefer exec form.)

### `CMD` vs `ENTRYPOINT`: which to use?

| Use | When |
|---|---|
| **`CMD` alone** | Most applications: a sensible default that people may replace (`bash`, `sh`, a test command) |
| **`ENTRYPOINT` + `CMD`** | The image *is* a tool (like `pinger`, `curl`, `terraform`): users supply just arguments |
| **`ENTRYPOINT` script + `CMD`** | Run **setup**, then start the app: the script ends with `exec "$@"` |

The **entrypoint-script pattern** is used by many official images:

`docker-entrypoint.sh`:

```sh
#!/bin/sh
set -e
echo "Preparing configuration..."
# ... setup work: generate config from environment variables, wait for a database, etc.
exec "$@"          # replace this script with the CMD, so the app is PID 1 and gets signals
```

`Dockerfile`:

```dockerfile
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["go", "run", "server.go"]
```

Now `docker run IMG` runs setup and then the server, while `docker run IMG sh` runs setup and then a shell.

---

## 7. PID 1, signals and graceful shutdown (intermediate → expert)

- The main process is **PID 1** inside the container's PID namespace (Chapter 4).
- `docker stop` sends **SIGTERM** to PID 1, waits (default **10 seconds**, `-t` to change), then sends **SIGKILL**.
- The Linux kernel gives PID 1 special treatment: **signals with no handler are ignored** for PID 1 (unlike other processes). So an app that doesn't install a SIGTERM handler won't stop on SIGTERM when it runs as PID 1, and you'll see a 10-second delay followed by exit code **137**.
- With **shell form**, `sh` is PID 1 and usually **doesn't forward** signals to your app. Use exec form.
- PID 1 must also **reap zombie processes** (children that exit). Programs that spawn children may need a tiny init:

```bash
docker run --init IMAGE          # Docker injects a small init (tini) as PID 1
```

```dockerfile
# or bake it in
RUN apt-get update && apt-get install -y --no-install-recommends tini && rm -rf /var/lib/apt/lists/*
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["myapp"]
```

- The stop signal can be changed with `STOPSIGNAL` in the Dockerfile (nginx uses SIGQUIT for graceful stop).
- Test graceful shutdown: `docker run -d --name s IMAGE; time docker stop s`. It should take well under 10 seconds. If it takes ~10 seconds, your app isn't handling SIGTERM as PID 1.

---

## 8. Common patterns

```dockerfile
# 1. Run an application with fixed arguments
CMD ["python", "app.py", "--host", "0.0.0.0", "--port", "8080"]

# 2. Run a script (make sure it is executable and has a #! line)
COPY start.sh /app/start.sh
RUN chmod +x /app/start.sh
CMD ["/app/start.sh"]

# 3. Need several steps? Put them in ONE script (end with exec)
# start.sh:  #!/bin/sh
#            python setup.py && exec python app.py

# 4. Default to a shell for a "toolbox" image
CMD ["bash"]

# 5. Use a variable in the command (requires a shell)
CMD ["sh", "-c", "exec python app.py --port ${PORT:-8080}"]
```

Why not `CMD ["python setup.py && python app.py"]`? Exec form has no shell, so `&&` is just a weird argument. Use a script or an explicit `sh -c`.

**Multiple `CMD` lines:** only the **last** counts, so write exactly one. The same goes for `ENTRYPOINT`.

---

## 9. Hands-on lab

**Lab 1: a `CMD` that runs automatically**

```bash
mkdir cmd-lab && cd cmd-lab
cat > Dockerfile << 'EOF'
FROM alpine:3.20
CMD ["echo", "Hello from CMD"]
EOF
docker build -t cmd-lab .
docker run --rm cmd-lab                       # Hello from CMD
docker run --rm cmd-lab echo "I replaced it" # I replaced it
docker run --rm cmd-lab uname -a              # any command works
```

**Lab 2: exec form vs shell form (variable expansion)**

```bash
cat > Dockerfile << 'EOF'
FROM alpine:3.20
ENV NAME=world
CMD echo "shell form: hello $NAME"
EOF
docker build -t form-shell . && docker run --rm form-shell     # shell form: hello world

cat > Dockerfile << 'EOF'
FROM alpine:3.20
ENV NAME=world
CMD ["echo", "exec form: hello $NAME"]
EOF
docker build -t form-exec . && docker run --rm form-exec       # exec form: hello $NAME   (no expansion!)
```

**Lab 3: the quoting mistake**

```bash
printf 'FROM alpine:3.20\nCMD ["echo hi"]\n' > Dockerfile
docker build -t bad . && docker run --rm bad
# docker: Error response from daemon: ... exec: "echo hi": executable file not found in $PATH
```

**Lab 4: see who is PID 1**

```bash
printf 'FROM alpine:3.20\nCMD sleep 300\n' > Dockerfile      # shell form
docker build -t pid-shell . && docker run -d --name p1 pid-shell
docker exec p1 ps                    # PID 1 is "/bin/sh -c sleep 300", with sleep as PID 7 or so
docker rm -f p1

printf 'FROM alpine:3.20\nCMD ["sleep","300"]\n' > Dockerfile   # exec form
docker build -t pid-exec . && docker run -d --name p2 pid-exec
docker exec p2 ps                    # PID 1 is "sleep 300"
time docker stop p2                  # BusyBox sleep has no handler as PID 1: takes ~10 s, exit 137
docker rm -f p2
```

**Lab 5: ENTRYPOINT + CMD**

```bash
cat > Dockerfile << 'EOF'
FROM alpine:3.20
ENTRYPOINT ["echo", "Result:"]
CMD ["default"]
EOF
docker build -t ep . 
docker run --rm ep              # Result: default
docker run --rm ep custom       # Result: custom
docker run --rm --entrypoint date ep   # runs date instead
```

**Lab 6: the Go server**: add `CMD ["go","run","server.go"]` to the Chapter 15 Dockerfile, build, run with `-p 8080:8080`, and `curl` it. Then run `docker run --rm -it go-server:1.0.1 bash` and start the server by hand to see how override and default differ.

---

## 10. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Container exits immediately, `Exited (0)` | The command finished (or daemonized) | Run a long-lived foreground process; check `docker logs` |
| `executable file not found in $PATH` (exit 127) | Wrong exec-form quoting, program not installed, or wrong path | `["prog","arg"]`, install the program, use the full path |
| `permission denied` / exit 126 | Script has no execute bit or wrong `#!` line | `chmod +x` (in the Dockerfile), check the shebang and line endings |
| `no such file or directory` for a script that exists | Windows CRLF line endings in the script's shebang, or wrong interpreter (e.g. `#!/bin/bash` on Alpine) | Convert to LF; use `#!/bin/sh` or install bash |
| `$VARIABLE` printed literally | Exec form doesn't expand variables | Use shell form or `["sh","-c","..."]` |
| `docker stop` hangs 10 s, exit 137 | App as PID 1 ignores SIGTERM or shell form swallows it | Exec form, handle SIGTERM in the app, `--init`, `exec` in scripts |
| The app runs but I can't reach it | Not published, or listening on `127.0.0.1` | `-p host:container`; bind to `0.0.0.0` |
| Running `docker run IMG bash` doesn't start the app | You replaced the CMD | That's expected: omit the command |
| `ENTRYPOINT` ignores my arguments | ENTRYPOINT in shell form | Use exec form |

---

## 11. Best practices

1. **One `CMD`, exec form**, at the end of the Dockerfile.
2. Prefer the app as **PID 1** (or use `--init`), and handle SIGTERM for graceful shutdown.
3. Run in the **foreground**; log to **stdout/stderr**.
4. Use **`ENTRYPOINT` + `CMD`** when the image is a tool; use an **entrypoint script ending in `exec "$@"`** for setup steps.
5. Keep **configuration** in environment variables (`ENV`, `-e`) rather than hard-coding it into the command.
6. Don't rely on overriding: a good default command means `docker run IMAGE` just works.
7. Official images often already define `CMD` (e.g. `node`, `python`, `nginx`); check with `docker inspect` before writing your own.

---

## 12. Summary

- **`CMD`** = the default command a container runs at start-up; **stored** at build time, **executed** at run time; only the last one counts.
- **Exec form** (`["prog","arg"]`) runs directly (signals work, no shell features); **shell form** uses `sh -c` (variables work, but signals may not).
- Arguments after the image name in `docker run` **override** `CMD`.
- **`ENTRYPOINT`** fixes the program; `CMD` (or run-time arguments) supplies default arguments; override it with `--entrypoint`.
- A container lives as long as its **foreground main process (PID 1)**; `docker stop` sends SIGTERM then SIGKILL after 10 s.

---

## 13. Check your understanding

1. When is `RUN` executed, and when is `CMD` executed?
2. What is wrong with `CMD ["python app.py"]`?
3. Given `ENTRYPOINT ["ping","-c","2"]` and `CMD ["localhost"]`, what runs for `docker run IMG example.com`?
4. Why does `CMD nginx` (with a daemonizing nginx) make the container stop immediately?
5. Why might `docker stop` take exactly ten seconds for a container started with shell-form `CMD`?
6. How do you run a shell in an image whose `ENTRYPOINT` is a fixed program?
7. Does `CMD ["echo", "$HOME"]` print the home directory?

<details>
<summary>Answers</summary>

1. `RUN` at image build time; `CMD` when a container starts.
2. The whole string is treated as one executable name. Use `["python", "app.py"]`.
3. `ping -c 2 example.com` (your argument replaces the CMD).
4. The nginx process forks to the background and the original main process exits, so the container stops. Run it in the foreground with `daemon off;`.
5. `sh` is PID 1 and doesn't forward SIGTERM to the app, so Docker waits the full timeout and then kills it (exit code 137).
6. `docker run -it --entrypoint sh IMG`.
7. No. Exec form has no shell, so it prints the literal `$HOME`. Use `["sh","-c","echo $HOME"]`.
</details>

**Practice**

1. Write a Dockerfile for a small Python (or Node) HTTP server that starts with `docker run` and no extra arguments; test it with `curl`.
2. Make an image `greeter` with `ENTRYPOINT ["echo","Hello,"]` and `CMD ["World"]`; run it with and without an argument.
3. Write an entrypoint script that prints a message, then does `exec "$@"`; verify that `docker exec IMG ps` shows your app (not the script) as PID 1.
4. Time `docker stop` for shell-form vs exec-form containers running a program that handles SIGTERM (`python -c "import signal,time; signal.signal(signal.SIGTERM, lambda *a: exit(0)); time.sleep(999)"`).

---

**Next:** [Chapter 17 – `WORKDIR` Deep Dive](17_workdir_deep_dive.md)
