# Chapter 23: CMD Deep Dive - Default Container Commands

## Overview

In the previous chapter, we learned how to automate container creation using Dockerfiles. We wrote a Dockerfile that installed Go, copied our server code, and set up the working directory. But there was still one manual step: after starting the container, we had to run `go run server.go` ourselves.

This chapter solves that final piece of the puzzle. We'll learn about the `CMD` instruction—one of the most important Dockerfile commands—which tells Docker what command to run automatically when a container starts. By the end of this chapter, you'll be able to create containers that start your application automatically, without any manual intervention.

### What You'll Learn

- The problem with manual command execution
- What the `CMD` instruction does and when it executes
- How to write `CMD` instructions in JSON array format
- The difference between build-time and runtime execution
- How `CMD` can be overridden
- When to override vs when to keep defaults
- Practical patterns for using `CMD` effectively

### Prerequisites

- Completed Chapter 22 (Towards The Dockerfile)
- Understanding of `FROM`, `RUN`, `WORKDIR`, and `COPY` instructions
- Familiarity with `docker build` and `docker run`
- Basic understanding of how containers start

---

## The Problem: Manual Command Execution

Let's revisit where we left off in Chapter 22. We had this Dockerfile:

```dockerfile
FROM ubuntu:24.04
RUN apt update
RUN apt install -y golang
WORKDIR /app
COPY . ./server.go
```

When we built and ran this image:

```bash
docker build -t new-go-server:1.0.0 .
docker run -it new-go-server:1.0.0 bash
```

We got a bash shell inside the container, but our Go server wasn't running. We had to manually execute:

```bash
go run server.go
```

### Why Is This a Problem?

**1. Not Truly Automated**: You still need to know what command to run

**2. Error-Prone**: Easy to forget the command or type it incorrectly

**3. Not Production-Ready**: In production, containers should start applications automatically

**4. Defeats the Purpose**: If we're automating setup, why not automate execution too?

**5. Scalability Issues**: Imagine deploying 100 containers and manually starting each one

### What We Want

We want this workflow:

```bash
docker run new-go-server:1.0.1
# Server automatically starts and listens on port 8080
# No manual commands needed!
```

This is where `CMD` comes in.

---

## Understanding CMD: The Default Command

The `CMD` instruction in a Dockerfile specifies the **default command** to run when a container starts from your image.

### Key Concepts

**1. Build Time vs Runtime**

- `RUN` executes during **build time** (when you run `docker build`)
- `CMD` executes at **runtime** (when you run `docker run`)

**2. Execution Timing**

```
Dockerfile → docker build → Image (CMD stored, not executed)
Image → docker run → Container (CMD executes now!)
```

**3. Default Nature**

`CMD` provides a **default** command. It can be overridden when you run the container.

### Basic Syntax

The `CMD` instruction has several formats, but the recommended one is the **JSON array format**:

```dockerfile
CMD ["executable", "param1", "param2"]
```

For our Go server:

```dockerfile
CMD ["go", "run", "server.go"]
```

---

## Adding CMD to Our Dockerfile

Let's update our Dockerfile to include the `CMD` instruction:

```dockerfile
FROM ubuntu:24.04

RUN apt update

RUN apt install -y golang

WORKDIR /app

COPY . ./server.go

CMD ["go", "run", "server.go"]
```

### Understanding the CMD Line

```dockerfile
CMD ["go", "run", "server.go"]
```

**Breaking it down**:

- `CMD` - The Dockerfile instruction
- `[...]` - JSON array syntax (square brackets)
- `"go"` - The executable/program to run
- `"run"` - First argument to the go command
- `"server.go"` - Second argument (the file to run)

**Why JSON array format?**

The command `go run server.go` has three parts:
1. `go` - the command
2. `run` - first argument
3. `server.go` - second argument

Each part goes in its own string inside the array, separated by commas.

### Alternative Formats (Not Recommended)

You might see these formats, but they're less explicit:

