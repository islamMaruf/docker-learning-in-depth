# Chapter 34: CIDR, Subnet and Subnet Mask: Understanding the Differences

> **In one sentence:** A **subnet** is the *thing* (a slice of address space that forms one network), a **subnet mask** is a *tool* (a 32-bit pattern that marks the network/host boundary) and **CIDR** is a *notation and addressing scheme* (`192.168.1.0/24`, prefix length instead of classes, which also allows aggregating routes); people mix the three words up constantly, and this chapter untangles them.

**Level:** 🟡 Intermediate · **Reading time:** ~50 minutes

**Prerequisite:** [Chapter 33](33_subnetting_and_subnet_masks_in_details.md).

---

## What you will learn

- The exact meaning of **subnet**, **subnet mask** and **CIDR**, and when to use each word
- **Why CIDR was invented** (classful waste and routing-table explosion) and what it changed
- Reading and writing **CIDR blocks**, converting to masks and back
- **Route aggregation (supernetting)**: how many prefixes become one
- **Wildcard masks**, **`/32`**, **`0.0.0.0/0`**: the shapes you see in firewalls, ACLs, security groups and routing tables
- How the three concepts appear in **real tools**: Linux, Windows, cloud, Docker, Kubernetes
- A full worked planning example, and a Python toolkit to check your work

---

## 1. Three words, three roles

| Term | What it is | Kind of thing | Example |
|---|---|---|---|
| **Subnet** | A range of addresses forming one logical IP network | A **concept / object** | "the office subnet" |
| **Subnet mask** | 32-bit pattern: 1s for network bits, 0s for host bits | A **value / tool** | `255.255.255.0` |
| **CIDR** | Classless Inter-Domain Routing: addressing without classes, written `address/prefix-length` | A **notation + scheme** | `192.168.1.0/24` |

Analogy: a **subnet** is a neighborhood; the **subnet mask** is the map's boundary line that says which houses belong to it; **CIDR** is the shorthand you write on the envelope, "Oak Quarter/24".

Correct sentences:
- "Create a **subnet** for the servers." (the network)
- "The **mask** is 255.255.255.0." (the tool/value)
- "Allow the **CIDR** 10.0.5.0/24 in the firewall." (the notation for that range)

Also correct: "`/24` is the **prefix length**", the number of leading 1s in the mask.

They describe **the same information three ways**:

```
Subnet (idea):   "the network containing 192.168.1.77, with 8 host bits"
CIDR (notation): 192.168.1.0/24
Mask (value):    255.255.255.0
```

---

## 2. Why CIDR exists: a short history

### Classful addressing (before 1993)
The first octet decided the mask automatically:

| Class | First octet | Fixed mask | Hosts per network |
|---|---|---|---|
| A | 1–126 | /8 | 16,777,214 |
| B | 128–191 | /16 | 65,534 |
| C | 192–223 | /24 | 254 |
| (D multicast 224–239; E reserved 240–255) | | | |

Two big problems in the early 1990s:

1. **Waste.** An organization needing 500 addresses was too big for a Class C (254) so it received a **Class B** (65,534), wasting ~99%. IPv4 space drained fast.
2. **Routing table explosion.** Every Class C was a separate routing entry on backbone routers; tables grew exponentially and routers ran out of memory.

### CIDR (RFC 1519, 1993; updated by RFC 4632)
- **Any prefix length** from `/0` to `/32`: a company needing 500 hosts gets a **`/23`** (510 hosts).
- **Aggregation:** contiguous blocks can be advertised as **one route**, shrinking the global table.
- Classes disappear as a routing concept.

CIDR slowed IPv4 exhaustion and kept the global routing table workable (still roughly 1 million IPv4 prefixes today, versus hundreds of millions of networks if there were no aggregation). IPv4 free pools were still eventually exhausted (IANA in 2011; regional registries after), which is why NAT and IPv6 matter.

---

## 3. CIDR notation

```
   192.168.1.0 / 24
   └─address─┘   └─ prefix length: number of leading network bits
```
- **prefix length** `n` → mask = `n` ones followed by `32−n` zeros.
- Network part: `n` bits; host part: `32−n` bits.
- Block size: `2^(32−n)` addresses.

