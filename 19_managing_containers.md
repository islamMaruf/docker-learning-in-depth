# Chapter 19: Managing Containers - Complete Lifecycle Control

## Overview

Throughout your Docker journey, you've learned to build images, run containers, and use various Docker features. However, managing containers goes far beyond just running them. In production environments and even during development, you need comprehensive control over container lifecycles—starting, stopping, restarting, removing, inspecting, and monitoring containers efficiently.

This chapter focuses on **container management**—the practical, day-to-day commands and workflows you'll use constantly when working with Docker. While individual commands might seem simple, understanding when and how to use them effectively is what separates novice Docker users from professionals.

Think of containers as living entities with lifecycles. They're born (created), live (running), can pause (stopped), come back to life (restarted), and eventually die (removed). Your job as a Docker practitioner is to manage these lifecycles smoothly and efficiently.

In this chapter, we'll cover:
- Complete container lifecycle management (create, start, stop, restart, remove)
- Listing and filtering containers
- Named vs anonymous containers
- Inspecting container details and logs
- Cleaning up and maintaining a healthy Docker environment
- Best practices for production container management
- Common management patterns and workflows
- Troubleshooting container issues

By the end of this chapter, you'll have mastered the essential commands and techniques for managing Docker containers like a professional.

## Prerequisites

To get the most from this chapter, you should have:

- **Docker installed and running** - Docker Desktop or Docker Engine
- **Basic Docker knowledge** - Running containers with `docker run`
- **Previous chapters completed** - Especially understanding of detached mode (Chapter 25)
- **A test image** - Use the Go server image from previous chapters or any Docker image
- **Terminal access** - Comfortable working with command-line interfaces

## Container Lifecycle Overview

Before diving into specific commands, let's understand the container lifecycle.

### The Container States

A Docker container can exist in several states:

```
    Created  →  Running  ←→  Paused
                  ↓
               Stopped
                  ↓
               Removed
```

**State Descriptions:**

1. **Created** - Container exists but not started (rarely used directly)
2. **Running** - Container is executing its process
3. **Paused** - Container execution is temporarily suspended (advanced use case)
4. **Stopped/Exited** - Container has stopped but still exists
5. **Removed** - Container is completely deleted

### State Transitions

```
docker run     → Creates and starts container (Created → Running)
docker stop    → Stops running container (Running → Stopped)
docker start   → Starts stopped container (Stopped → Running)
docker restart → Stops and starts container (Running → Stopped → Running)
docker rm      → Removes stopped container (Stopped → Removed)
docker rm -f   → Forces removal of running container (Running → Removed)
```

Understanding these states and transitions is crucial for effective container management.

## Listing Containers: docker ps

The `docker ps` command is your window into the container world—it shows you what's running, what's stopped, and the state of your Docker environment.

### Basic Usage: Active Containers

```bash
docker ps
```

Shows only **currently running** containers.

**Example output:**

```
CONTAINER ID   IMAGE                    COMMAND              CREATED         STATUS         PORTS     NAMES
a1b2c3d4e5f6   new-go-server:1.0.3     "go run server.go"   2 minutes ago   Up 2 minutes   8080/tcp  vigilant_tesla
```

**Column explanations:**

- **CONTAINER ID**: Short unique identifier (first 12 chars of full ID)
- **IMAGE**: Image used to create container
- **COMMAND**: Command running inside container
- **CREATED**: When container was created
- **STATUS**: Current state and uptime
- **PORTS**: Port mappings (if any)
- **NAMES**: Container name (auto-generated or user-specified)

### Showing All Containers

```bash
docker ps -a
```

Shows **all containers**, including stopped/exited ones.

**Example output:**

```
CONTAINER ID   IMAGE                    STATUS                      NAMES
a1b2c3d4e5f6   new-go-server:1.0.3     Up 5 minutes                vigilant_tesla
b2c3d4e5f6a7   new-go-server:1.0.2     Exited (0) 2 hours ago      romantic_curie
c3d4e5f6a7b8   ubuntu:latest           Exited (130) 1 day ago      awesome_wright
```

This is crucial for seeing containers that have stopped but still exist.

### Useful ps Flags

```bash
# Show only container IDs
docker ps -q

# Show last created container (including stopped)
docker ps -l

# Show n last created containers
docker ps -n 3

# Filter by status
docker ps -a --filter "status=exited"

# Filter by name
docker ps --filter "name=my-server"

# Custom format
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}"
```

**Example: Show only IDs of running containers**

```bash
docker ps -q
```

