# Chapter 21: Docker Hands On - Practical Docker Operations

## Overview

Welcome to the hands-on chapter where we'll move from theory to practice! In the previous chapters, we learned about containers, Docker fundamentals, Linux basics, package management, and user permissions. Now it's time to roll up our sleeves and work directly with Docker.

This chapter will guide you through the essential Docker commands you'll use daily. We'll learn how to manage Docker images, run containers, interact with running containers, name them for easy reference, and even build our first custom Docker image. By the end of this chapter, you'll have practical experience with the most important Docker operations.

### What You'll Learn

- How to view and manage Docker images on your system
- How to check running and stopped containers
- Running containers in interactive mode
- Understanding the `-i` and `-t` flags
- Naming containers for easier management
- Executing commands in running containers with `docker exec`
- Building your first custom Docker image
- Understanding the Dockerfile basics
- Using `docker build` to create images

### Prerequisites

Before diving into this chapter, you should:
- Have Docker Desktop installed (covered in Chapter 17)
- Understand basic Docker concepts like images and containers
- Be familiar with basic Linux commands from Chapter 19
- Have access to a terminal or command line

---

## Understanding Docker Images on Your System

### Viewing Available Images

When you work with Docker, the first thing you need to know is what images are available on your local system. Remember from earlier chapters that images are like templates or blueprints that Docker uses to create containers.

To see all the images currently stored on your computer, use the `docker images` command:

```bash
docker images
```

This command will display a table with several columns:

**1. REPOSITORY** - The name of the image. For example, `ubuntu` is the official Ubuntu image from Docker Hub.

**2. TAG** - The version or variant of the image. For example, `24.04` refers to Ubuntu version 24.04. Tags help you choose specific versions.

**3. IMAGE ID** - A unique identifier for the image. This is a hash value that Docker generates. You can use either the name or ID to reference an image.

**4. CREATED** - When the image was created or downloaded to your system.

**5. SIZE** - How much disk space the image occupies.

### Example Output

```
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
ubuntu        24.04     7ea6a91e1234   2 weeks ago    101MB
```

In this example:
- We have an Ubuntu image
- It's version 24.04
- The image ID starts with `7ea6a91e1234`
- It was created 2 weeks ago
- It takes up 101 megabytes of space

### Important Note About Image IDs

The full image ID is actually much longer than what's displayed. Docker shows only the first 12 characters for convenience. The complete ID might be 64 characters long, but you rarely need the full ID. The shortened version is unique enough for most operations.

### Pulling Images from Docker Hub

If you don't have an image locally, you can download (or "pull") it from Docker Hub using:

```bash
docker pull ubuntu:24.04
```

This command:
1. Connects to Docker Hub
2. Finds the Ubuntu image with tag 24.04
3. Downloads all the necessary layers to your computer
4. Stores it in your local image cache

After pulling, when you run `docker images` again, you'll see the newly downloaded image in the list.

### Removing Images

If you want to remove an image from your system to free up space, use `docker rmi` (remove image):

```bash
docker rmi ubuntu:24.04
```

You can also use the image ID:

```bash
docker rmi 7ea6a91e1234
```

**Force Removal**: Sometimes Docker won't let you remove an image because containers are using it (even stopped containers). In such cases, you can force the removal:

```bash
docker rmi -f ubuntu:24.04
```

The `-f` flag stands for "force" and tells Docker to remove the image regardless of dependencies.

---

## Managing Containers: Running vs Stopped

### Viewing Running Containers

Containers are the running instances of images. To see which containers are currently active and running on your system, use:

```bash
docker ps
```

The `ps` stands for "process status" - a term borrowed from Linux. This command shows only containers that are currently running.

### Example Output

```
CONTAINER ID   IMAGE          COMMAND   CREATED         STATUS         NAMES
a1b2c3d4e5f6   ubuntu:24.04   "bash"    10 seconds ago  Up 9 seconds   my_ubuntu
```

Let's understand each column:

**1. CONTAINER ID** - A unique identifier for the container (shortened, like image IDs)

**2. IMAGE** - Which image this container was created from

**3. COMMAND** - The command that's running inside the container