```dockerfile
# Shell form (not recommended)
CMD go run server.go

# Exec form (recommended - what we use)
CMD ["go", "run", "server.go"]
```

**Always use the exec form (JSON array)** for clarity and to avoid shell interpretation issues.

---

## How CMD Works: Build vs Run

Let's trace what happens with our new Dockerfile:

### During `docker build`

```bash
docker build -t new-go-server:1.0.1 .
```

**What Docker does**:

1. **Executes FROM**: Pulls Ubuntu 24.04
2. **Executes RUN apt update**: Updates package lists (creates layer)
3. **Executes RUN apt install**: Installs Go (creates layer)
4. **Executes WORKDIR**: Sets working directory (creates layer)
5. **Executes COPY**: Copies server.go (creates layer)
6. **Stores CMD**: **Does NOT execute**, just saves the command in image metadata

**Critical point**: The `CMD` instruction doesn't run during build! It's stored in the image as metadata saying "when someone runs this image, execute this command."

### During `docker run`

```bash
docker run new-go-server:1.0.1
```

**What Docker does**:

1. Creates a container from the image
2. All layers from build are already present (Ubuntu + Go + /app + server.go)
3. **Now executes CMD**: Runs `go run server.go`
4. Your server starts automatically!

---

## Building and Running with CMD

Let's see this in action step by step.

### Step 1: Create the Updated Dockerfile

Your project structure:

```
docker-learning/
├── Dockerfile
└── server.go
```

Dockerfile content:

```dockerfile
FROM ubuntu:24.04

RUN apt update

RUN apt install -y golang

WORKDIR /app

COPY . ./server.go

CMD ["go", "run", "server.go"]
```

### Step 2: Build the Image

```bash
docker build -t new-go-server:1.0.1 .
```

**What you'll see**:

```
[+] Building 32.5s (9/9) FINISHED
 => [1/5] FROM ubuntu:24.04
 => [2/5] RUN apt update
 => [3/5] RUN apt install -y golang
 => [4/5] WORKDIR /app
 => [5/5] COPY . ./server.go
 => exporting to image
 => => naming to docker.io/library/new-go-server:1.0.1
```

**Notice**: No step for `CMD` execution! It's stored but not run.

### Step 3: Run Without CMD Override

**The magic moment**:

```bash
docker run new-go-server:1.0.1
```

**Output**:

```
Server listening on port 8080...
```

**What happened?**

1. Docker created a container
2. Docker automatically executed: `go run server.go`
3. The server started!
4. The terminal is "held" by the running process

### Step 4: Verify It's Running

Open a new terminal:

```bash
docker ps
```

Output:

```
CONTAINER ID   IMAGE                   COMMAND                  STATUS
a1b2c3d4e5f6   new-go-server:1.0.1    "go run server.go"      Up 20 seconds
```

**See that?** The `COMMAND` column shows what's running: `"go run server.go"`. This is our `CMD` instruction in action!

### Step 5: Test the Server

Get into the running container:

```bash
docker exec -it <container-id> bash
```

Install curl and test:

```bash
apt update
apt install -y curl
curl localhost:8080
# Output: Hello World
```

Success! The server is running, and we never had to manually start it.

---

## CMD Override: Taking Back Control

The `CMD` instruction provides a **default** command, but you can override it if needed.

### Running with the Default CMD

```bash
docker run new-go-server:1.0.1
# Executes: go run server.go (from CMD)
```

### Overriding CMD

```bash
docker run -it new-go-server:1.0.1 bash
# Executes: bash (overrides CMD)
```

**What happens**: When you specify a command after the image name, Docker uses your command instead of the `CMD` from the Dockerfile.

### Understanding the Override

```bash
docker run [OPTIONS] IMAGE [COMMAND]
```

- If you omit `[COMMAND]`, Docker uses `CMD` from Dockerfile
- If you provide `[COMMAND]`, it **overrides** `CMD`

**Examples**:

