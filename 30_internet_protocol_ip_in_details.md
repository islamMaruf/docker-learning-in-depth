# Chapter 30: The Internet Protocol (IP) in Detail

> **In one sentence:** IP gives every network interface an **address** and delivers **packets** hop by hop from the sender to the destination across many networks, using each router's **routing table**, with no promises about delivery (it is *best effort*); TCP and UDP build reliability and ports on top.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~65 minutes

**Prerequisites:** [Chapter 21](21_philosophy_of_osi_model.md) (layers, encapsulation), [Chapter 22](22_tcp_ip_model.md), and binary/hex basics (explained as needed).

---

## What you will learn

- What IP does at Layer 3 and what it explicitly does **not** do
- The **IPv4 header**, every field, decoded from real bytes, including the checksum
- **IPv4 addresses**: binary and dotted decimal, classes (historical), private, loopback, link-local, CGNAT, multicast
- **IPv6**: why it exists, address format and types, the simpler header, SLAAC, and how it differs from IPv4
- How a packet travels: **routing tables, default gateway, longest-prefix match, next hops** and how the L2 header changes at each hop
- **TTL / hop limit** and how `traceroute` uses it
- **MTU, fragmentation and Path MTU Discovery** (and their real-world failure modes)
- **ICMP**: ping, unreachable, time exceeded
- **NAT** (and why it is not a firewall), **routing protocols** (static, RIP, OSPF, **BGP**)
- IP in **Docker** and **Kubernetes**: bridge subnets, container IPs, address overlap problems
- Hands-on: `ip`, `ping`, `traceroute`, `tracepath`, `mtr`, `tcpdump`, Python's `ipaddress`, and Docker networks

---

## 1. What IP is, and what it is not

The Internet Protocol lives at **Layer 3 (Network layer)**. Its job is **host-to-host delivery across networks**, answering: *"How does this packet get from that computer over there to this one, across dozens of networks I don't control?"*

| IP provides | IP does *not* provide |
|---|---|
| **Addressing**: an address per interface | **Reliability**: packets may be lost, duplicated, reordered, corrupted (TCP fixes it) |
| **Routing**: hop-by-hop forwarding by destination address | **Ports** / delivery to a program (TCP/UDP) |
| **Encapsulation** of transport data into packets | **Flow/congestion control** (TCP) |
| **Fragmentation** support (IPv4) | **Security or privacy** (use IPsec/TLS) |
| An error-reporting companion protocol: **ICMP** | **Sequencing, guaranteed timing** |

This is the *end-to-end principle* behind the internet: keep the network core simple and dumb (IP just forwards packets), and put intelligence at the ends (TCP, applications).

### Names of the data units (recap)
| Layer | Unit |
|---|---|
| L4 | Segment (TCP) / datagram (UDP) |
| **L3** | **Packet** (IP packet, also called an IP datagram) |
| L2 | Frame |

```
Application data ─► [TCP/UDP header | data] segment ─► [IP header | segment] PACKET ─► [Ethernet hdr | packet | FCS] frame ─► bits
```

---

## 2. A packet's journey (the big picture)

You send a message from Dhaka to a friend in New York. Every hop works the same way:

```
Your PC (192.168.1.50)
   │  "destination 72.21.91.29 is not on my subnet → send to my default gateway 192.168.1.1"
   ▼   Ethernet frame: dst MAC = gateway's MAC     IP packet: 192.168.1.50 → 72.21.91.29
Home router (NAT)      rewrites the source to its public IP; looks up 72.21.91.29 → forwards to the ISP
   ▼   new frame (new MACs), same destination IP; TTL 64 → 63
ISP router → ... → submarine cable → ... → US backbone → destination ISP → destination network
   ▼   each router: strip the L2 header, read the destination IP, decrement TTL, choose the next hop, build a NEW L2 header
Friend's phone           IP dst matches me → hand the payload to TCP/UDP (by protocol number and port)
```

Rules to remember:

1. **The destination IP address stays the same** all the way (except when NAT rewrites it).
2. **The L2 (MAC) addresses change at every hop.** Each link uses its own frame addressed to the *next* device.
3. **Routers look only at the IP header** (mostly the destination address and TTL). They forward packets independently, so packets of one connection may take different paths, and the return path may differ from the outgoing path (**asymmetric routing**).
4. Each router only needs to know **the next hop**, not the whole path.

---

## 3. The IPv4 header

