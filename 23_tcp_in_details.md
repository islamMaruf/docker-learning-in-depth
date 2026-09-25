# Chapter 23: TCP in Detail

> **In one sentence:** TCP turns the unreliable, packet-by-packet network into a **reliable, ordered byte stream** by numbering every byte, acknowledging what arrives, retransmitting what doesn't, and slowing down when the receiver or the network can't keep up.

**Level:** 🟡 Intermediate → 🔴 Expert (built up from basics) · **Reading time:** ~60 minutes

**Prerequisites:** [Chapter 21](21_philosophy_of_osi_model.md) and [Chapter 22](22_tcp_ip_model.md) (layers, ports, sockets, TCP vs UDP).

---

## What you will learn

- The **TCP header**, field by field, and what a segment looks like on the wire
- **Ports** in more detail: well-known, registered and **ephemeral**; how the OS uses them
- The **three-way handshake** (with real sequence numbers) and why it exists
- **Sequence** and **acknowledgment** numbers: how bytes are tracked; cumulative ACKs
- How TCP handles **loss**: timeouts, duplicate ACKs, fast retransmit, SACK
- **Flow control** (receive window) vs **congestion control** (slow start, congestion avoidance)
- **Connection termination**: the four-step close, half-close, `RST`, and `TIME_WAIT`
- The **checksum**, worked through with real numbers
- **TCP states**, and how to read them with `ss`
- Real-world behavior: Nagle, delayed ACK, keep-alive, SYN floods, backlog, and Docker implications
- Hands-on with `tcpdump`, `ss`, Python and `nc` so you can *see* all of it

---

## 1. What problem does TCP solve?

IP (Chapter 22) is *best effort*: packets can be **lost, duplicated, corrupted, delayed or reordered**. Yet applications such as web browsers want to hand over a stream of bytes and get **exactly those bytes, in order** at the other end, without worrying about the network.

TCP provides, on top of IP:

| Service | Mechanism |
|---|---|
| **Connection** (both ends know about each other) | Three-way handshake, connection state |
| **Reliability** (no missing data) | Sequence numbers, acknowledgments, retransmission |
| **Ordering, no duplicates** | Sequence numbers, receiver reassembly buffer |
| **Error detection** | Checksum |
| **Flow control** (don't overwhelm the receiver) | Receive window |
| **Congestion control** (don't overwhelm the network) | Congestion window, slow start, backoff |
| **Multiplexing** (many programs on one host) | Ports |

What TCP does **not** provide: message boundaries (it's a byte stream), security or encryption (that's TLS), or guaranteed timing.

---

## 2. The TCP segment and header

A **segment** = a **TCP header** followed by **payload** bytes. It travels inside an IP packet (IP protocol number **6**).

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
┌───────────────────────────────┬───────────────────────────────┐
│        Source Port (16)       │     Destination Port (16)     │
├───────────────────────────────┴───────────────────────────────┤
│                     Sequence Number (32)                      │
├───────────────────────────────────────────────────────────────┤
│                  Acknowledgment Number (32)                   │
├───────┬───────┬───────────────────────┬───────────────────────┤
│ Data  │ Rsvd  │ Flags (9 bits):       │                       │
│Offset │ (3)   │ NS CWR ECE URG ACK    │   Window Size (16)    │
│ (4)   │       │ PSH RST SYN FIN       │                       │
├───────────────────────────────┬───────────────────────────────┤
│         Checksum (16)         │      Urgent Pointer (16)      │
├───────────────────────────────┴───────────────────────────────┤
│                  Options (0 – 40 bytes)                       │
├───────────────────────────────────────────────────────────────┤
│                         Payload (data)                        │
└───────────────────────────────────────────────────────────────┘
```

| Field | Size | Purpose |
|---|---|---|
| **Source port** | 16 bits | Sender's port (clients: ephemeral) |
| **Destination port** | 16 bits | Receiver's port (servers: well-known/configured) |
| **Sequence number** | 32 bits | Position in the sender's byte stream of the **first payload byte** of this segment (or the ISN on a `SYN`) |
| **Acknowledgment number** | 32 bits | **The next byte the sender of this segment expects to receive.** Valid only if `ACK` is set |
| **Data offset** | 4 bits | Header length in **32-bit words**. Header bytes = offset × 4. Minimum 5 (20 bytes), maximum 15 (60 bytes) |
| **Reserved** | 3 bits | Must be zero (reserved for future use) |
| **Flags** | 9 bits | `NS`, `CWR`, `ECE`, `URG`, `ACK`, `PSH`, `RST`, `SYN`, `FIN` (see section 5) |
| **Window size** | 16 bits | How many more bytes the sender of this segment can currently **receive** (flow control). Scaled by an option for large windows |
| **Checksum** | 16 bits | Error detection over header + payload + IP "pseudo-header" |
| **Urgent pointer** | 16 bits | Only with `URG`; effectively unused today |
| **Options** | 0–40 bytes | **MSS**, **window scale**, **SACK**, **timestamps**, ... |

Sizes: header **20–60 bytes**. With IPv4 (20 bytes) on Ethernet (MTU 1500), the largest payload per segment is **MSS = 1500 − 20 − 20 = 1460 bytes**.

---

## 3. Ports revisited

A **port** is a 16-bit number (0–65535) that selects a *program* on a host. Together with the IP address it forms a **socket address** (`203.0.113.5:443`).

| Range | Name | Notes |
|---|---|---|
| **0–1023** | **Well-known** (system) ports | Assigned by IANA to standard services: 22 SSH, 25 SMTP, 53 DNS, 80 HTTP, 443 HTTPS. On Linux, binding needs root (or `CAP_NET_BIND_SERVICE`, or `net.ipv4.ip_unprivileged_port_start`) |
| **1024–49151** | **Registered** ports | Assigned to applications on request: 3306 MySQL, 5432 PostgreSQL, 6379 Redis, 8080 alt-HTTP |
| **49152–65535** | **Dynamic / private / ephemeral** | The IANA-suggested range for temporary client ports. **Linux uses 32768–60999 by default** (check: `cat /proc/sys/net/ipv4/ip_local_port_range`); Windows and BSDs use 49152+ |

### Ephemeral ports: the client side
A browser doesn't run on a fixed port. When it connects, the OS **chooses an unused ephemeral port** as the *source* port. The reply comes back to that port, and the OS hands it to the right process.

```
Browser tab 1: 192.168.1.23:51724 ──►  93.184.216.34:443
Browser tab 2: 192.168.1.23:51725 ──►  93.184.216.34:443     ← same destination, different source port
```

A TCP connection is identified by the **4-tuple/5-tuple** (source IP, source port, destination IP, destination port [+ protocol]), so the number of simultaneous connections is *not* limited to the number of ephemeral ports: the limit applies **per destination** (per `dst IP:port`). One client can hold ~28,000 (Linux default) connections **to the same server port** but many more in total to different servers. Busy proxies and load balancers hit "ephemeral port exhaustion" when they open too many connections to a *single* upstream.

Each server connection is another socket, but they all share the same **listening** port (443), which is why one server can serve thousands of clients.

---

## 4. The three-way handshake

Before data flows, both sides must agree on **initial sequence numbers (ISNs)** and prove they can reach each other. The handshake takes **one round trip**.

```
   Client                                                Server
     │                                                     │  (server is in LISTEN state)
     │ ──── SYN            Seq = x                    ───► │  1. "I'd like to connect; my numbering starts at x"
     │                                                     │
     │ ◄─── SYN+ACK        Seq = y, Ack = x+1         ──── │  2. "OK; mine starts at y; I expect x+1 next"
     │                                                     │
     │ ──── ACK            Seq = x+1, Ack = y+1       ───► │  3. "Got it; I expect y+1 next"
     │                                                     │
     │           ESTABLISHED               ESTABLISHED     │
```

With concrete (illustrative) numbers, client ISN `x = 1000`, server ISN `y = 5000`:

| # | Direction | Flags | Seq | Ack | Meaning |
|---|---|---|---|---|---|
| 1 | C → S | `SYN` | 1000 | 0 (unused) | "I want to connect. My first byte will be numbered 1001." |
| 2 | S → C | `SYN, ACK` | 5000 | 1001 | "I got your SYN (1000 + 1). Mine starts at 5000; my first byte will be 5001." |
| 3 | C → S | `ACK` | 1001 | 5001 | "I got your SYN (5000 + 1)." |

Key points:

- **A `SYN` consumes one sequence number** (as does a `FIN`). That's why the ACK is `x+1`, even though no data was sent.
- **ISNs are (pseudo)random**, not zero: an unpredictable ISN makes it much harder for an off-path attacker to forge segments, and stops stray old packets from a previous connection with the same ports from being accepted. (Real ISNs are chosen from the full 32-bit range, 0 to 4,294,967,295.)
- Options are negotiated in the `SYN`s: **MSS** (largest segment each side accepts), **window scale**, **SACK permitted**, **timestamps**. Each side uses the *smaller* MSS.
- The handshake also fills the listening server's **SYN queue / accept queue** (below).

### Watch it
```bash
sudo tcpdump -i any -nn 'tcp port 5000' &
nc -l 5000 &        # (some netcat versions need: nc -l -p 5000)
echo hi | nc 127.0.0.1 5000
```
You will see (exact ports/numbers differ): `[S]` (SYN), `[S.]` (SYN+ACK), `[.]` (ACK), `[P.]` (data + push + ack), `[F.]` and more. In tcpdump, `.` means ACK. Add `-S` to show absolute sequence numbers instead of relative ones.

---

## 5. Flags: the control bits

| Flag | Name | Meaning |
|---|---|---|
| **SYN** | Synchronize | Start a connection; carries the ISN |
| **ACK** | Acknowledge | The acknowledgment number is valid (set on almost every segment after the first SYN) |
| **FIN** | Finish | Sender has no more data; begins closing its direction |
| **RST** | Reset | Abort the connection immediately (or reject an unwanted one) |
| **PSH** | Push | Deliver buffered data to the application now (don't wait to fill a buffer) |
| **URG** | Urgent | The urgent pointer field is valid (obsolete in practice) |
| **ECE**, **CWR** | ECN Echo, Congestion Window Reduced | Explicit Congestion Notification signalling |
| **NS** | Nonce Sum | Experimental, rarely used |

Common combinations: `SYN` (connect), `SYN+ACK` (accept), `ACK` (plain acknowledgment or data), `PSH+ACK` (data to deliver now), `FIN+ACK` (close), `RST` / `RST+ACK` (reject or abort).

**When you see an RST:** connecting to a port where nothing listens ("connection refused"), a firewall or proxy actively rejecting, an application closing a socket with unread data, an abrupt `kill` of a process, or a middlebox timing out an idle flow ("connection reset by peer").

---

## 6. Sequence and acknowledgment numbers

TCP numbers **every byte** of the stream, not every segment.

- A segment's **sequence number** = the number of its **first payload byte**.
- The receiver's **acknowledgment number** = "**the next byte I expect**". Equivalently, "I have received **everything before this number**."
- Acknowledgments are **cumulative**: an ACK of 1012 confirms all bytes up to 1011, no matter how many segments they came in.

### Worked example: sending "Hello World" (11 bytes)
After the handshake above, the client's next byte is numbered 1001. Suppose (for teaching) that it sends three segments:

```
Client                                                          Server
  │ ── Seq=1001  "Hello"  (5 bytes: 1001–1005) ──────────────►  │
  │ ◄─ Ack=1006 ─────────────────────────────────────────────── │   "I have everything before 1006; send 1006 next"
  │ ── Seq=1006  " "      (1 byte : 1006)     ───────────────►  │
  │ ◄─ Ack=1007 ─────────────────────────────────────────────── │
  │ ── Seq=1007  "World"  (5 bytes: 1007–1011) ──────────────►  │
  │ ◄─ Ack=1012 ─────────────────────────────────────────────── │
```

Rule: **ACK = highest contiguous byte received + 1**. In real life TCP would send "Hello World" in **one** segment, and would acknowledge **every second segment** (delayed ACK), so this trace is for understanding only.

### Loss and retransmission
Suppose the second segment (" ") is lost, but the third arrives:

```
Client                                                          Server
  │ ── Seq=1001 "Hello"  ───────────────────────────────────►  │  ✓ Ack=1006
  │ ── Seq=1006 " "      ───────────X  (lost)                   │
  │ ── Seq=1007 "World"  ───────────────────────────────────►  │  out of order: buffered, but a gap at 1006
  │ ◄─ Ack=1006  (duplicate ACK: "still waiting for 1006")  ─── │
```

The sender learns about the loss in two ways:

1. **Timeout:** no ACK arrives within the **retransmission timeout (RTO)**, computed from measured round-trip times and doubled after each failure (**exponential backoff**). Retransmit from the missing byte.
2. **Fast retransmit:** **three duplicate ACKs** for the same number signal that later segments arrived but one was lost, so the sender retransmits immediately without waiting for the timer.

With **SACK** (Selective Acknowledgment, option negotiated at the handshake) the receiver also reports blocks it *did* receive beyond the gap ("I have 1007–1011"), so the sender retransmits **only** the missing bytes rather than everything after the gap.

The receiver holds out-of-order data in a buffer and delivers to the application **only in order**. That in-order guarantee is why one lost packet can stall a whole TCP stream (**head-of-line blocking**), which is one reason QUIC exists.

---

## 7. Flow control: the receive window

The receiver has limited buffer space. Every segment it sends carries a **window size**: the number of bytes it is currently willing to accept beyond the acknowledged point. The sender must never have more than that **in flight (sent but unacknowledged)**.

```
   acknowledged      in flight (≤ window)        not yet sendable
──────────────────┬──────────────────────────┬───────────────────────►
                  ▲                          ▲
             Ack number             Ack number + window
                  └───── the window slides right as ACKs arrive ─────┘
```

- **Sliding window:** instead of "send one segment, wait for its ACK" (stop-and-wait), the sender streams many segments and the window slides forward as ACKs come back, which keeps the pipe full.
- **Window updates:** a slow application that isn't reading drains its buffer slowly → the advertised window **shrinks**. At **window = 0** the sender pauses and probes periodically (**zero-window probes**) until space frees up.
- **Window scaling:** the 16-bit field limits a window to 65,535 bytes, which is far too small for fast, long-distance links. The **window scale option** (in the `SYN`) multiplies it (up to about 1 GB).
- **Throughput limit** ≈ window ÷ round-trip time. With a 64 KB window and a 100 ms RTT, the maximum is 64 KB / 0.1 s ≈ 5 Mbit/s no matter how fast the link is. That's the **bandwidth-delay product** problem, solved by window scaling.

## 8. Congestion control: protecting the network

Flow control protects the *receiver*. **Congestion control** protects the *network*: if everyone sends as fast as the receivers allow, links overflow and everyone suffers. The sender keeps a private **congestion window (cwnd)**, and may send at most `min(cwnd, receiver window)` bytes in flight.

| Phase | Behavior |
|---|---|
| **Slow start** | Start small (typically 10 segments) and **double cwnd every round trip** until reaching a threshold (`ssthresh`) or seeing loss. ("Slow" only relative to sending everything at once.) |
| **Congestion avoidance** | Past the threshold, grow **linearly** (about +1 segment per round trip) |
| **On loss** | Treat as a congestion signal: **cut cwnd** (classically halve it with fast recovery; a timeout resets it to a small value) |
| **Algorithms** | Loss-based **CUBIC** (Linux default), delay/bandwidth-model based **BBR**, older **Reno** |

The effect: TCP flows converge to a *fair share* of a bottleneck link, and a fresh connection begins slowly, which is another reason **reusing connections** (HTTP keep-alive, connection pools) matters.

---

## 9. The checksum

TCP protects header and data with a 16-bit **Internet checksum**:

1. Build a **pseudo-header** (source IP, destination IP, protocol, TCP length) + the TCP header (checksum field = 0) + payload. Pad with a zero byte if the total length is odd.
2. Treat everything as 16-bit words and add them with **one's-complement addition** (any carry out of bit 16 is wrapped around and added back).
3. Take the **one's complement** (flip all bits) of the sum. That's the checksum.

The receiver adds all words *including* the checksum; if nothing changed, the result is **`0xFFFF`** (all ones).

### A small worked example
Three 16-bit words: `0x1234`, `0xABCD`, `0x0F0F`.

```
   0x1234
 + 0xABCD
 --------
   0xBE01
 + 0x0F0F
 --------
   0xCD10           ← sum (no carry beyond 16 bits)
 checksum = ~0xCD10 = 0x32EF

Receiver:  0x1234 + 0xABCD + 0x0F0F + 0x32EF = 0xCD10 + 0x32EF = 0xFFFF   ✓
```

Wrap-around example: `0xFFFF + 0x0002 = 0x10001` → carry the top bit back: `0x0001 + 1 = 0x0002`.

If a bit flips in transit, the receiver's sum is no longer `0xFFFF`, so it **silently discards** the segment, and the missing ACK causes the sender to retransmit.

> The checksum is weak by modern standards: it can miss some multi-bit errors (for example two errors that cancel out, or swapped 16-bit words). Ethernet's stronger CRC (the FCS, layer 2) and application/TLS integrity checks (MACs) add more protection. So "TCP guarantees the data is perfect" really means "an excellent, but not mathematically absolute, level of protection".

---

## 10. Closing a connection

TCP connections are **full-duplex**: two independent byte streams. Each direction is closed separately with a `FIN`, hence the classic **four-step close**:

```
   Client                                              Server
     │ ── FIN, Seq=u            ─────────────────────► │  1. "I have no more data to send"
     │ ◄─ ACK, Ack=u+1          ────────────────────── │  2. "OK" (server may still send data: the connection is half-closed)
     │ ◄─ FIN, Seq=v            ────────────────────── │  3. "I'm done too"
     │ ── ACK, Ack=v+1          ─────────────────────► │  4. "OK"
     │        TIME_WAIT (≈ 60 s on Linux)          CLOSED
```

Notes:

- Steps 2 and 3 can be **merged** into one `FIN+ACK` if the server has nothing more to send (a three-segment close).
- The side that closes **first** ends in **`TIME_WAIT`** for 2×MSL (Linux: 60 s). This ensures the last ACK is delivered (if it is lost, the peer resends its `FIN`) and that stray delayed packets of the old connection can't be mistaken for a new one using the same 4-tuple. Many `TIME_WAIT` sockets on a busy *client/proxy* are normal but can contribute to port exhaustion. Prefer connection reuse over tuning.
- **Half-close:** `shutdown(SHUT_WR)` sends a `FIN` but keeps reading, which is used by protocols that signal "end of request" and then wait for the response.
- **`RST` = abrupt close:** no graceful shutdown, and unsent data may be discarded.

---

## 11. Connection states

A socket moves through these states (`ss -tan` and `netstat -an` show them):

```
        (server)  LISTEN ──SYN in──► SYN-RECEIVED ──ACK──► ESTABLISHED ◄── ACK ── SYN-SENT ◄─ connect() (client)

  active closer:   ESTABLISHED ─FIN→ FIN-WAIT-1 ─ACK→ FIN-WAIT-2 ─FIN in→ TIME-WAIT ─timeout→ CLOSED
  passive closer:  ESTABLISHED ─FIN in→ CLOSE-WAIT ─close()→ LAST-ACK ─ACK→ CLOSED
```

| State | Meaning | If you see many of them... |
|---|---|---|
| **LISTEN** | Server waiting for connections | Normal |
| **SYN-SENT** | Client sent `SYN`, waiting for `SYN+ACK` | Target unreachable/filtered, or firewall dropping |
| **SYN-RECV** | Server got `SYN`, sent `SYN+ACK`, waiting for the final ACK | **SYN flood** attack or lossy path |
| **ESTABLISHED** | Data flows | Normal (your active connections) |
| **FIN-WAIT-1/2** | You sent `FIN`; waiting | Peer not closing its side |
| **CLOSE-WAIT** | Peer sent `FIN`; **your application hasn't closed the socket** | **Application bug**: a connection leak (missing `close()`) |
| **LAST-ACK** | You sent your `FIN`; waiting for the final ACK | Rarely a problem |
| **TIME-WAIT** | Waiting 2×MSL after closing | Normal on busy clients; see above |

**CLOSE-WAIT is the one that points at *your* code**; TIME-WAIT is usually harmless.

---

## 12. Options, and other real-world behaviors

### Options
| Option | Purpose |
|---|---|
| **MSS** | Largest payload each side accepts (typically 1460 on Ethernet/IPv4) |
| **Window scale** | Allows windows larger than 64 KB |
| **SACK permitted / SACK** | Selective acknowledgments |
| **Timestamps** | Better RTT estimates; protects against wrapped sequence numbers |

### Behaviors that surprise people
- **Nagle's algorithm** batches small writes: while there's unacknowledged data in flight, tiny writes are coalesced. Together with the receiver's **delayed ACK** (~40 ms) this can add latency to request/response protocols with many small writes. Latency-sensitive apps set **`TCP_NODELAY`** on the socket.
- **Keep-alive:** by default, an idle TCP connection can stay open **forever** without any traffic and neither side notices if the other vanished (a power cut, a NAT/firewall dropping state). Enable **TCP keep-alive probes** (`SO_KEEPALIVE`, `tcp_keepalive_time` typically 2 h on Linux) or application-level heartbeats. Load balancers and NAT gateways often drop idle flows after 60 s to 15 min, which is why long-lived idle connections mysteriously "reset".
- **Backlog / accept queue:** a listening socket has a queue of completed handshakes waiting for `accept()`. If the app is slow to `accept()`, the queue fills and new connections get delayed or dropped (`ss -ltn` shows `Recv-Q`/`Send-Q` for listeners).
- **SYN flood** attack: send many `SYN`s and never complete the handshake to exhaust the server's half-open state. **SYN cookies** (kernel default on Linux) encode the state in the `SYN+ACK`'s sequence number so nothing needs to be stored.
- **Head-of-line blocking:** one lost segment delays everything behind it in the same connection (HTTP/2 over TCP suffers from this; HTTP/3 over QUIC avoids it).
- **Handshake cost:** each new connection costs at least one RTT (plus TLS: 1–2 more). **Reuse** connections: HTTP keep-alive, database connection pools, HTTP/2 multiplexing.
- **Server restarts:** after a crash and restart, binding to the same port may fail with `Address already in use` because of `TIME_WAIT`; servers set **`SO_REUSEADDR`**.

---

## 13. TCP in a Docker/Kubernetes world

- Containers on the same **user-defined bridge network** connect over ordinary TCP, resolving each other by name (Docker's built-in DNS). No `-p` is needed for container-to-container traffic.
- `-p 8080:80` publishes a **TCP** port through **NAT (DNAT)**; conntrack tracks each connection. A **UDP** port needs `-p 8080:80/udp`.
- **`localhost` inside a container** is the container itself.
- A server bound to `127.0.0.1` inside a container is unreachable from outside its network namespace; bind **`0.0.0.0`**.
- **Graceful shutdown**: on `docker stop` (SIGTERM), a well-behaved server stops accepting, finishes in-flight requests, and closes connections cleanly (with `FIN`) instead of dying with `RST`s. Chapter 16 covers PID 1 and signals.
- **Connection tracking table** limits (`nf_conntrack_max`) and **ephemeral port ranges** matter for busy proxies and NAT gateways.
- Idle-timeout mismatches between a client's connection pool and a **load balancer's** idle timeout cause "connection reset" errors on reused connections; keep client timeouts *shorter* than the server/LB's.
- Kubernetes Services (kube-proxy) balance **connections**, not requests: a long-lived HTTP/2 or gRPC connection sticks to one pod (use L7 load balancing for per-request balancing).

---

## 14. Hands-on labs

**Lab 1: Read a real handshake and teardown**

```bash
sudo tcpdump -i lo -nn -S 'tcp port 5000'      # -S: absolute sequence numbers
# in another terminal:
python3 -m http.server 5000 &                  # a TCP server
curl -s http://127.0.0.1:5000/ > /dev/null
```

Identify: the three handshake segments (`Flags [S]`, `[S.]`, `[.]`), the `seq`/`ack` values (verify `ack = seq + 1` for the SYNs), the `mss` option, the data (`[P.]` with `length N`), and the `[F.]` segments that close the connection.

**Lab 2: See states**

```bash
python3 -m http.server 5000 &
ss -tlnp | grep 5000                 # LISTEN
curl -s localhost:5000 >/dev/null; ss -tan | grep 5000    # TIME-WAIT on the closing side
# leave a connection half-open:
nc localhost 5000 &               # ESTABLISHED (nothing sent yet)
ss -tan | grep 5000
```

**Lab 3: Connection refused vs timeout**

```bash
nc -vz 127.0.0.1 59999           # nobody listening → immediate "Connection refused" (RST)
sudo iptables -A INPUT -p tcp --dport 6000 -j DROP      # simulate a firewall silently dropping
nc -vz -w 3 127.0.0.1 6000       # times out (SYN-SENT, no reply)
sudo iptables -D INPUT -p tcp --dport 6000 -j DROP      # clean up
```

**Lab 4: Look at a live connection's internals**

```bash
curl -s https://example.com >/dev/null &     # or any longer download
ss -ti dst example.com                        # -i shows cwnd, rtt, retrans, mss, wscale, sack, cubic...
```

**Lab 5: Watch the window and slow start.** Download a large file (`curl -o /dev/null https://speed.hetzner.de/100MB.bin`) while running `watch -n0.5 'ss -ti | grep -A1 <ip>'` and watch `cwnd` grow.

**Lab 6: Simulate loss and delay (Linux, root)**

```bash
sudo tc qdisc add dev lo root netem loss 5% delay 50ms
curl -o /dev/null -s -w '%{time_total}s\n' http://127.0.0.1:5000/bigfile
sudo tc qdisc del dev lo root                # remove it!
```

With tcpdump you'll see retransmissions and duplicate ACKs (`tcpdump` marks them; Wireshark labels them "TCP Retransmission" / "TCP Dup ACK").

**Lab 7: Sockets from Python** (Chapter 22's echo server), then try `nc localhost 5000` twice at the same time to see two established connections with different ephemeral ports.

**Lab 8: Docker**

```bash
docker network create tcp-net
docker run -d --name server --network tcp-net nginx:1.27-alpine
docker run --rm -it --network tcp-net alpine:3.20 sh -c '
  apk add --no-cache netcat-openbsd >/dev/null
  printf "GET / HTTP/1.0\r\nHost: server\r\n\r\n" | nc server 80 | head -5'
docker rm -f server
```

`nc server 80` resolves the container name via Docker DNS, then opens a TCP connection (handshake, request, response, close), all invisible until you capture it.

---

## 15. Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| `Connection refused` | Nothing listening (RST) | `ss -tlnp` on the server; port/host correct? bound to `0.0.0.0`? |
| `Connection timed out` (or hangs in SYN-SENT) | SYNs dropped: firewall/security group, wrong IP, routing, host down | `tcpdump` shows only SYNs going out; `traceroute`; firewall rules |
| `Connection reset by peer` | Peer aborted (RST): app crash, LB/idle timeout, protocol error | Server logs; LB idle timeout vs client keep-alive |
| Long delays before the *first byte* | DNS delay, TLS handshake, Nagle + delayed ACK | `curl -w` timing breakdown (see below) |
| Slow transfers on high-latency links | Small window / no window scaling, packet loss | `ss -ti` (cwnd, rtt, retrans), test with `iperf3` |
| Many `CLOSE-WAIT` | App isn't closing sockets | Fix code (close in `finally`), check connection pool leaks |
| Many `TIME-WAIT`, "cannot assign requested address" | Ephemeral port exhaustion when hammering one destination | Reuse connections; widen `ip_local_port_range`; more source IPs |
| Many `SYN-RECV` | SYN flood or overloaded accept queue | SYN cookies, increase backlog, rate limit |
| Occasional resets on reused idle connections | Idle timeout mismatch | Lower client idle timeout, enable keep-alive |

`curl` timing breakdown:

```bash
curl -o /dev/null -s -w 'dns %{time_namelookup}  tcp %{time_connect}  tls %{time_appconnect}  first-byte %{time_starttransfer}  total %{time_total}\n' https://example.com
```

---

## 16. Common misconceptions

| Misconception | Reality |
|---|---|
| "TCP is slow" | It has overhead and setup cost, but tuned TCP saturates fast links. Slowness usually comes from latency, loss, small windows, or no connection reuse |
| "ACK means I got *this* byte" | ACK = "I have **everything before** this number; send this one next" |
| "Every segment is acknowledged individually" | Cumulative ACKs (and delayed ACKs) acknowledge many segments at once |
| "Sequence numbers count segments" | They count **bytes** |
| "Sequence numbers start at 0/1" | Random ISN. (Tools display *relative* numbers unless you use `-S`) |
| "Ports limit me to 16,384 connections" | Connections are identified by the 4-tuple, so limits are per destination |
| "TCP preserves message boundaries" | It's a byte stream. Two `send()`s may arrive as one `recv()`, and vice versa |
| "TCP checksum makes data perfect" | It catches most, not all, errors; TLS/apps add integrity |
| "Closing a connection always takes four packets" | Often three (combined FIN+ACK); an `RST` takes one |
| "50% of internet traffic is TCP" (or any specific figure) | Shares vary by measure and year, and today much traffic runs on QUIC over UDP. Don't quote a number without a source |
| "TCP sequence numbers protect against command injection" | They only help against blind spoofing; use TLS/SSH for security |

---

## 17. Summary

- **TCP header:** ports, 32-bit **sequence** and **acknowledgment** numbers, data offset (header size in words), flags, **window**, checksum, options. 20–60 bytes.
- **Handshake:** `SYN` → `SYN+ACK` → `ACK`; agree on random **ISNs**, MSS, window scale, SACK. A `SYN`/`FIN` consumes one sequence number.
- **Reliability:** every byte numbered; cumulative **ACK = next byte expected**; loss detected by **timeout** or **3 duplicate ACKs** (fast retransmit); **SACK** avoids resending what arrived.
- **Flow control** = receiver **window** (protects the receiver); **congestion control** = **cwnd** with slow start, congestion avoidance and backoff (protects the network).
- **Checksum:** 16-bit one's-complement sum over pseudo-header + segment; the receiver expects `0xFFFF`.
- **Close:** `FIN`/`ACK` each way (half-close possible), `TIME_WAIT` for the active closer; `RST` aborts.
- **States** (`ss -tan`) diagnose problems: **CLOSE-WAIT** = your app leaks sockets; **SYN-RECV** floods; **TIME-WAIT** usually harmless.
- **Practical rules:** reuse connections, bind to `0.0.0.0` in containers, keep idle timeouts consistent, shut down gracefully.

---

## 18. Check your understanding

1. What is the minimum TCP header size, and what does a Data Offset of 8 mean?
2. In the handshake the client's SYN has Seq=7000. What are the Ack value of the SYN+ACK and the Seq/Ack of the third segment (if the server's ISN is 9000)?
3. A receiver acknowledges "Ack = 4001". Which bytes has it received?
4. What are the two ways a sender detects a lost segment?
5. What is the difference between flow control and congestion control? Which window belongs to each?
6. Why is the side that initiates the close left in `TIME_WAIT`?
7. You see thousands of connections in `CLOSE-WAIT`. Where is the bug likely to be?
8. Compute the one's-complement checksum of the words `0x00FF` and `0x0F00`.
9. Why does a fresh TCP connection to a far-away server transfer slowly at first?
10. `nc -vz host 80` returns instantly with "refused" from one machine, and hangs on another machine to the same host. What does each result tell you?

<details>
<summary>Answers</summary>

1. 20 bytes. Data Offset 8 means an 8 × 4 = 32-byte header (12 bytes of options).
2. SYN+ACK: Ack = 7001, Seq = 9000. Third segment: Seq = 7001, Ack = 9001.
3. All bytes numbered below 4001 (bytes up to and including 4000). It expects 4001 next.
4. A retransmission timeout (no ACK in time), or three duplicate ACKs (fast retransmit).
5. Flow control protects the receiver's buffers (receiver's advertised window). Congestion control protects the network (sender's congestion window, cwnd). The sender uses the minimum of the two.
6. To make sure the final ACK reached the peer (so it can retransmit its FIN if needed) and to let old duplicate packets die out before the same 4-tuple is reused.
7. In the application: it received the peer's FIN but never closed its socket (a connection leak, missing `close()`).
8. Sum = 0x00FF + 0x0F00 = 0x0FFF; checksum = ~0x0FFF = 0xF000. (Check: 0x0FFF + 0xF000 = 0xFFFF.)
9. Slow start begins with a small congestion window and doubles each round trip; on a long RTT it takes many round trips to fill the pipe. Reusing connections avoids this.
10. Instant "refused": an RST came back, so you reached a host with nothing listening on port 80. A hang/timeout: SYNs are being silently dropped (firewall, routing, host down); it is not reaching a listening process.
</details>

**Practice**

1. Write down the handshake, a 3-segment data transfer and the four-way close for ISNs of your choice (client 100, server 300; the client sends 20 bytes then 30 bytes) with Seq/Ack values. Then capture a real one with `tcpdump -S` and compare.
2. Use `ss -tan state established` and `ss -tan state time-wait | wc -l` on your machine while browsing; explain each count.
3. Reproduce a `CLOSE-WAIT` pile-up: write a tiny Python server that `accept()`s but never `close()`s; connect and disconnect clients with `nc`; observe with `ss`.
4. Use `tc netem` to add 100 ms latency and 2% loss, download something, and describe what `ss -ti` and Wireshark show (retransmits, `cwnd`).

---

**Next:** [Chapter 24 – UDP in Detail](24_udp_in_details.md)