**4. CREATED** - When the container was created

**5. STATUS** - The current state. "Up X seconds" means running

**6. NAMES** - The name of the container (auto-generated or user-specified)

### Viewing All Containers (Including Stopped)

By default, `docker ps` only shows running containers. But containers can be in a stopped state as well. To see ALL containers, including stopped ones, add the `-a` flag:

```bash
docker ps -a
```

The `-a` stands for "all". Now you'll see output like this:

```
CONTAINER ID   IMAGE          COMMAND   CREATED          STATUS                      NAMES
a1b2c3d4e5f6   ubuntu:24.04   "bash"    2 minutes ago    Up 2 minutes               my_ubuntu
x9y8z7w6v5u4   ubuntu:24.04   "bash"    31 seconds ago   Exited (0) 30 seconds ago  myubuntu_01
```

Notice the second container has a STATUS of "Exited". This means it was running but has stopped. The number in parentheses (0) is the exit code. Zero typically means it exited normally without errors.

### Why Do Containers Stop?

When you run a container with a command like `bash`, the container stays alive as long as that bash session is active. Once you exit bash (by typing `exit`), the main process ends, and the container stops.

This is a fundamental Docker principle: **A container runs as long as its main process is running. When that process ends, the container stops.**

---

## Running Containers Interactively

### The Basic `docker run` Command

To create and start a new container from an image, we use `docker run`:

```bash
docker run ubuntu:24.04 bash
```

Let's break this down:

**1. `docker run`** - The main command telling Docker to create and start a container

**2. `ubuntu:24.04`** - The image to use as the template

**3. `bash`** - The program to run inside the container once it starts

### The Problem: No Interaction

If you run the command above without any flags, something strange happens - the container starts bash, but immediately exits! Why?

By default, containers run in a **non-interactive mode**. This means:
- They don't accept input from your keyboard
- They don't provide a proper terminal interface
- The process starts, finds no work to do, and exits

This is fine for containers that run background services, but not useful when you want to interact with the container.

### The Solution: Interactive Mode with `-i` and `-t`

To interact with a container, we need to add flags to `docker run`:

```bash
docker run -it ubuntu:24.04 bash
```

Let's understand these flags:

#### The `-i` Flag (Interactive)

The `-i` flag stands for **interactive**. It tells Docker:
- Keep the standard input (stdin) open
- Allow you to type commands
- Accept input from your keyboard

Without `-i`, Docker closes stdin, and you can't type anything.

#### The `-t` Flag (TTY)

The `-t` flag stands for **TTY** (teletypewriter - a historical term for terminal). It tells Docker:
- Allocate a pseudo-terminal
- Provide a nice formatted terminal interface
- Show you prompts like `root@container-id:/#`
- Display colors and formatting properly

Without `-t`, you get input capability but no proper terminal formatting.

### Using Flags Together

You can write the flags separately or together:

```bash
# Together (most common)
docker run -it ubuntu:24.04 bash

# Separately (same result)
docker run -i -t ubuntu:24.04 bash
```

### What Happens with Different Flag Combinations?

**Only `-i` (no `-t`)**:
```bash
docker run -i ubuntu:24.04 bash
```
- You can type commands
- Commands execute
- But no nice prompt or formatting
- Looks plain and unformatted

**Only `-t` (no `-i`)**:
```bash
docker run -t ubuntu:24.04 bash
```
- You get a nice terminal interface
- You see prompts
- But you can't type anything! (stdin is closed)
- The terminal is "read-only" in a sense

**Both `-i` and `-t` (recommended)**:
```bash
docker run -it ubuntu:24.04 bash
```
- You can type commands
- You get a beautiful terminal interface
- Full interactivity
- This is what you want 99% of the time

### Practical Example

When you run:

```bash
docker run -it ubuntu:24.04 bash
```

You'll see something like:

```
root@a1b2c3d4e5f6:/#
```

This is the bash prompt inside the container! You can now:

```bash
# List files
ls

# Change directories
cd /bin

# List files in /bin
ls

# Return to root
cd /

# Exit the container
exit
```

When you type `exit`, you leave the bash session, which causes the container to stop (because bash was the main process).