An IPv4 packet is a header of **20 to 60 bytes** plus the payload. Each row below is 32 bits:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
┌───────┬───────┬───────────────┬───────────────────────────────┐
│Version│  IHL  │  DSCP  │ ECN  │         Total Length          │
├───────┴───────┴───────────────┼─────┬─────────────────────────┤
│        Identification         │Flags│     Fragment Offset     │
├───────────────┬───────────────┼─────┴─────────────────────────┤
│  Time to Live │   Protocol    │        Header Checksum        │
├───────────────┴───────────────┴───────────────────────────────┤
│                       Source IP Address                       │
├───────────────────────────────────────────────────────────────┤
│                    Destination IP Address                     │
├───────────────────────────────────────────────────────────────┤
│                     Options (if IHL > 5)                      │
└───────────────────────────────────────────────────────────────┘
```

| Field | Bits | Meaning |
|---|---|---|
| **Version** | 4 | `4` for IPv4 (`6` for IPv6). Tells the receiver how to parse the rest |
| **IHL** (Internet Header Length) | 4 | Header length in **32-bit words**: bytes = IHL × 4. Minimum **5** (20 bytes), maximum **15** (60 bytes) |
| **DSCP** | 6 | *Differentiated Services Code Point*: traffic class for QoS (0 = best effort, 46 = EF, used for voice; 34 = AF41, video). Formerly the "Type of Service" field |
| **ECN** | 2 | *Explicit Congestion Notification*: `00` not ECN-capable, `01`/`10` capable, `11` congestion experienced (routers mark instead of dropping) |
| **Total Length** | 16 | Whole packet (header + data) in bytes. Max **65,535** |
| **Identification** | 16 | Identifies which original packet a **fragment** belongs to |
| **Flags** | 3 | bit 0 reserved (0); **DF** (Don't Fragment); **MF** (More Fragments) |
| **Fragment Offset** | 13 | Position of this fragment's data in the original packet, in **8-byte units** |
| **TTL** (Time To Live) | 8 | Hop counter, decremented by each router; at 0 the packet is dropped (section 8) |
| **Protocol** | 8 | Which protocol is in the payload: **1** ICMP, **6** TCP, **17** UDP, 41 IPv6-in-IPv4, 47 GRE, 50 ESP (IPsec), 89 OSPF, ... |
| **Header Checksum** | 16 | 16-bit one's-complement checksum of the **header only** (recomputed at every hop because TTL changes) |
| **Source / Destination Address** | 32 each | The sender's and receiver's IPv4 addresses |
| **Options** | 0–40 bytes | Rarely used today (record route, timestamp, source routing); most routers drop or slow-path packets with options for security/performance reasons |

### 3.1 Decode a real header

A sample header in hex (20 bytes):

```
45 00 00 73  00 00 40 00  40 11 b8 61  c0 a8 00 01  c0 a8 00 c7
```

| Bytes | Value | Meaning |
|---|---|---|
| `4` `5` | 0x45 | Version 4, IHL 5 → 20-byte header |
| `00` | 0x00 | DSCP 0, ECN 0 (best effort) |
| `00 73` | 115 | Total length 115 bytes → payload 95 bytes |
| `00 00` | 0 | Identification 0 |
| `40 00` | 0x4000 | Flags `010` = **DF set**, MF clear; Fragment offset 0 |
| `40` | 64 | **TTL = 64** |
| `11` | 17 | Protocol = **UDP** |
| `b8 61` | | Header checksum |
| `c0 a8 00 01` | 192.168.0.1 | **Source** (192=0xc0, 168=0xa8, 0, 1) |
| `c0 a8 00 c7` | 192.168.0.199 | **Destination** (0xc7 = 199) |

**Checksum check:** add all ten 16-bit words including the checksum: `4500 + 0073 + 0000 + 4000 + 4011 + b861 + c0a8 + 0001 + c0a8 + 00c7 = 0x2FFFD`; fold the carry (`0x2` + `0xFFFD` = `0xFFFF`) → **`0xFFFF`** ✓. A valid header always sums to `0xFFFF`. To *compute* the checksum, set the field to 0, sum the header, fold carries, and take the bitwise complement.

### 3.2 Try it: decode with Python

```python
import struct, socket
hdr = bytes.fromhex("45000073000040004011b861c0a80001c0a800c7")
ver_ihl, tos, total_len, ident, flags_frag, ttl, proto, csum, src, dst = struct.unpack("!BBHHHBBH4s4s", hdr)
print("version", ver_ihl >> 4, "IHL", ver_ihl & 0xF, "len", total_len, "TTL", ttl, "proto", proto)
print("DF" if flags_frag & 0x4000 else "", "MF" if flags_frag & 0x2000 else "", "offset", (flags_frag & 0x1FFF) * 8)
print(socket.inet_ntoa(src), "->", socket.inet_ntoa(dst))
```
(You can also capture real packets: `sudo tcpdump -i any -nn -v -c 3 icmp` prints `ttl`, `id`, `flags [DF]`, `proto`, `length` for each packet.)

---

## 4. IPv4 addresses

An IPv4 address is **32 bits**, written as four **octets** in dotted decimal:

```
192      .  168      .  1        .  10
11000000 . 10101000 .  00000001 .  00001010     (each octet 0–255)
```
**2³² ≈ 4.3 billion** addresses in total.

### 4.1 Network part and host part
Every address has a **network prefix** (which network) and a **host part** (which device on it). The split is given by the **prefix length** (CIDR, `/24`) or equivalently a **subnet mask** (`255.255.255.0`):

```
192.168.1.10/24   →   network 192.168.1.0    host .10
mask 255.255.255.0 = 11111111.11111111.11111111.00000000   (24 ones)
```
A device decides "is the destination on my local network?" by comparing the **network parts** (`(my_ip AND mask) == (dst_ip AND mask)`). Subnetting and CIDR have their own chapters (33 and 34), so only the essentials here:

| Prefix | Mask | Addresses | Usable hosts |
|---|---|---|---|
| /8 | 255.0.0.0 | 16,777,216 | 16,777,214 |
| /16 | 255.255.0.0 | 65,536 | 65,534 |
| /24 | 255.255.255.0 | 256 | 254 |
| /30 | 255.255.255.252 | 4 | 2 (point-to-point links) |
| /31 | 255.255.255.254 | 2 | 2 (point-to-point, RFC 3021) |
| /32 | 255.255.255.255 | 1 | a single host route |

In a normal subnet the **first address** (host bits all 0) is the **network address** and the **last** (all 1) is the **directed broadcast**; neither can be assigned to a host, hence "− 2".

### 4.2 Historical classes
Originally addresses were split into fixed **classes** by their first bits:

| Class | First bits | First octet | Default prefix | Notes |
|---|---|---|---|---|
| A | `0` | 1–126 | /8 | 127 is loopback; 0 is reserved |
| B | `10` | 128–191 | /16 | |
| C | `110` | 192–223 | /24 | |
| D | `1110` | 224–239 | | Multicast |
| E | `1111` | 240–255 | | Reserved/experimental |

**Classful addressing is obsolete** (wasteful: a company needing 300 addresses had to take a /16 with 65,534). Since 1993 the internet uses **CIDR** (classless), with arbitrary prefix lengths and route aggregation. You'll still see the class names in old material and exams.

### 4.3 Special-purpose IPv4 addresses

| Range | Purpose |
|---|---|
| **10.0.0.0/8**, **172.16.0.0/12** (172.16–172.31), **192.168.0.0/16** | **Private** (RFC 1918): reusable inside any network, **not routable on the public internet**. Need NAT to reach it |
| **127.0.0.0/8** (usually `127.0.0.1`) | **Loopback**: packets never leave the host; `localhost` |
| **169.254.0.0/16** | **Link-local / APIPA**: self-assigned when DHCP fails (also `169.254.169.254` = cloud metadata service) |
| **100.64.0.0/10** | **Carrier-grade NAT** shared space (RFC 6598): ISPs' internal use; Tailscale also uses it |
| **0.0.0.0** | "This host / any address" (e.g. a server bound to `0.0.0.0` listens on all interfaces; a client with no address yet); `0.0.0.0/0` = the **default route** |
| **255.255.255.255** | Limited broadcast (this local network only; used by DHCP discover) |
| **224.0.0.0/4** | **Multicast** (`224.0.0.251` mDNS, `224.0.0.1` all hosts) |
| **240.0.0.0/4** | Reserved |
| **192.0.2.0/24**, **198.51.100.0/24**, **203.0.113.0/24** | **Documentation** (TEST-NET-1/2/3): safe for examples |
| **198.18.0.0/15** | Benchmark testing |

Everything else is (in principle) **public**, globally unique, and allocated by **IANA → five Regional Internet Registries (ARIN, RIPE NCC, APNIC, LACNIC, AFRINIC) → ISPs and organizations**. The free pool of IPv4 addresses ran out: IANA allocated its last blocks in **2011**, and the regional registries followed. Addresses are now traded and rented, and shared via NAT.

---

## 5. IPv6

### 5.1 Why
IPv4's 4.3 billion addresses can't cover billions of people and tens of billions of devices. **IPv6** (RFC 8200) uses **128-bit** addresses: 2¹²⁸ ≈ 3.4 × 10³⁸. That is about **4 × 10²⁸ addresses for every person on Earth**, enough that every device can have a globally unique address with **end-to-end connectivity**, and no NAT is needed.

### 5.2 Address format
Eight groups of four hexadecimal digits (16 bits each), separated by colons:

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
```

