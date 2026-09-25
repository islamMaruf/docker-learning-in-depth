# Chapter 21: The OSI Model: Why Networks Are Built in Layers

> **In one sentence:** The OSI model splits "sending data over a network" into **seven layers**, each with one job, so that different vendors' hardware and software can work together and so that you can reason about (and debug) a network one layer at a time.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~45 minutes

**Prerequisites:** none for the theory. The hands-on commands (section 9) run on any Linux/macOS terminal, or inside a container (Chapter 10), e.g. `docker run -it --rm ubuntu:24.04 bash` after `apt-get update && apt-get install -y iproute2 iputils-ping curl`.

---

## What you will learn

- The problem the OSI model solved, and what "model" really means
- The **seven layers**, their jobs, their data units, and their real-world examples
- How **encapsulation** works: data being wrapped on the way down and unwrapped on the way up
- Which addresses live at which layer (MAC, IP, port) and who reads them
- Why the real internet uses the simpler **TCP/IP** model (next chapter), and how the two relate
- How to use layers to **troubleshoot** and to understand phrases like "L4 load balancer"
- How this applies to Docker and Kubernetes networking
- Commands to *see* each layer on your own machine

---

## 1. Why a model? The problem of the 1970s and 80s

Early computer networks were built by individual vendors. IBM had its own architecture (SNA), Digital Equipment had DECnet, and other companies had their own. Each had its own rules ("protocols") for how bits become messages. A computer from one vendor typically **could not talk** to another's, which is like two people who speak different languages and have no translator.

Two standards efforts appeared: the **ISO** (International Organization for Standardization) published the **OSI reference model** (1984), while, in parallel, the internet community built the **TCP/IP** protocols. The OSI *protocols* themselves were mostly never adopted, but the OSI **model** became the universal vocabulary of networking. It is a *shared language for describing networks*.

### What does the name mean?
**O**pen **S**ystems **I**nterconnection:

- **System**: a computer or device with its own software.
- **Open**: its rules are publicly specified (as opposed to secret, proprietary ones), so anyone can implement them.
- **Interconnection**: making *different* systems ("inter") talk to one another.

### What is a "model"?
A model is a **framework for thinking**, not a program you install or a wire you plug in. Like a building blueprint, no layer is a physical thing you can point to; real protocols and devices are **implementations** that fit into the framework.

### The big idea: layering
Split a huge problem into layers where:

1. Each layer has **one clear job**.
2. Each layer **uses the services** of the layer below, and **provides services** to the layer above.
3. A layer can be **replaced** without changing the others. (You can switch from Wi-Fi to a cable, and your web browser doesn't care. You can switch from HTTP/1.1 to HTTP/2, and your cable doesn't care.)

This is the same reason a postal system works: you write a letter (content), put it in an envelope with an address (routing), and the postal truck drivers don't care what is inside.

---

## 2. The seven layers at a glance

```
 Layer  Name           Job (in one line)                          Data unit    Examples
 ─────  ─────────────  ─────────────────────────────────────────  ───────────  ──────────────────────────
  7     Application    What the user's program speaks             Data         HTTP, DNS, SMTP, SSH, FTP
  6     Presentation   Formatting, encoding, encryption, compress Data         TLS*, UTF-8, JPEG, JSON
  5     Session        Start/keep/end conversations               Data         (RPC sessions, NetBIOS)*
  4     Transport      Process-to-process delivery: ports,        Segment (TCP) TCP, UDP
                       reliability, ordering                      Datagram(UDP)
  3     Network        Host-to-host delivery across networks:     Packet       IP, ICMP, routers
                       IP addresses, routing
  2     Data Link      Delivery inside ONE network: MAC           Frame        Ethernet, Wi-Fi, ARP*,
                       addresses, switches, error detection                    switches
  1     Physical       Bits as electrical/light/radio signals     Bit          Cables, fiber, radio, hubs
```

*Layer placement of a few protocols is debated, see section 6.

### Memory aids
Top to bottom (7→1): **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing. Bottom to top (1→7): **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way, or the one this series used: **P**lease **D**o **N**ot **T**ell **S**ecret **P**asswords to **A**nyone. Any works: pick one and keep it.