**Output:**
```
a1b2c3d4e5f6
b2c3d4e5f6a7
```

This is useful for scripting and batch operations.

## Naming Containers: Why It Matters

By default, Docker assigns random names to containers like `romantic_curie` or `vigilant_tesla`. While amusing, these aren't practical for management.

### The Problem with Anonymous Names

```bash
# Run without name
docker run -d new-go-server:1.0.3
a1b2c3d4e5f6...

docker ps
```

**Output:**
```
CONTAINER ID   IMAGE                    NAMES
a1b2c3d4e5f6   new-go-server:1.0.3     suspicious_goldberg
```

Now you must remember or look up `suspicious_goldberg` or use the container ID `a1b2c3d4e5f6` for all operations. Not ideal!

### Using Named Containers

```bash
# Run with explicit name
docker run -d --name my-go new-go-server:1.0.3
```

```bash
docker ps
```

**Output:**
```
CONTAINER ID   IMAGE                    NAMES
b3c4d5e6f7a8   new-go-server:1.0.3     my-go
```

Much better! Now you can reference the container by name:

```bash
docker logs my-go
docker stop my-go
docker start my-go
docker rm my-go
```

### Naming Best Practices

1. **Descriptive names**: `backend-api`, `postgres-db`, `nginx-proxy`
2. **Include environment**: `dev-redis`, `prod-mysql`
3. **Include version if needed**: `api-v2`, `worker-v3`
4. **Keep it short**: `auth-svc` rather than `authentication-service-production-v2`
5. **Avoid spaces**: Use hyphens or underscores

**Examples:**

```bash
docker run -d --name backend-api my-backend:latest
docker run -d --name postgres-dev postgres:15
docker run -d --name redis-cache redis:alpine
```

## Stopping Containers

Stopping a container gracefully shuts it down without deleting it.

### Basic Stop Command

```bash
docker stop <container-id-or-name>
```

**Example:**

```bash
# Using name
docker stop my-go

# Using ID
docker stop a1b2c3d4e5f6
```

**What happens when you stop a container:**

1. Docker sends **SIGTERM** signal to the main process
2. Process has 10 seconds (by default) to gracefully shut down
3. If process doesn't exit, Docker sends **SIGKILL** to force termination
4. Container enters "Exited" state

### Verification

```bash
docker ps  # Container disappears from running list
docker ps -a  # Container appears with "Exited" status
```

**Output:**
```
CONTAINER ID   IMAGE                    STATUS                      NAMES
a1b2c3d4e5f6   new-go-server:1.0.3     Exited (0) 5 seconds ago    my-go
```

The `(0)` indicates the exit code—0 means clean exit.

### Stopping Multiple Containers

```bash
# Stop multiple by name/ID
docker stop container1 container2 container3

# Stop all running containers
docker stop $(docker ps -q)
```

### Custom Timeout

By default, Docker waits 10 seconds. You can customize:

```bash
# Wait 30 seconds before forcing kill
docker stop -t 30 my-go
```

## Starting Containers

The `docker start` command starts a container that was previously stopped.

### Basic Start Command

```bash
docker start <container-id-or-name>
```

**Example:**

```bash
docker ps -a  # Shows my-go is stopped
```

**Output:**
```
CONTAINER ID   IMAGE                    STATUS                     NAMES
a1b2c3d4e5f6   new-go-server:1.0.3     Exited (0) 2 minutes ago   my-go
```

```bash
docker start my-go
```

```bash
docker ps  # Now shows my-go is running
```

**Output:**
```
CONTAINER ID   IMAGE                    STATUS         NAMES
a1b2c3d4e5f6   new-go-server:1.0.3     Up 3 seconds   my-go
```

### Start vs Run

**Important distinction:**

- `docker run` - Creates **new** container from image
- `docker start` - Starts **existing** stopped container

```bash
# Creates NEW container each time
docker run -d --name server1 my-app:latest
docker run -d --name server2 my-app:latest  # Different container

# Starts EXISTING container
docker start server1  # Same container, same data
docker start server1  # Still same container
```

### Starting with Attached Output

By default, `docker start` runs in detached mode. To see output:

```bash
docker start -a my-go
```

The `-a` flag attaches your terminal to the container's output.

### Starting Multiple Containers

```bash
docker start container1 container2 container3
```

## Restarting Containers

The `docker restart` command stops and then starts a container—useful for applying configuration changes or recovering from issues.

### Basic Restart Command

```bash
docker restart <container-id-or-name>
```

**Example:**

```bash
docker restart my-go
```

**What happens:**

