# Chapter 33: Subnetting and Subnet Masks

> **In one sentence:** An IP address has two parts, a **network part** and a **host part**, and the **subnet mask** is the ruler that tells every device where one ends and the other begins, which is how a host decides "is this destination on my own network (deliver directly) or somewhere else (hand it to the router)?"

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~60 minutes

**Prerequisites:** [Chapter 30](30_internet_protocol_ip_in_details.md) (IP addresses) and [Chapter 32](32_first_computer_and_first_router_in_details.md) (the four settings every host needs).

---

## What you will learn

- Why IP addresses are split into a **network** part and a **host** part
- How to read a **subnet mask** in dotted-decimal and **CIDR `/24`** form
- The **AND** operation that turns "address + mask" into the **network address**
- How to find the **network address, broadcast address, first/last usable host, and host count** for any address (with a fast method that needs no binary)
- How a host uses the mask for **every packet it sends**: local vs remote
- **What breaks** when masks are wrong, and how to spot it
- How to **divide** one network into smaller ones (**subnetting**) with worked examples
- Doing all of it with `ipcalc`, `sipcalc` and Python's `ipaddress` module

---

## 1. The problem a mask solves

A computer that just received an address from DHCP faces the question every time it sends a packet:

> "The destination is `192.168.1.77`. Is it **on my own local network** (I can ARP for it and send a frame straight to it), or is it **somewhere else** (I must send the packet to my router)?"

Delivering directly and asking the router are completely different actions. A host must be able to decide **using only its own configuration**: its address, its mask and its gateway. It cannot ask "is the other machine near me?". It has no map of the internet. So the address must carry, within its own structure, the answer to *"which network do I belong to?"*.

Analogy: a postal address `12 Oak Street, Springfield`. The **street/town** part identifies the neighborhood (network); the **house number** identifies the home (host). Mail for your street can be walked next door; mail for another town goes to the post office (the router). The subnet mask is the rule telling you where the "town" part stops and the "house" part starts.

---

## 2. IP addresses in binary

An IPv4 address is **32 bits**, written as four **octets** (8 bits, values 0–255) separated by dots:

```
192      .  168      .  1        .  77
11000000 . 10101000 . 00000001 . 01001101
```

Quick way to convert an octet: the bit positions are worth `128 64 32 16 8 4 2 1`.
- `77` = 64 + 8 + 4 + 1 → `01001101`
- `168` = 128 + 32 + 8 → `10101000`

```bash
python3 -c "print(format(77,'08b'), int('11000000',2))"      # 01001101 192
```

---

## 3. What is a "network"?

A **network** (here: an IP subnet) is a group of interfaces that:
- can reach each other **directly at Layer 2** (same switch/VLAN/Wi-Fi network, no router between them), and
- share the same **leading bits** in their addresses.

All devices at home connected to your router have addresses starting `192.168.1.`; they are one network. Your neighbor's `192.168.1.x` network is a *different, separate* network that happens to use the same numbers (private addresses, Chapter 30). The **router** sits between networks.

So an address = **network bits** (identical for every member) + **host bits** (unique per member):

```
192.168.1.77 with 24 network bits:

network part:  192.168.1      |  host part: 77
bits:          11000000 10101000 00000001 | 01001101
                     24 bits              |   8 bits
```

Where the line goes is **not fixed by the address itself**; the mask says. (Section 12 shows why the old "class" rules stopped mattering.)

---

## 4. The subnet mask

A **subnet mask** is 32 bits: **1s for the network part, then 0s for the host part**. Always a solid run of 1s followed by a solid run of 0s.

```
255.255.255.0  =  11111111 . 11111111 . 11111111 . 00000000
                  └──────── 24 ones ────────┘  └─ 8 zeros ─┘
```

### Two ways to write it
| Dotted decimal | CIDR / prefix length | Meaning |
|---|---|---|
| `255.255.255.0` | `/24` | first 24 bits are network |
| `255.255.0.0` | `/16` | first 16 bits |
| `255.0.0.0` | `/8` | first 8 bits |
| `255.255.255.128` | `/25` | first 25 bits |
| `255.255.255.192` | `/26` | first 26 bits |
| `255.255.255.252` | `/30` | first 30 bits |

`192.168.1.77/24` and `192.168.1.77` + `255.255.255.0` mean exactly the same thing. (CIDR is the topic of Chapter 34.)

### The masks you can memorize
Valid masks only have these values in the "interesting" octet:

| Octet value | Binary | Ones in it |
|---|---|---|
| 0 | 00000000 | 0 |
| 128 | 10000000 | 1 |
| 192 | 11000000 | 2 |
| 224 | 11100000 | 3 |
| 240 | 11110000 | 4 |
| 248 | 11111000 | 5 |
| 252 | 11111100 | 6 |
| 254 | 11111110 | 7 |
| 255 | 11111111 | 8 |

So `255.255.240.0` is `/20` (8+8+4). A mask like `255.255.253.0` is **invalid** (not contiguous ones).

---

## 5. The AND operation: finding the network

To find which network an address belongs to, the device performs a **bitwise AND** of the address with the mask. AND keeps a bit only if both bits are 1: so it **keeps the network bits and zeroes the host bits**.

```
  IP    192.168.1.77    11000000.10101000.00000001.01001101
  Mask  255.255.255.0   11111111.11111111.11111111.00000000
  AND   ------------------------------------------------------
  Net   192.168.1.0     11000000.10101000.00000001.00000000
```
Result: **network address `192.168.1.0`**.

**Two addresses are on the same network if AND-ing each with the mask gives the same result.** That is the entire rule.

```bash
python3 - <<'E'
import ipaddress as i
a = i.ip_interface("192.168.1.77/24")
print(a.network)                                     # 192.168.1.0/24
print(i.ip_address("192.168.1.200") in a.network)    # True   (same network)
print(i.ip_address("192.168.2.5")   in a.network)    # False  (different network)
E
```

---

## 6. Special addresses inside every subnet

In each subnet, the all-zeros and all-ones host patterns are reserved:

| Address | Host bits | Meaning | Example (`192.168.1.0/24`) |
|---|---|---|---|
| **Network address** | all **0** | Names the network itself; not assigned to a host | `192.168.1.0` |
| **First usable** | `…0001` | Often the gateway (convention only) | `192.168.1.1` |
| **Last usable** | `…1110` | Highest host | `192.168.1.254` |
| **Broadcast** | all **1** | "Everyone on this network" (directed broadcast) | `192.168.1.255` |

The **usable host count** is `2^h − 2` (h = host bits; the −2 removes network and broadcast):

| Prefix | Host bits | Total addresses | Usable hosts |
|---|---|---|---|
| /8 | 24 | 16,777,216 | 16,777,214 |
| /16 | 16 | 65,536 | 65,534 |
| /24 | 8 | 256 | **254** |
| /25 | 7 | 128 | 126 |
| /26 | 6 | 64 | 62 |
| /27 | 5 | 32 | 30 |
| /28 | 4 | 16 | 14 |
| /29 | 3 | 8 | 6 |
| /30 | 2 | 4 | **2** |
| /31 | 1 | 2 | 2 (special: point-to-point, RFC 3021) |
| /32 | 0 | 1 | 1 (a single host route) |

`/30` was the classic point-to-point link (2 usable), and **`/31`** (RFC 3021) now saves addresses on router-to-router links by dropping the network/broadcast reservation.

### Broadcast addresses: which one?
- **`255.255.255.255`** (limited broadcast): "everyone on *my* link", never forwarded by routers. This is what DHCP Discover uses (Chapter 35) because the sender doesn't even have an address yet.
- **`192.168.1.255`** (directed broadcast): "everyone on that subnet". Routers normally **do not** forward directed broadcasts from outside (disabled by default for security, e.g. Smurf-attack prevention).
- At Layer 2, broadcast is MAC `ff:ff:ff:ff:ff:ff` (Chapter 31). Any IPv4 broadcast is sent to that MAC.

---

## 7. Fast mental calculation (no binary needed)

Take the **interesting octet**, the one where the mask is neither 255 nor 0. Compute the **block size** = `256 − mask octet`. Subnets start at multiples of the block size.

Example: `192.168.1.77/26` → mask `255.255.255.192`, interesting octet is the 4th, block size = 256 − 192 = **64**.

```
Subnet starts: 0, 64, 128, 192
77 lies in 64..127  →  network 192.168.1.64
broadcast = next network − 1 = 192.168.1.127
usable = 192.168.1.65 – 192.168.1.126  (62 hosts)
```

Harder example: `172.16.50.75` with `255.255.240.0` (/20):