```bash
# Uses CMD from Dockerfile
docker run new-go-server:1.0.1

# Overrides CMD with bash
docker run -it new-go-server:1.0.1 bash

# Overrides CMD with ls
docker run new-go-server:1.0.1 ls

# Overrides CMD with custom command
docker run -it new-go-server:1.0.1 go version
```

---

## When to Override CMD

### Use Case 1: Debugging

**Scenario**: Your application isn't working correctly, and you want to inspect the container.

**Without override**:
```bash
docker run myapp
# App starts, but you can't explore
```

**With override**:
```bash
docker run -it myapp bash
# You get a shell to investigate
```

**Example debugging session**:

```bash
# Start container with bash instead of app
docker run -it new-go-server:1.0.1 bash

# Check if files are there
ls -la
# Output: server.go

# Check Go version
go version

# Try running manually to see errors
go run server.go
```

### Use Case 2: Running One-Off Commands

**Scenario**: You want to use the container environment for a specific task.

```bash
# Check what version of Go is installed
docker run new-go-server:1.0.1 go version

# List files in the container
docker run new-go-server:1.0.1 ls -la

# Check environment variables
docker run new-go-server:1.0.1 env
```

### Use Case 3: Development vs Production

**During development**:
```bash
# Override to get shell for testing
docker run -it myapp:dev bash
```

**In production**:
```bash
# Use default CMD to start app
docker run myapp:prod
```

### When NOT to Override

**In normal production use**: 99% of the time, you want the default `CMD` to run. Overriding is primarily for:

- Debugging (1% of the time)
- Development testing
- Specific administrative tasks

**Rule of thumb**: If you find yourself frequently overriding `CMD`, you might have the wrong default command in your Dockerfile.

---

## CMD Best Practices

### 1. Always Include CMD

**Bad** (no CMD):
```dockerfile
FROM ubuntu:24.04
RUN apt update && apt install -y python3
COPY app.py /app/
WORKDIR /app
# Missing CMD!
```

Result: Container starts and immediately exits (no command to run).

**Good** (with CMD):
```dockerfile
FROM ubuntu:24.04
RUN apt update && apt install -y python3
COPY app.py /app/
WORKDIR /app
CMD ["python3", "app.py"]
```

Result: Container starts and runs your application.

### 2. Use Exec Form (JSON Array)

**Bad** (shell form):
```dockerfile
CMD go run server.go
```

Problems:
- Shell interpretation can cause issues
- Signal handling doesn't work properly
- Extra shell process running

**Good** (exec form):
```dockerfile
CMD ["go", "run", "server.go"]
```

Benefits:
- Direct execution (no shell)
- Proper signal handling
- Clean process tree

### 3. Quote Each Argument Separately

**Wrong**:
```dockerfile
CMD ["go run server.go"]
# This tries to run a program named "go run server.go" (doesn't exist!)
```

**Correct**:
```dockerfile
CMD ["go", "run", "server.go"]
# This runs: go with arguments: run, server.go
```

### 4. Put CMD at the End

```dockerfile
FROM ubuntu:24.04
RUN apt update && apt install -y golang
WORKDIR /app
COPY . ./server.go
CMD ["go", "run", "server.go"]  # ← At the end
```

Why? It's the last thing that happens conceptually (even though it doesn't execute during build).

### 5. Make CMD Match Your Use Case

**For web servers**:
```dockerfile
CMD ["python", "app.py"]
CMD ["node", "server.js"]
CMD ["go", "run", "server.go"]
```

**For long-running services**:
```dockerfile
CMD ["nginx", "-g", "daemon off;"]
CMD ["redis-server"]
```

**For batch jobs**:
```dockerfile
CMD ["python", "process_data.py"]
```

---

## Common CMD Patterns

### Pattern 1: Running a Script

```dockerfile
FROM ubuntu:24.04
RUN apt update && apt install -y python3
WORKDIR /app
COPY start.sh /app/
RUN chmod +x start.sh
CMD ["./start.sh"]
```

### Pattern 2: Running with Arguments

