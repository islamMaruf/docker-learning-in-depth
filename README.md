# Docker & Linux, From First Principles to Production

A beginner-friendly, hands-on learning path through **Docker, Linux and computer networking**. 43 chapters, about 151,000 words, each written to be read in order or on its own.

> Every chapter starts with a **one-sentence summary** and **what you will learn**, explains new terms in plain language before using them, includes **runnable labs**, and ends with **quiz questions (with answers) and practice tasks**.

## Who is this for?

- **Beginners** who have never used Docker, Linux or a terminal
- **Developers** who want to containerize applications and understand what happens underneath
- **DevOps / SRE / sysadmins** who want solid networking and Linux foundations
- **Students** preparing for cloud and infrastructure roles
- **Network-curious people** who want to follow a packet from browser to wire

## What you will be able to do

- Explain what containers are (namespaces, cgroups, images, layers) and how they differ from VMs
- Write Dockerfiles, run and manage containers, and debug them
- Work confidently in a Linux shell: packages, users, permissions, processes
- Explain and inspect the network stack: TCP, UDP, HTTP/1.x/2, DNS, TLS, IP, Ethernet, DHCP, ARP
- Plan subnets with CIDR, read routing tables, and predict which interface a packet uses
- Reproduce networks safely with Linux network namespaces

## Chapters


### Part 1: Docker Fundamentals (chapters 1-9)

| # | Chapter | What it covers |
|---|---|---|
| 1 | [What Is Docker?](01_what_is_docker.md) | Docker lets you pack an application together with everything it needs to run, so it behaves the same on your laptop, your teammate's laptop, and a production server. |
| 2 | [The Kernel](02_kernel.md) | The kernel is the core program of an operating system. It is the only software allowed to touch the hardware directly, and every other program must ask it for help. |
| 3 | [Virtual Machines](03_virtual_machine.md) | A virtual machine (VM) is a complete, pretend computer, with its own virtual CPU, memory, disk and its own operating system, running as software inside a real computer. |
| 4 | [Containers (Namespaces + cgroups)](04_container.md) | A container is an ordinary Linux process (or group of processes) that the kernel has put in a "bubble": namespaces control what it can *see*, and cgroups control how much it can *use*. |
| 5 | [Containers vs Virtual Machines, and the Docker Engine](05_container_vs_vm_docker_engine.md) | Containers and VMs both isolate software, but a VM carries its own OS and kernel while a container shares the host's kernel. Docker Engine is the program that makes those containers easy to create, and Docker Desktop is how Windows and macOS users get a Linux kernel to run them on. |
| 6 | [Docker Engine Internals](06_docker_engine_internals.md) | When you type `docker run`, the `docker` CLI sends an HTTP request to a background service (`dockerd`), which hands the work down through containerd and runc, and finally the Linux kernel creates the container. |
| 7 | [The Docker Ecosystem](07_docker_ecosystem.md) | "Docker" is not one program. It is a platform of cooperating parts: the CLI, the Engine, images, registries such as Docker Hub, Docker Desktop, Docker Compose, and more. |
| 8 | [Linux, Distributions, and Why Docker Needs Them](08_linux.md) | "Linux" is strictly a kernel; a distribution (Ubuntu, Debian, Alpine, ...) wraps that kernel with tools, libraries and a package manager to make a full operating system. Docker containers use the host's Linux kernel plus the *user-space files of a distribution* that come from the image. |
| 9 | [GNU Coreutils, Shells and Terminals](09_gnu_coreutils.md) | When you type `ls` in a terminal, a terminal shows your keystrokes, a shell interprets them and starts a program from coreutils, and that program asks the kernel to do the work. |

### Part 2: Linux Fundamentals (chapters 10-13)

| # | Chapter | What it covers |
|---|---|---|
| 10 | [Running Ubuntu in Docker (Your First Real Container)](10_running_ubuntu_on_docker.md) | In this chapter you download an Ubuntu image, start an interactive container from it, look around inside, exit, and learn what happens to the container afterward. |
| 11 | [Managing Packages on Linux](11_managing_packages_on_linux.md) | On Linux you install software with a package manager (like `apt`) that downloads verified packages from repositories, installs their dependencies automatically, and can update or remove everything cleanly. |
| 12 | [Linux Basic Commands](12_linux_basic_commands.md) | Learn to move around the Linux file system and to create, view, copy, move, search and delete files from the command line, the daily toolkit for working inside containers and on servers. |
| 13 | [Users, Groups and Permissions](13_managing_user_group_and_permission.md) | Linux decides who may read, change or run every file using three ideas: users (who you are), groups (teams you belong to) and permissions (`r`, `w`, `x` for the owner, the group, and everyone else). Docker containers use the exact same system, and getting it wrong is behind many "permission denied" errors and security problems. |

