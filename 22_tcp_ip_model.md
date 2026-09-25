# Chapter 22: The TCP/IP Model: How the Internet Really Works

> **In one sentence:** The internet runs on the **TCP/IP** protocol suite, a simpler, four-layer cousin of the OSI model (Application, Transport, Internet, Link), in which **IP** delivers packets between hosts and **TCP or UDP** delivers data between programs.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~45 minutes

**Prerequisites:** [Chapter 21 – The OSI Model](21_philosophy_of_osi_model.md) (layers, encapsulation, ports, MACs, IPs).

---

## What you will learn

- Where TCP/IP came from and why it exists
- The **four (or five) layers** of TCP/IP and how they map to OSI
- What each layer does and which protocols live there
- **TCP vs UDP** in depth: guarantees, costs, headers, and how to choose
- **Sockets**: how programs actually use TCP and UDP (with runnable code)
- How data moves through the stack, with real header fields
- Docker and Kubernetes networking seen through TCP/IP
- Debugging by layer, and how to *watch* TCP and UDP with tools

---

## 1. Where TCP/IP came from

- **1969:** ARPANET, a research network funded by the US Department of Defense's ARPA, connects its first four sites (UCLA, Stanford Research Institute, UC Santa Barbara, University of Utah). It initially used a protocol called NCP.
- **1970s:** Researchers Vint Cerf and Bob Kahn design a way to connect *different networks* into an "inter-net": the **Transmission Control Program**, published in 1974, later split into **TCP** (reliable delivery) and **IP** (addressing and routing).
- **1981:** IP and TCP are published as standards (RFC 791 and RFC 793).
- **1 January 1983 ("flag day"):** ARPANET switches from NCP to TCP/IP. This is often called the birthday of the modern internet.
- **1984:** The ISO publishes the OSI model. TCP/IP is already working and free, and it wins in practice. The OSI *protocols* fade, while the OSI *layer names* stay as vocabulary.

> **A myth to skip:** you may hear that the internet was designed to survive nuclear war. Packet switching (an idea from Paul Baran and Donald Davies) *did* consider resilience, but ARPANET and TCP/IP were mainly built to share expensive computers and connect different research networks. What is true is the design principle: **no central control, and many independent networks that just agree on a few simple rules.**

TCP/IP was **built by running code and publishing rules** (RFCs, "Request for Comments", maintained today by the IETF). OSI was designed by committee before implementations existed. That's why the internet uses TCP/IP.

---

## 2. The layers of TCP/IP

```
    OSI (7 layers)              TCP/IP (4 layers)              Protocols
┌───────────────────┐
│ 7 Application     │
│ 6 Presentation    │──►  APPLICATION            HTTP, HTTPS(TLS), DNS, SSH, SMTP, IMAP,
│ 5 Session         │                            FTP, DHCP, NTP, gRPC, MQTT
├───────────────────┤
│ 4 Transport       │──►  TRANSPORT              TCP, UDP  (QUIC over UDP)
├───────────────────┤
│ 3 Network         │──►  INTERNET               IPv4, IPv6, ICMP, IPsec
├───────────────────┤
│ 2 Data Link       │                            Ethernet, Wi-Fi (802.11), ARP,
│ 1 Physical        │──►  LINK / NETWORK ACCESS  PPP, cables, radio
└───────────────────┘
```

Some textbooks split the bottom layer into "Data Link" + "Physical", giving a **five-layer** TCP/IP model. That is the most convenient one to use in practice, and it maps to OSI as 5 → 7/6/5, 4 → 4, 3 → 3, 2 → 2, 1 → 1.

**Why fewer layers?** The internet's protocols never separated "presentation" and "session" as distinct stages. Applications and their libraries do formatting, encryption and session handling themselves (your browser + TLS library + HTTP), so TCP/IP lumps everything above the transport into **Application**.

**The hourglass:** many application protocols on top, *one* protocol (IP) in the middle, many link technologies below. Because everything speaks IP, any application can run over any link (Wi-Fi, Ethernet, 5G, satellite, a VPN tunnel), which is the secret of the internet's growth.