```
interesting octet = the 3rd (240); block size = 256 − 240 = 16
3rd-octet subnet starts: 0, 16, 32, 48, 64 …   50 lies in 48..63
network   = 172.16.48.0
broadcast = 172.16.63.255       (last of that block; the 4th octet is all 1s)
usable    = 172.16.48.1 – 172.16.63.254   (2^12 − 2 = 4,094 hosts)
```
Verify:

```bash
python3 -c "
import ipaddress as i
n=i.ip_interface('172.16.50.75/20').network
print(n, n.netmask, n.broadcast_address, n.num_addresses-2)"
# 172.16.48.0/20 255.255.240.0 172.16.63.255 4094
```

Tools: `ipcalc 172.16.50.75/20` (package `ipcalc`), `sipcalc`, or online calculators. Learn the method first, then use tools to check.

---

## 8. How a host uses the mask on every packet

Configuration of PC-A: `192.168.1.10`, mask `255.255.255.0`, gateway `192.168.1.1`.

**Sending to `192.168.1.77`:**
```
destination AND mask = 192.168.1.0 ; my network = 192.168.1.0  → SAME → LOCAL
→ ARP for 192.168.1.77's MAC → send the frame directly to it.
```
**Sending to `8.8.8.8`:**
```
8.8.8.8 AND 255.255.255.0 = 8.8.8.0 ≠ 192.168.1.0 → REMOTE
→ ARP for the gateway 192.168.1.1 → send the frame to the router's MAC (destination IP stays 8.8.8.8).
```
Modern operating systems implement this with the **routing table** (Chapter 42):

```bash
ip route
# default via 192.168.1.1 dev eth0            ← everything else → gateway
# 192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.10   ← my own network: direct delivery
```
The second line is created automatically from your address+mask. The mask thus *creates the routing table's first entry*.

**Same destination, different mask, different decision.** Destination `192.168.2.5`:

| Host mask | `192.168.2.5` is… | Action |
|---|---|---|
| `/24` (`255.255.255.0`) | outside `192.168.1.0/24` | send to gateway |
| `/16` (`255.255.0.0`) | inside `192.168.0.0/16` | ARP directly (which fails if the host really sits behind a router!) |

---

## 9. Why DHCP Discover can work before you have an address

A machine without configuration doesn't know a mask or a gateway (that's the point of DHCP). So it uses **source `0.0.0.0`** and **destination `255.255.255.255`** (limited broadcast) and Layer-2 broadcast MAC `ff:ff:ff:ff:ff:ff`. Every device on the link receives it; the DHCP server (a router or server) answers. Chapters 35–37 dissect this fully.

---

## 10. Subnetting: splitting one network into smaller ones

**Subnetting** = borrowing bits from the host part to create extra network bits, splitting one network into several smaller ones.

Example: split `192.168.1.0/24` into **4** equal subnets. 4 = 2², so borrow **2 bits** → `/26`.

```
192.168.1.0/24    network bits 24 | host bits 8
→ /26:            network bits 26 | host bits 6   (the 2 borrowed bits count 00, 01, 10, 11)

Subnet         Network          Usable range               Broadcast
#1 (00)        192.168.1.0/26   192.168.1.1  – .62         192.168.1.63
#2 (01)        192.168.1.64/26  192.168.1.65 – .126        192.168.1.127
#3 (10)        192.168.1.128/26 192.168.1.129 – .190       192.168.1.191
#4 (11)        192.168.1.192/26 192.168.1.193 – .254       192.168.1.255
```
Each subnet has 62 usable hosts (4 × 62 = 248 vs 254 before: the cost of extra network/broadcast addresses).

```bash
python3 -c "
import ipaddress as i
for s in i.ip_network('192.168.1.0/24').subnets(prefixlen_diff=2):
    print(s, s[1], s[-2], s.broadcast_address)"
```

**Why split networks?**
- **Smaller broadcast domains**: less broadcast noise (ARP, DHCP).
- **Security**: separate departments/guests/IoT/servers, and control traffic between them with firewall rules at the router.
- **Organization**: an address tells you the department, site or function.
- **Fault isolation** and simpler troubleshooting.
- **Efficient use of addresses**: match subnet sizes to needs.

### VLSM: subnets of different sizes
You are not forced to make equal parts. **Variable Length Subnet Masking** carves the block into pieces sized to need. Example, from `192.168.10.0/24`, sizes needed: 100 hosts, 50 hosts, 20 hosts, 2 (a router link).

Allocate **largest first** to keep the blocks aligned:

| Need | Smallest fit | Subnet | Range | Broadcast |
|---|---|---|---|---|
| 100 hosts | /25 (126) | `192.168.10.0/25` | .1–.126 | .127 |
| 50 hosts | /26 (62) | `192.168.10.128/26` | .129–.190 | .191 |
| 20 hosts | /27 (30) | `192.168.10.192/27` | .193–.222 | .223 |
| 2 (link) | /30 (2) | `192.168.10.224/30` | .225–.226 | .227 |

Left free: `192.168.10.228 – .255` for growth. Plan for growth (a common rule: 30–50% spare).

---

## 11. Choosing a subnet size in practice

| Situation | Typical choice |
|---|---|
| Home/small office | `/24` (254 hosts) |
| A department floor | `/24` or `/23` |
| Data-center server VLAN | `/24`–`/26` |
| Wi-Fi with many phones/laptops | `/22` or `/21` (bigger DHCP pool, lower churn issues) |
| Router-to-router link | `/31` (or `/30`) |
| Loopback/host route | `/32` |

Docker and cloud sensible defaults: Docker's `docker0` is `172.17.0.0/16`, user-defined bridge networks take `172.18.0.0/16`, `172.19.0.0/16` and so on (configurable via `default-address-pools`); an AWS VPC is commonly `10.0.0.0/16` sliced into `/24` subnets per availability zone. Overlapping ranges (e.g. your VPN using the same `172.17.0.0/16` as Docker) cause confusing failures.

---

## 12. About classes (A, B, C)

In the 1980s addresses were "classful": the first bits fixed the mask automatically: Class A (`1–126`, /8), Class B (`128–191`, /16), Class C (`192–223`, /24). Classes were wasteful (a company needing 300 hosts had to take a /16 = 65,534 addresses). In 1993 **CIDR** removed them (Chapter 34); **today the mask is always explicit** and "class" only survives in old exam questions and terms like "a /24 (Class C-sized)". Never assume a mask from the first octet.

---

## 13. Lab: watch the mask decide

Uses Linux network namespaces (Chapter 32 introduced them). Nothing touches your real network.

```bash
sudo ip netns add a; sudo ip netns add b
sudo ip link add a0 type veth peer name b0
sudo ip link set a0 netns a; sudo ip link set b0 netns b
sudo ip netns exec a ip link set a0 up; sudo ip netns exec b ip link set b0 up

# Same wire, same /24 → they can talk
sudo ip netns exec a ip addr add 192.168.1.10/24 dev a0
sudo ip netns exec b ip addr add 192.168.1.20/24 dev b0
sudo ip netns exec a ping -c 1 192.168.1.20                # works
sudo ip netns exec a ip route                              # a route for 192.168.1.0/24 exists because of the mask

# Now break it with a mask that puts b in a different network
sudo ip netns exec b ip addr flush dev b0
sudo ip netns exec b ip addr add 192.168.1.20/26 dev b0     # b thinks its network is 192.168.1.0/26 (.0–.63)
sudo ip netns exec a ip addr flush dev a0
sudo ip netns exec a ip addr add 192.168.1.100/26 dev a0    # a thinks .64–.127
sudo ip netns exec a ping -c 1 -W 1 192.168.1.20            # "Network is unreachable": a sees .20 as a DIFFERENT network, and has no gateway
sudo ip netns exec a ip route

# Asymmetric mismatch: one side /24, the other /26
sudo ip netns exec a ip addr flush dev a0
sudo ip netns exec a ip addr add 192.168.1.100/24 dev a0    # a is /24: thinks .20 is local → ARPs → b receives the request
sudo ip netns exec a ping -c 2 -W 1 192.168.1.20            # request arrives, b's REPLY... b thinks .100 is remote → no route → one-way failure
sudo ip netns exec b ip route
sudo ip netns del a; sudo ip netns del b
```
Lesson: **mask mismatches produce one-way or partial connectivity**, one of the most confusing failure types.

---

## 14. Troubleshooting

| Symptom | Likely mask-related cause |
|---|---|
| Can reach some local hosts but not others | Mask too long (e.g. `/26` set where the network is `/24`) so hosts think their neighbors are remote |
| "Network is unreachable" when pinging a neighbor | Different subnets and no route: mask mismatch |
| One-way connectivity (A→B works but replies are lost) | A and B disagree about the mask |
| Can ping internet by IP but not local devices | Gateway route works but the local subnet is misdefined |
| Two devices can't talk although on the same switch | Different networks configured; they need a router (or the same subnet) |
| Host reaches strange machines | Mask too short (`/8` instead of `/24`) claims a huge range is local → ARP for non-existent neighbors |