### Part 3: Docker in Practice (chapters 14-20)

| # | Chapter | What it covers |
|---|---|---|
| 14 | [Docker Hands-On (Images, Containers, `exec` and Your First Build)](14_docker_hands_on.md) | This is the practical chapter. You install Docker, learn the day-to-day commands for images and containers, open extra shells in running containers, and build your first custom image with a Dockerfile. |
| 15 | [Towards the Dockerfile](15_towards_the_dockerfile.md) | Instead of typing setup commands by hand inside a container and saving the result, you write them once in a text file (a Dockerfile) so that anyone can rebuild the exact same image with a single command. |
| 16 | [`CMD` (and `ENTRYPOINT`): What Runs When a Container Starts](16_cmd_deep_dive.md) | `CMD` stores the default command in an image so that `docker run IMAGE` starts your application by itself; you can replace it at run time, and `ENTRYPOINT` is its stricter sibling that fixes *what program* runs. |
| 17 | [`WORKDIR` Deep Dive](17_workdir_deep_dive.md) | `WORKDIR` sets the directory in which the following Dockerfile instructions run, and in which the container starts, so you can use short relative paths instead of repeating long absolute ones. |
| 18 | [Detached Mode: Running Containers in the Background](18_detach_mode.md) | `docker run -d` starts a container in the background and gives your terminal back; you then use `docker logs`, `docker exec`, `docker stop` and `-p` port publishing to work with it. |
| 19 | [Managing Containers: The Whole Life Cycle](19_managing_containers.md) | Containers are created, started, stopped, restarted and removed. This chapter is the practical toolkit for doing each of those things, for finding and inspecting containers, and for cleaning up, with the important rule that *most configuration changes require re-creating a container, not restarting it*. |
| 20 | [How Docker Builds Images: Layers, Cache and Multi-Stage Builds](20_building_magic_behind_dockerfile.md) | An image is a stack of read-only layers; `docker build` produces one layer per file-changing instruction, caches each layer, and reuses it as long as nothing above or in that step has changed. If you understand that, you can write Dockerfiles that build in seconds and ship small, safe images. |

### Part 4: Networking Fundamentals (chapters 21-23)

| # | Chapter | What it covers |
|---|---|---|
| 21 | [The OSI Model: Why Networks Are Built in Layers](21_philosophy_of_osi_model.md) | The OSI model splits "sending data over a network" into seven layers, each with one job, so that different vendors' hardware and software can work together and so that you can reason about (and debug) a network one layer at a time. |
| 22 | [The TCP/IP Model: How the Internet Really Works](22_tcp_ip_model.md) | The internet runs on the TCP/IP protocol suite, a simpler, four-layer cousin of the OSI model (Application, Transport, Internet, Link), in which IP delivers packets between hosts and TCP or UDP delivers data between programs. |
| 23 | [TCP in Detail](23_tcp_in_details.md) | TCP turns the unreliable, packet-by-packet network into a reliable, ordered byte stream by numbering every byte, acknowledging what arrives, retransmitting what doesn't, and slowing down when the receiver or the network can't keep up. |

### Part 5: Protocols and the Network Stack (chapters 24-34)