In practice engineers use **OSI layer numbers** ("L2, L3, L4, L7") while the software implements the **TCP/IP** stack.

---

## 3. The layers in detail

### 3.1 Application layer
Everything that runs above TCP/UDP and is specific to an application: the request formats, encodings, encryption (TLS), sessions, authentication.

| Protocol | Purpose | Transport | Default port |
|---|---|---|---|
| **HTTP** / **HTTPS** | Web, APIs | TCP (HTTP/3 uses QUIC/UDP) | 80 / 443 |
| **DNS** | Names → IPs | UDP (TCP for big answers and zone transfers) | 53 |
| **SSH** / **SFTP** | Remote shell / file transfer over SSH | TCP | 22 |
| **SMTP** | Send email | TCP | 25 (587 submission, 465 SMTPS) |
| **IMAP** / **POP3** | Read email | TCP | 143 (993 TLS) / 110 (995 TLS) |
| **FTP** | File transfer (old, unencrypted) | TCP | 21 (control), 20 (data, active mode) |
| **DHCP** | Auto-assign IP addresses | UDP | 67 (server) / 68 (client) |
| **NTP** | Time sync | UDP | 123 |
| **PostgreSQL / MySQL / Redis** | Databases | TCP | 5432 / 3306 / 6379 |

(Note: SFTP is *not* FTP over TLS; it runs inside SSH on port 22.) Port numbers ≤ 1023 are "well-known"; the IANA registry lists all of them. You can see the local list: `less /etc/services`.

An example of what "one layer does three OSI jobs" means: sending "Hello" in a chat app means building a JSON message (formatting), maybe compressing it and encrypting it with TLS, and managing the logged-in connection. All of that is **application-layer** code, before the transport layer sees a single byte.

### 3.2 Transport layer: TCP and UDP
Delivers data between **processes** using **ports** (Chapter 21). Two main protocols with opposite trade-offs, explained in detail in section 4.

### 3.3 Internet layer: IP
- **IP** (v4: `192.168.1.23`, 32-bit; v6: `2001:db8::1`, 128-bit) gives every host an address and **routes** packets hop-by-hop across networks. It is **best effort**: packets may be lost, duplicated, reordered or delayed. IP promises nothing, and reliability is the job of layers above (TCP) or the application.
- **ICMP** carries errors and diagnostics (`ping` = echo request/reply; "destination unreachable"; "TTL exceeded", which `traceroute` uses).
- **TTL** (time to live): a hop counter decremented by each router; it stops packets from looping forever.
- **Fragmentation:** if a packet is bigger than a link's **MTU** (usually 1500 bytes), IPv4 routers may split it. Modern TCP avoids this with **path MTU discovery**.
- **NAT:** because IPv4 addresses ran short, most homes and companies use **private** addresses (`10.x.x.x`, `172.16–31.x.x`, `192.168.x.x`) behind one public address, and a NAT device rewrites addresses/ports (Chapters 30 and later).

### 3.4 Link layer
Moves frames across **one network**: Ethernet, Wi-Fi, etc.; MAC addresses, switches, ARP (IP → MAC lookup on the local network), VLANs, and the physical medium.

---

## 4. TCP vs UDP

Both take a chunk of data from a program, put source and destination **ports** on it, and hand it to IP. What they add differs greatly.

| | **TCP** (Transmission Control Protocol) | **UDP** (User Datagram Protocol) |
|---|---|---|
| Model | **Connection-oriented byte stream**: connect first, then send a continuous stream | **Connectionless messages**: each datagram stands alone |
| Reliability | **Yes**: numbering, acknowledgements, retransmission of lost data | **No**: lost datagrams stay lost |
| Ordering | **Yes**: data is delivered in the order sent | **No**: may arrive out of order |
| Duplicates | Removed | Possible |
| Message boundaries | **None** (a stream: two `send()`s may arrive as one `recv()`) | **Preserved** (one `sendto()` = one `recvfrom()`) |
| Flow control | Yes: the receiver tells the sender how much it can take (window) | No |
| Congestion control | Yes: slows down when the network is overloaded | No (the application must behave) |
| Header size | 20 bytes (min) | **8 bytes** |
| Setup delay | Handshake (1 round-trip) before data | None |
| Broadcast / multicast | No | **Yes** |
| Typical uses | Web, APIs, email, SSH, file transfer, databases | DNS lookups, video/voice calls, games, streaming, DHCP, NTP, VPNs, QUIC |