Professionals use **numbers**: "L2 switch", "L3 router", "L4 load balancer", "L7 proxy". Learn the numbers.

---

## 3. The layers one by one

We follow a chat message, "Hello", sent from your phone to a friend's laptop.

### Layer 7: Application
- **Job:** the protocols that applications speak to each other and to users. It is where *your* software lives (browser, chat app, `curl`, an API server).
- **Examples:** **HTTP/HTTPS** (web), **DNS** (names → IPs), **SMTP/IMAP** (email), **SSH** (remote login), **FTP**, **MQTT**, gRPC.
- **Our example:** the chat app decides to send `{"text":"Hello"}` with an HTTP request `POST /messages`.

### Layer 6: Presentation
- **Job:** agree on how data is *represented*: character encoding (UTF-8), serialization (JSON, XML, protobuf), image/video formats, **compression** and **encryption**.
- **Real world:** these things are usually done inside the application or by a library (a JSON library, a TLS library). **TLS** encrypts application data and is often *described* at L6 (or L5), but it actually runs on top of TCP and below HTTP, so it doesn't fit cleanly (Chapter 29).

### Layer 5: Session
- **Job:** set up, maintain and tear down *conversations* between applications; checkpoints and resuming.
- **Real world:** rarely a separate layer today. Login sessions, cookies, WebSocket connections, RPC sessions and TLS sessions are handled by applications and libraries. Treat L5-L7 together as "the application side" (which is exactly what the TCP/IP model does).

### Layer 4: Transport
- **Job:** deliver data between **processes** (applications) on two hosts, identified by **port numbers**. Optionally add **reliability**: acknowledgements, retransmission, ordering, flow control and congestion control.
- **Protocols:** **TCP** (reliable stream, connection-oriented; web, email, SSH) and **UDP** (fast, no guarantees; DNS queries, video calls, games). QUIC (HTTP/3) is built over UDP.
- **Ports:** a computer has **one IP address but 65,535 ports** per protocol; the port says *which program* gets the data. **Servers listen on well-known ports** (80 HTTP, 443 HTTPS, 22 SSH, 53 DNS, 5432 PostgreSQL). **Clients use a temporary ("ephemeral") source port** picked by the OS (typically 32768-60999 on Linux). Your browser doesn't have "port 3000"; each connection gets a fresh random source port.
- **Data unit:** a **segment** (TCP) or a **datagram** (UDP): the application data plus a **transport header** (source port, destination port, and for TCP sequence numbers, flags, and so on).
- **Our example:** the message is split (if large) into segments, from source port 51724 to destination port 443.

### Layer 3: Network
- **Job:** get a packet from **any host to any other host**, possibly across many networks, using **IP addresses** and **routing**.
- **Protocols:** **IP** (IPv4/IPv6), **ICMP** (ping, error messages), routing protocols (BGP, OSPF).
- **Devices:** **routers** (they read the IP header and choose the next hop).
- **Data unit:** a **packet**: the segment plus an **IP header** (source IP, destination IP, TTL, protocol number...).
- **Address analogy:** an IP is like a *street address* and can change when you move (or as your network changes).
- **Our example:** from your phone's IP `192.168.1.23` (private, behind NAT) to the chat server's public IP.

### Layer 2: Data Link
- **Job:** deliver a **frame** between two devices that are directly connected on the **same local network** (same "link"), using **MAC addresses**; detect corrupted frames (a checksum called the **FCS**).
- **Technologies:** **Ethernet**, **Wi-Fi (802.11)**, PPP; **switches** and **bridges** work here; **ARP** helps find the MAC that owns an IP on the local network. VLANs also live here.
- **MAC address:** a 48-bit hardware address like `a4:83:e7:1c:9b:02`, assigned at manufacture (the first half identifies the vendor). It is only meaningful **on the local network**, and **it is not truly unchangeable**: operating systems can change or randomize it (phones do, for privacy), and virtual machines and containers get generated ones.
- **Data unit:** a **frame**: the packet wrapped by a **header** (destination MAC, source MAC, type) and a **trailer** (FCS).
- **Key point:** **MAC addresses change at every hop**, while IP addresses stay the same end to end. The frame from your phone to your router has the router's MAC as destination; the router builds a *new* frame for the next link.