```dockerfile
FROM python:3.9
WORKDIR /app
COPY app.py /app/
CMD ["python", "app.py", "--host", "0.0.0.0", "--port", "8080"]
```

### Pattern 3: Running Multiple Commands (Using a Script)

You can't directly chain commands in CMD, but you can use a script:

**start.sh**:
```bash
#!/bin/bash
python setup.py
python app.py
```

**Dockerfile**:
```dockerfile
COPY start.sh /app/
RUN chmod +x /app/start.sh
CMD ["/app/start.sh"]
```

### Pattern 4: Background Services

Some applications need to stay in foreground:

```dockerfile
# Wrong - nginx will daemonize and container will exit
CMD ["nginx"]

# Correct - keeps nginx in foreground
CMD ["nginx", "-g", "daemon off;"]
```

---

## Comparing RUN vs CMD

This is crucial to understand:

### RUN Instruction

```dockerfile
RUN apt update
RUN apt install -y golang
```

**Purpose**: Execute commands during image build

**Timing**: Build time (`docker build`)

**Result**: Creates new layer in image

**Usage**: Install software, create files, configure system

**Execution count**: Every time you build (or cached)

**Multiple allowed**: Yes, as many as you need

### CMD Instruction

```dockerfile
CMD ["go", "run", "server.go"]
```

**Purpose**: Specify default command for container startup

**Timing**: Runtime (`docker run`)

**Result**: Process runs in container

**Usage**: Start application, run service

**Execution count**: Every time container starts

**Multiple allowed**: Only one (last one wins)

### Side-by-Side Example

```dockerfile
FROM ubuntu:24.04

# RUN - executes during build
RUN apt update
RUN apt install -y golang

# RUN - executes during build
RUN mkdir /app

# CMD - executes at runtime
CMD ["go", "run", "server.go"]
```

**During build**:
- `RUN` commands execute
- `CMD` is stored but NOT executed

**During run**:
- Container starts with all `RUN` results present
- `CMD` executes

---

## Multiple CMD Instructions

**Important**: If you have multiple `CMD` instructions in a Dockerfile, **only the last one takes effect**.

```dockerfile
FROM ubuntu:24.04
RUN apt install -y golang
WORKDIR /app
COPY server.go ./

CMD ["echo", "Hello"]
CMD ["go", "version"]
CMD ["go", "run", "server.go"]  # ← Only this one is used!
```

Result: Only `go run server.go` executes. The others are ignored.

**Why?** Docker doesn't support multiple default commands. There's one default, which is the last `CMD` specified.

**Best practice**: Only have one `CMD` instruction in your Dockerfile.

---

## CMD Without Interactive Mode

One interesting behavior to understand:

### With Interactive Mode + Override

```bash
docker run -it new-go-server:1.0.1 bash
```

- `-it` keeps container alive with interactive shell
- `bash` overrides CMD
- You get a shell prompt

### Without Interactive Mode, Without Override

```bash
docker run new-go-server:1.0.1
```

- Uses CMD from Dockerfile
- Container runs `go run server.go`
- Terminal is "held" by the process
- Container stays alive as long as the process runs

### What If There's No CMD?

```bash
docker run ubuntu:24.04
```

- Ubuntu image has a default CMD
- Container starts and immediately exits
- No long-running process

**This is why CMD is essential** for your custom images!

---

## Real-World Example: Complete Workflow

Let's walk through a complete example from start to finish.

### Scenario

You're building a production-ready Node.js API server.

### Step 1: Application Code (app.js)

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello from Node.js!');
});

server.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

### Step 2: Dockerfile

```dockerfile
FROM node:18

WORKDIR /app

COPY app.js /app/

CMD ["node", "app.js"]
```

### Step 3: Build

```bash
docker build -t my-node-app:1.0 .
```

### Step 4: Run (Production)

```bash
docker run my-node-app:1.0
# Server automatically starts!
```

### Step 5: Debug (Development)

```bash
# Override CMD to inspect
docker run -it my-node-app:1.0 bash

# Inside container, verify setup
ls -la
node --version
cat app.js

# Manually test
node app.js
```