> **Corrections to two popular slogans.** "TCP guarantees delivery" really means: *either the data arrives complete and in order, or the sender is told the connection failed*. It can't force data through a broken network. And "UDP is faster" means *lower latency and less overhead*, not that data moves through cables faster: UDP simply skips the reliability work, so an application that needs reliability on top of UDP has to implement it (as **QUIC** does).

### 4.1 TCP in a nutshell
(Chapter 23 covers it in depth.)

1. **Handshake:** `SYN` → `SYN-ACK` → `ACK` (three-way) establishes a connection and agrees on starting sequence numbers.
2. **Data transfer:** bytes are numbered; the receiver **acknowledges**; unacknowledged data is **retransmitted**; a **window** limits how much can be in flight (flow control), and a **congestion window** adapts to the network.
3. **Teardown:** `FIN`/`ACK` in each direction (or `RST` to abort).

Imagine certified mail with tracking and re-sends, or a phone call (dial, both say hello, talk, both say goodbye).

**TCP header** (20+ bytes): source port, destination port, sequence number, acknowledgement number, header length, **flags** (SYN, ACK, FIN, RST, PSH, URG...), window size, checksum, urgent pointer, options.

### 4.2 UDP in a nutshell
A datagram = an **8-byte header** (source port, destination port, length, checksum) + payload. No connection, no memory of previous datagrams: fire and forget, like shouting a message or dropping postcards in a mailbox.

**UDP header:** `| src port (16) | dst port (16) | length (16) | checksum (16) |` and that's it.

### 4.3 Choosing
| Situation | Choice | Why |
|---|---|---|
| Bank transfer, web page, API call, file download, email, database | **TCP** | Every byte must arrive, correct and in order |
| Live video/voice call, online game position updates | **UDP** | A late packet is *useless*; better to skip it than to wait for a retransmission (head-of-line blocking) |
| DNS lookup | **UDP** | One tiny request and one tiny reply; a handshake would double the cost. Falls back to TCP for large responses |
| Sensor readings sent every second | **UDP** (or MQTT over TCP if you can't lose messages) | Newest value matters |
| Custom low-latency protocol with its own reliability (QUIC/HTTP/3, some games) | **UDP** as a base | Control retransmission and multiplexing yourself |

Applications often use **both**: a video call sends media over UDP, but sign-in, chat and file sharing over TCP.

---

## 5. Sockets: how programs actually use TCP/UDP

A **socket** is the operating system's API for network communication: an endpoint identified by (protocol, local IP, local port), plus, for connected TCP sockets, the remote IP and port. The kernel implements TCP/UDP/IP; your program just calls `socket()`, `bind()`, `listen()`, `accept()`, `connect()`, `send()`/`recv()` (system calls, Chapter 2).

A connection is uniquely identified by the **5-tuple**: `(protocol, src IP, src port, dst IP, dst port)`. That's how one server port (443) serves thousands of clients at once: each client has a different source IP/port.

### Try it: a TCP echo server and client (Python 3)

Terminal 1, the server:

```python
# server.py
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)   # SOCK_STREAM = TCP
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(("0.0.0.0", 5000))       # listen on all interfaces, port 5000
s.listen()
print("listening on 5000")
while True:
    conn, addr = s.accept()      # blocks until the TCP handshake completes
    print("connection from", addr)   # e.g. ('127.0.0.1', 51724)  ← client's ephemeral port
    with conn:
        data = conn.recv(1024)
        conn.sendall(b"echo: " + data)
```

Terminal 2, the client:

```python
# client.py
import socket
c = socket.create_connection(("127.0.0.1", 5000))
c.sendall(b"Hello")
print(c.recv(1024))              # b'echo: Hello'
c.close()
```

While the server runs: `ss -tlnp | grep 5000` shows it listening; `ss -tn | grep 5000` (while a client is connected) shows the 5-tuple. Or use netcat instead of Python: `nc -l 5000` in one terminal and `nc localhost 5000` in another; type, and text flows both ways.

### The UDP version

```python
# udp_server.py
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)    # SOCK_DGRAM = UDP
s.bind(("0.0.0.0", 5001))
while True:
    data, addr = s.recvfrom(1024)     # no accept(), no connection: just datagrams
    s.sendto(b"echo: " + data, addr)
```

```bash
nc -u -w1 127.0.0.1 5001      # type a line: the reply comes back; but nothing "connected"
```

Notice: UDP has no `listen/accept`; if you send to a port where nobody listens, you may get an ICMP "port unreachable", but often nothing at all.

**Message boundaries:** TCP is a stream. Sending `"AB"` then `"CD"` may be received as `"ABCD"` in a single `recv`. That's why application protocols add framing (a length prefix, a newline, HTTP's `Content-Length`). With UDP each datagram arrives whole (or not at all).