---

## Naming Containers for Easy Management

### The Problem with Auto-Generated Names

By default, Docker assigns random names to containers. These names are combinations of adjectives and famous scientists' names, like:
- `silly_einstein`
- `brave_darwin`
- `determined_turing`

While these names are fun, they're not practical. If you create multiple containers, you'll have trouble remembering which is which.

### Using the `--name` Flag

You can assign your own meaningful names to containers using the `--name` flag:

```bash
docker run --name my_ubuntu -it ubuntu:24.04 bash
```

Now your container is named `my_ubuntu` instead of a random name. When you run `docker ps`, you'll see:

```
CONTAINER ID   IMAGE          COMMAND   CREATED          STATUS         NAMES
a1b2c3d4e5f6   ubuntu:24.04   "bash"    5 seconds ago    Up 4 seconds   my_ubuntu
```

### Benefits of Naming Containers

**1. Easy to Remember**: Instead of remembering `silly_einstein`, you use descriptive names like `web_server` or `database_container`.

**2. Easy to Reference**: Other Docker commands can use the name instead of the container ID:

```bash
# Using container name
docker exec -it my_ubuntu bash

# Using container ID (harder to remember)
docker exec -it a1b2c3d4e5f6 bash
```

**3. Self-Documenting**: Names like `frontend_app`, `backend_api`, and `mysql_db` tell you what each container does.

### Naming Rules

Container names must:
- Be unique (no two containers can have the same name)
- Contain only lowercase letters, numbers, hyphens, and underscores
- Not start with a hyphen

### Practical Naming Example

```bash
# Create a container named myubuntu_01
docker run --name myubuntu_01 -it ubuntu:24.04 bash

# Exit it
exit

# Try to create another with the same name
docker run --name myubuntu_01 -it ubuntu:24.04 bash
# This will FAIL because the name is already taken

# Create with a different name
docker run --name myubuntu_02 -it ubuntu:24.04 bash
```

---

## Executing Commands in Running Containers

### The `docker exec` Command

Imagine you have a container that's already running. Maybe it's running a web server or database. You want to access that container and run commands inside it without stopping it. This is where `docker exec` comes in.

The `docker exec` command allows you to execute commands in a **running** container.

### Basic Syntax

```bash
docker exec [OPTIONS] CONTAINER COMMAND
```

For interactive access (which is most common), use:

```bash
docker exec -it my_ubuntu bash
```

Let's break this down:

**1. `docker exec`** - The command to execute something in a running container

**2. `-it`** - Same flags as before! Interactive mode with a proper terminal

**3. `my_ubuntu`** - The name (or ID) of the running container

**4. `bash`** - The command to run (typically bash for an interactive shell)

### Practical Scenario

**Step 1**: Start a container and give it a name:
```bash
docker run --name my_ubuntu -it ubuntu:24.04 bash
```

You're now inside the container with a bash prompt.

**Step 2**: Open another terminal window (keep the first one running)

**Step 3**: From the second terminal, access the same container:
```bash
docker exec -it my_ubuntu bash
```

**Amazing Result**: You now have TWO terminal sessions accessing the SAME container simultaneously! Any files you create in one session will be visible in the other.

### Demonstrating Multiple Access

**Terminal 1** (inside container):
```bash
# Create a file
echo "Hello from Terminal 1" > /test.txt

# Read it
cat /test.txt
# Output: Hello from Terminal 1
```

**Terminal 2** (same container via docker exec):
```bash
# Read the same file
cat /test.txt
# Output: Hello from Terminal 1

# Append to it
echo "Hello from Terminal 2" >> /test.txt

# Read it again
cat /test.txt
# Output:
# Hello from Terminal 1
# Hello from Terminal 2
```

Both terminals are working inside the same container environment!

### Why Is This Useful?

**1. Debugging**: Your application is running in a container. You can exec into it to check logs or inspect files without stopping the app.

**2. Maintenance**: Perform administrative tasks (cleaning temp files, checking configurations) while the service runs.

**3. Learning**: Great for exploring container internals and understanding how they work.

### Important Distinction: `docker run` vs `docker exec`

