# Chapter 42: The Routing Table

> **In one sentence:** The **routing table** is a list of rules of the form "*to reach destination network N, send the packet out interface I (via next-hop gateway G, if any)*" that **every** IP device consults for **every** packet it sends; it is filled automatically from your interfaces and DHCP, and manually or by routing protocols when needed.

**Level:** 🟡 Intermediate · **Reading time:** ~55 minutes

**Prerequisites:** Chapters [33](33_subnetting_and_subnet_masks_in_details.md) (masks), [34](34_cidr_subnet_subnet_mask_differences_in_details.md) (CIDR), [40](40_multiple_nics_in_single_computer_in_details.md) and [41](41_visualizing_multiple_nics_in_single_computer_in_details.md) (multiple NICs).

---

## What you will learn

- **What a routing table is**, and what each **column** means (destination, gateway, netmask, flags, metric, interface)
- How to **read it** on Linux, macOS and Windows
- The two kinds of routes: **directly connected** (gateway `0.0.0.0`/`on-link`) and **via a gateway**
- How the **network address** is calculated (`destination AND mask`), and the matching rules
- **How the table gets filled**: kernel (from addresses), DHCP, administrators, VPNs, routing protocols
- The **default route / default gateway**, **loopback route**, **metrics**, and special routes (blackhole, unreachable)
- How to **add, delete and persist** routes safely
- **Docker, VPN, cloud** routing tables in real life
- Labs and troubleshooting

---

## 1. The problem again

A computer with three NICs (Chapter 40) needs a **rulebook** to decide where each packet goes:

```
Destination is on my 192.168.1.x network  → out NIC 1, deliver directly
Destination is on my 192.168.2.x network  → out NIC 2, deliver directly
Destination is on my 192.168.3.x network  → out NIC 3, deliver directly
Anything else                              → NIC 1, hand to the router 192.168.1.1
```
That rulebook is the **routing table** (also called **forwarding table** or **FIB**, Forwarding Information Base). Every host and router has one: your phone, your laptop, a Docker container, your home router, and the largest internet backbone routers (whose tables hold ~1 million IPv4 prefixes).

---

## 2. Anatomy of a routing table

### Linux, the classic view (`route -n` / `netstat -rn`)

```
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    100    0        0 eth0
192.168.1.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
192.168.2.0     0.0.0.0         255.255.255.0   U     200    0        0 eth1
192.168.3.0     0.0.0.0         255.255.255.0   U     300    0        0 eth2
```
### Linux, modern (`ip route`)

```
default via 192.168.1.1 dev eth0 proto dhcp metric 100
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.10 metric 100
192.168.2.0/24 dev eth1 proto kernel scope link src 192.168.2.10 metric 200
192.168.3.0/24 dev eth2 proto kernel scope link src 192.168.3.10 metric 300
```

### The columns

| Column | Meaning |
|---|---|
| **Destination** | The **network** (or single host) this rule covers, e.g. `192.168.2.0` |
| **Genmask / Netmask** (or the `/24` in CIDR form) | Which bits of the destination count: identical to the subnet mask (Chapter 33). `0.0.0.0` = "match everything" (default route) |
| **Gateway** (`via`) | The **next-hop router** to hand the packet to. **`0.0.0.0` (Linux) / `on-link` (Windows) / no `via`** = the destination is **directly connected**: deliver straight to it with ARP |
| **Flags** | `U` route is up · `G` uses a **gateway** · `H` a **host** route (/32) · `!` reject (Linux) · `R`, `D`, `M` dynamic/redirect |
| **Metric** | **Cost** (lower = preferred). Used to choose between routes of equal specificity |
| **Iface** | The **outgoing interface** (NIC) |
| **src** (`ip route`) | The **source IP** used for packets taking this route |
| **proto** (`ip route`) | Who installed it: `kernel` (from an address), `dhcp`, `static`, `boot`, `ra` (IPv6 router advert), `bgp`, `ospf`… |
| **scope** | `link` = on-link, `global` = via a gateway, `host` = this machine |
| Ref / Use | Legacy counters |