### Layer 1: Physical
- **Job:** move raw **bits** as voltage, light pulses or radio waves. Connectors, cable types, frequencies, signal encoding.
- **Devices/media:** copper (Cat5e/6), fiber, radio (Wi-Fi, 5G), **hubs** and repeaters.
- **Data unit:** the bit.

---

## 4. Encapsulation: how data travels down and up

On the sending side, each layer takes what the layer above gave it and **adds its own header** (and at L2, a trailer). This is **encapsulation**. The receiver reverses it (**decapsulation**), each layer reading and removing *its own* header.

```
Sender                                                           Receiver
┌───────────────────────────────────────┐                        ┌─────────────────────────────────┐
│ L7 Application:      "Hello"          │                        │ L7  "Hello"                     │
│        ▼                              │                        │        ▲                        │
│ L4: [TCP hdr | Hello]      SEGMENT    │                        │ L4  strip TCP hdr, deliver to   │
│        ▼                              │                        │     the app listening on port   │
│ L3: [IP hdr | TCP hdr | Hello] PACKET │                        │ L3  IP dest is me? strip IP hdr │
│        ▼                              │                        │        ▲                        │
│ L2: [Eth hdr | IP | TCP | Hello | FCS]│                        │ L2  MAC dest is me? FCS ok?    │
│                                FRAME  │                        │     strip Eth hdr + FCS         │
│        ▼                              │      ── bits ──►       │        ▲                        │
│ L1: 1010011101...                     │ ═══════════════════════│ L1  signals → bits              │
└───────────────────────────────────────┘                        └─────────────────────────────────┘
```

Read the frame **from the outside in**. The order on the wire is:

```
| Ethernet header (dst MAC, src MAC, type) | IP header (src IP, dst IP, ...) | TCP header (src port, dst port, ...) | DATA | Ethernet trailer (FCS) |
```