| # | Chapter | What it covers |
|---|---|---|
| 24 | [UDP in Detail](24_udp_in_details.md) | UDP is the simplest transport protocol: it adds only ports, a length and a checksum to your data and hands it to IP, with no connection, no acknowledgments, no retransmission and no ordering, which makes it fast, lightweight and ideal for real-time or "one question, one answer" traffic. |
| 25 | [HTTP/1.0 in Detail](25_http_1_0_in_details.md) | HTTP is the plain-text request/response language that browsers and servers speak; version 1.0 (1996) is its first widely used form, in which every request opened a new TCP connection, got one response, and closed it, simple and clear, but costly. |
| 26 | [HTTP/1.1 in Detail](26_http_1_1_in_details.md) | HTTP/1.1 fixed HTTP/1.0's biggest waste by making TCP connections persistent (reused for many requests), and added the `Host` header (virtual hosting), chunked transfer, better caching, range requests and more methods, but it still delivers responses one at a time per connection, which is why browsers open several connections and why HTTP/2 was invented. |
| 27 | [HTTP/2 in Detail](27_http_2_in_details.md) | HTTP/2 keeps the *meaning* of HTTP (methods, headers, status codes) but changes how messages are sent: as binary frames belonging to independent streams that are multiplexed over one TCP connection, with compressed headers, so slow responses no longer block fast ones at the HTTP level. |
| 28 | [DNS (the Domain Name System) in Detail](28_dns_domain_name_system_in_details.md) | DNS is the internet's distributed, hierarchical, cached "phone book": it turns names people can remember (`www.example.com`) into the IP addresses (and other facts) that computers need, through a chain of resolvers and name servers that each know one small piece. |
| 29 | [TLS (Transport Layer Security) in Detail](29_tls_transport_layer_security_in_details.md) | TLS wraps a TCP connection in a secure channel that gives you confidentiality (nobody can read the data), integrity (nobody can change it unnoticed) and authentication (you are really talking to the server you think), by using certificates to prove identity, key exchange to agree on a secret, and fast symmetric encryption for the actual data. HTTPS = HTTP over TLS. |
| 30 | [The Internet Protocol (IP) in Detail](30_internet_protocol_ip_in_details.md) | IP gives every network interface an address and delivers packets hop by hop from the sender to the destination across many networks, using each router's routing table, with no promises about delivery (it is *best effort*); TCP and UDP build reliability and ports on top. |
| 31 | [The Data Link Layer and Ethernet Frames](31_data_link_layer_frame_in_details.md) | The Data Link layer (Layer 2) moves data across one link (one local network) by wrapping each IP packet in a frame with MAC addresses (who on this link should receive it), a type (what is inside) and a checksum (was it damaged); switches forward frames by MAC address, and the frame is rebuilt at every router. |
| 32 | [The First Computer and the First Router](32_first_computer_and_first_router_in_details.md) | To put a computer on a network you need a network interface (hardware plus a driver) with a MAC address, an IP address (from DHCP, manual settings, or a self-assigned link-local fallback), a subnet mask and a default gateway; connecting to other networks and the internet requires a router, which forwards packets between networks and (at home) also does DHCP, NAT, Wi-Fi and a firewall. |
| 33 | [Subnetting and Subnet Masks](33_subnetting_and_subnet_masks_in_details.md) | An IP address has two parts, a network part and a host part, and the subnet mask is the ruler that tells every device where one ends and the other begins, which is how a host decides "is this destination on my own network (deliver directly) or somewhere else (hand it to the router)?" |
| 34 | [CIDR, Subnet and Subnet Mask: Understanding the Differences](34_cidr_subnet_subnet_mask_differences_in_details.md) | A subnet is the *thing* (a slice of address space that forms one network), a subnet mask is a *tool* (a 32-bit pattern that marks the network/host boundary) and CIDR is a *notation and addressing scheme* (`192.168.1.0/24`, prefix length instead of classes, which also allows aggregating routes); people mix the three words up constantly, and this chapter untangles them. |

### Part 6: DHCP, Devices and Routing (chapters 35-43)