**`docker run`**:
- Creates a NEW container from an image
- Starts that container
- Runs a command in it

**`docker exec`**:
- Does NOT create a new container
- Accesses an EXISTING running container
- Runs additional commands in it

Think of `docker run` as building a new house and moving in, while `docker exec` is like visiting an existing house.

---

## Container Isolation and File Systems

### Each Container Has Its Own Filesystem

One of Docker's powerful features is **isolation**. Each container has its own isolated filesystem. Files you create in one container don't affect other containers, even if they're created from the same image.

### Demonstration

**Step 1**: Create first container and add a file:
```bash
docker run --name container1 -it ubuntu:24.04 bash

# Inside container1
echo "I am in container 1" > /myfile.txt
ls /
# You'll see myfile.txt

exit
```

**Step 2**: Create second container from the same image:
```bash
docker run --name container2 -it ubuntu:24.04 bash

# Inside container2
ls /
# myfile.txt does NOT exist here!

echo "I am in container 2" > /myfile.txt
exit
```

Even though both containers use the same Ubuntu image, their filesystems are isolated. Changes in one don't affect the other.

### Multiple Containers from One Image

You can run as many containers as you want from a single image:

```bash
docker run -it ubuntu:24.04 bash  # Container 1
docker run -it ubuntu:24.04 bash  # Container 2
docker run -it ubuntu:24.04 bash  # Container 3
```

Each will be an independent, isolated instance with its own filesystem, processes, and network interfaces.

---

## Building Your First Custom Docker Image

So far, we've been using pre-built images from Docker Hub (like `ubuntu:24.04`). But what if you want to create your own image? Maybe you want an Ubuntu image that already has certain files or programs installed.

This is where **Dockerfiles** come in.

### What Is a Dockerfile?

A Dockerfile is a text file that contains instructions for building a Docker image. It's like a recipe that tells Docker:
- What base image to start with
- What files to add
- What programs to install
- What commands to run

### Creating Your First Dockerfile

Let's create a custom Ubuntu image that has a text file already included.

**Step 1**: Create a directory for your Docker project:
```bash
# Navigate to Desktop
cd ~/Desktop

# Create a directory
mkdir docker-tutorial

# Go into it
cd docker-tutorial

# Check current location
pwd
# Output: /home/username/Desktop/docker-tutorial
```

**Step 2**: Create a file named `Dockerfile` (exactly this name, capital D):
```bash
# Open with your favorite text editor
# The filename must be exactly: Dockerfile
# Capital D, no extension, no spaces
```

**Step 3**: Write the following content in the Dockerfile:

```dockerfile
FROM ubuntu:24.04

RUN echo "Hello World" > /hello.txt
```

Let's understand each line:

#### Line 1: `FROM ubuntu:24.04`

The `FROM` instruction specifies the **base image**. This is the starting point for your custom image. We're saying:
- "Start with the official Ubuntu 24.04 image"
- "Build on top of it"

You can think of it like this: You're taking an existing Ubuntu installation and customizing it.

#### Line 2: `RUN echo "Hello World" > /hello.txt`

The `RUN` instruction executes a command during the image build process. In this case:
- It runs the bash command: `echo "Hello World" > /hello.txt`
- This creates a file named `hello.txt` in the root directory
- The file contains the text "Hello World"

This command runs **inside the container** during the build, not on your host machine.

### Building the Image

Now that we have a Dockerfile, let's build an image from it.

**Step 4**: Run the build command:
```bash
docker build .
```

Let's break this down:

**`docker build`** - The command to build an image from a Dockerfile

**`.` (dot)** - Means "current directory". Docker will look for a file named `Dockerfile` in the current directory.

### Understanding the Build Output

When you run `docker build .`, you'll see output like this:

```
[+] Building 2.3s (6/6) FINISHED
 => [internal] load build definition from Dockerfile
 => => transferring dockerfile: 120B
 => [internal] load metadata
 => [internal] load .dockerignore
 => [1/2] FROM docker.io/library/ubuntu:24.04
 => [2/2] RUN echo "Hello World" > /hello.txt
 => exporting to image
 => => exporting layers
 => => writing image sha256:7ea6a91e...
```