Two shorthand rules:
1. **Drop leading zeros** in each group: `2001:db8:85a3:0:0:8a2e:370:7334`
2. Replace **one** run of consecutive all-zero groups with `::` (only once per address): `2001:db8:85a3::8a2e:370:7334`

Special: **`::1`** = loopback, **`::`** = unspecified. In URLs, put IPv6 in brackets: `http://[2001:db8::1]:8080/`. Prefix notation is the same as CIDR: `2001:db8:abcd:12::/64`.

### 5.3 Address types

| Prefix | Type | Notes |
|---|---|---|
| `2000::/3` | **Global unicast** | Public, routable (currently allocated from `2000::/3`) |
| `fe80::/10` | **Link-local** | Automatically configured on every interface; valid only on the local link (used by neighbor discovery and routing protocols); written with a zone like `fe80::1%eth0` |
| `fc00::/7` (in practice `fd00::/8`) | **Unique local (ULA)** | The private-network equivalent (RFC 4193), not routed on the internet |
| `ff00::/8` | **Multicast** | Replaces broadcast (`ff02::1` all nodes, `ff02::2` all routers) |
| `::1/128` | Loopback | |
| `2001:db8::/32` | Documentation | For examples |
| `::ffff:0:0/96` | IPv4-mapped | Represents IPv4 addresses inside IPv6 sockets |

Typical allocation: an ISP gives a home a **/56** or **/48**, and each LAN uses a **/64** (64 bits of network + 64 bits of interface ID). There is **no broadcast**; it uses multicast.

### 5.4 The IPv6 header (simpler: fixed 40 bytes)