### Step 6: Test

```bash
# In another terminal
docker ps  # Get container ID
docker exec -it <container-id> bash
curl localhost:3000
# Output: Hello from Node.js!
```

---

## Troubleshooting CMD Issues

### Issue 1: Container Exits Immediately

**Symptoms**:
```bash
docker run myapp
# Container starts and immediately stops
```

**Check**:
```bash
docker ps -a
# Shows: Exited (0) 1 second ago
```

**Causes**:
1. No `CMD` instruction (or default CMD exits quickly)
2. CMD command completes immediately
3. CMD command fails

**Solutions**:
- Add a `CMD` that runs a long-lived process
- Check logs: `docker logs <container-id>`
- Override CMD to debug: `docker run -it myapp bash`

### Issue 2: Wrong Command Format

**Symptoms**: Error like "executable file not found"

**Wrong**:
```dockerfile
CMD ["go run server.go"]
```

**Correct**:
```dockerfile
CMD ["go", "run", "server.go"]
```

### Issue 3: CMD Not Running

**Symptoms**: Container starts but application doesn't

**Cause**: You're overriding CMD without realizing it

**Check**:
```bash
# Are you running with a command at the end?
docker run myapp bash  # ← This overrides CMD!

# Use without override
docker run myapp
```

### Issue 4: Can't Debug Because CMD Runs

**Solution**: Override CMD temporarily

```bash
# Instead of this (runs CMD)
docker run myapp

# Do this (overrides with bash)
docker run -it myapp bash
```

---

## Advanced: CMD vs ENTRYPOINT (Preview)

You might hear about another instruction called `ENTRYPOINT`. Here's a quick comparison:

### CMD

- Provides **default** command
- **Can be overridden** completely
- Good for providing a default that users might change

### ENTRYPOINT

- Provides **fixed** executable
- **Cannot be overridden** easily (without `--entrypoint`)
- Good for commands that should always run

We'll cover `ENTRYPOINT` in depth in a later chapter. For now, `CMD` is perfect for most use cases.

---

## Practical Exercises

### Exercise 1: Basic CMD

1. Create a simple Python script that prints "Hello World"
2. Write a Dockerfile that:
   - Uses `python:3.9` as base
   - Copies the script
   - Uses `CMD` to run it
3. Build and run
4. Verify it prints "Hello World" automatically

### Exercise 2: CMD Override Practice

Using your image from Exercise 1:
1. Run with default CMD
2. Run overriding CMD with `bash`
3. Run overriding CMD with `python --version`
4. Understand when each is useful

### Exercise 3: Web Server

1. Create a simple HTTP server (Python, Node, or Go)
2. Write a Dockerfile with appropriate `CMD`
3. Build and run without `-it` or command override
4. Verify server starts automatically
5. Test it with `curl` from another terminal

### Exercise 4: Debugging

1. Create a Dockerfile with an intentional error in CMD
2. Build the image
3. Try to run it (it will fail)
4. Use CMD override to debug
5. Fix the Dockerfile and rebuild

### Exercise 5: Multiple Processes

1. Create a script that runs two commands sequentially
2. Make the script executable
3. Use CMD to run the script
4. Verify both commands execute

---

## Best Practices Summary

### Writing CMD

1. ✅ **Always use exec form**: `CMD ["executable", "param1"]`
2. ✅ **One CMD per Dockerfile**: Last one wins, so only write one
3. ✅ **Put CMD at the end**: Logical position in Dockerfile
4. ✅ **Quote each element separately**: `["go", "run", "file.go"]` not `["go run file.go"]`
5. ✅ **Make it meaningful**: Run your actual application, not just a shell

### Using CMD

1. ✅ **Normal run**: `docker run image` (uses CMD)
2. ✅ **Debug**: `docker run -it image bash` (overrides CMD)
3. ✅ **One-off commands**: `docker run image ls` (overrides CMD)
4. ❌ **Don't habitually override**: If you always override, fix the Dockerfile