---

## 6. A complete journey through TCP/IP

`curl http://example.com/` (simplified):

| Step | Layer | What happens |
|---|---|---|
| 1 | Application | curl builds `GET / HTTP/1.1`, `Host: example.com` |
| 2 | Application (DNS) | Resolver asks for `example.com` → gets `93.184.x.x` (usually a UDP query to port 53) |
| 3 | Transport | TCP: the OS picks an **ephemeral source port** (say 51724) and performs the **handshake** to port 80 |
| 4 | Transport | HTTP request bytes become **TCP segments** (source port 51724, dest port 80, sequence numbers) |
| 5 | Internet | Each segment goes into an **IP packet**: source = your IP, destination = `93.184.x.x` |
| 6 | Link | The packet is put in an **Ethernet frame** addressed to your **gateway's MAC** (found through **ARP**), because the destination isn't on the local network |
| 7 | Physical | Bits leave over Wi-Fi/Ethernet |
| 8 | Routers | Each router removes the old frame, looks at the **IP destination**, decrements the TTL, and puts the packet in a **new frame** for the next link |
| 9 | Server | Frame → packet → segment; port 80 → the web server process; reply flows back the same way |

On the wire, a segment is laid out like this (headers **in front** of payload):

```
| Ethernet: dst MAC, src MAC, type | IPv4: src IP, dst IP, TTL, proto=6(TCP) | TCP: src port, dst port, seq, ack, flags | HTTP data | FCS |
```

`proto=6` says "the payload is TCP"; `17` means UDP; `1` means ICMP. The IP header carries this **protocol number**, and the TCP/UDP header carries **port numbers** to say which upper protocol/program gets the data. That's how each layer knows what to hand the data to.

---

## 7. TCP/IP and Docker

| TCP/IP layer | In a Docker/Kubernetes world |
|---|---|
| **Application** | The app inside the container (HTTP server, DB), DNS-based service discovery, reverse proxies and Ingress |
| **Transport** | Ports in the container, **`-p 8080:80`** publish mapping, Kubernetes Services (connection-level load-balancing) |
| **Internet** | Container IPs (`172.17.0.0/16` on the default bridge), routing between networks, NAT (masquerade) for outbound traffic, NetworkPolicy |
| **Link** | Virtual **veth** pairs, the **`docker0`** Linux bridge (a virtual switch), VXLAN overlays between hosts |

`docker run -d -p 8080:80 nginx` in TCP/IP terms:

- nginx (application) listens on **TCP port 80** *inside its own network namespace*.
- The container has its own IP (e.g. `172.17.0.2`) on the `docker0` bridge.
- Docker adds a **NAT (DNAT) rule** so that TCP connections to *host port 8080* are rewritten to `172.17.0.2:80`. Return traffic is un-NATed.
- Outbound container traffic is **masqueraded** (source NAT) to the host's IP.

To publish a UDP port: `-p 5353:53/udp`.

---

## 8. Troubleshooting by layer (with Docker)

Scenario: *"My app container can't reach the database container."*