(Many simplified diagrams place "sender info" on the left and "receiver info" on the right of the payload. In reality each layer's header goes in **front of** its payload, and only the Ethernet layer has a trailer.)

Typical sizes: Ethernet header 14 bytes, IPv4 header 20 bytes (min), TCP header 20 bytes (min), Ethernet's maximum payload (the **MTU**) 1500 bytes, so a full frame carries at most about 1460 bytes of TCP data.

### The receiver's questions
Each layer answers one question and then hands the data upward:

| Layer | Question | If "no" |
|---|---|---|
| L2 | Is this frame **addressed to my MAC** (or broadcast), and is it intact (FCS)? | Ignore or drop |
| L3 | Is this **IP addressed to me**? (A router instead forwards it.) | Drop, or forward |
| L4 | **Which application** (port) wants this? Is anyone listening? | Send "port unreachable" or a TCP reset |
| L7 | Is it a valid request for this app? | The app replies with an error, e.g. HTTP 400 |

### Which device looks at which layer?

| Device | Highest layer it reads | Decision it makes |
|---|---|---|
| Hub / repeater | L1 | Copies bits to all ports |
| **Switch** | L2 | Forwards a frame by destination MAC |
| **Router** | L3 | Forwards a packet by destination IP (rewrites the L2 header at each hop) |
| Firewall | L3–L4 (stateful), up to L7 (next-gen/WAF) | Allow/deny by IP, port, connection state, or content |
| **L4 load balancer** | L4 | Spreads TCP/UDP connections by IP + port (doesn't read HTTP) |
| **L7 load balancer / reverse proxy** | L7 | Routes by HTTP host, path, header, cookie (nginx, Envoy, HAProxy, ingress controllers) |
| Your OS kernel | L2-L4 (network stack) | Also implements ARP, IP, TCP, UDP |

---

## 5. Data-unit vocabulary (PDU)

| Layer | Name | Contains |
|---|---|---|
| 7-5 | **Data** / message | Application payload |
| 4 | **Segment** (TCP) / **Datagram** (UDP) | Payload + ports (+ TCP details) |
| 3 | **Packet** (IP datagram) | Segment + IP addresses |
| 2 | **Frame** | Packet + MAC addresses + FCS |
| 1 | **Bits** | Signals |

People often say "packet" loosely for any of these. In an interview or a design review it's worth using them precisely.

### Three address types, three jobs

| Address | Layer | Scope | Example | Changes hop by hop? |
|---|---|---|---|---|
| **MAC** | 2 | One local network (link) | `a4:83:e7:1c:9b:02` | **Yes** |
| **IP** | 3 | Whole internet | `192.168.1.23`, `2001:db8::1` | No (except NAT) |
| **Port** | 4 | One computer's processes | `443` | No (except NAT/PAT) |

A full "address" of a connection is the **5-tuple**: protocol, source IP, source port, destination IP, destination port. Firewalls, NAT and load balancers track connections with it.

---

## 6. OSI vs TCP/IP: the model versus the real thing

The internet was built with the **TCP/IP** protocol suite, which has **four** (sometimes five) layers. The OSI model is used as a *teaching and vocabulary* framework:

```
   OSI (7 layers)              TCP/IP (4 layers)          Examples
┌─────────────────┐
│ 7 Application   │
│ 6 Presentation  │ ──────►  Application               HTTP, DNS, TLS, SSH, SMTP
│ 5 Session       │
├─────────────────┤
│ 4 Transport     │ ──────►  Transport                 TCP, UDP, QUIC
├─────────────────┤
│ 3 Network       │ ──────►  Internet                  IP, ICMP
├─────────────────┤
│ 2 Data Link     │ ──────►  Link (Network Access)     Ethernet, Wi-Fi, ARP
│ 1 Physical      │
└─────────────────┘
```

Chapter 22 covers TCP/IP in detail. Points to remember:

- Real protocols **don't respect the layer boundaries perfectly**: ARP sits between L2 and L3, TLS between L4 and L7, MPLS is "L2.5", QUIC merges transport, encryption and session setup, VPNs put entire packets inside other packets (**tunneling**).
- Some protocols place **layers inside layers**: an HTTP request is inside TLS inside TCP inside IP inside Ethernet.
- "Layer 8" is a joke about the user.

---

## 7. Layers and Docker / Kubernetes

| Concept | Layer | What it means for containers |
|---|---|---|
| Container's `eth0` and its **MAC** | L2 | Each container has a virtual Ethernet interface (one end of a **veth pair**) attached to a **Linux bridge** (`docker0`), which behaves like a virtual **switch** |
| Container's **IP** (e.g. `172.17.0.2`) | L3 | Assigned from the Docker network's subnet; the host **routes** between the bridge and the outside world |
| **NAT / port publishing** `-p 8080:80` | L3–L4 | Docker installs firewall/NAT rules (iptables/nftables) that translate host `IP:8080` → container `IP:80` |
| **Ports** the app listens on | L4 | `EXPOSE 80`, `-p`, `ss -tlnp` |
| **Docker's embedded DNS** (container names → IPs) | L7 (DNS) | Service discovery on user-defined networks |
| **Reverse proxy / Ingress** (nginx, Traefik) | L7 | Route by host/path |
| **Kubernetes Service** (kube-proxy) | L4 | Virtual IP load-balancing connections |
| **Ingress / Gateway** | L7 | HTTP routing, TLS termination |
| **NetworkPolicy** | L3–L4 | Which pods may talk to which, on which ports |
| **Service mesh** (Istio, Linkerd) | L7 (and mTLS) | Per-request routing, retries, encryption between services |
| **Network namespace** | L2–L4 | Each container has its own interfaces, IPs, routing table, port space (Chapter 4) |

Docker network drivers: **bridge** (default: virtual switch on one host), **host** (share the host's network stack), **none**, **overlay** (multi-host virtual network, encapsulating L2 frames inside UDP packets: VXLAN), **macvlan** (containers appear as separate devices with their own MAC on the physical LAN).

---

## 8. Troubleshooting with layers

Work **bottom-up** (or use divide and conquer: start in the middle with a ping or a connection test):

| Layer | Question | Command / clue |
|---|---|---|
| **1 Physical** | Is the link up? Cable, Wi-Fi, interface enabled? | `ip link show` (look for `UP`, `LOWER_UP`); link lights; `ethtool eth0` |
| **2 Data Link** | Right VLAN/switch? Can I see the neighbor's MAC? | `ip neigh` (ARP table); `arp -a`; `bridge fdb` |
| **3 Network** | Do I have an IP, a route, a gateway? Can I ping the destination IP? | `ip addr`, `ip route`, `ping 192.168.1.1`, `ping 8.8.8.8`, `traceroute` / `mtr` |
| **3 (names)** | Does the name resolve? | `getent hosts example.com`, `dig example.com`, `nslookup` |
| **4 Transport** | Is anything listening? Is the port reachable/filtered? | `ss -tlnp` (server side), `nc -vz host 443`, `curl -v telnet://host:443` |
| **5-6** | TLS/certificates OK? | `openssl s_client -connect host:443 -servername host`, `curl -v https://...` |
| **7 Application** | Does the app answer correctly? | `curl -i https://host/path`, logs, HTTP status codes |

Typical symptoms and where to look:

| Symptom | Likely layer |
|---|---|
| "Network cable unplugged", no Wi-Fi | 1 |
| Can ping the router but not other LAN hosts; duplicate IP | 2 |
| `Network is unreachable`, no default route; ping by IP fails | 3 |
| `Temporary failure in name resolution`, ping by IP works | 7 (DNS) |
| `Connection refused` (instant): host is reachable but nothing listens on that port | 4 |
| `Connection timed out`: packets dropped, usually a firewall or a wrong route | 3/4 |
| `certificate verify failed`, `SSL_ERROR...` | TLS (6/7) |
| HTTP 404/500 | 7 |

`refused` vs `timeout` is one of the most useful distinctions in networking: **refused** = you reached the machine and it said no; **timeout** = you got no answer at all.

---

## 9. Hands-on: see the layers on your machine

Use Linux (or WSL2, or a container with `iproute2 iputils-ping curl tcpdump` installed).

**Layer 1/2: interfaces and MACs**

```bash
ip -br link
# lo    UNKNOWN  00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
# eth0  UP       02:42:ac:11:00:02 <BROADCAST,MULTICAST,UP,LOWER_UP>     ← 02:42:... is a Docker-generated MAC
```

**Layer 2 ↔ 3: IP addresses and the neighbor (ARP) table**

```bash
ip -br addr             # IP per interface
ip neigh                # IP → MAC entries your machine has learned
```

**Layer 3: routing**

```bash
ip route                # "default via 172.17.0.1 dev eth0": where packets to the internet go
ping -c 3 172.17.0.1    # L3 reachability of the gateway
traceroute -n 8.8.8.8   # each router (hop) along the way; (apt-get install traceroute)
```

**Layer 4: ports and connections**

```bash
ss -tlnp                # TCP listening sockets (which ports have servers)
ss -tn                  # established TCP connections (local ip:port ↔ remote ip:port = the 5-tuple)
nc -vz example.com 443  # can I open a TCP connection to port 443?
```

**Layer 7 with all the layers underneath**

```bash
curl -v https://example.com 2>&1 | head -30
# * Trying 93.184.x.x:443...           ← L3/L4: resolved IP, connecting
# * Connected to example.com port 443  ← L4: TCP handshake done
# * TLSv1.3 (OUT), TLS handshake ...   ← TLS
# > GET / HTTP/2                       ← L7: your request
# < HTTP/2 200                         ← L7: response
```

**Watching real frames (needs root):** in one terminal run `sudo tcpdump -i any -nn -e -c 20 port 80`, then in another `curl -s http://example.com >/dev/null`. The `-e` flag shows **MAC addresses (L2)**, and the lines show **IPs (L3)** and **ports and TCP flags (L4)**. **Wireshark** displays each layer's header as an expandable tree; it is the best way to make encapsulation real.

**Docker view:**

```bash
docker network ls
docker run -d --name web nginx:1.27-alpine
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{.MacAddress}}{{end}}' web   # its L3 and L2 addresses
ip -br link | grep -E 'docker0|veth'         # the virtual switch and the container's veth end (on the host)
docker exec web ip addr                      # container's own eth0
docker exec web netstat -tln 2>/dev/null || docker exec web ss -tln     # L4: listening on :80
```

---

## 10. Common misconceptions

| Misconception | Reality |
|---|---|
| "The internet uses the OSI model" | It uses TCP/IP. OSI is a reference model and vocabulary |
| "Each layer is a separate program" | Layers are roles. The kernel implements L2–L4; libraries and apps handle L5–L7; hardware does L1–L2 |
| "The receiver adds nothing; only the sender wraps the data" | Both do: sender adds headers going down; the receiver strips them going up. And routers rewrite L2 headers at every hop |
| "MAC addresses are permanent and unique forever" | Usually assigned once, but can be changed, spoofed, or randomized, and only matter on the local link |
| "IP addresses identify a device forever" | They are assigned (often by DHCP) and change; many devices share one public IP through NAT |
| "A port belongs to an application forever, e.g. WhatsApp is port 3000" | Servers use well-known/configured ports; **clients get random ephemeral source ports** |
| "Packet, frame and segment all mean the same" | They are layer-specific names (L3, L2, L4) |
| "HTTPS is layer 6/7 only" | TLS sits between TCP and HTTP. Categorization is fuzzy |
| "Presentation = JSON/HTML" | Those are application-level formats; L6 is about representation concerns. In practice, applications handle it |
| "Layer numbers tell the order of a request" | Data is processed from 7 down to 1 on the sender and 1 up to 7 on the receiver |

---

## 11. Summary

- OSI = a **seven-layer reference model**: 7 Application, 6 Presentation, 5 Session, 4 Transport, 3 Network, 2 Data Link, 1 Physical.
- Each layer has one job and offers a service to the layer above; that makes networks **interchangeable, testable and debuggable**.
- **Encapsulation:** headers are added going down, removed coming up: data → segment/datagram (ports) → packet (IPs) → frame (MACs + FCS) → bits.
- **MAC** = link-local, **IP** = end-to-end, **port** = which process. MACs change at every hop; IPs don't.
- Real networks run **TCP/IP**; OSI numbers (L2 switch, L3 router, L4/L7 load balancer) are the industry vocabulary.
- Docker: veth + bridge (L2), container IPs and NAT (L3), ports (L4), DNS/proxies (L7).
- Troubleshoot by layer: link → IP/route → port → TLS → application; **refused** vs **timeout** narrows it fast.

---

## 12. Check your understanding

1. Name the seven layers from 7 to 1 and the data unit at layers 4, 3 and 2.
2. Which layer are `ping`'s ICMP messages, an Ethernet switch, a router and a TCP port at?
3. As a packet crosses three routers, which addresses change: source/destination MAC, source/destination IP?
4. What is the difference between "connection refused" and "connection timed out"?
5. Why do we say a browser doesn't "use port 3000"? Which port does it use to connect to an HTTPS site, and which on its own side?
6. A team says "we need an L7 load balancer". What can it do that an L4 load balancer can't?
7. Where do Docker's `-p 8080:80` rules operate, and why isn't it just a port "opening"?

<details>
<summary>Answers</summary>

1. Application, Presentation, Session, Transport, Network, Data Link, Physical. Segment (datagram for UDP) at 4, packet at 3, frame at 2.
2. ICMP: L3. Switch: L2. Router: L3. TCP port: L4.
3. The MAC addresses (a new L2 frame per hop). Source and destination IP stay the same (barring NAT).
4. Refused: the host answered, but nothing listens on that port (or it sent a reset). Timeout: no response at all, typically a firewall drop, wrong route, or host down.
5. Servers use well-known ports; clients get a random ephemeral source port. HTTPS server side: 443; client side: a random high port.
6. It can read the HTTP request (host, path, headers, cookies) and route/modify based on it; an L4 balancer only sees IPs and ports.
7. At L3/L4: NAT rules in iptables/nftables translate host `IP:8080` to container `IP:80`; a forwarding rule, not just an opened port.
</details>

**Practice**

1. On your machine, run `ip -br addr`, `ip route`, `ip neigh`, `ss -tlnp`. For each line of output, write down its OSI layer and which addresses (MAC/IP/port) it contains.
2. Start a container, and from the host, find its IP (`docker inspect`) and its MAC; ping it; find it in `ip neigh` on the host. Which layers did you just touch?
3. Run `sudo tcpdump -i any -nn -e port 80` while you `curl` an HTTP site. Identify in one captured line each layer's information.
4. Draw the encapsulation of `curl http://example.com` as a nested set of boxes, with the header fields you found in step 3.

---

**Next:** [Chapter 22 – The TCP/IP Model](22_tcp_ip_model.md)