### Common Mistakes

1. ❌ Forgetting CMD entirely
2. ❌ Using shell form instead of exec form
3. ❌ Not quoting arguments separately
4. ❌ Multiple CMD instructions (forgetting only last counts)
5. ❌ CMD that exits immediately (container will stop)

---

## Key Takeaways

### Conceptual Understanding

1. **CMD executes at runtime**, not build time
2. **RUN executes at build time**, not runtime
3. **CMD can be overridden** by providing a command to `docker run`
4. **Only one CMD** is active (the last one in Dockerfile)
5. **CMD is stored in image metadata**, not in a layer

### Practical Skills

1. You can write `CMD` instructions in correct JSON format
2. You can build images with default commands
3. You can run containers that auto-start applications
4. You can override CMD when debugging
5. You can distinguish when to use CMD vs when to override

### Professional Impact

**Before CMD**: Containers need manual command execution
**After CMD**: Containers auto-start applications

This is essential for:
- Production deployments (can't manually start services)
- Container orchestration (Kubernetes, Docker Swarm)
- Scaling (launch many containers automatically)
- CI/CD pipelines (automated deployments)

---

## Visual Understanding: RUN vs CMD Timeline

```
docker build -t myapp .
├── FROM ubuntu:24.04          → Pull base image
├── RUN apt update             → EXECUTES (creates layer)
├── RUN apt install -y golang  → EXECUTES (creates layer)
├── WORKDIR /app               → Creates directory (creates layer)
├── COPY server.go ./          → Copies file (creates layer)
└── CMD ["go", "run", ...]     → STORED (no execution, no layer)
                                 ↓
                            IMAGE CREATED
                                 ↓
docker run myapp
├── Create container from image
├── All RUN results present
└── CMD EXECUTES NOW → go run server.go
                       ↓
                  SERVER RUNNING
```

---

## The Path Forward

### What We've Mastered

In this chapter, you've learned:

✅ What CMD does and when it executes
✅ How to write CMD in JSON array format
✅ The difference between build-time (RUN) and runtime (CMD) execution
✅ How to override CMD for debugging
✅ When to use CMD vs when to override
✅ Best practices for CMD instructions
✅ Common CMD patterns and pitfalls

### Why This Matters

**CMD transforms your containers from interactive environments into self-contained applications.** This is crucial because:

1. **Automation**: Containers can start without human intervention
2. **Orchestration**: Systems like Kubernetes can manage your containers
3. **Scaling**: Deploy hundreds of containers automatically
4. **Reliability**: Consistent startup behavior every time

You've moved from "containers that require manual commands" to "containers that are true application packages."

### What's Next

We've learned several Dockerfile instructions, but there's more to explore:

- **WORKDIR deep dive**: Understanding working directories thoroughly
- **ENTRYPOINT**: The fixed-command alternative to CMD
- **ENV**: Setting environment variables
- **EXPOSE**: Declaring ports
- **Detached mode**: Running containers in background
- **Container management**: Starting, stopping, and monitoring

Each concept builds on what you've learned, creating a complete understanding of container lifecycle management.

---

## Final Thoughts

Remember the journey we've taken:

**Chapter 21**: Manual container operations
**Chapter 22**: Automating setup with Dockerfiles
**Chapter 23**: Automating execution with CMD

We started with completely manual processes and have now automated both **setup** and **execution**. Your containers now:

1. Install dependencies automatically (RUN)
2. Copy files automatically (COPY)
3. Start applications automatically (CMD)

This is the foundation of modern containerization. Every professional Dockerfile includes a `CMD` instruction. Every production container uses it to auto-start services.

You're no longer just using Docker—you're crafting production-ready container images that can be deployed anywhere, scaled infinitely, and managed automatically.

**The CMD instruction is small but mighty**. Three lines in your Dockerfile, but it transforms your containers from development toys into production powerhouses.

Keep building, keep learning, and remember: great Docker images have great default commands!