Let's understand what's happening:

**1. Load build definition**: Docker reads your Dockerfile

**2. Load metadata**: Docker prepares the build context

**3. FROM ubuntu:24.04**: Docker either uses the local Ubuntu image (if you already have it) or downloads it

**4. RUN echo...**: Docker executes your RUN command, creating the hello.txt file

**5. Exporting to image**: Docker saves the result as a new image

**6. Writing image sha256...**: Docker assigns a unique ID to your new image

### The Image ID

At the end, you'll see something like:

```
=> => writing image sha256:7ea6a91e298f...
```

This is your new image's ID. You can use this ID to run a container from your custom image.

### Viewing Your Custom Image

Run `docker images` to see your newly created image:

```bash
docker images
```

Output:
```
REPOSITORY    TAG       IMAGE ID       CREATED          SIZE
<none>        <none>    7ea6a91e298f   2 minutes ago    101MB
ubuntu        24.04     ...            ...              101MB
```

Notice:
- **REPOSITORY** and **TAG** are `<none>`. This is because we didn't give the image a name or tag.
- **IMAGE ID** is the unique identifier (starting with 7ea6a91e in this example)
- **SIZE** is 101MB (same as Ubuntu base because we only added a tiny text file)

### Building with a Name and Tag

Having `<none>` as the repository name is not useful. Let's rebuild with a proper name:

```bash
docker build -t custom_ubuntu .
```

The `-t` flag (tag) lets you specify a name (and optionally a tag) for your image. Here we're naming it `custom_ubuntu`.

Now run `docker images` again:

```
REPOSITORY       TAG       IMAGE ID       CREATED          SIZE
custom_ubuntu    latest    7ea6a91e298f   2 minutes ago    101MB
ubuntu           24.04     ...            ...              101MB
```

Much better! Now:
- **REPOSITORY** is `custom_ubuntu`
- **TAG** is `latest` (Docker's default if you don't specify)
- Everything else is the same

### Specifying a Custom Tag

You can also specify a specific tag:

```bash
docker build -t custom_ubuntu:1.0 .
```

This creates an image named `custom_ubuntu` with tag `1.0`:

```
REPOSITORY       TAG       IMAGE ID       CREATED          SIZE
custom_ubuntu    1.0       7ea6a91e298f   2 minutes ago    101MB
```

Tags are useful for versioning your images (e.g., `myapp:1.0`, `myapp:1.1`, `myapp:2.0`).

---

## Running Your Custom Image

Now let's run a container from your custom image and verify that the hello.txt file exists.

```bash
docker run -it custom_ubuntu bash
```

You're now inside a container created from your custom image. Let's verify:

```bash
# List files in the root directory
ls /

# You should see hello.txt among the files!

# Read the contents
cat /hello.txt
# Output: Hello World
```

**Amazing!** The file was created during the image build and is now present in every container you create from this image.

### The Magic of Docker Images

Here's the beautiful part:
- You created the file ONCE during the build
- It's now permanently part of the image
- Every container created from this image will have the file
- You don't have to create it manually each time

This is how Docker images work. They capture a state (filesystem, installed programs, configurations) and let you create identical containers from that state repeatedly.

---

## Understanding the Dockerfile Components

While we'll cover Dockerfiles in much more depth in upcoming chapters, let's quickly understand what we just did:

### The Build Process Flow

```
Dockerfile → docker build → Docker Image → docker run → Container
```

1. You write a Dockerfile with instructions
2. `docker build` reads the Dockerfile and creates an image
3. The image is stored locally
4. `docker run` creates containers from that image

### Anatomy of Our Dockerfile

```dockerfile
FROM ubuntu:24.04
RUN echo "Hello World" > /hello.txt
```

**FROM** - Always the first instruction. Specifies the base image.

**RUN** - Executes commands during build time. Used for:
- Installing packages (`RUN apt-get install ...`)
- Creating files
- Downloading resources
- Setting up the environment

Think of each `RUN` instruction as a step in your setup process.

### What Happens During Build?

When you run `docker build .`, Docker:

1. **Starts with the base image** (ubuntu:24.04)
2. **Creates a temporary container** from that image
3. **Runs each instruction** (like `RUN echo...`) in that container
4. **Captures the changes** (new files, modified files)
5. **Saves the result** as a new image layer
6. **Repeats** for each instruction
7. **Produces the final image** with all changes

### The Power of Layers

Docker images are built in layers. Each instruction creates a new layer:

```
Base Image: ubuntu:24.04
  ↓
Layer 1: RUN echo "Hello World" > /hello.txt
  ↓
Final Image: custom_ubuntu
```

This layered approach makes Docker efficient:
- Layers are cached and reused
- If you change only one layer, Docker rebuilds only that layer and those after it
- Multiple images can share base layers (saving space)

---

## Comparing docker run and docker build

Let's clarify these two important commands:

### `docker run`

**Purpose**: Create and start a container from an image

**Usage**:
```bash
docker run [OPTIONS] IMAGE [COMMAND]
```

**What it does**:
- Takes an existing image
- Creates a new container
- Starts the container
- Runs the specified command (or default command)

**Example**:
```bash
docker run -it ubuntu:24.04 bash
```

### `docker build`

**Purpose**: Create an image from a Dockerfile

**Usage**:
```bash
docker build [OPTIONS] PATH
```

**What it does**:
- Reads a Dockerfile
- Executes the instructions
- Creates a new image
- Stores it locally

**Example**:
```bash
docker build -t myimage .
```

### Key Difference

- **`docker build`** creates **images** (templates)
- **`docker run`** creates **containers** (instances)

Think of it like:
- **`docker build`** is like creating a blueprint for a house
- **`docker run`** is like building actual houses from that blueprint

---

## Practical Summary: Commands You Learned

Let's recap all the Docker commands we covered in this hands-on session:

### Image Management

```bash
# View all local images
docker images

# Pull an image from Docker Hub
docker pull ubuntu:24.04

# Remove an image
docker rmi ubuntu:24.04

# Force remove an image
docker rmi -f ubuntu:24.04
```

### Container Management

```bash
# View running containers
docker ps

# View all containers (including stopped)
docker ps -a
```

### Running Containers

```bash
# Run a container (exits immediately if not interactive)
docker run ubuntu:24.04 bash

# Run a container in interactive mode
docker run -it ubuntu:24.04 bash

# Run with a custom name
docker run --name my_ubuntu -it ubuntu:24.04 bash
```

### Accessing Running Containers

```bash
# Execute commands in a running container
docker exec -it my_ubuntu bash

# Using container ID instead of name
docker exec -it a1b2c3d4e5f6 bash
```

### Building Custom Images

```bash
# Build an image from Dockerfile in current directory
docker build .

# Build with a name
docker build -t custom_ubuntu .

# Build with name and tag
docker build -t custom_ubuntu:1.0 .
```

---

## Practice Exercises

Now it's your turn to practice! Try these exercises to reinforce your learning:

### Exercise 1: Image Management

1. Pull the `alpine` image from Docker Hub (a lightweight Linux distribution)
2. List all your images
3. Note the size difference between Alpine and Ubuntu
4. Remove the Alpine image

### Exercise 2: Container Exploration

1. Run an Ubuntu container in interactive mode
2. Inside the container:
   - Create a directory: `mkdir /mydata`
   - Create a file: `echo "Test data" > /mydata/test.txt`
   - Verify the file: `cat /mydata/test.txt`
3. Exit the container
4. Run `docker ps -a` to see the stopped container
5. Start the same container again (use its name or ID)
6. Check if your file still exists

### Exercise 3: Multiple Container Access

1. Start a named container: `docker run --name web_container -it ubuntu:24.04 bash`
2. Create a file inside: `echo "Web Server Data" > /data.txt`
3. Open a new terminal window
4. Access the same container: `docker exec -it web_container bash`
5. Read the file from the second terminal: `cat /data.txt`
6. Modify it from the second terminal: `echo "Updated" >> /data.txt`
7. Return to the first terminal and verify the changes

### Exercise 4: Building Custom Images