```bash
# Link / Internet: are they on the same network, and do they have IPs?
docker network ls
docker network inspect mynet | less
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' db

# Internet: can the app container reach the db's IP?  (may need: apt-get install -y iputils-ping)
docker exec app ping -c 2 db          # name resolution + ICMP (only works on user-defined networks by NAME)
docker exec app getent hosts db       # DNS: does the name resolve?

# Transport: is the port open? (no telnet needed)
docker exec app sh -c 'nc -vz db 5432'      # or:  bash -c '</dev/tcp/db/5432 && echo open'
docker exec db ss -tlnp                     # is the DB really listening, and on which address (0.0.0.0, not 127.0.0.1)?

# Application: credentials, config, logs
docker logs db
docker exec app env | grep -i database
```

Error text ↔ layer:

| Message | Meaning | Layer |
|---|---|---|
| `Name or service not known` / `Temporary failure in name resolution` | DNS failed | App (DNS) |
| `Network is unreachable` / `No route to host` | No route or ARP failure | Internet / Link |
| `Connection timed out` | Packets silently dropped: firewall, wrong IP, or host down | Internet/Transport |
| **`Connection refused`** (`ECONNREFUSED`) | Host reachable, but **nothing listening** on that port (TCP RST). Check the port number, that the server is running, and that it binds to `0.0.0.0`, not `127.0.0.1` | **Transport** |
| `Connection reset by peer` | The other side aborted an existing connection | Transport/App |
| `SSL/TLS handshake failure`, `certificate verify failed` | TLS problem | App (TLS) |
| HTTP `502/503/504` | A proxy/gateway can't reach the upstream | App |