### Convert prefix ↔ mask
Method: fill octets left to right, 8 bits at a time.

| Prefix | Mask | Working |
|---|---|---|
| /8 | 255.0.0.0 | 8 ones |
| /12 | 255.240.0.0 | 8 + 4 ones → 11110000 = 240 |
| /16 | 255.255.0.0 | 16 ones |
| /20 | 255.255.240.0 | 16 + 4 |
| /22 | 255.255.252.0 | 16 + 6 → 11111100 = 252 |
| /23 | 255.255.254.0 | 16 + 7 |
| /24 | 255.255.255.0 | 24 |
| /26 | 255.255.255.192 | 24 + 2 → 11000000 = 192 |
| /28 | 255.255.255.240 | 24 + 4 |
| /30 | 255.255.255.252 | 24 + 6 |
| /32 | 255.255.255.255 | all ones |

Reverse: count the 1s. `255.255.248.0` → 8 + 8 + 5 (248 = 11111000) = **/21**.

### The address in front of the slash
- `192.168.1.0/24`: a **network** (host bits zero), a clean CIDR *block*.
- `192.168.1.77/24`: an **interface** (host + its network), a common way of writing an *address with its prefix*, exactly what `ip addr` shows: `inet 192.168.1.77/24`.

A block must be **aligned**: `192.168.1.64/26` is valid (64 is a multiple of the block size 64); `192.168.1.70/26` is not a proper network address (the tools will complain or normalize to `.64/26`).

### Block sizes at a glance
| Prefix | Addresses | Typical use |
|---|---|---|
| /8 | 16,777,216 | Whole `10.0.0.0/8` private block |
| /12 | 1,048,576 | `172.16.0.0/12` private |
| /16 | 65,536 | `192.168.0.0/16` private; `docker0` |
| /20–/22 | 4096–1024 | Big cloud/office subnets |
| /24 | 256 | Standard LAN |
| /28 | 16 | Small subnet |
| /30–/31 | 4–2 | Links |
| /32 | 1 | One host |
| /0 | all 4,294,967,296 | **Everything** (`0.0.0.0/0`, "default route") |

---

## 4. Subnets in more depth

A **subnet** is created when you decide "this group of devices is one IP network": choose a prefix, assign hosts, give them a gateway. You can carve subnets out of a larger block (subnetting) or combine several into a bigger block (supernetting).

### Why have many subnets?
1. **Broadcast containment:** broadcasts (ARP, DHCP, mDNS) only reach the local subnet.
2. **Security boundaries:** rules between subnets at a router/firewall: HR ≠ servers ≠ guests ≠ IoT.
3. **Organization:** predictable addressing per site/function.
4. **Performance/failure isolation:** a loop or flood affects one subnet only.
5. **Cloud design:** public/private/database subnets, per availability zone.

### How many subnets and hosts when you split?
Borrowing `b` bits from the host part gives `2^b` subnets, each with `2^(h−b) − 2` usable hosts.

Example: `10.0.0.0/8` split into `/10` blocks → `2^2 = 4` subnets (`10.0.0.0/10`, `10.64.0.0/10`, `10.128.0.0/10`, `10.192.0.0/10`).

### Worked example: split `172.16.0.0/16` into subnets of at least 500 hosts
500 hosts → need 9 host bits (2^9 − 2 = 510) → prefix `/23`. Borrowed bits: 23 − 16 = 7 → `2^7 = 128` subnets:
`172.16.0.0/23`, `172.16.2.0/23`, `172.16.4.0/23`, … `172.16.254.0/23`.

```bash
python3 -c "
import ipaddress as i
s=list(i.ip_network('172.16.0.0/16').subnets(new_prefix=23))
print(len(s), s[0], s[1], s[-1], s[0].num_addresses-2)"
# 128 172.16.0.0/23 172.16.2.0/23 172.16.254.0/23 510
```

---

## 5. Subnet masks in tools

The same fact appears in whatever form each tool prefers:

| Tool | You will see |
|---|---|
| `ip addr` (Linux) | `inet 192.168.1.77/24` (CIDR) |
| `ifconfig` (legacy) | `netmask 255.255.255.0` |
| Windows `ipconfig` | `Subnet Mask . . . : 255.255.255.0` |
| Windows GUI | prefix length or mask |
| Cloud (AWS/Azure/GCP) | `10.0.1.0/24` (CIDR) |
| Cisco (classic IOS) | `ip address 192.168.1.1 255.255.255.0`; ACLs use *wildcard masks* |
| Docker | `docker network create --subnet 172.30.0.0/16 mynet` |
| Kubernetes | Pod CIDR `10.244.0.0/16`, Service CIDR `10.96.0.0/12` |
| DHCP server (dnsmasq/ISC) | `subnet 192.168.1.0 netmask 255.255.255.0 { range … }` |

Linux examples of setting it (temporary):

```bash
sudo ip addr add 192.168.1.77/24 dev eth0                   # CIDR form
sudo ifconfig eth0 192.168.1.77 netmask 255.255.255.0       # legacy tool, dotted mask
```

### Wildcard masks (the inverse)
Some ACL syntaxes (Cisco, OSPF `network` statements) use the **inverse** of the mask: 0-bits mean "must match", 1-bits mean "don't care".

```
/24 mask     255.255.255.0
wildcard     0.0.0.255         (subtract from 255.255.255.255)
```
`/26` → mask `255.255.255.192` → wildcard `0.0.0.63`. Not to be confused with the subnet mask.

---

## 6. Aggregation (supernetting): CIDR's superpower

Contiguous, aligned blocks can be summarized into one bigger prefix:

```
192.168.0.0/24 + 192.168.1.0/24 + 192.168.2.0/24 + 192.168.3.0/24  →  192.168.0.0/22
```
Why it works:

```
192.168.0.0  = 11000000.10101000.000000|00.00000000
192.168.1.0  = 11000000.10101000.000000|01.00000000
192.168.2.0  = 11000000.10101000.000000|10.00000000
192.168.3.0  = 11000000.10101000.000000|11.00000000
                                 first 22 bits are shared  →  /22
```
Rules: the blocks must be **contiguous**, and the group **must start at a boundary aligned to its size**. `192.168.1.0/24 + 192.168.2.0/24` **cannot** merge into one `/23` because `192.168.1.0` isn't a multiple of 512 (a `/23` starting at `192.168.1.0` isn't a valid block).

Benefits:
- **Smaller routing tables** at the ISP/backbone: one route instead of four.
- **Stability:** changes inside the block don't ripple outward.
- **Firewall rules and cloud security groups:** one CIDR instead of many.

```bash
python3 - <<'E'
import ipaddress as i
nets=[i.ip_network(f"192.168.{k}.0/24") for k in range(4)]
print(list(i.collapse_addresses(nets)))                        # [192.168.0.0/22]
print(list(i.collapse_addresses([i.ip_network("10.0.0.0/24"),
      i.ip_network("10.0.1.0/24"), i.ip_network("10.0.2.0/24")])))  # [10.0.0.0/23, 10.0.2.0/24]  (3 don't fit one block)
E
```
`0.0.0.0/0` is the ultimate aggregate ("the whole Internet"), used as the **default route**. A **longest-prefix match** picks the most specific route (Chapter 42): `10.0.5.0/24` beats `10.0.0.0/8` beats `0.0.0.0/0`.

### Special CIDR values you will meet constantly
| CIDR | Meaning |
|---|---|
| `0.0.0.0/0` | All IPv4 addresses ("anywhere"; default route; open-to-world in a security group) |
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | Private ranges (RFC 1918) |
| `100.64.0.0/10` | Carrier-grade NAT shared space |
| `169.254.0.0/16` | Link-local (Chapter 32) |
| `127.0.0.0/8` | Loopback |
| `203.0.113.0/24`, `198.51.100.0/24`, `192.0.2.0/24` | Documentation-only ranges (safe for examples) |
| `x.x.x.x/32` | Exactly one address ("allow just this host") |

---

## 7. Comparing the three