### The two kinds of route

| Kind | Example | Meaning | Next step |
|---|---|---|---|
| **Directly connected** ("on-link", *no gateway*) | `192.168.2.0/24 dev eth1` | "This network is attached to me" | **ARP for the destination itself**, send the frame directly |
| **Via a gateway** | `default via 192.168.1.1 dev eth0` | "Hand it to that router" | **ARP for the gateway**, send the frame to the gateway's MAC; the IP destination is unchanged |

---

## 3. Viewing the table on each OS

| OS | Command | Notes |
|---|---|---|
| **Linux** | `ip route` (or `ip r`), `ip -6 route`, `route -n`, `netstat -rn` | `ip route show table all` includes local/other tables |
| **macOS** | `netstat -rn`, `netstat -rn -f inet`, `route -n get default` | `Gateway` may show `link#N` (on-link) or a MAC for hosts |
| **Windows** | `route print`, `netstat -r`, `Get-NetRoute` | `On-link` = directly connected; each row also lists the **interface IP** |
| **Cisco/routers** | `show ip route` | Codes: `C` connected, `S` static, `O` OSPF, `B` BGP |

**Ask the kernel what it would do** (the most useful command):

```bash
ip route get 192.168.3.10      # 192.168.3.10 dev eth2 src 192.168.3.10 uid 1000
ip route get 8.8.8.8           # 8.8.8.8 via 192.168.1.1 dev eth0 src 192.168.1.10
route -n get 8.8.8.8           # macOS
Find-NetRoute -RemoteIPAddress 8.8.8.8      # Windows PowerShell
```

---

## 4. The network address: `destination AND mask`

Each route says "if `destination AND mask == route's network`, then this rule applies". The OS does that AND (Chapter 33) between the **packet's destination IP** and each route's mask.

Example: packet to **`192.168.3.10`**; route `192.168.3.0/24` (mask `255.255.255.0`):

```
  192.168.3.10  = 11000000.10101000.00000011.00001010
& 255.255.255.0 = 11111111.11111111.11111111.00000000
--------------------------------------------------------
  192.168.3.0     ← equals the route's destination → MATCH
```
Same packet against `192.168.2.0/24`: `192.168.3.10 & mask = 192.168.3.0 ≠ 192.168.2.0` → no match. Against the default route (`0.0.0.0/0`, mask all zeros): `anything & 0.0.0.0 = 0.0.0.0` → **always matches**, which is precisely why it works as a catch-all.

```bash
python3 - <<'E'
import ipaddress as ip
d = ip.ip_address("192.168.3.10")
for net in ["192.168.1.0/24","192.168.2.0/24","192.168.3.0/24","0.0.0.0/0"]:
    print(f"{net:16} {'MATCH' if d in ip.ip_network(net) else '-'}")
E
```

---

## 5. The decision algorithm (short version; Chapter 43 has the full one)

For each outgoing packet the OS:

1. Takes the **destination IP**.
2. **Finds every route that matches** (destination AND mask = route network).
3. Chooses the **most specific** match: the **longest prefix** (`/32` beats `/24` beats `/0`).
4. If several routes have the same prefix length, the **lowest metric** wins.
5. Uses that route's **interface** (and gateway, if any) to send; the **source IP** is normally the interface's address.
6. If **no route** matches: **"Network is unreachable"** (no default route) and the packet isn't sent.

### Worked examples with the three-NIC table from §2

| Destination | Matching routes | Winner | What happens |
|---|---|---|---|
| **192.168.3.10** | `192.168.3.0/24` (eth2), `default` | `/24` is longer | Out **eth2**, ARP for `192.168.3.10` itself |
| **192.168.2.10** | `192.168.2.0/24` (eth1), `default` | `/24` | Out **eth1**, ARP for `.2.10` |
| **8.8.8.8** | only `default` | default | Out **eth0**, ARP for the **gateway 192.168.1.1**, IP dst stays 8.8.8.8 |
| **192.168.1.77** | `192.168.1.0/24` (eth0), `default` | `/24` | eth0, direct |
| **127.0.0.1** | `local 127.0.0.0/8 dev lo` (in the `local` table) | loopback | Never leaves the machine |

