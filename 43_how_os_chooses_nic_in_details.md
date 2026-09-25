# Chapter 43: How the OS Chooses a NIC: The Complete Algorithm

> **In one sentence:** For every outgoing packet the OS takes the **destination IP**, finds every routing-table entry it **matches** (`destination AND netmask == route network`), keeps the one with the **longest prefix** (ties broken by **lowest metric**), and sends the packet out that entry's **interface** to that entry's **gateway** (or directly to the destination if the route is on-link), using that interface's address as the **source IP**.

**Level:** 🟡 Intermediate → 🔴 Advanced · **Reading time:** ~55 minutes

**Prerequisite:** [Chapter 42](42_routing_table_in_details.md). This chapter is the payoff of Chapters 33–42.

---

## What you will learn

- The **complete algorithm**, step by step, with a runnable **Python implementation** you can test against real `ip route get`
- **Longest prefix match** vs **metric**: exactly what wins and why
- What the **source IP** becomes, and what **ARP** targets (destination or gateway?)
- The identity **subnet mask = netmask = Genmask = prefix length**
- Where the algorithm gets **more complex in real life**: policy routing (`ip rule`), multiple tables, **bound sockets**, **reverse-path filtering**, **ARP flux**, **ECMP**, **weak vs strong host model**
- Common mistakes, complete worked examples, and a **lab** you can run

---

## 1. The setup

The scenario from Chapters 40–42: a computer with three NICs on three networks.

```
                 ┌── NIC1 eth0 192.168.1.10/24 ── LAN1 ── router 192.168.1.1 ── Internet
 Computer ───────┼── NIC2 eth1 192.168.2.10/24 ── LAN2 ── router 192.168.2.1
                 └── NIC3 eth2 192.168.3.10/24 ── LAN3 ── router 192.168.3.1
```
Routing table:

```
Destination      Netmask          Gateway        Iface   Metric
0.0.0.0          0.0.0.0          192.168.1.1    eth0    100      ← default route
192.168.1.0      255.255.255.0    (on-link)      eth0    100
192.168.2.0      255.255.255.0    (on-link)      eth1    200
192.168.3.0      255.255.255.0    (on-link)      eth2    300
127.0.0.0        255.0.0.0        (on-link)      lo      0
```
An application does: `connect(("192.168.3.10", 80))` or `ping 8.8.8.8`. Layers 7 and 4 don't know about NICs: the app names a destination and TCP/UDP builds a segment. **The choice is made at Layer 3.**

---

## 2. Terminology cleanup: mask, netmask, genmask

| Term | Where you see it | Meaning |
|---|---|---|
| **Subnet mask** | Windows `ipconfig`, textbooks, DHCP option 1 | 32-bit mask of network bits |
| **Netmask** | Linux `ifconfig` / `netmask 255.255.255.0`, `route -n` (as **Genmask**), macOS | Same thing |
| **Genmask** | `route -n` column | Same thing, historical name ("generality mask") |
| **Prefix length** | `/24` in `ip addr` / `ip route` | Same thing, counted in bits |

