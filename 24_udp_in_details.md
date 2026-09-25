# Chapter 24: UDP in Detail

> **In one sentence:** UDP is the simplest transport protocol: it adds only **ports, a length and a checksum** to your data and hands it to IP, with **no connection, no acknowledgments, no retransmission and no ordering**, which makes it fast, lightweight and ideal for real-time or "one question, one answer" traffic.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~40 minutes

**Prerequisites:** [Chapter 22](22_tcp_ip_model.md) (TCP vs UDP overview, sockets) and [Chapter 23](23_tcp_in_details.md) (TCP, so you can compare).

---

## What you will learn

- The **8-byte UDP header**, field by field (and why each field exists)
- What UDP *doesn't* do, and what that means for applications
- **Connectionless** and **stateless**: what the OS remembers (and doesn't)
- The **checksum** and the pseudo-header, and when it may be zero
- **Datagram sizes**, MTU and fragmentation (why DNS worries about 1232 bytes)
- Where UDP is used: DNS, DHCP, NTP, VoIP, games, **QUIC/HTTP/3**, WireGuard, mDNS and more
- How reliability is built *on top of* UDP
- **Security** issues: spoofing, reflection and amplification attacks
- UDP with **Docker** and **NAT**
- Hands-on: Python sockets, `nc`, `tcpdump`, `ss`, `iperf3`, `dig`

---

## 1. The philosophy: do as little as possible

TCP builds a reliable, ordered stream (connection setup, numbering, ACKs, retransmission, windows). That is wonderful for downloads, but costs latency and state, and it forces "in order, always" semantics even when the application doesn't want them.

UDP takes the opposite approach: **give applications direct access to IP's best-effort service, plus port numbers.** Anything more, they build themselves if they need it.

Use these two mental pictures:

| TCP | UDP |
|---|---|
| A phone call (dial, both confirm, talk, hang up) | Sending postcards (write, address, drop in the mailbox) |
| Certified mail with signatures | Shouting across a room |

A single UDP unit is called a **datagram**. Each datagram is **independent**: it may arrive, be lost, be duplicated, or arrive after later ones.

---

## 2. The UDP header: 8 bytes

```
 0      7 8     15 16    23 24    31
┌────────┴────────┬────────┴────────┐
│  Source Port    │ Destination Port│   16 bits each
├─────────────────┼─────────────────┤
│     Length      │    Checksum     │   16 bits each
├─────────────────┴─────────────────┤
│              Data (payload)       │
└───────────────────────────────────┘
```

| Field | Size | Purpose |
|---|---|---|
| **Source port** | 16 bits | Sender's port. Where replies should go. **Optional** for one-way senders (may be 0), but nearly always set to an ephemeral port |
| **Destination port** | 16 bits | The receiving program's port (53 DNS, 67/68 DHCP, 123 NTP, 443 for QUIC, 51820 WireGuard) |
| **Length** | 16 bits | Header **plus** data, in bytes. Minimum 8 (empty datagram). Payload size = Length − 8 |
| **Checksum** | 16 bits | Detects corruption (section 6) |

That's all. **Compare TCP's 20–60 byte header** (sequence, acknowledgment, flags, window, urgent pointer, options).

### What the missing fields mean

| TCP field | Why UDP has no equivalent |
|---|---|
| Sequence number | No ordering, no reassembly, no duplicate detection |
| Acknowledgment number | No delivery confirmation |
| Flags (SYN/FIN/RST...) | No connection to open, close or reset |
| Window size | No flow control |
| Data offset / options | Fixed-size header; no negotiation or extensions |

### Size limits
- **Length** is 16 bits → up to 65,535 bytes total, so at most **65,527** bytes of UDP payload in theory.
- Over IPv4, the whole IP packet is also limited to 65,535 bytes, so the practical maximum payload is 65,535 − 20 (IP) − 8 (UDP) = **65,507 bytes**.
- The **path MTU** (typically **1500** on Ethernet) is the real limit. To avoid fragmentation, keep the payload ≤ 1500 − 20 − 8 = **1472 bytes** (IPv4) or **1452 bytes** (IPv6, 40-byte header). Many applications stay near **~1200 bytes** to be safe across tunnels, VPNs and PPPoE (QUIC requires a minimum datagram size of 1200 bytes).

### Overhead comparison (headers only)

| Payload | TCP (20-byte header) | UDP (8-byte header) |
|---|---|---|
| 20 bytes | 50% overhead | 29% |
| 100 bytes | 17% | 7% |
| 1000 bytes | 2% | 0.8% |

(This ignores the IP and Ethernet headers common to both.) The bigger difference is not header bytes, but **round trips and state**, discussed next.

---

## 3. What UDP does *not* do

### 3.1 No connection setup
TCP needs a handshake (one round trip) before any data, and more (TLS) on top. With UDP you send data **immediately**:

```
TCP:  SYN ─►  ◄─ SYN+ACK  ACK+data ─►  ◄─ response      (data reaches server after 1 round trip; answer after 2)
UDP:  request ─►  ◄─ response                            (answer after 1 round trip)
```

A DNS lookup is tiny: one small question, one small answer. With TCP the exchange costs about twice the round-trip time and roughly 7–10 packets (handshake, query, response, close); with UDP it costs **1 round trip and 2 packets**. That is why DNS uses UDP by default.

### 3.2 No acknowledgments
The sender never learns whether a datagram arrived. Benefits: no ACK traffic, no waiting, latency stays low. If the application cares, it must ask for its own confirmation (for example a DNS resolver simply **retries** after a timeout).

### 3.3 No retransmission
A lost datagram is gone. For real-time media this is right: a voice packet that arrives 300 ms late is useless. The application can choose to conceal the loss (audio interpolation), tolerate it (a video frame glitch), or add its own recovery (selective retransmission, forward error correction).

### 3.4 No ordering or duplicate suppression
Datagrams may be delivered out of order or twice. Applications that care attach their own sequence numbers (RTP, QUIC).

### 3.5 No flow control and no congestion control
A UDP sender can transmit at whatever rate it likes, which can overflow the receiver's socket buffer (datagrams are then silently dropped by the OS) or congest the network. Well-behaved UDP protocols (QUIC, WebRTC) implement **their own congestion control**. Badly-behaved ones can harm other traffic (which is one reason some networks rate-limit UDP).

### 3.6 No termination
There is no "close". A sender simply stops. If the receiver needs to know "the sender is finished", the application defines it (an end-of-stream message, or a timeout).

### 3.7 Message boundaries are preserved
Unlike TCP's byte stream, **one `sendto()` produces exactly one datagram, and one `recvfrom()` returns exactly one datagram**. If the buffer you give `recvfrom()` is smaller than the datagram, the excess is **truncated and lost**, so use a large enough buffer (e.g. 65535).

---

## 4. "Connectionless" and "stateless"

TCP keeps a table entry for every connection: state, sequence numbers, windows, timers, buffers (several KB each). UDP keeps **no per-conversation state** in the protocol.

A UDP server needs just **one socket** bound to a port. It reads datagrams from *anyone* (each with its sender address), replies to that address, and forgets. Consequences:

- **Scalability:** one socket can serve enormous numbers of clients; there is no `accept()` loop and no per-client TCP state (this is one reason DNS servers cope with huge query rates).
- **Simplicity:** no lifecycle, no `TIME_WAIT`.
- **Flexibility:** an endpoint can send to many destinations (or many senders to one) with a single socket, use **broadcast** and **multicast**, and change patterns instantly.
- **Weakness:** no built-in verification that the sender is who they claim to be (see section 9).

Notes:

- Operating systems and **firewalls/NAT** *do* keep pseudo-state for UDP ("flows", **conntrack** entries) so replies can be matched to the earlier request, expiring after a timeout (typically 30–180 s, often 30 s for unreplied flows). That's why UDP applications behind NAT send **keepalives** every 15–25 s.
- **`connect()` on a UDP socket** doesn't handshake; it merely records a default destination and filters incoming datagrams to that peer. It also lets the socket receive **ICMP errors**: sending to a closed port often provokes an ICMP *port unreachable*, which a *connected* UDP socket reports as `ECONNREFUSED` on a later call.

---

## 5. What happens when nobody is listening?

Send a datagram to a port where no program is bound:

1. The receiving host's OS finds no socket for that port.
2. It usually replies with **ICMP "Destination unreachable / Port unreachable"** (unless a firewall drops or rate-limits it).
3. The sender's application **usually sees nothing** (a plain `sendto()` succeeded long ago). Only connected sockets or some APIs surface the ICMP error.

Contrast with TCP, where a closed port immediately produces an `RST` and "connection refused". This is why UDP port scans are slow and ambiguous: silence can mean *open*, *filtered*, or *lost*.

---

## 6. The checksum

- A 16-bit **Internet checksum** (the same one's-complement algorithm as TCP, Chapter 23) computed over: a **pseudo-header** (source IP, destination IP, zero byte, protocol = **17**, UDP length) + the UDP header (checksum field set to 0) + the data, padded with a zero byte if the length is odd.
- The pseudo-header makes the checksum cover **which hosts** the datagram is for, so a datagram delivered to the wrong host because of a corrupted IP header is rejected.
- **IPv4:** the checksum is **optional**. A sender may put `0x0000` to mean "not computed". (If the computed value happens to be `0x0000`, it is sent as `0xFFFF`.)
- **IPv6:** the checksum is **mandatory** (the IPv6 header has no checksum of its own). Only specific tunnel protocols may use zero checksums under strict rules.
- **On failure** the receiver **silently discards** the datagram. No error, no retransmission.
- It is a **weak, non-cryptographic** check: it catches most random corruption but not all, and it does nothing against deliberate tampering. Use **DTLS**, QUIC's built-in encryption, or an application-level MAC for integrity and authenticity.

A worked example with tiny numbers: words `0x0001` and `0xF203` → sum `0xF204` → checksum `~0xF204 = 0x0DFB`. The receiver adds everything including the checksum: `0xF204 + 0x0DFB = 0xFFFF` ✓.

---

## 7. Fragmentation, MTU and DNS

If a UDP datagram plus IP header exceeds the link **MTU** (usually 1500), IPv4 may **fragment** it into several IP packets, which the receiver must reassemble. Problems:

- **If any fragment is lost, the whole datagram is lost.** UDP won't retransmit.
- Fragments are dropped by many firewalls and middleboxes, and are a vector for attacks.
- IPv6 routers don't fragment at all; only the sender may, after discovering the path MTU.

Therefore applications keep datagrams below the MTU. **DNS** is the classic case: it originally limited UDP replies to **512 bytes**; with **EDNS(0)** clients advertise a larger buffer, and since 2020 the recommendation ("DNS Flag Day") is to use about **1232 bytes** to avoid fragmentation. If a response doesn't fit, the server sets the **TC (truncated)** bit and the client **retries over TCP**. (You can force TCP: `dig +tcp example.com`.) Large answers (DNSSEC, many records) are one reason DNS over TCP is increasingly common; DNS over TLS/HTTPS also runs over TCP (or QUIC).

---

## 8. Where UDP is used

| Use | Why UDP | Notes |
|---|---|---|
| **DNS** (port 53) | Tiny query/answer; no handshake; huge fan-in of clients | Falls back to TCP for large answers and zone transfers |
| **DHCP** (67/68) | Client has **no IP address yet**, so it must broadcast (`0.0.0.0` → `255.255.255.255`); TCP can't broadcast | DORA: Discover, Offer, Request, Acknowledge (Chapters 35–37) |
| **NTP** (123) | Precise timestamps; TCP retransmissions would ruin timing | |
| **mDNS / SSDP / service discovery** | Multicast to "whoever is out there" | Bonjour/Avahi, UPnP |
| **VoIP / video calls** (RTP/SRTP, WebRTC) | Low latency > completeness; a late packet is useless | WebRTC also needs NAT traversal (STUN/ICE) and DTLS/SRTP encryption |
| **Online gaming** | Position/state updates every few ms; latest data wins | Many games add selective reliability for important events |
| **QUIC / HTTP/3** (443/UDP) | Build a better transport: multiple independent streams (no head-of-line blocking), faster handshakes (0-RTT/1-RTT including TLS 1.3), connection migration across networks | Standardized in RFC 9000 (2021); HTTP/3 in RFC 9114 (2022); widely deployed by major CDNs and browsers |
| **VPNs** (WireGuard, OpenVPN-UDP, IPsec NAT-T) | Tunnelling TCP inside TCP makes "TCP meltdown" (two layers of retransmission fight each other) | |
| **IoT / telemetry** (CoAP, syslog, SNMP, StatsD) | Lightweight, loss-tolerant, many small messages | |
| **Broadcast / multicast** (IPTV, market data feeds, LAN discovery) | One-to-many delivery; TCP is strictly point-to-point | |
| **Live/interactive streaming** | Low latency (WebRTC, SRT, RIST) | |

> **A common misconception:** *video streaming* services like Netflix or YouTube (on-demand) mostly use **HTTP over TCP** (or QUIC), not raw UDP: they download chunks of video and buffer ahead, so reliability matters more than latency. Raw UDP is used for **live, interactive** media (calls, cloud gaming, low-latency live).

### Reliability *on top of* UDP
When applications need reliability without TCP's constraints, they add it themselves:

- **Sequence numbers** and **acknowledgments** for the messages that matter (game events, chat).
- **Timeout and retry** (DNS resolvers, DHCP, TFTP lock-step).
- **Forward error correction** (send redundant data so the receiver can rebuild losses: voice/video).
- **QUIC** implements essentially all of TCP's guarantees (streams, retransmission, congestion control, flow control), plus encryption, in user space over UDP, and is deployed without needing changes in routers or operating-system kernels (which are hard to update). Because middleboxes already pass UDP, **UDP became the "programmable substrate" for new transports.**

---

## 9. Security considerations

| Threat | Why UDP is exposed | Mitigation |
|---|---|---|
| **IP spoofing** | No handshake, so nothing proves the source address is real | Application-level tokens/cookies, cryptography, **BCP 38** ingress filtering at ISPs |
| **Reflection / amplification DDoS** | Attacker sends a small request with the **victim's forged source IP** to a public UDP service; the service sends a much larger answer to the victim. Historic amplifiers: open **DNS resolvers**, **NTP** `monlist`, **memcached**, **SSDP**, **CLDAP**. Amplification factors range from ~10× to tens of thousands× | Don't run open resolvers or reflectors; rate-limit responses (RRL); require validation (QUIC's address-validation token; DNS cookies); disable dangerous features |
| **Floods** | Cheap to send, no state to slow the attacker | Rate limiting, DDoS scrubbing, anycast |
| **Eavesdropping / tampering** | Plain UDP is unencrypted with a weak checksum | **DTLS**, QUIC, WireGuard, SRTP |
| **Port-scan ambiguity** | Silence ≠ closed | Stateful firewall with default-deny |
| **Socket buffer overflow** | Fast senders overrun slow receivers | Bigger `SO_RCVBUF`, receiver-side rate control, monitor `netstat -su` |

Firewall tip: statefully allow UDP replies only to requests you sent (conntrack `ESTABLISHED,RELATED`), and only expose the UDP ports you truly need.

---

## 10. UDP with Docker and NAT

- **Publishing UDP ports** must be explicit: `docker run -p 5353:53/udp ...`. `-p 8080:80` publishes **TCP only**. Publish both with `-p 53:53/tcp -p 53:53/udp`.
- **Container DNS:** on user-defined networks, Docker runs an embedded DNS resolver at **`127.0.0.11`** inside each container; your container's `getaddrinfo()` sends **UDP port 53** queries to it, and it forwards to the host's resolvers. (If DNS "randomly" fails inside containers, check MTU/firewall for UDP/53 and the `/etc/resolv.conf` inside the container.)
- **NAT and UDP:** since there's no connection, NAT gear uses timeouts. Long-idle UDP flows (VPNs, game sessions) need keepalives, otherwise return traffic is dropped after the mapping expires.
- **Kubernetes:** Services support `protocol: UDP`; some cloud load balancers have limited UDP support/health checks. Check your provider's docs.
- **Docker Desktop:** UDP port publishing works but goes through extra proxying; measure before relying on it for high packet rates.

---

## 11. Code: UDP in Python

**Server**

```python
# udp_server.py
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)     # DGRAM = UDP
sock.bind(("0.0.0.0", 10000))
print("UDP server listening on :10000")

while True:
    data, client = sock.recvfrom(65535)          # one call = one datagram (with sender address)
    print(f"{len(data)} bytes from {client}: {data!r}")
    sock.sendto(b"ACK: " + data, client)         # reply to whoever sent it
```

**Client (with a timeout and a retry: reliability *you* add)**

```python
# udp_client.py
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.settimeout(1.0)
msg = b"Hello, UDP!"

for attempt in range(1, 4):
    sock.sendto(msg, ("127.0.0.1", 10000))            # no connect(), no handshake
    try:
        data, server = sock.recvfrom(65535)
        print("reply:", data, "from", server)
        break
    except socket.timeout:
        print(f"attempt {attempt}: no reply, retrying")
else:
    print("gave up: the datagram or its reply was lost, or nobody is listening")
```

Differences from the TCP version (Chapter 22): `SOCK_DGRAM`, **no** `listen()`/`accept()`/`connect()`, `sendto`/`recvfrom` carry the address every time, and each call is one datagram. Run the client with the server stopped: you'll see the timeouts (a plain UDP socket doesn't learn about the closed port).

**Try `connect()` on a UDP socket** (add `sock.connect(("127.0.0.1", 10000))`, then `sock.send(msg)`, `sock.recv(...)`): with the server **stopped**, the second call raises `ConnectionRefusedError`, because the ICMP port-unreachable is now delivered to your connected socket.

**Message boundaries demo:** send `b"AAA"` and `b"BBB"` back to back; `recvfrom` returns them as **two separate** datagrams (compare with TCP, which may merge them).

---

## 12. Hands-on lab

**1. Netcat**

```bash
# terminal 1
nc -u -l 10000            # (some versions need: nc -u -l -p 10000)
# terminal 2
echo "hello" | nc -u -w1 127.0.0.1 10000
```

**2. See the packets: header fields and length**

```bash
sudo tcpdump -i lo -nn -X udp port 10000
```

Read `UDP, length 6` (payload bytes). In the `-X` hex dump, find the 8 header bytes: source port, destination port `2710` (= 10000 in hex), length `000e` (= 14 = 8 + 6), then the checksum. Verify Length = 8 + payload.

**3. Nobody listening: ICMP port unreachable**

```bash
sudo tcpdump -i lo -nn 'udp port 10001 or icmp' &
echo test | nc -u -w1 127.0.0.1 10001      # no server there
```

You will see the UDP datagram, followed by `ICMP 127.0.0.1 udp port 10001 unreachable`. (`nc` may not show an error, but a connected Python socket would.)

**4. Sockets and counters**

```bash
ss -uan                    # UDP sockets: State is UNCONN (unconnected) or ESTAB (connected UDP)
ss -uanp | grep 10000
netstat -su                # UDP statistics: "packet receive errors", "receive buffer errors", "packets to unknown port received"
```

**5. DNS over UDP and TCP**

```bash
sudo tcpdump -i any -nn port 53 &
dig example.com                 # one UDP query + one UDP response
dig +tcp example.com            # full TCP handshake, query, response, close: compare the packet counts
dig +bufsize=512 +ignore example.com ANY    # experiment with truncation (TC) responses (may vary by resolver)
```

**6. Throughput and loss: `iperf3`**

```bash
iperf3 -s &
iperf3 -c 127.0.0.1 -u -b 100M -l 1200        # UDP at 100 Mbit/s, 1200-byte datagrams; the report shows jitter and loss %
```

Try `-b 5G` on a slow link and watch **loss** appear: UDP does not slow down for you.

**7. Simulate a bad network (Linux, root)**

```bash
sudo tc qdisc add dev lo root netem loss 20% delay 20ms 5ms reorder 25% 
python3 udp_client.py            # see retries; datagrams are lost or reordered
sudo tc qdisc del dev lo root    # always clean up
```

**8. Docker**

```bash
docker run -d --name udpsrv -p 10000:10000/udp alpine:3.20 sh -c 'apk add --no-cache netcat-openbsd >/dev/null && nc -u -l -p 10000'
echo hi | nc -u -w1 127.0.0.1 10000
docker logs udpsrv
docker rm -f udpsrv
# now publish WITHOUT /udp (-p 10000:10000) and notice it does not work: only TCP was published
```

---

## 13. TCP vs UDP: full comparison

| | TCP | UDP |
|---|---|---|
| Unit | Byte **stream** | **Datagram** (message) |
| Connection | Handshake, state | None |
| Reliability | Retransmission until acknowledged | None |
| Ordering | Guaranteed | None |
| Duplicates | Removed | Possible |
| Flow / congestion control | Yes | No (app's job) |
| Header | 20–60 bytes | 8 bytes |
| First data after | ≥ 1 round trip | Immediately |
| Message boundaries | Not preserved | Preserved |
| Broadcast / multicast | No | Yes |
| Per-peer state on server | Yes | None (one socket serves all) |
| NAT/firewall friendliness | Well understood | Needs timeouts/keepalives |
| Typical uses | HTTP/1–2, SSH, SMTP, databases, downloads | DNS, DHCP, NTP, VoIP, games, QUIC, VPNs, discovery |

---

## 14. Common misconceptions

| Misconception | Reality |
|---|---|
| "UDP is unreliable, therefore bad" | It is **minimal**: it lets the application choose its own reliability. QUIC is reliable *and* runs over UDP |
| "UDP is always faster" | Lower latency and setup cost, but bulk throughput over a lossy path can be *worse* if the app has no good congestion control. Speed doesn't come from faster bits, but from skipping work |
| "UDP packets are unordered because they take different paths" | They *may* arrive out of order (multi-path, queues); UDP makes no attempt to fix it |
| "Video streaming = UDP" | On-demand streaming mostly uses HTTP over TCP/QUIC; UDP dominates *live/interactive* media |
| "UDP has no checksum" | It has one (optional in IPv4, mandatory in IPv6) |
| "A UDP send error means the packet was lost" | `sendto()` succeeding says nothing about arrival |
| "UDP can't be secured" | DTLS, QUIC, WireGuard and SRTP secure it |
| "DNS only uses UDP" | It falls back to TCP (truncation, large answers, zone transfers, DoT/DoH) |
| "`docker run -p 8080:80` publishes UDP too" | Only TCP, unless `/udp` is specified |
| "UDP needs no ports opened in firewalls" | Firewalls still filter UDP by port; NAT needs mappings |

---

## 15. Summary

- UDP = **source port, destination port, length, checksum** (8 bytes) + data. Nothing else.
- **No** connection, ACKs, retransmission, ordering, duplicate suppression, flow or congestion control, or termination. **Yes** to message boundaries, broadcast/multicast, low latency and tiny per-peer cost.
- The checksum covers a pseudo-header, is optional over IPv4 and mandatory over IPv6; failures are silently dropped.
- Keep datagrams under the path MTU (~1200–1472 bytes); fragmentation multiplies loss risk. DNS truncates and falls back to TCP.
- Great for DNS, DHCP, NTP, calls, games, discovery, VPNs and **QUIC/HTTP/3**; applications add reliability where needed.
- Beware spoofing, reflection/amplification and floods; secure with DTLS/QUIC/WireGuard and validate sources.
- Docker: publish UDP with `-p host:container/udp`; NAT and firewalls treat UDP with timeouts.

---

## 16. Check your understanding

1. List the four fields of the UDP header and their sizes. What is the smallest possible UDP datagram?
2. A UDP header's Length field says 53. How many payload bytes are there?
3. Why must DHCP use UDP?
4. What happens if you send a UDP datagram to a port where nothing is listening? How does that differ from TCP?
5. Why is a reflection attack easier with UDP than with TCP?
6. Which is better for a live voice call, and why: TCP or UDP?
7. A colleague's containerized DNS server works over TCP but not UDP when published with `-p 53:53`. Why?
8. Your client sends 3000-byte UDP datagrams and loses many. Why, and what would you change?
9. How can an application get reliability over UDP?

<details>
<summary>Answers</summary>

1. Source port, destination port, length, checksum: 16 bits each (8 bytes). The smallest datagram is 8 bytes (empty payload, length = 8).
2. 53 − 8 = 45 bytes.
3. The client doesn't have an IP address yet and must broadcast; TCP can't broadcast or work without an established address and connection.
4. The OS usually returns an ICMP port unreachable, but a plain sender doesn't notice. TCP would immediately answer a SYN with RST ("connection refused").
5. No handshake validates the source address, so a forged source IP gets a (larger) reply sent to the victim. TCP's handshake requires the attacker to see the SYN+ACK.
6. UDP. A retransmitted voice packet arrives too late to be useful; stalling for it (head-of-line blocking) makes audio worse than a small glitch.
7. `-p 53:53` publishes TCP only. Publish UDP with `-p 53:53/udp`.
8. 3000 bytes exceeds the MTU, so IP fragments each datagram; losing any fragment loses the whole datagram. Send datagrams ≤ ~1200–1472 bytes (split the data yourself).
9. Add sequence numbers, acknowledgments, timeouts/retries, and possibly forward error correction (or use a protocol like QUIC that does it).
</details>

**Practice**

1. Run the Python server/client, capture with `tcpdump -X`, and decode the four header fields by hand from the hex dump. Confirm the checksum field is non-zero.
2. Write a UDP "chat" where two terminals send messages to each other with one socket each. Then stop one side and observe.
3. Use `dig` with and without `+tcp` and count packets in each case with `tcpdump`; compute the latency difference using `dig`'s `Query time`.
4. Use `iperf3 -u` with increasing bandwidth (`-b 10M`, `100M`, `1G`) and report the loss and jitter; explain what changed.
5. Publish a UDP echo server in Docker and prove that the `/udp` suffix matters.

---

**Next:** [Chapter 25 – HTTP/1.0 in Detail](25_http_1_0_in_details.md)