---

## 6. How the table gets its entries

| Source | Route created | Example |
|---|---|---|
| **The kernel, from interface addresses** ("connected routes") | When you configure `192.168.2.10/24` on `eth1`, the kernel adds `192.168.2.0/24 dev eth1 proto kernel scope link src 192.168.2.10` | Automatic; the reason your address **and mask** matter (Chapter 33) |
| **DHCP** (option 3 Router, option 121 classless routes) | `default via 192.168.1.1 dev eth0 proto dhcp` | Automatic on each lease |
| **IPv6 Router Advertisements** | `default via fe80::1 dev eth0 proto ra` | Automatic |
| **Administrator (static routes)** | `ip route add 10.5.0.0/16 via 192.168.1.254` | Reach a remote network behind another router |
| **VPN clients** | Adds routes for corporate ranges via `tun0`/`wg0`, sometimes replaces the default route (**full tunnel**) or adds only the corporate CIDRs (**split tunnel**) | `10.0.0.0/8 dev tun0` |
| **Docker / hypervisors / CNI** | Bridge subnets appear as connected routes | `172.17.0.0/16 dev docker0` |
| **Routing protocols (OSPF, BGP, RIP, IS-IS)** | Routers exchange reachability and install routes automatically | Enterprise/ISP networks; `proto bgp` via FRR/BIRD |
| **ICMP redirects** (rare today) | A router says "use that gateway instead" | Usually disabled for security |

### Local and loopback routes
Linux keeps some routes in a separate **`local` table**, consulted first:

```bash
ip route show table local
# local 127.0.0.0/8 dev lo proto kernel scope host src 127.0.0.1
# local 127.0.0.1 dev lo proto kernel scope host src 127.0.0.1
# broadcast 127.255.255.255 dev lo ...
# local 192.168.1.10 dev eth0 proto kernel scope host src 192.168.1.10     ← packets to MY OWN address go via lo, they never hit the wire
# broadcast 192.168.1.255 dev eth0 ...
```
So a ping to `127.0.0.1` **or to your own LAN address** is delivered internally over the loopback path; no frame is ever sent.

### Special route types (Linux)
```bash
ip route add blackhole 203.0.113.0/24        # silently drop
ip route add unreachable 198.51.100.0/24     # reply ICMP "host unreachable"
ip route add prohibit 192.0.2.0/24           # reply ICMP "administratively prohibited"
```
Used for sinkholing bad ranges, or in BGP setups to anchor aggregate routes.

---

## 7. Default route and default gateway

- **Default route** = the entry `0.0.0.0/0`: matches everything, but is the **least specific**, so it is only used when nothing better matches ("if you don't know where it belongs, give it to the router").
- **Default gateway** = the **next-hop address** of the default route: normally your router, `192.168.1.1`.

They are two views of the same thing: *the default route's `via` address is the default gateway.*

### Why one default route (usually)?
Two default routes with equal metrics are ambiguous ("which way to the Internet?"). The kernel picks the **lowest metric** (or, with equal metrics, the first/oldest, unless you configure **multipath/ECMP** deliberately). Typical laptop: Ethernet has metric 100, Wi-Fi 600, so wired is preferred and Wi-Fi is a backup. Check with:

```bash
ip route show default
# default via 192.168.1.1 dev enp3s0 proto dhcp metric 100
# default via 192.168.1.1 dev wlp2s0 proto dhcp metric 600
```
Windows computes **automatic metrics** from link speed; macOS uses the **Service Order** (System Settings → Network → ⋯ → Set Service Order).

A **VPN full tunnel** installs a default route via the tunnel (or two `/1` routes `0.0.0.0/1` and `128.0.0.0/1` that override the real default without deleting it: more specific, hence they win).