They are **all the same value** in different notation: `255.255.255.0` = `/24` (Chapters 33–34). Each interface has its **own** mask (it comes from that interface's configuration/lease), and each **route** carries a mask too: for connected routes it is copied from the interface; for static routes you write it; for the default route it's `0.0.0.0` (`/0`).

---

## 3. The algorithm

For a packet with destination **D** (and, optionally, a source address S and output-interface constraint if the socket was bound):

```
1. LOCAL?    Is D one of my own addresses (or loopback/broadcast)?        → deliver locally (table "local")
2. CANDIDATES: for each route R in the table:
                 if (D AND R.mask) == R.network      → R is a candidate
3. NONE?     No candidates                            → error "Network is unreachable"
4. SPECIFIC: keep the candidate(s) with the longest prefix (most 1-bits in the mask)
5. TIE-BREAK: among equal prefix lengths, choose the lowest metric
              (if still tied: first installed, or ECMP hash if multipath is configured)
6. NEXT HOP: if the chosen route has a gateway G → next hop = G
             else (on-link)                       → next hop = D itself
7. INTERFACE: out_if = R.interface
8. SOURCE IP: if the socket isn't bound to an address, use the "preferred source"
              of the route (R.src) or the primary address of out_if
9. ARP:      resolve the MAC of the next hop on out_if (cache, else ARP request)
10. FRAME:   Ethernet header: dst = MAC(next hop), src = MAC(out_if); IP: dst = D (unchanged), src = from step 8
             then hand the frame to the NIC driver of out_if
```
Two crucial facts:

- **Step 6:** the IP destination is **always the final destination D**. Only the **MAC** target changes: the gateway's MAC when the route has a gateway, or D's own MAC when D is on-link.
- **Steps 4–5 order:** *prefix length first, metric second.* A `/24` with metric 300 beats a `/0` with metric 1.

---

## 4. Worked examples (using the table above)

### Example 1: `192.168.3.20` (a host on LAN3)

| Route | `192.168.3.20 & mask` | Equals route network? |
|---|---|---|
| `0.0.0.0/0` | `0.0.0.0` | ✅ candidate (prefix 0) |
| `192.168.1.0/24` | `192.168.3.0` | ✗ |
| `192.168.2.0/24` | `192.168.3.0` | ✗ |
| `192.168.3.0/24` | `192.168.3.0` | ✅ candidate (prefix 24) |
| `127.0.0.0/8` | `192.0.0.0` | ✗ |

Candidates: default (/0) and `192.168.3.0/24`. **Longest prefix → `192.168.3.0/24` → eth2**, on-link → ARP for **192.168.3.20** itself.

```
Packet:  IP  src 192.168.3.10 (eth2's address)   dst 192.168.3.20
Frame :  dst MAC = MAC of 192.168.3.20 (via ARP on LAN3), src MAC = eth2's MAC
```
(If D were *your own* address `192.168.3.10`, step 1 would catch it: delivered locally via loopback.)

### Example 2: `8.8.8.8`

Only the default route matches (`8.8.8.8 & 0.0.0.0 = 0.0.0.0`). Route: gateway `192.168.1.1`, **eth0**. ARP for the **gateway** `192.168.1.1`.
```
Packet:  IP  src 192.168.1.10   dst 8.8.8.8
Frame :  dst MAC = router's MAC (192.168.1.1)   src MAC = eth0's MAC
```
### Example 3: `192.168.2.10`
`192.168.2.0/24` (eth1) beats the default → out **eth1**, direct ARP for the destination; source IP `192.168.2.10` (eth1's address).

### Example 4: `10.1.2.3` with an extra static route
Add `10.0.0.0/8 via 192.168.2.254 dev eth1 metric 50`. Candidates: default and `10.0.0.0/8`. Longest prefix → `10.0.0.0/8` → **eth1**, ARP for the **gateway 192.168.2.254**. (The *metric* 50 vs 100 isn't even consulted, because prefix lengths differ.)

### Example 5: same prefix length, different metrics
Two routes to `172.16.0.0/16`: via eth0 (metric 100) and via eth1 (metric 200). Same prefix → **lowest metric** → eth0. If eth0 goes down (link lost), its route is removed and eth1's takes over: that's how **failover** works.

### Example 6: `127.0.0.1` and your own IP
`127.0.0.0/8` on `lo`: delivered internally. Same for `192.168.1.10` (your own address): the `local` table matches first and delivers via loopback; nothing is transmitted.

---

## 5. Python implementation (verify it yourself)

```python
import ipaddress as ip

# (network, gateway or None, interface, metric)
ROUTES = [
    ("0.0.0.0/0",       "192.168.1.1",   "eth0", 100),
    ("192.168.1.0/24",  None,            "eth0", 100),
    ("192.168.2.0/24",  None,            "eth1", 200),
    ("192.168.3.0/24",  None,            "eth2", 300),
    ("127.0.0.0/8",     None,            "lo",   0),
    ("10.0.0.0/8",      "192.168.2.254", "eth1", 50),
]
IF_ADDR = {"eth0": "192.168.1.10", "eth1": "192.168.2.10", "eth2": "192.168.3.10", "lo": "127.0.0.1"}

def route_lookup(dst):
    d = ip.ip_address(dst)
    cands = [(ip.ip_network(n), gw, ifc, m) for n, gw, ifc, m in ROUTES if d in ip.ip_network(n)]   # step 2
    if not cands:
        return None                                                                                    # step 3: unreachable
    cands.sort(key=lambda r: (-r[0].prefixlen, r[3]))       # step 4+5: longest prefix first, then lowest metric
    net, gw, ifc, metric = cands[0]
    next_hop = gw or dst                                    # step 6
    return {"route": str(net), "iface": ifc, "next_hop": next_hop, "src": IF_ADDR[ifc],
            "arp_for": next_hop, "candidates": [str(c[0]) for c in cands]}

for dst in ["192.168.3.20", "8.8.8.8", "192.168.2.77", "10.1.2.3", "127.0.0.1", "192.168.1.55"]:
    print(dst, route_lookup(dst))
```
Output (abridged; run it to see the full dictionaries):

```
192.168.3.20 -> route 192.168.3.0/24  iface eth2  next hop 192.168.3.20 (on-link)   candidates: 192.168.3.0/24, 0.0.0.0/0
8.8.8.8      -> route 0.0.0.0/0       iface eth0  next hop 192.168.1.1              candidates: 0.0.0.0/0
192.168.2.77 -> route 192.168.2.0/24  iface eth1  next hop 192.168.2.77 (on-link)
10.1.2.3     -> route 10.0.0.0/8      iface eth1  next hop 192.168.2.254            candidates: 10.0.0.0/8, 0.0.0.0/0
127.0.0.1    -> route 127.0.0.0/8     iface lo    (loopback)
192.168.1.55 -> route 192.168.1.0/24  iface eth0  next hop 192.168.1.55 (on-link)
```
Real kernels do the same thing with a much faster data structure (a compressed **LC-trie** for Linux IPv4 FIB), so lookups on tables with a million routes take well under a microsecond, and results are cached per socket/destination for speed. So the "routing decision" is **per packet in theory, but cached in practice**; changing the table invalidates the caches.

---

## 6. Why this algorithm works

| Property | Explanation |
|---|---|
| **Deterministic** | Same table + same destination = same answer, always |
| **Most specific wins** | Specific knowledge ("this /24 is on eth2") overrides general knowledge ("everything is via eth0"); you can override the default just by adding a more specific route |
| **Default route as a safety net** | `0.0.0.0/0` matches every address so nothing is left undecided, unless you deliberately have none (then "unreachable" tells you so) |
| **Efficient** | Just bitwise AND + compare; hardware/tries make it fast |
| **Composable** | The same rule works for a phone, a Docker container, and a backbone router; each hop only decides the *next* hop |
| **Robust** | Metrics + removal of routes on link loss give automatic failover |

---

## 7. Common mistakes and corrections

| Mistake | Correction |
|---|---|
| "The OS picks the **first** matching route" | It picks the **longest prefix**; the order in the table doesn't matter for that |
| "**Metric** decides which NIC" | Only between routes of **equal** prefix length |
| "Subnet mask differs from netmask" | Same thing: `/24` = `255.255.255.0` = Genmask |
| "The routing decision is made once per destination, forever" | Each packet/flow is looked up (with caching); table changes take effect immediately |
| "Gateway `0.0.0.0` means no route" | It means on-link (no gateway needed) |
| "If a route matches, the packet will be delivered" | Only if the next hop answers ARP, forwards it, and the return path exists |
| "The IP destination changes at the gateway" | Only the MAC changes (unless NAT); the IP destination stays D |
| "The app decides the NIC" | The kernel does, by routing (apps can bind to influence it) |
| "Two NICs on the same subnet just work" | Requires care (see ARP flux, §9) |

---

## 8. Source address selection

Once the interface is chosen, the **source IP** must be chosen too:

- **Bound socket** (`bind("192.168.2.10")` or `curl --interface eth1`/`--local-addr`): the app's choice is kept, but the packet still leaves via the interface **the routing table selects for that destination**, which may not be the interface owning that source address.
- **Unbound socket:** the kernel uses the route's `src` (Linux `ip route` shows `src 192.168.1.10`) or, failing that, the **primary address of the outgoing interface**. For IPv6, a full selection algorithm (RFC 6724) prefers matching scope, avoiding deprecated addresses, etc.
- **UDP `connect()` trick:** connect a UDP socket to a destination (no packets are sent) and then read `getsockname()` to find out which source IP the kernel *would* use, a common way to learn your outbound IP.

```python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.connect(("8.8.8.8", 53))          # nothing is sent; the kernel just does the route lookup
print(s.getsockname()[0])           # the source address the kernel chose for that destination
```

---

## 9. When real life gets more complex

### 9.1 Policy routing (`ip rule`) and multiple tables
Linux can consult **several routing tables** and choose which one by rules on the packet's **source address, incoming interface, mark, uid, port**, etc. before doing the longest-prefix match *inside the chosen table*:

```bash
ip rule
# 0:      from all lookup local
# 32766:  from all lookup main
# 32767:  from all lookup default
sudo ip route add default via 10.9.0.1 dev eth1 table 100        # a separate table
sudo ip rule add from 192.168.2.10 lookup 100                     # traffic sourced from eth1's address uses table 100
```
Uses: **multi-WAN** (each ISP's replies must leave the right NIC), VPN split by application (`ip rule add uidrange …`), Docker/Kubernetes (Calico, Cilium, and Kubernetes use marks), Tailscale/WireGuard (`fwmark`) and containers' own tables. In routers, **VRF** does the same with isolated tables.

### 9.2 Equal-cost multipath (ECMP)
Two routes with the same prefix **and** the same metric can be installed as one multipath route; the kernel spreads **flows** (by a hash of addresses/ports) across next hops. Used in data centers and multi-uplink setups.
```bash
sudo ip route add 172.16.0.0/16 nexthop via 10.0.0.1 dev eth0 weight 1 nexthop via 10.0.1.1 dev eth1 weight 1
```

### 9.3 Reverse-path filtering (`rp_filter`)
When a packet **arrives** on eth1 from source S, the kernel can check "would I route a reply to S out of eth1?". If not, the packet may be dropped (anti-spoofing). On multihomed hosts with **asymmetric routing** this silently drops legitimate traffic: `sysctl net.ipv4.conf.all.rp_filter` (`1` strict, `2` loose, `0` off). Classic symptom: "packets arrive (tcpdump) but the app never sees them".

### 9.4 Weak vs strong host model
Linux is a **weak host**: it will accept a packet for *any of its own addresses on any interface* (e.g. eth0 can answer ARP for eth1's address), and choose outputs purely by routing. Other systems (BSD variants, Windows as configured) can be stricter.

**ARP flux:** with two NICs **on the same LAN**, Linux may answer an ARP request for NIC2's address using NIC1's MAC (both are "mine"), so switches/neighbors learn the wrong port. Mitigate with:
```bash
sudo sysctl -w net.ipv4.conf.all.arp_ignore=1        # answer ARP only if the address is on the receiving interface
sudo sysctl -w net.ipv4.conf.all.arp_announce=2      # use the best local address for ARP source
```

### 9.5 Redirects and PMTU
Routers can send **ICMP redirects** ("use gateway X for that destination"), which the kernel may cache as a per-destination exception route; and **Path MTU discovery** stores per-destination MTU exceptions (`ip route show cache`, `ip route get … ` shows `mtu`). Both are "cached exceptions" layered on top of the table.

---

## 10. Lab: prove the algorithm on real kernels

Use the two-network host from Chapter 40 (namespace `mh` with `eth1`=10.1.0.10/24 and `eth2`=10.2.0.10/24 and hosts `n1`, `n2`), then add overlapping routes. (If you deleted it, rebuild it with Chapter 40 §5.)

```bash
# Add three routes to the SAME destination range to test prefix vs metric
sudo ip netns exec mh ip route add 10.9.0.0/16 via 10.1.0.50 dev eth1 metric 500
sudo ip netns exec mh ip route add 10.9.0.0/16 via 10.2.0.50 dev eth2 metric 100      # same prefix: lower metric wins
sudo ip netns exec mh ip route get 10.9.1.1                    # → via 10.2.0.50 dev eth2
sudo ip netns exec mh ip route add 10.9.1.0/24 via 10.1.0.50 dev eth1 metric 900      # longer prefix: wins despite the HIGHER metric
sudo ip netns exec mh ip route get 10.9.1.1                    # → via 10.1.0.50 dev eth1
sudo ip netns exec mh ip route get 10.9.2.1                    # → via 10.2.0.50 dev eth2 (falls to the /16)
sudo ip netns exec mh ip route add 10.9.1.1/32 dev eth2                                # host route: even more specific
sudo ip netns exec mh ip route get 10.9.1.1                    # → dev eth2 (direct, no gateway)

# Take the interface with the best route DOWN and watch failover
sudo ip netns exec mh ip route del 10.9.1.1/32; sudo ip netns exec mh ip route del 10.9.1.0/24
sudo ip netns exec mh ip link set eth2 down
sudo ip netns exec mh ip route get 10.9.2.1                    # the eth2 routes vanished → now via eth1 (metric 500)
sudo ip netns exec mh ip link set eth2 up

# Source address selection: which src does the kernel pick?
sudo ip netns exec mh ip route get 10.1.0.50                   # src 10.1.0.10
sudo ip netns exec mh ip route get 10.1.0.50 from 10.2.0.10    # force another source → shows how source affects the result
sudo ip netns exec mh python3 -c "
import socket; s=socket.socket(socket.AF_INET,socket.SOCK_DGRAM); s.connect(('10.2.0.50',9)); print(s.getsockname()[0])"   # 10.2.0.10

# Compare with the Python model: paste the same routes into ROUTES and run route_lookup('10.9.1.1') etc.
```
And check real machines: `ip route get 1.1.1.1`, `ip route get <docker container IP>`, `ip route get <your own IP>` (shows `local … dev lo`).

**Capture proof** that the *IP destination stays the same but the MAC changes* when going through a gateway:
```bash
sudo tcpdump -nn -e -i eth0 icmp &          # on your real machine
ping -c 1 8.8.8.8                            # dst IP 8.8.8.8, dst MAC = your router's MAC (compare with `ip neigh show <gateway>`)
```

---

## 11. Full picture: from application to wire

```
Application:   send to 192.168.3.20:80
Transport:     TCP segment (src port 51234 → 80)
Network:       route lookup → 192.168.3.0/24 on eth2, on-link → src IP 192.168.3.10, dst IP 192.168.3.20
Neighbor:      ARP for 192.168.3.20 on eth2 → MAC
Data link:     Ethernet frame: dst MAC (target), src MAC (eth2), type 0x0800
Physical:      out of NIC3's port
```
And for `8.8.8.8`: same, but the route is the default: `eth0`, next hop 192.168.1.1, ARP for the **gateway**, frame dst MAC = gateway's MAC, IP dst 8.8.8.8.

---

## 12. Summary

- **Algorithm:** local? → find all matching routes (`dst AND mask == network`) → **longest prefix** → **lowest metric** → next hop = gateway or the destination itself → interface → source IP → ARP the next hop → build the frame.
- **Prefix length beats metric**; metric only breaks ties among equal prefixes; no match means **Network is unreachable**.
- **IP destination is constant end to end; the MAC is the next hop's.**
- **Subnet mask = netmask = Genmask = prefix length.**
- Reality adds **policy routing**, **multiple tables**, **ECMP**, **source-address selection**, **rp_filter**, **ARP flux** and **cached exceptions**, but every one of them is a refinement of this same lookup.
- `ip route get <dst>` shows exactly what your kernel decides.

---

## 13. Check your understanding

1. State the algorithm in your own words in six steps.
2. Route A: `10.0.0.0/8` metric 1; Route B: `10.20.0.0/16` metric 500; destination `10.20.5.5`. Which wins? Why?
3. Two routes `172.16.0.0/16` via eth0 metric 100 and via eth1 metric 200. Which is used? What happens if eth0 loses link?
4. For destination `8.8.8.8`, what is the destination MAC? The destination IP?
5. How does the kernel choose the source IP of an unbound socket?
6. What is the difference between subnet mask, netmask and Genmask?
7. A multi-homed server receives requests on eth1 but replies leave via eth0 and the client never gets them. Which mechanisms could be involved and how do you fix it?
8. What does `ip route get 192.168.1.10` (your own IP) show, and why?

<details>
<summary>Answers</summary>

1. Check local → collect matching routes → choose longest prefix → tie-break by metric → determine next hop (gateway or destination) and interface → choose source IP, ARP the next hop, frame and send.
2. Route B (`/16`): it's the most specific match; metric is irrelevant across different prefix lengths.
3. The eth0 route (lower metric). If eth0 loses link its routes are removed and the eth1 route takes over.
4. The router's MAC (ARPed for the default gateway); the destination IP is 8.8.8.8.
5. The route's preferred `src` or the primary address of the outgoing interface (or RFC 6724 rules for IPv6).
6. Nothing: three names for the same mask (Windows/Linux/route-command wording), equal to the `/N` prefix.
7. Asymmetric routing: the default route sends replies via eth0. Fix with policy routing (`ip rule from <eth1 IP> lookup 100` with a table having eth1's gateway); check `rp_filter` and firewalls too.
8. `local 192.168.1.10 dev lo src 192.168.1.10`: the `local` table routes your own addresses through loopback; no packet leaves the machine.
</details>

**Practice**

1. Extend the Python model to support metrics ties, a `/32` host route and "unreachable" results; test 10 destinations and compare each with `ip route get` on a real machine (rebuild your table with the same routes in a namespace).
2. In the lab, create two default routes with different metrics; bring the preferred interface down/up and log `ip route get 8.8.8.8` at each step.
3. Implement the policy-routing example: a namespace with two uplinks where traffic sourced from each address must leave through its own uplink; prove it with `ip route get X from Y` and `tcpdump`.
4. Reproduce an `rp_filter` drop: send a packet with a spoofed source and watch it vanish (`sysctl net.ipv4.conf.all.rp_filter=1`, counters in `nstat -az | grep -i IPReversePathFilter`).
5. Draw the complete journey for `curl http://192.168.3.20` and `curl http://8.8.8.8` on your own machine with real MACs and IPs from `ip route get` and `ip neigh`.

---

**You've completed the networking section (Chapters 21–43).** You can now follow a packet from application to wire: DNS, TCP/UDP, IP, addressing, DHCP, ARP, switching, routing. These are the same building blocks Docker uses for its bridge networks, port publishing and DNS. Revisit [Chapter 10](10_running_ubuntu_on_docker.md) or [Chapter 14](14_docker_hands_on.md) with this knowledge and inspect a container's interfaces, routes and neighbors.