| | **Subnet** | **Subnet mask** | **CIDR** |
|---|---|---|---|
| Nature | A network / address range | A 32-bit bit-pattern | A notation and allocation scheme |
| Answers | *Which group of addresses?* | *Where is the boundary?* | *How do I write/allocate/aggregate it?* |
| Example | The `192.168.1.x` LAN | `255.255.255.0` | `192.168.1.0/24` |
| Used by | Designers, admins | Host network stack, routers | Cloud consoles, firewalls, routing, docs |
| Created by | Subnetting | Derived from prefix | Choosing the prefix |
| Related to | Broadcast domain | AND with an address | Aggregation, routing tables |
| Computation | Range, hosts | AND, wildcard | Prefix ↔ mask |

They are **not** rivals: every subnet has a mask; every mask has a prefix; the CIDR block *names* the subnet.

---

## 8. Complete worked example: a small office

**Requirements:** sites and hosts needed (with growth): Staff 100, Guests 50, Servers 20, VoIP 30, Management/link 2. Available: `192.168.10.0/24`.

**Step 1: plan (subnets).** Round each need up to a power-of-two block and **count the total before assigning anything**:

```
Staff   100 hosts → /25 (126 usable) = 128 addresses
Guests   50 hosts → /26 (62)         =  64
VoIP     30 hosts → /26 (62)         =  64   (a /27 gives exactly 30: no growth)
Servers  20 hosts → /27 (30)         =  32
Link      2 hosts → /30 (2)          =   4
                                total  292  > 256
```
The requirement does not fit in a `/24`. Options: choose a **`/23`** (512 addresses), or shrink some blocks. We take `192.168.10.0/23` and, to keep things tidy, size VoIP as a `/27` (30 hosts). Allocate **largest first** so every block stays aligned:

With `192.168.10.0/23`:

| Subnet | CIDR | Range | Broadcast |
|---|---|---|---|
| Staff | `192.168.10.0/25` | .10.1–.10.126 | .10.127 |
| Guests | `192.168.10.128/26` | .10.129–.10.190 | .10.191 |
| VoIP | `192.168.10.192/27` | .10.193–.10.222 | .10.223 |
| Servers | `192.168.10.224/27` | .10.225–.10.254 | .10.255 |
| Link | `192.168.11.0/30` | .11.1–.11.2 | .11.3 |
| Spare | `192.168.11.4` onward | growth | |

**Step 2: allocate (CIDR notation)**, the table above is your allocation record.

**Step 3: masks**, for configuration on legacy devices: `/25` = `255.255.255.128`, `/26` = `255.255.255.192`, `/27` = `255.255.255.224`, `/30` = `255.255.255.252`.

**Step 4: configure a Linux server in the Servers subnet**

```bash
sudo ip addr add 192.168.10.230/27 dev eth0
sudo ip route add default via 192.168.10.225
```
**Step 5: document** in a table, spreadsheet or IPAM tool (NetBox, phpIPAM): CIDR, purpose, VLAN ID, gateway, DHCP range, owner.

**Step 6: verify**

```bash
ip route get 192.168.10.100      # different subnet → via gateway
ping -c 1 192.168.10.226         # same subnet → direct (ARP)
ip neigh show                    # neighbors
```
Verify the design programmatically (overlap checks!):

```python
import ipaddress as i
plan = {"staff":"192.168.10.0/25","guests":"192.168.10.128/26","voip":"192.168.10.192/27",
        "servers":"192.168.10.224/27","link":"192.168.11.0/30"}
nets = {k:i.ip_network(v) for k,v in plan.items()}
for a in nets:
    for b in nets:
        if a<b and nets[a].overlaps(nets[b]): print("OVERLAP",a,b)
print({k:v.num_addresses-2 for k,v in nets.items()})
```

---

## 9. CIDR in the wild

**Cloud security groups**

```
Allow SSH  from 203.0.113.25/32      ← only my office IP
Allow HTTPS from 0.0.0.0/0            ← the whole Internet
Allow DB   from 10.0.1.0/24           ← the app subnet only
```
**Docker**

```bash
docker network create --subnet 172.30.0.0/24 --gateway 172.30.0.1 appnet
docker run -d --network appnet --ip 172.30.0.10 --name web nginx
docker network inspect appnet | grep -A3 IPAM
```
The daemon's pools are configurable in `/etc/docker/daemon.json`:
`{"default-address-pools":[{"base":"10.200.0.0/16","size":24}]}` → each new network gets a `/24` from `10.200.0.0/16`. Change it when Docker's `172.x` ranges collide with your VPN or corporate network.