1. Container is stopped (SIGTERM, then SIGKILL if needed)
2. Brief pause
3. Container is started again

This is equivalent to:

```bash
docker stop my-go
docker start my-go
```

### When to Use Restart

1. **Container is hanging** - Process is unresponsive
2. **Configuration changes** - After updating environment variables (in some cases)
3. **Memory leaks** - Temporary workaround until fix is deployed
4. **Scheduled maintenance** - Periodic restarts in production
5. **Troubleshooting** - "Have you tried turning it off and on again?"

### Verification

```bash
docker ps
```

**Before restart:**
```
STATUS
Up 10 minutes
```

**After restart:**
```
STATUS
Up 5 seconds
```

Notice the uptime resets, confirming the restart occurred.

## Removing Containers

Removing a container deletes it permanently. Stopped containers still consume disk space, so regular cleanup is important.

### Basic Remove Command

```bash
docker rm <container-id-or-name>
```

**Example:**

```bash
# First, stop the container
docker stop my-go

# Then remove it
docker rm my-go
```

**Output:**
```
my-go
```

The container name is echoed, confirming deletion.

### Verification

```bash
docker ps -a  # Container no longer appears
```

### Cannot Remove Running Container

**Attempting to remove running container:**

```bash
docker rm my-go
```

**Error:**
```
Error response from daemon: Cannot remove container my-go: 
Container is running. Stop the container before removing or force remove
```

Docker protects you from accidentally deleting running containers.

**Two solutions:**

1. Stop first, then remove:
   ```bash
   docker stop my-go
   docker rm my-go
   ```

2. Force remove:
   ```bash
   docker rm -f my-go
   ```

### Force Remove: docker rm -f

The `-f` flag forces removal of running containers:

```bash
docker rm -f my-go
```

This is equivalent to:

```bash
docker kill my-go   # Immediately terminates (SIGKILL)
docker rm my-go     # Then removes
```

**Use cases for force removal:**

- Development: Quick cleanup without manual stopping
- Stuck containers: Process won't respond to SIGTERM
- Batch cleanup scripts

**Warning**: Force removal doesn't allow graceful shutdown. Use cautiously in production.

### Removing Multiple Containers

```bash
# Remove specific containers
docker rm container1 container2 container3

# Remove all stopped containers
docker rm $(docker ps -a -q -f status=exited)

# Remove all containers (running and stopped) - USE WITH CAUTION!
docker rm -f $(docker ps -a -q)
```

### Container Removal Best Practices

1. **Remove after testing** - Don't accumulate stopped containers
2. **Use meaningful names** - Makes identification easier before removal
3. **Check before removing** - `docker ps -a` to see what you're deleting
4. **Regular cleanup** - Automate removal of old containers
5. **Preserve important data** - Use volumes for persistent data

## Practical Management Workflow

Let's walk through a complete, realistic workflow demonstrating all these commands together.

### Scenario: Managing a Development Environment

You're developing an application with a backend service. Let's manage its complete lifecycle.

**Step 1: Run the container with a name**

```bash
docker run -d --name backend-dev -p 8080:8080 new-go-server:1.0.3
```

**Step 2: Verify it's running**

```bash
docker ps
```

**Output:**
```
CONTAINER ID   IMAGE                    PORTS                    NAMES
f1a2b3c4d5e6   new-go-server:1.0.3     0.0.0.0:8080->8080/tcp   backend-dev
```

**Step 3: Test the service**

```bash
curl localhost:8080
# Hello World
```

**Step 4: Check logs**

```bash
docker logs backend-dev
```

**Output:**
```
Server running on port 8080
```

**Step 5: Container is misbehaving—restart it**

```bash
docker restart backend-dev
```

**Step 6: Need to stop for configuration changes**

```bash
docker stop backend-dev
```

**Step 7: Verify it's stopped**

```bash
docker ps  # Not in running list
docker ps -a  # Shows as Exited
```

**Step 8: Start it again after changes**

```bash
docker start backend-dev
```

**Step 9: Done with this version—clean up**

```bash
docker stop backend-dev
docker rm backend-dev
```

**Step 10: Verify cleanup**

```bash
docker ps -a  # Container gone
```

### Scenario: Managing Multiple Microservices

**Start three services:**

```bash
docker run -d --name auth-service -p 8001:8080 auth:latest
docker run -d --name user-service -p 8002:8080 users:latest
docker run -d --name order-service -p 8003:8080 orders:latest
```

**Check all services:**

```bash
docker ps
```