---

## 8. Managing routes

### Linux

```bash
ip route add 10.5.0.0/16 via 192.168.1.254               # via a gateway (must be on a connected network)
ip route add 10.6.0.0/16 dev eth1                        # directly connected network
ip route add default via 192.168.1.1 metric 100
ip route replace default via 192.168.1.2                 # add-or-change
ip route del 10.5.0.0/16
ip route flush cache                                     # rarely needed today
sysctl net.ipv4.ip_forward                               # 1 = this box also forwards packets for others (is a router)
```
**Persistence:** these commands vanish on reboot. Make them permanent with your network manager: **netplan** (`routes:` with `to:`/`via:`), **NetworkManager** (`nmcli con mod eth0 +ipv4.routes "10.5.0.0/16 192.168.1.254"`), **systemd-networkd** (`[Route]` sections), or `/etc/network/interfaces` (`up ip route add …`).

### macOS
```bash
sudo route -n add -net 10.5.0.0/16 192.168.1.254
sudo route -n delete -net 10.5.0.0/16
```
### Windows
```powershell
route add 10.5.0.0 mask 255.255.0.0 192.168.1.254 metric 10      # temporary
route -p add 10.5.0.0 mask 255.255.0.0 192.168.1.254             # persistent (-p)
route delete 10.5.0.0
New-NetRoute -DestinationPrefix 10.5.0.0/16 -NextHop 192.168.1.254 -InterfaceAlias "Ethernet"
```
**Caution:** a wrong default route or a deleted connected route can cut off your own SSH session. Test on a console or keep a rollback (`ip route replace` back), and prefer testing in namespaces.

---

## 9. Routing tables in Docker, VPNs and the cloud

**Docker host**
```
default via 192.168.1.1 dev eth0
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
172.18.0.0/16 dev br-2f4a9c1e7b3d proto kernel scope link src 172.18.0.1
```
- Traffic to any container goes through its bridge (connected route); everything else via `eth0`.
- Inside a container: `docker exec c ip route` → `default via 172.17.0.1 dev eth0` and `172.17.0.0/16 dev eth0 src 172.17.0.2`: the **host's bridge IP is the container's default gateway**.
- Outgoing container traffic is NATed (`MASQUERADE`), and Docker enables `ip_forward`.
- If a VPN or office network uses `172.17.0.0/16`, the two rules conflict: change Docker's pools.

**Kubernetes node:** additional routes per pod CIDR (`10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink`) or BGP-installed (`proto bird`) routes, one per other node.

**Cloud VPC:** you configure **route tables** (a managed, per-subnet version of exactly this idea): "10.0.0.0/16 local, 0.0.0.0/0 → internet gateway (public subnet) or NAT gateway (private subnet), 10.1.0.0/16 → VPC peering / transit gateway". Longest-prefix match applies there too.

**Home router:** two routes essentially: `192.168.1.0/24` connected on the LAN side, `default` via the ISP on the WAN side (plus NAT).

---

## 10. Lab: a two-router network and static routes

We'll build `pc — R1 — R2 — server` where R1 and R2 need routes to reach far networks.