1. Create a new directory called `my-first-image`
2. Inside it, create a Dockerfile with these instructions:
   ```dockerfile
   FROM ubuntu:24.04
   RUN apt-get update
   RUN apt-get install -y curl
   RUN echo "This is my custom image" > /readme.txt
   ```
3. Build the image: `docker build -t my-first-image .`
4. Run a container from it: `docker run -it my-first-image bash`
5. Verify curl is installed: `curl --version`
6. Read your custom file: `cat /readme.txt`

### Exercise 5: Container Isolation

1. Create two containers from Ubuntu:
   ```bash
   docker run --name container_a -it ubuntu:24.04 bash
   docker run --name container_b -it ubuntu:24.04 bash
   ```
2. In container_a, create a file: `echo "I am A" > /identity.txt`
3. In container_b, try to read the file: `cat /identity.txt`
4. You'll get an error! This proves containers are isolated.

---

## Common Patterns and Best Practices

### 1. Always Name Your Containers

```bash
# Bad (hard to manage)
docker run -it ubuntu:24.04 bash

# Good (easy to reference)
docker run --name dev_environment -it ubuntu:24.04 bash
```

### 2. Use Meaningful Image Names

```bash
# Bad
docker build -t myimg .

# Good
docker build -t company-webapp:1.0 .
```

### 3. Tag Your Images Properly

```bash
# Development version
docker build -t myapp:dev .

# Production version
docker build -t myapp:1.0 .
docker build -t myapp:latest .
```

### 4. Clean Up Regularly

```bash
# Remove stopped containers
docker container prune

# Remove unused images
docker image prune

# Remove everything unused
docker system prune
```

### 5. Check Container Status Frequently

```bash
# Quick check of running containers
docker ps

# Full view including stopped
docker ps -a
```

---

## Troubleshooting Common Issues

### Issue 1: Container Exits Immediately

**Problem**: When you run `docker run ubuntu:24.04`, it exits right away.

**Solution**: Use `-it` flags for interactive mode:
```bash
docker run -it ubuntu:24.04 bash
```

**Why**: Without `-it`, the container has no work to do and exits.

### Issue 2: Can't Remove Image

**Problem**: `docker rmi` says image is being used.

**Solution**: Check for containers using that image:
```bash
docker ps -a  # Find containers
docker rm <container-id>  # Remove the container
docker rmi <image>  # Then remove the image
```

**Alternative**: Force remove:
```bash
docker rmi -f ubuntu:24.04
```

### Issue 3: Container Name Already Exists

**Problem**: Error saying "name already in use" when running `docker run --name mycontainer`.

**Solution**: Either:
- Choose a different name
- Remove the existing container:
  ```bash
  docker rm mycontainer
  ```

### Issue 4: Can't Connect to Docker

**Problem**: Error "Cannot connect to Docker daemon".

**Solution**:
- Make sure Docker Desktop is running
- On Linux, you might need `sudo`:
  ```bash
  sudo docker ps
  ```

### Issue 5: Build Fails

**Problem**: `docker build` fails with errors.

**Solutions**:
- Check your Dockerfile syntax
- Ensure you're in the correct directory (where Dockerfile exists)
- Check the error message carefully - it usually tells you which line failed

---

## Understanding Flags: A Quick Reference

### `-i` (Interactive)

- Keeps stdin open
- Allows keyboard input
- Essential for interactive programs

### `-t` (TTY)

- Allocates a pseudo-terminal
- Provides terminal formatting
- Shows prompts properly

### `-it` (Combined)

- Most common combination
- Interactive terminal
- Use for bash sessions

### `--name`

- Assigns a custom name
- Makes management easier
- Must be unique

### `-f` (Force)

- Forces operations
- Use with caution
- Can bypass safety checks

---