Two common Docker mistakes behind `ECONNREFUSED`: connecting to **`localhost`** from inside a container (that's *the container itself*, not the host or another container; use the service/container name on a shared network, or `host.docker.internal` on Docker Desktop), and a server listening only on `127.0.0.1` inside its container.

> `EXPOSE` in a Dockerfile, and container-to-container traffic on the same user-defined network, do **not** need `-p`. Publishing is only for traffic coming from **outside** the Docker network.

---

## 9. Hands-on: watch TCP and UDP

You need `tcpdump` (or Wireshark), `nc` (netcat), and root. (In a container: `apt-get update && apt-get install -y tcpdump netcat-openbsd iproute2 iputils-ping curl`, and run with `--cap-add NET_RAW --cap-add NET_ADMIN` if needed.)

**1. See the TCP handshake and teardown**

```bash
# terminal 1
sudo tcpdump -i lo -nn 'tcp port 5000'
# terminal 2
nc -l 5000 &      # server (option syntax differs between netcat variants: `nc -l -p 5000` on some)
echo hi | nc 127.0.0.1 5000
```

You will see lines with flags: `[S]` (SYN), `[S.]` (SYN-ACK), `[.]` (ACK), `[P.]` (data with PUSH), `[F.]` (FIN), matching section 4.1.

**2. See UDP**: `sudo tcpdump -i lo -nn udp port 5001` while you run the UDP server and `echo hello | nc -u -w1 127.0.0.1 5001`: no handshake, no flags, just datagrams.

**3. See DNS over UDP**: `sudo tcpdump -i any -nn udp port 53` then `dig example.com` or `getent hosts example.org`.

**4. Sockets and states**

```bash
ss -tan | head           # TCP states: LISTEN, ESTAB, TIME-WAIT, SYN-SENT ...
ss -uan                  # UDP sockets
ss -s                    # summary counts
ss -tlnp                 # which process listens on which TCP port
```

**5. Compare cost:** `time (for i in $(seq 100); do getent hosts example.com >/dev/null; done)` (mostly UDP DNS), versus the same number of `curl -s http://example.com -o /dev/null` calls (TCP handshake + HTTP each time). Compare, and think about connection reuse (keep-alive).

**6. Wireshark:** capture on your interface, filter `tcp.port == 443` or `udp`, click a packet, and expand **Frame → Ethernet → IP → TCP** to see every header field from this chapter.

---

## 10. Common misconceptions

| Misconception | Reality |
|---|---|
| "The internet uses OSI" | It uses TCP/IP; OSI is the vocabulary |
| "TCP/IP is one protocol" | It's a *suite* (IP, TCP, UDP, ICMP, ARP, DNS, HTTP...), named after two of them |
| "TCP guarantees delivery no matter what" | It guarantees ordered, complete delivery *or* a reported failure |
| "UDP is unreliable so it is bad" | It is *minimal*. Great for real-time and for building custom protocols (QUIC) |
| "UDP is always faster" | Lower latency for small exchanges; bulk transfer over TCP is usually just as fast and much safer |
| "IP is reliable" | IP is best effort. Loss, duplication and reordering are normal |
| "HTTP is always TCP" | HTTP/1.1 and 2 use TCP; **HTTP/3 uses QUIC over UDP** |
| "A port belongs to one application forever" | It's a per-host, per-protocol address space; clients use random ephemeral ports |
| "TCP/IP has no session or presentation layer, so it does not support them" | The functions exist; they're done by applications and libraries (TLS, cookies, JSON) |
| "`localhost` in a container is my computer" | It's the container's own loopback |

---

## 11. Summary

- TCP/IP was **built and deployed** (ARPANET, 1970s–1983) while OSI was designed by committee; TCP/IP won, and OSI supplied the layer vocabulary.
- **Four layers:** Application (OSI 5–7), Transport (4), Internet (3), Link (1–2); or five with Physical separated.
- **IP** = addresses + routing, best effort. **TCP** = reliable ordered byte stream with handshake, flow and congestion control. **UDP** = tiny, connectionless datagrams with no guarantees.
- A **socket** (protocol + IP + port) is how programs use the stack; a connection is identified by the **5-tuple**.
- Headers are added going down (segment → packet → frame) and removed going up; routers rewrite the link layer at each hop but keep the IP addresses.
- Debug by layer: DNS → route/ping → port (`nc`, `ss`) → TLS → app. `refused` = reached but nothing listening; `timeout` = no answer.

---

## 12. Check your understanding

1. Map the four TCP/IP layers to the OSI layers and give one protocol for each.
2. Give two reasons a video call uses UDP for media but TCP for sign-in.
3. What identifies a TCP connection uniquely?
4. Why can a web server on port 443 serve thousands of clients at once?
5. Your app in container A gets `ECONNREFUSED` connecting to `localhost:5432`, but the database runs in container B. What's wrong?
6. Why does DNS normally use UDP?
7. What does the `proto` field of the IP header tell the receiver? What tells it which *program* gets the data?

<details>
<summary>Answers</summary>

1. Application (OSI 5–7): HTTP; Transport (4): TCP; Internet (3): IP; Link (1–2): Ethernet.
2. Real-time media: a late packet is useless, so retransmitting just adds delay; loss of a frame is acceptable. Sign-in/chat/files must arrive complete and correct, so TCP.
3. The 5-tuple: protocol, source IP, source port, destination IP, destination port.
4. Each client connection has a different (source IP, source port), so each is a distinct 5-tuple, though the server-side port is the same.
5. `localhost` inside container A is A itself, and nothing listens there. Connect to container B's name (on a shared user-defined network) or its IP instead.
6. Queries and answers are tiny; UDP avoids the handshake round-trip. If the answer is too large it falls back to TCP.
7. `proto` tells the OS which transport protocol handles the payload (6 = TCP, 17 = UDP, 1 = ICMP); the destination **port** in the TCP/UDP header selects the program.
</details>

**Practice**

1. Run the TCP and UDP echo programs; capture each with `tcpdump` and count the packets for one exchange in each case.
2. Use `ss -tan` while a browser is open; identify LISTEN, ESTAB and TIME-WAIT sockets, and find the ephemeral port of one connection.
3. Start `nginx` in Docker with `-p 8080:80`; from the host `curl` it; use `sudo iptables -t nat -L -n | grep 8080` (or `nft list ruleset`) to find the DNAT rule.
4. In two containers on a user-defined network (`docker network create lab`), start `nc -l 5000` in one and connect to it by container name from the other.

---

**Next:** [Chapter 23 – TCP in Detail](23_tcp_in_details.md)