```
pc 10.0.1.10/24 ── R1 (10.0.1.1 | 10.0.12.1) ── R2 (10.0.12.2 | 10.0.2.1) ── srv 10.0.2.10/24
```
```bash
for n in pc r1 r2 srv; do sudo ip netns add $n; sudo ip netns exec $n ip link set lo up; done
link() { # ns1 if1 ip1 ns2 if2 ip2
  sudo ip link add $2 type veth peer name $5
  sudo ip link set $2 netns $1; sudo ip link set $5 netns $4
  sudo ip netns exec $1 ip addr add $3 dev $2; sudo ip netns exec $4 ip addr add $6 dev $5
  sudo ip netns exec $1 ip link set $2 up;     sudo ip netns exec $4 ip link set $5 up; }
link pc pc0 10.0.1.10/24  r1 r1a 10.0.1.1/24
link r1 r1b 10.0.12.1/24  r2 r2a 10.0.12.2/24
link r2 r2b 10.0.2.1/24   srv s0  10.0.2.10/24
sudo ip netns exec r1 sysctl -qw net.ipv4.ip_forward=1
sudo ip netns exec r2 sysctl -qw net.ipv4.ip_forward=1

# Look at each table: only CONNECTED routes exist
for n in pc r1 r2 srv; do echo "== $n"; sudo ip netns exec $n ip route; done

sudo ip netns exec pc ping -c1 -W1 10.0.2.10      # "Network is unreachable": pc has no route to 10.0.2.0/24
sudo ip netns exec pc ip route add default via 10.0.1.1          # pc: everything else → R1
sudo ip netns exec pc ping -c1 -W1 10.0.2.10      # still fails (no reply): now R1 doesn't know 10.0.2.0/24
sudo ip netns exec r1 ip route add 10.0.2.0/24 via 10.0.12.2     # R1: that network is behind R2
sudo ip netns exec pc ping -c1 -W1 10.0.2.10      # still fails: the reply can't come back, srv doesn't know 10.0.1.0/24
sudo ip netns exec srv ip route add default via 10.0.2.1         # srv: everything else → R2
sudo ip netns exec r2 ip route add 10.0.1.0/24 via 10.0.12.1     # R2: pc's network is behind R1
sudo ip netns exec pc ping -c2 10.0.2.10          # works!  ttl=62 (two routers)
sudo ip netns exec pc traceroute -n 10.0.2.10     # hop1 10.0.1.1 (R1), hop2 10.0.12.2 (R2), hop3 10.0.2.10

# read the decisions
sudo ip netns exec pc  ip route get 10.0.2.10     # via 10.0.1.1 dev pc0
sudo ip netns exec r1  ip route get 10.0.2.10     # via 10.0.12.2 dev r1b
sudo ip netns exec r2  ip route get 10.0.2.10     # dev r2b (connected)
```
**Lessons:** routing must work **in both directions** (the reply needs a path back); every router only needs to know the **next hop**, not the whole path; a missing route anywhere shows up as "unreachable", timeouts, or replies that never return. Try removing routes one at a time and predicting the symptom. Add a **blackhole** route on R2 for `10.0.2.10/32` to see the difference between "unreachable" and "timeout".

Clean up: `for n in pc r1 r2 srv; do sudo ip netns del $n; done`.

---

## 11. Troubleshooting with the routing table

| Symptom | Check |
|---|---|
| **"Network is unreachable"** (immediately) | No matching route (usually no default route): `ip route`, DHCP/gateway config |
| **"Destination host unreachable"** from the gateway | The gateway has no route or can't ARP the target |
| **Timeout** (no error) | Route exists but the path is broken later: firewall, missing return route, wrong gateway, ARP failure (`ip neigh`) |
| **Works locally, not to the Internet** | Missing/incorrect default route, router WAN down, NAT |
| **Traffic goes out the wrong NIC** | Metrics/overlapping routes: `ip route get <dst>` |
| **VPN up but no corporate access** | Route for corporate CIDRs not installed or overridden; check `ip route get <corp-ip>` |
| **Docker containers can't reach a VPN network** | Overlapping subnets or no masquerade for the tunnel |
| **After adding a route, SSH drops** | You changed the route your session depends on; `ip route get <your-client-ip>` |
| **Asymmetric routing** | Requests come in one NIC, replies leave another (multi-homed hosts, reverse-path filter drops: `rp_filter`) |

Tools: `ip route get`, `traceroute -n` / `tracepath` / `mtr` (Windows: `tracert`, `pathping`), `ip -s neigh`, `tcpdump -i <if>`, `ping -I <if>`.

---

## 12. Common misconceptions