**Output:**
```
CONTAINER ID   PORTS                    NAMES
a1b2c3d4e5f6   0.0.0.0:8001->8080/tcp   auth-service
b2c3d4e5f6a7   0.0.0.0:8002->8080/tcp   user-service
c3d4e5f6a7b8   0.0.0.0:8003->8080/tcp   order-service
```

**Restart one service:**

```bash
docker restart user-service
```

**Stop all services:**

```bash
docker stop auth-service user-service order-service
```

**Remove all services:**

```bash
docker rm auth-service user-service order-service
```

**Alternative: Stop and remove all in one command:**

```bash
docker rm -f auth-service user-service order-service
```

## Advanced Container Management

### Container Stats and Monitoring

Monitor resource usage in real-time:

```bash
docker stats
```

**Output:**
```
CONTAINER ID   NAME        CPU %    MEM USAGE / LIMIT     NET I/O
a1b2c3d4e5f6   my-go       0.25%    15.5MiB / 1.952GiB    1.2kB / 0B
```

To monitor specific container:

```bash
docker stats my-go
```

### Container Inspection

Get detailed information about a container:

```bash
docker inspect my-go
```

Returns JSON with comprehensive details:
- IP address
- Network settings
- Volumes
- Environment variables
- State and status
- Image used
- And much more

**Get specific information:**

```bash
# Get IP address
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-go

# Get status
docker inspect -f '{{.State.Status}}' my-go
```

### Container Logs

View container output:

```bash
# View all logs
docker logs my-go

# Follow logs (like tail -f)
docker logs -f my-go

# Show last 100 lines
docker logs --tail 100 my-go

# Show logs with timestamps
docker logs -t my-go

# Show logs since specific time
docker logs --since 2024-01-01T10:00:00 my-go
```

### Executing Commands in Running Containers

Run commands inside containers without stopping them:

```bash
# Start interactive bash session
docker exec -it my-go bash

# Run single command
docker exec my-go ls -la

# Run as specific user
docker exec -u root my-go apt-get update
```

## Container Cleanup Strategies

Over time, stopped containers accumulate. Regular cleanup is essential.

### Manual Cleanup

```bash
# Remove all stopped containers
docker container prune

# You'll be prompted to confirm
WARNING! This will remove all stopped containers.
Are you sure you want to continue? [y/N] y
```

### Automatic Cleanup: --rm Flag

Run containers that auto-remove after exit:

```bash
docker run --rm -d --name temp-server my-app:latest
```

When this container stops, it's automatically removed.

**Use cases:**

- One-time tasks
- Testing
- Short-lived processes
- CI/CD pipelines

### Cleanup Scripts

**Remove all stopped containers:**

```bash
docker rm $(docker ps -a -q -f status=exited)
```

**Remove all containers (caution!):**

```bash
docker rm -f $(docker ps -a -q)
```

**Remove containers older than 24 hours (requires formatting):**

```bash
docker ps -a --filter "before=24h" --format "{{.ID}}" | xargs docker rm
```

## Common Patterns and Best Practices

### Pattern 1: Development Workflow

```bash
# Start with meaningful name and port mapping
docker run -d --name dev-server -p 8080:8080 my-app:dev

# Work on code, test changes
curl localhost:8080

# Rebuild image
docker build -t my-app:dev .

# Remove old container, start new one
docker rm -f dev-server
docker run -d --name dev-server -p 8080:8080 my-app:dev
```

### Pattern 2: Production Deployment

```bash
# Run with restart policy
docker run -d \
  --name prod-api \
  --restart unless-stopped \
  -p 80:8080 \
  my-app:latest

# Monitor
docker stats prod-api
docker logs -f prod-api
```

### Pattern 3: Batch Operations

```bash
# Start multiple containers
for i in {1..3}; do
  docker run -d --name worker-$i worker:latest
done

# Stop all workers
docker stop $(docker ps -q --filter "name=worker")

# Remove all workers
docker rm $(docker ps -a -q --filter "name=worker")
```

### Pattern 4: Graceful Upgrades

```bash
# Current version running
docker ps
# NAME: api-v1

# Start new version on different port
docker run -d --name api-v2 -p 8081:8080 api:v2

# Test new version
curl localhost:8081

# If good, stop old version
docker stop api-v1

# Point traffic to new version (update load balancer/proxy)
# ...

# Clean up old version
docker rm api-v1
```

## Troubleshooting Common Issues

### Issue 1: Container Exits Immediately

**Symptom:**

```bash
docker run -d --name test my-app:latest
docker ps  # Container not listed
```

**Diagnosis:**