```bash
ip -br addr              # address and prefix per interface
ip route                 # the derived routes
ip route get 192.168.1.77   # which route/interface WOULD be used, and the source address
ipcalc 192.168.1.77/26   # what network am I in?
ping -c 1 <gw>; arp -n / ip neigh    # are local and gateway ARP entries resolving?
```
Windows: `ipconfig /all`, `route print`, `Test-NetConnection`. 

---

## 15. Common misconceptions

| Misconception | Reality |
|---|---|
| "The first octet decides the mask" | Only in obsolete classful addressing; masks are explicit |
| "A /24 has 255 hosts" | 254 usable (256 − network − broadcast) |
| "The gateway is always `.1`" | Convention only; it can be any usable address |
| "The mask is a second address" | It's a bit pattern: contiguous 1s then 0s |
| "Subnetting is only a math exercise" | It sets broadcast domains, security boundaries and routing |
| "Hosts on different subnets on one switch can talk" | Not without a router (layer 3) |
| "Routers forward broadcasts" | Not by default |
| "/31 and /32 are invalid" | Valid for point-to-point links and host routes |
| "Bigger subnet is always better" | Large broadcast domains create noise and blast radius |

---

## 16. Summary

- An IPv4 address = **network bits + host bits**; the **subnet mask** (`255.255.255.0` = `/24`) marks the boundary.
- **Address AND mask = network address**. Same result → same network → deliver directly (ARP); otherwise send to the **default gateway**.
- In each subnet, host-bits-all-0 = **network address**, all-1 = **broadcast**; usable hosts = `2^h − 2`.
- **Block size = 256 − mask octet** is the fast way to find networks by hand.
- **Subnetting** borrows host bits to make more, smaller networks; **VLSM** lets the sizes differ.
- Wrong masks cause one-way, partial or "network unreachable" failures.

---

## 17. Check your understanding

1. What does the subnet mask tell a host?
2. Compute network, broadcast, first/last usable and host count for `10.20.30.200/27`.
3. Are `192.168.5.130/25` and `192.168.5.100/25` on the same network? What if the masks were `/24`?
4. Why is `255.255.255.100` not a valid mask?
5. Split `10.0.0.0/22` into four equal subnets.
6. A host has `192.168.1.50/16` but sits behind a router on a real `/24`. What symptom appears?
7. Which broadcast does DHCP Discover use and why?
8. How many usable hosts in `/29`? In `/31`?

<details>
<summary>Answers</summary>

1. Which bits of an address identify the network vs the host, so it can decide local vs remote delivery.
2. /27 → block size 32; `.200` lies in 192..223. Network `10.20.30.192`, broadcast `10.20.30.223`, usable `.193–.222`, 30 hosts.
3. `.130` is in `.128–.255`, `.100` in `.0–.127`: **different** networks with /25. With /24 both are `192.168.5.0/24`: same network.
4. 100 = `01100100`, not a contiguous run of 1s followed by 0s.
5. `10.0.0.0/24`, `10.0.1.0/24`, `10.0.2.0/24`, `10.0.3.0/24`.
6. It thinks the whole `192.168.0.0/16` is local, so for destinations like `192.168.2.5` it ARPs directly instead of using the gateway: no reply, unreachable.
7. Limited broadcast `255.255.255.255` (MAC `ff:ff:ff:ff:ff:ff`), because the client has no address, mask or gateway yet.
8. `/29` → 6 usable; `/31` → 2 (point-to-point, RFC 3021).
</details>

**Practice**

1. For each of `172.31.100.9/20`, `192.168.77.200/28`, `10.1.1.1/30`, write network, broadcast, range, host count by hand; verify with Python.
2. Design a plan for `192.168.20.0/24` with VLSM: 60 hosts, 25 hosts, 10 hosts, and two 2-host links.
3. Read `ip route` on your machine and explain each entry using the mask.
4. Run the namespace lab and add a third namespace with mask `/25` to reproduce a one-way failure; capture with `tcpdump -nn -e -i a0 arp or icmp`.
5. Check which subnets Docker uses on your machine (`docker network ls`, `docker network inspect bridge | grep -i subnet`) and whether they overlap with your LAN/VPN.

---

**Next:** [Chapter 34 – CIDR, Subnet, and Subnet Mask: Understanding the Differences](34_cidr_subnet_subnet_mask_differences_in_details.md)