## Visual Understanding: Container Lifecycle

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  IMAGE (ubuntu:24.04)                              │
│  ┌─────────────────────────────────────────┐       │
│  │ Blueprint / Template                     │       │
│  │ Contains: OS files, programs, configs   │       │
│  └─────────────────────────────────────────┘       │
│           │                                         │
│           │ docker run -it ubuntu:24.04 bash       │
│           ↓                                         │
│  ┌─────────────────────────────────────────┐       │
│  │ CONTAINER (running)                      │       │
│  │ ┌─────────────────────────────────┐     │       │
│  │ │ Instance of image                │     │       │
│  │ │ Has its own filesystem           │     │       │
│  │ │ Running bash process             │     │       │
│  │ └─────────────────────────────────┘     │       │
│  └─────────────────────────────────────────┘       │
│           │                                         │
│           │ exit command                            │
│           ↓                                         │
│  ┌─────────────────────────────────────────┐       │
│  │ CONTAINER (stopped)                      │       │
│  │ ┌─────────────────────────────────────┐ │       │
│  │ │ Filesystem preserved                 │ │       │
│  │ │ No running processes                 │ │       │
│  │ │ Can be restarted                     │ │       │
│  │ └─────────────────────────────────────┘ │       │
│  └─────────────────────────────────────────┘       │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## The Path Forward

### What We've Accomplished

In this hands-on chapter, you've:

✅ Learned to view and manage Docker images
✅ Understood how to check running and stopped containers
✅ Mastered interactive container access with `-it` flags
✅ Learned to name containers for easier management
✅ Used `docker exec` to access running containers
✅ Built your first custom Docker image
✅ Understood the basics of Dockerfiles
✅ Learned the difference between `docker run` and `docker build`

### Why This Matters

These fundamentals form the foundation of working with Docker. Every complex Docker workflow builds on these basic operations. Whether you're deploying microservices, setting up development environments, or building CI/CD pipelines, you'll use these commands daily.

### What's Next?

In the coming chapters, we'll dive deeper into:

- **Dockerfile Best Practices**: Writing production-ready Dockerfiles
- **The CMD Instruction**: Understanding how containers decide what to run
- **Working Directories**: The WORKDIR instruction and container filesystem organization
- **Detached Mode**: Running containers in the background
- **Container Management**: Advanced operations like starting, stopping, and monitoring
- **Docker Networking**: How containers communicate
- **Docker Volumes**: Persisting data beyond container lifetime

Each chapter will build on what you learned here, taking you from beginner to confident Docker user.

---

## Key Takeaways

### Conceptual Understanding

1. **Images are templates**, containers are instances
2. **Containers are isolated** - each has its own filesystem
3. **Interactive mode** (-it) is essential for bash access
4. **Naming** makes container management practical
5. **docker exec** lets you access running containers
6. **Dockerfiles** automate image creation

### Practical Skills

1. You can view and manage images with `docker images` and `docker rmi`
2. You can check containers with `docker ps` and `docker ps -a`
3. You can run interactive containers with `docker run -it`
4. You can name containers with `--name`
5. You can access running containers with `docker exec`
6. You can build custom images with `docker build`

### Mental Model

Think of Docker as:
- **Images** = Recipe or blueprint
- **Containers** = The dish you cook from the recipe
- **Dockerfile** = The instructions for creating a new recipe
- **docker build** = Following the instructions to create the recipe
- **docker run** = Cooking a dish from the recipe
- **docker exec** = Adding ingredients to a dish that's already cooking

---

## Final Thoughts

Docker might seem complex at first, but it's actually quite logical once you understand the basic concepts:

**Images** hold the template
**Containers** are created from images
**Dockerfiles** define how to build images
**Commands** let you manage everything

You've now got hands-on experience with all the essential Docker operations. The best way to solidify this knowledge is to practice. Create containers, build images, experiment with different configurations. Don't worry about making mistakes - containers are disposable and easy to recreate!

Remember: The reason people find Docker difficult is not because Docker is hard - it's because they don't understand:
- Operating systems fundamentals
- Networking basics
- Process management
- Filesystem concepts

But you've been learning these fundamentals in previous chapters! You're now equipped with the background knowledge that makes Docker make sense. Everything you learned about Linux, file systems, users, and permissions directly applies to working with Docker containers.

In the next chapters, we'll go deeper into Dockerfiles, explore how images are built layer by layer, understand container lifecycle management in detail, and work with real applications like web servers. You're well on your way to Docker mastery!

Keep experimenting, keep learning, and don't hesitate to revisit this chapter whenever you need a refresher on the fundamentals. Happy Dockering!
