# Chapter 1: What Is Docker?

> **In one sentence:** Docker lets you pack an application together with everything it needs to run, so it behaves the same on your laptop, your teammate's laptop, and a production server.

**Level:** 🟢 Beginner (no prior knowledge needed) · **Reading time:** ~20 minutes

---

## What you will learn

- The problem Docker was built to solve ("it works on my machine")
- What a **container** and an **image** are, in plain words
- Where the name and the whale logo come from
- A short history: dotCloud, Solomon Hykes, PyCon 2013
- What Docker is (and is not) — a few myths cleared up
- Where Docker fits in a real workflow, from beginner use to production

---

## 1. The problem: "It works on my machine"

Imagine you build a small web app on your laptop.

| Where it runs | OS | Python | Result |
|---|---|---|---|
| Your laptop | Windows | 3.12 | Works |
| Teammate's laptop | macOS | 3.9 | One library fails to install |
| Company server | CentOS / Ubuntu | 3.6 | Crashes on startup |

Nothing is wrong with your *code*. The **environment** is different. An environment is everything around your code:

- the operating system and its system libraries
- the language runtime (Python, Node.js, Java, ...) and its version
- installed packages and their versions
- configuration files and environment variables
- file paths, permissions, and so on

Before containers, teams fought this with long setup documents ("install X, then Y, then set Z...") that were always slightly out of date. Every new developer lost a day or two just getting the project to run.

> **Key idea:** Most "bugs" of this kind are not bugs in your code. They are differences between environments. If you could ship the environment *together with* the code, the problem would disappear.

That is exactly what Docker does.

---

## 2. The analogy: shipping containers

Docker's name and logo come from a real-world story.

### Before 1956
Cargo was loaded by hand: barrels, sacks, crates, all different shapes. Loading a ship took days, goods got damaged or stolen, and every port handled things differently.

### After the standard shipping container
In the 1950s, Malcolm McLean pioneered the **standard steel shipping container** (his first container ship sailed in 1956). Standard sizes (typically 20 ft or 40 ft long, about 8 ft wide) meant that:

- Cranes, trucks, trains and ships were all built to handle the *same* box.
- Nobody needed to know what was **inside** the box. They only moved the box.
- Loading time dropped from days to hours, and global shipping became far cheaper.

### The same idea for software

| Shipping world | Docker world |
|---|---|
| Standard steel box | Standard **container** format |
| Goods inside the box | Your app + its libraries + config |
| Any ship, truck or train can carry it | Any machine with Docker can run it |
| Dock workers ("dockers") load and unload | The **Docker** software starts and stops containers |

The box does not care what is inside. Docker does not care whether your app is Python, Java, or Go. It just runs the box.

---

## 3. What Docker is

**Docker is a platform for building, shipping, and running applications in containers.**

"Platform" means it is not one single program. It is a set of tools that work together:

- the **Docker Engine** (the background service that actually runs containers)
- the **`docker` command-line tool (CLI)** you type commands into
- **Docker Desktop** (a friendly app for Windows and macOS that bundles the above)
- **Docker Hub** (a public website where people share ready-made images)
- **Docker Compose** (a tool for running several containers together)

You will meet each of these in later chapters.

### 3.1 Container

A **container** is a running, isolated process (or group of processes) that has its own view of the file system, network, and process list. Inside it live:

1. Your application code
2. Its dependencies (exact Python/Node version, libraries, ...)
3. A minimal set of OS files (for example a tiny Alpine Linux or Debian file system)

> **Common misunderstanding:** a container does **not** contain a full operating system with its own kernel. It shares the kernel of the machine it runs on. That is why containers start in about a second and use little memory. We explain this properly in Chapters 2–5.

### 3.2 Image

An **image** is a read-only package (a template) from which containers are created. It holds the app, its dependencies and the file system layout.

The easiest way to remember it:

| Concept | Analogy | Changes? |
|---|---|---|
| **Image** | A recipe / a class / an installer file | No, it is read-only |
| **Container** | The cooked dish / an object / the installed and running program | Yes, while running |

One image can start **many** containers:

```
              ┌──> Container A (running)
Image  ───────┼──> Container B (running)
(read-only)   └──> Container C (stopped)
```

### 3.3 Solving the earlier problem

Without Docker, two apps that need different Python versions fight over one machine. With Docker each app gets its own container:

```
        One laptop or server
┌────────────────────┬────────────────────┐
│ Container 1        │ Container 2        │
│ App A + Python 3.6 │ App B + Python 3.12│
└────────────────────┴────────────────────┘
      isolated, no conflicts
```

You give a teammate the **image** (or, more commonly, the small text recipe that builds it, called a Dockerfile — Chapter 15). They run it. It behaves the same, whether they use Windows, macOS or Linux.

---

## 4. A first look (optional, 2 minutes)

If Docker is already installed you can try this right now. If not, skip it; Chapter 14 walks through installation.

```bash
docker run hello-world
```

What happens behind the scenes:

1. The `docker` CLI asks the Docker Engine to run an image called `hello-world`.
2. The Engine does not have the image locally, so it downloads it from Docker Hub.
3. It creates a container from the image and runs it.
4. The program prints a welcome message and exits.

You will see text beginning with `Hello from Docker!`. That one command showed the whole idea: **download an image, start a container from it**.

---

## 5. A short history