```bash
docker ps -a  # Shows "Exited (1) 2 seconds ago"
docker logs test  # Check error messages
```

**Common causes:**

- Application error/crash
- Missing dependencies
- Configuration error
- No long-running process in container

### Issue 2: Cannot Remove Container

**Symptom:**

```bash
docker rm my-go
Error: Container is running
```

**Solution:**

```bash
# Option 1: Stop first
docker stop my-go
docker rm my-go

# Option 2: Force remove
docker rm -f my-go
```

### Issue 3: Container Name Already in Use

**Symptom:**

```bash
docker run -d --name my-server my-app:latest
Error: Conflict. The container name "/my-server" is already in use
```

**Diagnosis:**

```bash
docker ps -a --filter "name=my-server"
```

**Solution:**

```bash
# Remove old container
docker rm my-server

# Or use different name
docker run -d --name my-server-v2 my-app:latest
```

### Issue 4: Lost Container ID

**Symptom:**

Ran container without name, forgot ID.

**Solution:**

```bash
# Show last created container
docker ps -l

# Or filter by image
docker ps --filter "ancestor=my-app:latest"
```

## Command Reference Summary

### Essential Commands

```bash
# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# Run container with name
docker run -d --name my-container image:tag

# Stop container
docker stop container-name

# Start stopped container
docker start container-name

# Restart container
docker restart container-name

# Remove stopped container
docker rm container-name

# Force remove running container
docker rm -f container-name

# View logs
docker logs container-name

# Follow logs
docker logs -f container-name

# Execute command in container
docker exec -it container-name bash

# View resource usage
docker stats

# Inspect container
docker inspect container-name

# Remove all stopped containers
docker container prune
```

### Useful Flags

```bash
-d              # Detached mode
--name          # Assign name
-p              # Port mapping
--rm            # Auto-remove after exit
--restart       # Restart policy
-f              # Force operation
-a              # Attach or show all
-q              # Quiet (show only IDs)
-it             # Interactive terminal
```

## Summary and Key Takeaways

### What We Learned

1. **Container Lifecycle** - Containers have states: created, running, stopped, removed

2. **Essential Management Commands**:
   - `docker ps` - List containers
   - `docker stop` - Stop running container
   - `docker start` - Start stopped container
   - `docker restart` - Stop and start container
   - `docker rm` - Remove container
   - `docker rm -f` - Force remove running container

3. **Naming Containers** - Always use `--name` for easier management

4. **Viewing Container Information**:
   - `docker ps -a` - All containers
   - `docker logs` - Container output
   - `docker inspect` - Detailed information
   - `docker stats` - Resource usage

5. **Cleanup Strategies**:
   - `docker container prune` - Remove stopped containers
   - `--rm` flag - Auto-remove after exit
   - Regular maintenance prevents disk space issues

6. **Best Practices**:
   - Name containers meaningfully
   - Clean up regularly
   - Use restart policies for production
   - Monitor resource usage
   - Check logs for troubleshooting

### The Foundation is Complete

You now have comprehensive knowledge of container management—the daily operations every Docker user performs. Combined with your understanding of:

- Building images (Chapters 21-22)
- Dockerfile instructions (CMD, WORKDIR - Chapters 23-24)
- Detached mode (Chapter 25)
- Container management (this chapter)

You're equipped to handle most practical Docker scenarios!

## What's Next

In the next chapter, **Building Magic Behind the Dockerfile**, we'll dive deep into how Docker actually builds images. You'll learn:

- Docker build process internals
- Layer caching and optimization
- Build context and .dockerignore
- Multi-stage builds
- Build arguments and variables
- Optimizing image size and build speed

This knowledge will transform you from someone who writes Dockerfiles to someone who understands exactly how Docker builds images and how to optimize that process.

## Conclusion

Container management is the bread and butter of working with Docker. These commands—`ps`, `stop`, `start`, `restart`, `rm`—you'll use hundreds of times. Mastering them makes Docker feel intuitive and manageable rather than overwhelming.

Remember:
- **Name your containers** - Makes everything easier
- **Clean up regularly** - Prevents clutter and disk issues
- **Check status frequently** - `docker ps` and `docker ps -a` are your friends
- **Use force removal carefully** - Convenient but can cause data loss
- **Monitor resource usage** - `docker stats` helps catch issues early

With solid container management skills, you're ready to tackle more advanced Docker concepts. Keep practicing, and these commands will become second nature.

---

**Next Chapter**: Building Magic Behind the Dockerfile - Discover how Docker actually builds images, optimize your build process, and create production-ready, efficient Docker images.