**Kubernetes:** `kubeadm init --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12`. Pod, Service and node ranges must **not overlap** with each other or your LAN.

**Routing:** `ip route add 10.5.0.0/16 via 192.168.1.254` = "to reach any address whose first 16 bits are 10.5, go via this gateway".

---

## 10. Common mistakes

| Mistake | Fix |
|---|---|
| Writing an unaligned block (`10.0.0.130/25` as a network) | Network addresses have host bits zero: `10.0.0.128/25` |
| Assuming subnets in a plan don't overlap | Verify with `ip_network().overlaps()` |
| Choosing subnet sizes without growth room | Plan for 30–100% growth |
| Overlapping ranges between VPN/Docker/Kubernetes and the LAN | Pick distinct private ranges early |
| Treating `/24` as always right | Match size to need: `/28`–`/20` all exist |
| Mixing up the mask and the wildcard | Wildcard is the inverse |
| Using `0.0.0.0/0` in a security rule accidentally | Means the entire Internet |
| Summarizing non-adjacent or unaligned blocks | Only aligned contiguous blocks aggregate |
| Saying "a /24 subnet mask" | The mask is `255.255.255.0`; `/24` is the prefix length. (Casual speech tolerates it.) |

---

## 11. Summary

- **Subnet** = a network (range of addresses), **mask** = the boundary pattern (`255.255.255.0`), **CIDR** = the `address/prefix` notation and classless allocation scheme (`192.168.1.0/24`); three views of the same thing.
- CIDR ended classful waste and made **route aggregation** possible.
- Block size is `2^(32−prefix)`; usable hosts subtract 2 (except /31, /32); blocks must be **aligned**.
- **Longest-prefix match** decides between overlapping routes; `0.0.0.0/0` is the default; `/32` is one host.
- Plan by **counting first**, allocate **largest to smallest**, leave room, avoid overlaps, document.

---

## 12. Check your understanding

1. State in one sentence the difference between a subnet, a subnet mask and CIDR.
2. What are the two problems classful addressing caused, and how did CIDR help?
3. Convert `/19` to a mask, and `255.255.255.224` to a prefix.
4. Can `10.1.2.0/23` be a valid network address? What about `10.1.3.0/23`?
5. Aggregate `172.16.8.0/24`, `172.16.9.0/24`, `172.16.10.0/24`, `172.16.11.0/24`.
6. What is the wildcard mask of `/27`?
7. What does `203.0.113.7/32` mean? And `0.0.0.0/0`?
8. Which route wins for destination `10.5.5.5` among `10.0.0.0/8`, `10.5.0.0/16`, `0.0.0.0/0`?

<details>
<summary>Answers</summary>

1. A subnet is the network itself; the mask is the bit pattern marking network vs host bits; CIDR is the prefix notation/classless scheme that names it.
2. Address waste and huge routing tables; CIDR allows any prefix length and route aggregation.
3. `/19` = `255.255.224.0`; `255.255.255.224` = `/27`.
4. `10.1.2.0/23`: yes (2 is even, aligned to 512). `10.1.3.0/23`: no, that's inside `10.1.2.0/23`.
5. `172.16.8.0/22`.
6. `0.0.0.31`.
7. Exactly one host; all IPv4 addresses (default route / anywhere).
8. `10.5.0.0/16` (longest matching prefix).
</details>

**Practice**

1. Plan `10.20.0.0/16` for three sites, each with 4 subnets (200, 100, 50, 10 hosts). Use Python to generate and check for overlaps.
2. Take five real CIDR blocks from your cloud console, Docker networks or Kubernetes config; convert each to mask, range, and size.
3. Find two subnets in your environment that could be aggregated, and two that can't (and explain why).
4. Change Docker's `default-address-pools` on a test machine and confirm new networks follow it.
5. Write a script that reads a list of CIDRs and reports overlaps and total addresses.

---

**Next:** [Chapter 35 – DHCP Discover Deep Dive](35_dhcp_discover_deep_dive_in_details.md)