| Misconception | Reality |
|---|---|
| "The routing table lists all the hops to the destination" | It lists only the **next hop** for each destination network |
| "Gateway `0.0.0.0` means no route" | It means **directly connected** (no gateway needed) |
| "The default gateway is separate from routing" | It's the `via` of the default route |
| "The first matching route wins" | The **longest prefix** wins, then the lowest metric (Chapter 43) |
| "Only routers have routing tables" | Every IP host does (even a container) |
| "A route to a network means it's reachable" | Only if the next hop and the **return route** exist and firewalls allow it |
| "Routes I add stay after reboot" | Not unless persisted in the network manager config |
| "Metric decides over prefix length" | Prefix length first; metric only breaks ties |
| "Traffic to my own IP goes out to the router and back" | The local table delivers it internally via `lo` |

---

## 13. Summary

- The **routing table** maps **destination networks** to **interface + next hop**; each row has **destination, mask/prefix, gateway, flags, metric, interface** (`ip route`, `route print`, `netstat -rn`).
- **Directly connected** networks (no gateway) → ARP for the destination; **via gateway** → ARP for the gateway, IP destination unchanged.
- Matching uses **`destination AND mask`**; the **longest prefix** wins, then **lowest metric**; no match = "Network is unreachable".
- Entries come from **interface addresses (kernel)**, **DHCP**, **static** config, **VPNs**, **routing protocols**; loopback/own addresses live in the **local** table.
- **Default route** `0.0.0.0/0` (its `via` is the default gateway) catches everything else; keep exactly one preferred.
- Docker, Kubernetes, VPNs and cloud VPCs all use this same mechanism. **`ip route get <dst>`** is your friend.

---

## 14. Check your understanding

1. Name the columns of a routing table and what each means.
2. What does gateway `0.0.0.0` (or `on-link`) tell you?
3. How does the OS decide whether route `192.168.2.0/24` matches destination `192.168.2.77`?
4. Which route wins for `10.1.2.3` among `10.0.0.0/8` (metric 10), `10.1.0.0/16` (metric 100), `0.0.0.0/0` (metric 1)?
5. Explain default route vs default gateway.
6. Where do routes come from?
7. Why does a packet to your own LAN IP never appear on the network?
8. In the lab, after adding the route on R1, `ping` still failed. Why?
9. What does "Network is unreachable" indicate compared with a timeout?

<details>
<summary>Answers</summary>

1. Destination, netmask/prefix, gateway (next hop or none for connected), flags, metric, interface (plus src/proto in `ip route`).
2. The destination network is directly attached; ARP for the destination itself.
3. AND the destination with the route's mask: `192.168.2.77 & 255.255.255.0 = 192.168.2.0` equals the route's destination.
4. `10.1.0.0/16` (longest matching prefix wins regardless of metrics).
5. The default route is `0.0.0.0/0`, matching anything; its next-hop address is the default gateway.
6. Kernel (from interface addresses), DHCP/RA, static admin config, VPN clients, Docker/CNI, routing protocols.
7. The `local` table routes it via loopback internally.
8. The return path was missing: `srv` (and R2) didn't yet know how to send replies back to `10.0.1.0/24`.
9. "Unreachable" means no route was found locally (or an ICMP unreachable came back); a timeout means the packet was sent but the reply never came (filtered, no return route, host down).
</details>

**Practice**

1. Print your routing table (all OSes you have) and annotate each line: type (connected/gateway/default), interface, source (kernel/DHCP/VPN/Docker).
2. Predict then verify with `ip route get` for: your gateway, a Docker container IP, `8.8.8.8`, your own IP, `127.0.0.1`.
3. Connect to a VPN and diff the table before and after (`ip route > before.txt` ...). Full or split tunnel?
4. Extend the lab with a third router and network; add routes only where needed; then replace all static routes on R1 with a single default.
5. Add two default routes with different metrics; take the preferred interface down and watch the failover.

---

**Next:** [Chapter 43 – How the OS Chooses a NIC: The Complete Algorithm](43_how_os_chooses_nic_in_details.md)