| # | Chapter | What it covers |
|---|---|---|
| 35 | [DHCP Discover, Layer by Layer](35_dhcp_discover_deep_dive_in_details.md) | When a device joins a network with no address, it sends a DHCP Discover: a broadcast that walks down the whole stack: a DHCP message (Layer 7) inside a UDP datagram from port 68 to 67 (Layer 4) inside an IP packet from `0.0.0.0` to `255.255.255.255` (Layer 3) inside an Ethernet frame to `ff:ff:ff:ff:ff:ff` (Layer 2) sent as signals (Layer 1), asking "is there a DHCP server out there?". |
| 36 | [DHCP Offer, Layer by Layer](36_dhcp_offer_deep_dive_in_details.md) | The DHCP Offer is the server's reply to a Discover: "here is an address (`yiaddr`) I'm willing to lend you, plus the mask, router, DNS and lease time to go with it", sent from UDP port 67 to 68, from the server's real IP to either the offered address (unicast to the client's MAC) or to `255.255.255.255` (broadcast), because the client still can't be addressed the normal way. |
| 37 | [DHCP Request and Acknowledge](37_dhcp_request_and_acknowledge_in_details.md) | After picking an Offer, the client broadcasts a DHCP Request ("I accept *this* address from *that* server"), and the chosen server answers with a DHCP ACK that turns the provisional offer into a lease with a timer; the client then checks the address is really free, configures its interface, and starts the lease clock that drives renewal (T1) and rebinding (T2). |
| 38 | [Hub, Switch and Router](38_hub_switch_router_network_devices_in_details.md) | A hub repeats every signal out of every port (Layer 1, no intelligence), a switch learns which MAC lives on which port and forwards frames only where needed (Layer 2), and a router forwards packets between different IP networks (Layer 3); a home "router" is really a router, a switch and a Wi-Fi access point in one box. |
| 39 | [Networking Inside a Network: ARP](39_networking_inside_a_network_arp_protocol_in_details.md) | ARP (Address Resolution Protocol) answers the question "I know the IP address of the machine I want to reach on my own network; what is its MAC address?", by broadcasting a question to everyone on the local link and receiving a unicast answer from the owner, then caching the result so it doesn't have to ask again. |
| 40 | [Multiple NICs in a Single Computer](40_multiple_nics_in_single_computer_in_details.md) | Almost every real computer has more than one network interface (Ethernet + Wi-Fi + loopback + VPN + Docker bridge + more), each connected to a different network with its own IP address, mask and possibly gateway, which raises the question every OS must answer for every single packet: "*which interface should this go out of?*" (the answer is the routing table, the topic of Chapters 42–43). |
| 41 | [Visualizing Multiple NICs in a Single Computer](41_visualizing_multiple_nics_in_single_computer_in_details.md) | Your computer already has several network interfaces; this chapter teaches you to list them on Linux, macOS and Windows, to decode their names (`eth0`, `enp3s0`, `wlp2s0`, `en0`, `docker0`, `veth…`, `utun3`), and to tell physical from virtual interfaces. |
| 42 | [The Routing Table](42_routing_table_in_details.md) | The routing table is a list of rules of the form "*to reach destination network N, send the packet out interface I (via next-hop gateway G, if any)*" that every IP device consults for every packet it sends; it is filled automatically from your interfaces and DHCP, and manually or by routing protocols when needed. |
| 43 | [How the OS Chooses a NIC: The Complete Algorithm](43_how_os_chooses_nic_in_details.md) | For every outgoing packet the OS takes the destination IP, finds every routing-table entry it matches (`destination AND netmask == route network`), keeps the one with the longest prefix (ties broken by lowest metric), and sends the packet out that entry's interface to that entry's gateway (or directly to the destination if the route is on-link), using that interface's address as the source IP. |

## Suggested learning paths

| Goal | Read |
|---|---|
| **Complete beginner** | 1 → 2 → 3 → 4 → 5, then 8 → 12 → 13, then 14 → 20 |
| **Docker in depth** | 1, 4, 5, 6, 7, then 14-20, then 40-43 to understand container networking |
| **Linux essentials** | 2, 8-13 |
| **Networking fundamentals** | 21-24, 28, 30-34 |
| **How the web works** | 22, 23, 25-29 |
| **Local networks and routing** | 30-34, 38-43 |
| **Everything** | 1 → 43 in order |

## Chapter format

Each chapter contains, in this order: a one-sentence summary, level and reading time, prerequisites, "what you will learn", the explanation (with analogies, diagrams and real command output), labs you can run, common misconceptions, a summary, and a self-check quiz with answers plus practice tasks. A "Next" link points to the following chapter.

## Requirements

- Any computer with a terminal; Linux is best, macOS and Windows (WSL2) work
- [Docker](https://docs.docker.com/get-docker/) for chapters 1-20 and the container examples
- For the networking labs (chapters 32-43): a Linux machine or VM with `sudo`, `iproute2`, `tcpdump` and `iputils-ping`. The labs use network namespaces, so your real network is not touched
- About 4 GB RAM and an internet connection for pulling images

## Using this repository

```bash
git clone https://github.com/islamMaruf/docker-learning-in-depth.git
cd docker-learning-in-depth
grep -l "namespace" *.md      # find chapters that mention a topic
```
Read on GitHub, or in any Markdown viewer (VS Code, Obsidian).

## Statistics

| Part | Chapters | Words |
|---|---|---|
| Part 1: Docker Fundamentals | 1-9 | ~20,835 |
| Part 2: Linux Fundamentals | 10-13 | ~12,655 |
| Part 3: Docker in Practice | 14-20 | ~21,070 |
| Part 4: Networking Fundamentals | 21-23 | ~13,262 |
| Part 5: Protocols and the Network Stack | 24-34 | ~52,637 |
| Part 6: DHCP, Devices and Routing | 35-43 | ~30,714 |
| **Total** | **43** | **~151,173** |

## Where to go next

Docker Compose for multi-container apps, image security scanning, CI/CD with containers, Kubernetes, cloud networking (VPCs, load balancers), IPv6 deployment, VPNs and firewalls.

## License and contributing

No license file has been added yet; add one (for example MIT or CC BY 4.0) before reusing the content. Corrections and improvements are welcome through issues and pull requests.

---

**Start here:** [Chapter 1: What Is Docker?](01_what_is_docker.md)