| Year | Event |
|---|---|
| 1956 | First container ship, based on Malcolm McLean's shipping container idea |
| 2008–2010 | Linux gains the building blocks of containers (cgroups, namespaces) and tools like LXC appear. The company **dotCloud**, a Platform-as-a-Service startup, is founded (2010) by Solomon Hykes and others |
| March 2013 | Solomon Hykes gives a short "lightning talk" at **PyCon** and demos the internal tool dotCloud used. Docker is released as open source |
| 2013 | dotCloud renames itself **Docker, Inc.** because the tool became more popular than the platform |
| 2014 | Google open-sources **Kubernetes**, a system for running many containers across many machines |
| 2015 | The **Open Container Initiative (OCI)** is founded to standardise the image and runtime formats, so containers are not owned by one company |
| Today | Containers are a standard way to deploy software across the industry |

**Why "Docker"?** In British English a *docker* is a dock worker who loads and unloads ships. Docker does that job for software.

**Why a whale?** The logo (a whale carrying containers, nicknamed *Moby Dock*) is the same idea: a big ship-like animal carrying a stack of containers.

> **Note on accuracy:** Docker did not invent the underlying Linux features. Its big contribution was making them easy to use, with a simple CLI, a portable image format and a shared registry (Docker Hub).

---

## 6. How Docker fits into a real workflow

```
  Write code ──► Build image ──► Test it locally ──► Push to a registry
                                                            │
        Run in production  ◄──  Pull image on the server  ◄─┘
```

1. **Build:** you describe your app in a `Dockerfile` and run `docker build`. That produces an image.
2. **Ship:** you push the image to a **registry** (Docker Hub, GitHub Container Registry, AWS ECR, ...).
3. **Run:** any server pulls the image and runs `docker run`. The container that runs in production is the *same* image you tested.

Benefits at each level:

| You are... | Docker helps you... |
|---|---|
| A beginner | Try software (databases, web servers) without installing it on your computer |
| A developer | Start a whole project with one command; give every teammate an identical setup |
| A tester | Create clean, throw-away environments for each test run |
| A DevOps / platform engineer | Deploy the exact artifact that passed the tests, and scale it with orchestrators like Kubernetes |

---

## 7. Myths and common misunderstandings

**"Docker is just a tool."**
It is a platform made of several parts (engine, CLI, desktop app, registry, Compose, ...).

**"A container is a small virtual machine."**
Not quite. A VM runs a full guest operating system on virtual hardware. A container is an isolated process that shares the host's kernel. Chapters 3–5 cover the difference in depth.

**"Docker is only for DevOps people."**
Developers, testers, data scientists and students use it too, for reproducible environments.

**"Docker will make my app faster."**
Not by itself. Containers add very little overhead, but the main benefits are consistency, isolation and easy deployment.

**"Kubernetes needs Docker."**
Kubernetes runs containers, but since version 1.24 (2022) it no longer talks to Docker Engine directly. It uses runtimes such as containerd or CRI-O. Images built with Docker still run fine there, because they follow the OCI standard. (Chapter 6 explains containerd.)

**"Containers are automatically secure."**
Isolation helps, but containers share a kernel with the host, so they need careful configuration (for example, do not run as root unless required).

**"The whale means Docker is heavy."**
Only if you run too many containers on too little RAM. A single container is light.

---

## 8. Key terms (mini glossary)

| Term | Meaning |
|---|---|
| **Container** | An isolated running instance of an image |
| **Image** | A read-only template used to create containers |
| **Dockerfile** | A text file with instructions for building an image |
| **Registry** | A server that stores and distributes images (Docker Hub is the best known) |
| **Docker Engine** | The background service (`dockerd`) that builds and runs containers |
| **Docker CLI** | The `docker` command you type in a terminal |
| **Docker Desktop** | Docker's desktop app for Windows, macOS and Linux |
| **OCI** | Open Container Initiative, the standard for image and runtime formats |
| **Kubernetes** | A system that schedules containers across many machines |

---

## 9. Check your understanding

1. In one sentence, what problem does Docker solve?
2. What is the difference between an image and a container?
3. Does a container include its own kernel? Why does that matter for start-up speed?
4. Why is the shipping-container analogy a good one? Name two similarities.
5. True or false: you need Docker to run Kubernetes.

<details>
<summary>Answers</summary>

1. It makes an app run the same way in every environment by packaging the app with its dependencies.
2. An image is a read-only template; a container is a running (or stopped) instance created from it. One image can create many containers.
3. No, it shares the host kernel. Because no extra OS has to boot, containers start in about a second and use less memory than VMs.
4. Any two of: standard format, contents don't matter to the carrier, works across many "transport" systems, moves quickly, cheap to handle.
5. False. Kubernetes needs a container runtime such as containerd or CRI-O; Docker Engine is not required.
</details>

---

## 10. Where to go next

- **Chapter 2 – Kernel:** what the kernel is. You need this to understand why containers are light.
- **Chapter 3 – Virtual Machine:** the older way of isolating software.
- **Chapter 4 – Container:** how containers really work (namespaces and cgroups).
- **Chapter 14 – Docker hands-on:** install Docker and run your first containers.

**Further reading:** the [official Docker overview](https://docs.docker.com/get-started/docker-overview/) and the [OCI website](https://opencontainers.org/).