```
┌───────┬───────────────┬───────────────────────────────────────┐
│Version│ Traffic Class │              Flow Label               │
├───────┴───────────────┼───────────────┬───────────────────────┤
│     Payload Length    │  Next Header  │       Hop Limit       │
├───────────────────────┴───────────────┴───────────────────────┤
│                      Source Address (128 bits)                │
├───────────────────────────────────────────────────────────────┤
│                   Destination Address (128 bits)              │
└───────────────────────────────────────────────────────────────┘
```
- **Hop Limit** replaces TTL; **Next Header** replaces Protocol and chains **extension headers** (routing, fragment, hop-by-hop, ...) instead of IPv4 Options.
- **No header checksum** (link layers and TCP/UDP checksums cover it; UDP's checksum is mandatory over IPv6): less work per hop.
- **No fragmentation by routers**: only the *sender* fragments (with a Fragment extension header), relying on **Path MTU Discovery**. The minimum IPv6 MTU is **1280 bytes**.

### 5.5 Configuration and neighbors
- **SLAAC** (Stateless Address Autoconfiguration): a host builds its own address from the **prefix advertised by the router** (Router Advertisements, ICMPv6) plus an interface identifier (random "privacy addresses" by default). **DHCPv6** is optional for extra settings or stateful assignment.
- **NDP (Neighbor Discovery Protocol)** over ICMPv6 replaces ARP (Chapter 39) and also does router discovery and duplicate address detection.
- **IPsec is not mandatory** in IPv6 (an older claim); it is supported, like in IPv4.

### 5.6 IPv4 vs IPv6

| | IPv4 | IPv6 |
|---|---|---|
| Address | 32 bits, dotted decimal | 128 bits, hex with colons |
| Header | 20–60 bytes (options) | 40 bytes fixed (+ extension headers) |
| Checksum in header | Yes | No |
| Fragmentation | Sender **and** routers | Sender only (PMTUD required) |
| Broadcast | Yes | No (multicast) |
| Address configuration | DHCP / manual | SLAAC (+ DHCPv6) |
| ARP | Yes | NDP (ICMPv6) |
| NAT | Ubiquitous | Not needed (though NPTv6 exists) |
| Adoption | Universal | Large and growing (national and per-service adoption varies widely; check e.g. Google's IPv6 statistics) |

Most systems run **dual stack** (both IPv4 and IPv6), and clients use **Happy Eyeballs** (RFC 8305): try IPv6 first, fall back to IPv4 within milliseconds if it's slow. IPv6 is not inherently faster; the gains are end-to-end connectivity, no CGNAT, and simpler forwarding.

---

## 6. Routing: how routers choose

### 6.1 The routing table
Every host and router has a **routing table**: a list of `destination prefix → next hop / interface`. See yours:

```bash
ip route          # Linux (modern);  also:  route -n / netstat -rn
# default via 192.168.1.1 dev wlan0 proto dhcp metric 600
# 192.168.1.0/24 dev wlan0 proto kernel scope link src 192.168.1.50 metric 600
# 172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
ip -6 route
```

- `192.168.1.0/24 dev wlan0` is a **connected route**: "this subnet is directly reachable on that interface; deliver by ARP."
- `default via 192.168.1.1` (`0.0.0.0/0`) is the **default route** / **default gateway**: "for anything not matched more specifically, send it to this router."

### 6.2 Longest-prefix match
When several routes match a destination, the router picks the **most specific one**: the one with the **longest prefix**. (Ties broken by administrative distance/metric.)

```
Destination 192.168.1.50; routes:
  0.0.0.0/0         → router C   (matches everything, /0)
  192.168.0.0/16    → router A   (matches, /16)
  192.168.1.0/24    → router B   (matches, /24)   ← longest → chosen
```
This rule lets the internet **aggregate** millions of networks into compact tables (an ISP advertises one big prefix), while still allowing specific exceptions.

### 6.3 Direct vs indirect delivery
For each outgoing packet the sender does:

1. Is the destination in a **directly connected** subnet? → **ARP** for the destination's MAC (IPv6: NDP) and send the frame straight to it.
2. Otherwise → look up the route; the packet goes to the **next-hop router's MAC** (the destination IP is unchanged).

Try `ip route get 8.8.8.8` (shows which route, interface, next hop and source address would be used) and `ip route get 192.168.1.20`.

### 6.4 Learning routes
| Method | How | Where |
|---|---|---|
| **Connected** | Automatic, from interface addresses | everywhere |
| **Static** | `ip route add 10.0.0.0/8 via 192.168.1.254` | small networks, hosts, containers |
| **Dynamic** routing protocols | Routers exchange information and compute paths | larger networks |

Dynamic protocols: **RIP** (distance-vector, hop count ≤ 15; old), **OSPF** (link-state, shortest path by cost; common inside organizations), **IS-IS** (link-state; ISPs), **EIGRP** (Cisco), and **BGP** (**Border Gateway Protocol**), the routing protocol of the internet: it exchanges reachability between **Autonomous Systems (ASes)**, the independently managed networks, each with an **AS number** (ISPs, cloud providers, large companies). BGP is *policy-based* (business relationships: customers, peers, transit providers), not just shortest path. A misconfigured or hijacked BGP announcement can redirect or black-hole traffic for large parts of the internet, which has happened several times. Private ASNs (64512–65534) are used inside organizations and in examples.

### 6.5 Anycast
Many servers can announce the **same** IP prefix from different places (DNS root servers, CDNs, `1.1.1.1`, `8.8.8.8`); routing delivers each client's packets to the **nearest** instance.

---

## 7. What a router does to a packet

1. Receive the frame, check the FCS, **strip the L2 header**.
2. Check the IP header (version, length, checksum). **Drop** if invalid.
3. **Decrement TTL**. If it reaches 0, drop the packet and send **ICMP Time Exceeded** to the source.
4. **Look up** the destination in the routing table (longest prefix); pick the next hop and outgoing interface.
5. **Recompute the header checksum** (TTL changed).
6. If the packet is **larger than the outgoing link's MTU** → fragment it (IPv4, if DF=0) or drop it and send **ICMP Fragmentation Needed** (DF=1).
7. Build a **new L2 frame** (source = router's MAC on that link, destination = next hop's MAC via ARP) and transmit.

Also: forwarding may apply **ACLs/firewall rules**, **QoS by DSCP**, **NAT**, or **policy routing**.

---

## 8. TTL and traceroute

**TTL** (IPv4) / **Hop Limit** (IPv6) stops packets from circulating forever in a routing loop. Senders choose an initial value (commonly **64** on Linux/macOS/Android/iOS, **128** on Windows, **255** on some routers); each router subtracts 1. You can guess the hop count and the remote OS family from a received TTL (e.g. TTL 117 → started at 128 → 11 hops away).

**traceroute** exploits this to map the path:

```
send probe with TTL=1 → the first router drops it and replies ICMP "Time Exceeded" (revealing its address)
send probe with TTL=2 → the second router replies
...
until the destination replies (Echo Reply / port unreachable)
```
```bash
traceroute -n 8.8.8.8            # UDP probes by default on Linux; -I for ICMP, -T for TCP
tracepath 8.8.8.8                # also discovers the path MTU
mtr -n -c 20 --report 8.8.8.8    # traceroute + ping statistics per hop (loss, latency)
```
Reading results: `* * *` at a hop may only mean that router doesn't answer probes (rate-limited or filtered), not that traffic stops there; **latency at an intermediate hop** can be misleading (routers deprioritize replying to probes); look at loss/latency from the **destination** backward.

---

## 9. MTU, fragmentation and Path MTU Discovery

**MTU** (Maximum Transmission Unit) = the largest L3 packet a link can carry: **Ethernet 1500**, PPPoE 1492, many VPN/tunnels ~1400–1450, jumbo frames 9000. Overheads matter: an IPv4/TCP segment of **1460 bytes** of data = 1500 with headers (**MSS** 1460).

### 9.1 IPv4 fragmentation
If a packet is bigger than the next link's MTU **and DF = 0**, the router splits it:

```
Original: 3020 bytes of payload (+20 header) over a link with MTU 1500
  Fragment 1: header + 1480 data, ID=12345, MF=1, offset=0
  Fragment 2: header + 1480 data, ID=12345, MF=1, offset=185     (1480 ÷ 8)
  Fragment 3: header +   60 data, ID=12345, MF=0, offset=370     (2960 ÷ 8)
```
Rules: all fragments carry the **same Identification**; data sizes (except the last) are **multiples of 8** because the offset is in 8-byte units; **only the destination reassembles** (routers don't). Problems: **losing one fragment loses the whole packet**; the extra load, firewalls that drop fragments, and security issues. Fragmentation is therefore avoided.

### 9.2 Path MTU Discovery (PMTUD)
The sender sets **DF = 1** and sends full-sized packets. If a router can't forward it, it drops the packet and returns **ICMP "Destination Unreachable, Fragmentation Needed (type 3, code 4)"** stating the next-hop MTU; the sender lowers its packet size. (IPv6: ICMPv6 "Packet Too Big".) TCP also negotiates **MSS** and adapts to PMTU.

**The classic failure: PMTU black holes.** If a firewall blocks ICMP (a common but harmful practice), the sender never learns that packets are too big, so small packets work but **large ones hang forever** (a website loads its header but the page stalls; SSH logs in but `ls` of a big directory hangs; a VPN "connects" but big transfers stall). Fixes: **allow the needed ICMP types**, **clamp the MSS** on the tunnel/router (`iptables ... -j TCPMSS --clamp-mss-to-pmtu`), or lower the MTU on the interface.

---

## 10. ICMP: IP's error and diagnostics helper

**ICMP** (Internet Control Message Protocol; IP protocol 1; ICMPv6 = 58) reports problems and supports diagnostics. It isn't a transport for applications.

| Type (code) | Message | Used by |
|---|---|---|
| 8 / 0 | **Echo Request** / **Echo Reply** | `ping` |
| 3 | **Destination Unreachable** (0 net, 1 host, 3 **port**, 4 **fragmentation needed**, 13 admin prohibited) | UDP "port unreachable", PMTUD |
| 11 | **Time Exceeded** (TTL expired) | `traceroute` |
| 5 | Redirect ("use another gateway") | (usually disabled) |
| 4 (old) / — | Source quench (obsolete) | |

```bash
ping -c 4 8.8.8.8                     # RTT and TTL of replies
ping -c 3 -M do -s 1472 8.8.8.8       # -M do = set DF; 1472 + 8 ICMP + 20 IP = 1500. Try 1473: "message too long" or fragmentation needed
ping -6 -c 3 ipv6.google.com          # IPv6
```
Don't blanket-block ICMP: allow echo, "fragmentation needed/packet too big", "time exceeded" and (IPv6) neighbor discovery.

---

## 11. NAT (Network Address Translation)

Because of IPv4 scarcity, most homes and offices use **private addresses** internally and share **one (or a few) public IPs**. A **NAT** device (your router) rewrites addresses/ports:

```
Inside                       NAT router                              Internet
192.168.1.50:54321 ────►  translates source ──►  203.0.113.50:60001 ─────────► 142.250.185.206:443
                          table: 203.0.113.50:60001  ↔  192.168.1.50:54321
reply to 203.0.113.50:60001 ─► router looks it up ─► rewrites destination to 192.168.1.50:54321 ─► inside
```

- The common form is **PAT / NAPT / "masquerading"**: many inside hosts share one public IP, told apart by **translated source ports**.
- **Outbound is easy** (the router creates the mapping). **Inbound is not**: an unsolicited packet has no table entry, so it's dropped. To reach a service inside you need **port forwarding** (a static mapping such as public `:8080` → `192.168.1.50:80`), UPnP/NAT-PMP, or a relay.
- **NAT is not a firewall.** It incidentally blocks unsolicited inbound traffic, but security requires real stateful firewall rules. It also **breaks the end-to-end model**: peer-to-peer apps need **STUN/TURN/ICE** hole-punching; some protocols carry IPs inside payloads (FTP, SIP), needing ALGs.
- **Double NAT / CGNAT:** ISPs increasingly put customers behind carrier-grade NAT (`100.64.0.0/10`), so you can't port-forward at all.
- **Docker publishing `-p 8080:80`** is exactly **DNAT** (destination NAT) in the host's firewall; container outbound traffic is **masqueraded** (SNAT). See rules with `sudo iptables -t nat -L -n -v` (or `nft list ruleset`).
- **Connection tracking** (`conntrack`) is the table behind it: `sudo conntrack -L`.
- IPv6 doesn't need NAT; use a stateful firewall instead.

---

## 12. IP in Docker and Kubernetes

| Concept | Details |
|---|---|
| **Default bridge network** | Subnet **172.17.0.0/16**, gateway `172.17.0.1` (the host's `docker0` interface); containers get addresses like `172.17.0.2`; outbound traffic is masqueraded to the host's IP |
| **User-defined networks** | Docker allocates subnets from a pool (default `172.18.0.0/16`, `172.19.0.0/16`, ... then `192.168.x.0/20`); choose your own with `docker network create --subnet 10.10.0.0/24 mynet` and a specific container address with `--ip` |
| **Inspect** | `docker network ls`, `docker network inspect bridge -f '{{json .IPAM.Config}}'`, `docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}} gw={{.Gateway}}{{end}}' NAME` |
| **Host view** | `ip -br addr` shows `docker0` and `br-<id>` interfaces plus `veth...` peers; `ip route` shows a route per Docker subnet |
| **Address conflicts** | Docker's default 172.16/12 ranges **overlap with corporate VPNs and cloud VPCs**. Symptoms: containers can't reach an internal `172.x.x.x` service, or the VPN breaks after starting Docker. Fix: set `default-address-pools` in `/etc/docker/daemon.json` (e.g. `{"default-address-pools":[{"base":"10.200.0.0/16","size":24}]}`) or create networks with explicit non-overlapping subnets |
| **Container-to-container** | Same subnet on the same bridge → direct (L2 via the Linux bridge); different networks → isolated unless a container is attached to both or routed |
| **`localhost`** | Inside a container, `127.0.0.1` is the container itself |
| **IPv6** | Off by default; enable in `daemon.json` (`"ipv6": true, "fixed-cidr-v6": "fd00:db8:1::/64"`) or per network (`docker network create --ipv6 ...`) |
| **Kubernetes** | Every **pod** gets its own IP from the cluster's **pod CIDR** (a CNI plugin provides routing/overlay; pods can talk to any pod without NAT); **Services** get virtual **cluster IPs** from the service CIDR; `NodePort`/`LoadBalancer` expose them; plan CIDRs so pod, service, node and VPC ranges **don't overlap** |

---

## 13. Hands-on labs

**Lab 1: Your addresses and routes**

```bash
ip -br addr                  # interfaces, states, IPv4/IPv6 with prefix lengths
ip route                     # default gateway and connected routes
ip -6 route
ip route get 8.8.8.8         # "via <gw> dev <if> src <your ip>" → which route wins and why
ip route get 127.0.0.1
```
Identify: your private IP, the mask/prefix, the default gateway, whether the docker0 subnet appears, and any IPv6 global and link-local addresses.

**Lab 2: See IP headers on the wire**

```bash
sudo tcpdump -i any -nn -v -c 4 icmp &
ping -c 2 8.8.8.8
```
Read: `ttl 64`, `id`, `flags [DF]`, `proto ICMP (1)`, `length 84` (20 IP + 8 ICMP + 56 data). Then `sudo tcpdump -i any -nn -XX -c 1 icmp` to see the raw bytes; find `45 00` at the start of the IP header.

**Lab 3: TTL and traceroute**

```bash
ping -c 1 -t 1 8.8.8.8            # (Linux: -t sets TTL) → "Time to live exceeded" from the first router
ping -c 1 -t 2 8.8.8.8            # second router answers
traceroute -n 8.8.8.8
mtr -n -c 10 --report 1.1.1.1
```
For each hop note the private-vs-public addresses (your NAT gateway, ISP internal `100.64.x.x`/`10.x.x.x`, then public transit).

**Lab 4: MTU and fragmentation**

```bash
ip link show | grep mtu                         # interface MTUs (docker0 is 1500)
ping -c 2 -M do -s 1472 8.8.8.8                 # fits exactly 1500 → works
ping -c 2 -M do -s 1473 8.8.8.8                 # 1501 → "Message too long" (local) or ICMP frag needed
ping -c 2 -s 3000 8.8.8.8 &                     # DF not set: fragmented by the sender
sudo tcpdump -i any -nn -v icmp                 # see "frag ... (frag 1234:1480@0+)" then "(frag 1234:1528@1480)"
tracepath -n 8.8.8.8                            # reports "pmtu 1500"
```

**Lab 5: Subnet arithmetic with Python's `ipaddress`**

```python
import ipaddress as ip
n = ip.ip_network("192.168.1.0/24")
print(n.netmask, n.broadcast_address, n.num_addresses, list(n.hosts())[0], list(n.hosts())[-1])
print(list(ip.ip_network("192.168.1.0/24").subnets(new_prefix=26)))        # 4 subnets of 64
print(ip.ip_address("192.168.1.77") in ip.ip_network("192.168.1.64/26"))   # True
print(ip.ip_address("10.1.2.3").is_private, ip.ip_address("8.8.8.8").is_global, ip.ip_address("::1").is_loopback)
print(ip.ip_address("2001:db8::1").exploded, ip.ip_address("2001:0db8:0000:0000:0000:0000:0000:0001").compressed)
```

**Lab 6: Longest-prefix match by hand.** Add temporary routes to a scratch namespace (safe; nothing touches your real routing):

```bash
sudo ip netns add lab
sudo ip netns exec lab ip link set lo up
sudo ip netns exec lab ip link add d0 type dummy && sudo ip netns exec lab ip link set d0 up
sudo ip netns exec lab ip addr add 10.9.0.1/24 dev d0
sudo ip netns exec lab ip route add 192.168.0.0/16 via 10.9.0.2
sudo ip netns exec lab ip route add 192.168.1.0/24 via 10.9.0.3
sudo ip netns exec lab ip route add default via 10.9.0.4
sudo ip netns exec lab ip route get 192.168.1.50      # → via 10.9.0.3  (the /24 wins)
sudo ip netns exec lab ip route get 192.168.7.50      # → via 10.9.0.2  (the /16)
sudo ip netns exec lab ip route get 8.8.8.8           # → via 10.9.0.4  (default)
sudo ip netns del lab
```

**Lab 7: Docker networking**

```bash
docker network create --subnet 10.77.0.0/24 labnet
docker run -d --name a --network labnet --ip 10.77.0.10 alpine:3.20 sleep 3600
docker run -d --name b --network labnet --ip 10.77.0.11 alpine:3.20 sleep 3600
docker exec a ip -br addr; docker exec a ip route; docker exec a ping -c 2 10.77.0.11
ip -br addr | grep br-; ip route | grep 10.77
docker exec a traceroute -n 8.8.8.8 2>/dev/null || docker exec a sh -c 'apk add --no-cache busybox-extras >/dev/null; traceroute -n 8.8.8.8'   # 1st hop: the bridge 10.77.0.1, then your LAN gateway
sudo iptables -t nat -L POSTROUTING -n | grep 10.77            # the MASQUERADE rule for outbound traffic
docker rm -f a b; docker network rm labnet
```

**Lab 8: IPv6**

```bash
ip -6 addr                     # fe80:: link-local always; a global 2xxx: address if your network has IPv6
ping -6 -c 2 ::1
curl -6 -sI https://ipv6.google.com | head -1        # works only with IPv6 connectivity
curl -s https://api64.ipify.org                       # which address family do you use to reach the internet?
```

**Lab 9: What is my public IP, and what does NAT change?** `curl -s https://ifconfig.me` (public) vs `ip -br addr` (private). Explain why they differ, then `curl` the same from inside a Docker container and note both are the same public IP.

---

## 14. Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| `Network is unreachable` | No route (missing default gateway or interface down) | `ip route`, `ip -br addr`, `ip link` |
| Can ping the gateway but not the internet | Default route wrong, ISP or gateway problem, NAT/DNS issue | `ping 8.8.8.8` (IP) vs `ping example.com` (DNS); `traceroute -n 8.8.8.8` |
| Ping works but the website hangs after connecting | **MTU/PMTU black hole** | `ping -M do -s 1472 host`, `tracepath`, MSS clamping, allow ICMP |
| Two hosts on a LAN can't talk | Different subnets/masks, VLAN, duplicate IP | `ip addr`, `arping -D`, mask consistency |
| Random connection failures/duplicates | **Duplicate IP addresses** on the LAN | `arping`, DHCP scope/static overlap |
| `169.254.x.x` address | DHCP failed | Check DHCP server/cable/Wi-Fi |
| Can't reach a private IP behind a VPN in Docker | Docker's subnet overlaps the VPN's | Change `default-address-pools` / network subnet |
| IPv6 sites slow (timeouts, then fine) | Broken IPv6 path, Happy Eyeballs fallback delay | Test with `-4`/`-6` in curl; fix IPv6 routing or disable it on that path |
| Asymmetric routing breaks a stateful firewall | Reply takes another path | Check both directions with `traceroute` from each end |
| Cannot reach a server behind NAT from outside | No port forwarding, or CGNAT | Configure a forward, use a tunnel/relay |
| `ttl exceeded` in traceroute loops | Routing loop | Check routes between the repeating hops |

---

## 15. Common misconceptions

| Misconception | Reality |
|---|---|
| "IP guarantees delivery" | It is best effort; TCP (or the app) adds reliability |
| "An IP address identifies a device permanently" | Addresses are assigned (often by DHCP), change, and are shared through NAT; an interface can have several |
| "Routers read the whole packet" | Routers look at the IP header (and optionally L4 for firewalling/ECMP); they don't need the payload |
| "MAC addresses are used across the internet" | MACs only matter on each local link and change at every hop |
| "The path out is the path back" | Routing is per hop and can be asymmetric |
| "IPv6 is faster/more secure by itself" | Simpler forwarding and no NAT help, but speed is similar and security depends on configuration |
| "NAT is a firewall" | It's address rewriting; use real firewall rules |
| "Private addresses are secret/unreachable" | They are just non-routable on the public internet; anyone on your network can reach them |
| "`/24` means 24 hosts" | It means **24 network bits**, leaving 8 host bits (254 usable) |
| "IPv6 IPsec is mandatory" | It's supported, not mandatory |
| "Fragmentation is normal and fine" | It hurts performance and reliability; PMTUD and MSS avoid it |
| "The internet runs out of IPv4, so it stopped working" | Scarcity is handled by NAT, address trading, and IPv6 adoption |

---

## 16. Summary

- **IP** = addressing + hop-by-hop routing at **Layer 3**; **best effort**, with no reliability, ports or ordering. Unit: the **packet**; ICMP reports errors.
- **IPv4 header:** version, IHL, DSCP/ECN, total length, identification/flags/fragment offset, **TTL**, **protocol** (1/6/17), **header checksum**, **source**, **destination**, options. Decode it from hex; the checksum of a valid header sums to `0xFFFF`.
- **IPv4 addresses:** 32 bits; prefix/mask split network and host; private ranges (10/8, 172.16/12, 192.168/16), loopback 127/8, link-local 169.254/16, CGNAT 100.64/10, multicast 224/4; classes are historical; CIDR is current.
- **IPv6:** 128-bit addresses with `::` shorthand, link-local/ULA/global types, fixed 40-byte header, no checksum, no router fragmentation, SLAAC, NDP, multicast instead of broadcast.
- **Routing:** hosts/routers use tables; **longest-prefix match** picks the route; the **destination IP stays the same, the MACs change per hop**; TTL decrements each hop; BGP glues autonomous systems together.
- **MTU/PMTUD:** avoid fragmentation with DF + ICMP "frag needed"; blocked ICMP creates black holes; clamp MSS.
- **NAT** shares public addresses (DNAT for Docker `-p`, SNAT/masquerade outbound) but is not a firewall.
- **Docker:** `docker0` = 172.17.0.0/16; watch out for subnet overlap with VPNs; Kubernetes gives each pod an IP.

---

## 17. Check your understanding

1. In an IPv4 header `IHL = 8` and Total Length = 300. How long is the header and how much payload does the packet carry?
2. Which header field tells the receiver the payload is TCP? Which number?
3. A packet crosses 5 routers. Which of (source IP, destination IP, source MAC, destination MAC, TTL) change, and how?
4. Routes: `10.0.0.0/8 → A`, `10.1.0.0/16 → B`, `10.1.2.0/24 → C`, `0.0.0.0/0 → D`. Where do packets to 10.1.2.9, 10.1.9.9, 10.9.9.9 and 8.8.8.8 go?
5. Compress `2001:0db8:0000:0000:0000:ff00:0042:8329`.
6. A 4,000-byte payload must cross a link with MTU 1500 (IPv4, DF=0, 20-byte header). How many fragments, with what data sizes and offsets?
7. Why can a website hang after the TCP handshake completes when a firewall blocks all ICMP?
8. Why can't two containers on different user-defined Docker networks talk by default, and what would allow it?
9. Why is NAT not a security feature? What does `docker run -p 8080:80` do at the IP layer?
10. Give two reasons that IPv6 doesn't need NAT.

<details>
<summary>Answers</summary>

1. Header = 8 × 4 = 32 bytes (12 bytes of options); payload = 300 − 32 = 268 bytes.
2. The Protocol field, value 6 (UDP = 17, ICMP = 1).
3. Source and destination IP: unchanged (barring NAT). Source and destination MAC: rewritten at every hop (new frame per link). TTL: decremented by 1 at each router (64 → 59 after five routers).
4. 10.1.2.9 → C (/24); 10.1.9.9 → B (/16); 10.9.9.9 → A (/8); 8.8.8.8 → D (default).
5. `2001:db8::ff00:42:8329`.
6. Data per fragment must be ≤ 1480 and a multiple of 8: 1480 fits. 4000 ÷ 1480 → fragments of 1480, 1480, 1040 bytes (three fragments) with offsets 0, 185, 370 and MF = 1, 1, 0. (Total header bytes = 3 × 20.)
7. Large packets need Path MTU Discovery, which relies on ICMP "fragmentation needed"/"packet too big". If it's blocked, the sender keeps sending oversized DF packets that are dropped silently (a PMTU black hole), while small packets, like the handshake, work.
8. They are separate subnets/bridges with no route or forwarding between them (isolation). Attach a container to both networks, connect one container to the other network (`docker network connect`), or route/publish ports.
9. NAT only rewrites addresses/ports and incidentally drops unsolicited inbound traffic; it doesn't inspect or authorize traffic like a firewall. `-p 8080:80` installs DNAT (and forwarding) rules mapping host port 8080 to the container's IP:80.
10. IPv6 has vastly more addresses (every device can be globally unique), so no address sharing is needed; and end-to-end addressing is part of its design (privacy and security handled by firewalls and address privacy extensions).
</details>

**Practice**

1. Capture a `ping` and a `curl` with tcpdump, and decode the IP headers of one packet by hand from the hex (version, IHL, total length, TTL, protocol, addresses); verify the checksum with a script.
2. Use `traceroute`/`mtr` to two destinations (one nearby, one on another continent), annotate each hop as private/CGNAT/public, and estimate where the biggest latency jump occurs.
3. Find your path MTU to three destinations with `ping -M do -s` and `tracepath`; explain any results below 1500.
4. Create two Docker networks with non-overlapping subnets, attach a container to both, and prove it can route between them (`sysctl net.ipv4.ip_forward=1` inside it, and add routes in the peers).
5. Enable IPv6 on a Docker network, start two containers, and ping each other's `fd00:` addresses.

---

**Next:** [Chapter 31 – The Data Link Layer and Ethernet Frames](31_data_link_layer_frame_in_details.md)
