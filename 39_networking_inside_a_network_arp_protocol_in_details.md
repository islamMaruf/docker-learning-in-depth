# Chapter 39: Networking Inside a Network: ARP

> **In one sentence:** **ARP** (Address Resolution Protocol) answers the question "I know the **IP address** of the machine I want to reach on my own network; what is its **MAC address**?", by **broadcasting a question** to everyone on the local link and receiving a **unicast answer** from the owner, then caching the result so it doesn't have to ask again.

**Level:** 🟡 Intermediate · **Reading time:** ~55 minutes

**Prerequisites:** Chapters [30](30_internet_protocol_ip_in_details.md) (IP), [31](31_data_link_layer_frame_in_details.md) (frames, MACs), [33](33_subnetting_and_subnet_masks_in_details.md) (same network vs different) and [38](38_hub_switch_router_network_devices_in_details.md) (hubs, switches, routers).

---

## What you will learn

- **Why ARP must exist:** IP addresses are useless to an Ethernet NIC, which only understands MACs
- The **ARP packet** format, byte by byte, for **request** and **reply**
- How a request and reply **travel through hubs, switches and a router** (who floods, who learns, who ignores)
- The **ARP cache** (neighbor table): states, timeouts, and the commands to inspect it
- ARP when the destination is **remote**: you ARP for the **gateway**, not the server
- **Gratuitous ARP**, **ARP probe/announcement**, **proxy ARP**
- **ARP spoofing/poisoning** and how to defend against it
- **IPv6** replaces ARP with **NDP**; **Docker/VM** angles; a hands-on lab with real packets

---

## 1. The problem: two kinds of addresses

Every packet you send on a LAN has **two** destination addresses:

| Layer | Address | Example | Used by |
|---|---|---|---|
| 3 (network) | **IP** | `192.168.1.20` | Applications, routing decisions |
| 2 (data link) | **MAC** | `bb:bb:bb:bb:bb:bb` | The NIC, switches |

Applications only know an IP (or a name that DNS resolves to an IP). But the NIC builds an Ethernet frame and must put a **destination MAC** in it. The switch also forwards **by MAC**. The kernel is stuck between them: it has the IP, but needs the MAC.

```
HTTP request for 192.168.1.20:80 ready to send
Ethernet frame:  dst MAC = ???     src MAC = aa:bb:cc:11:22:33
                       ▲
                       └─ ARP fills this in
```

Why not just embed the MAC in the IP address, or have DHCP tell you all MACs? Because the two are **independent by design** (IPs are assigned/logical, MACs are hardware/local), which lets layers be swapped without changing others. ARP is the glue that connects them **on one link**.

Analogy: you know a colleague's name (IP), but to hand them a letter in a big open-plan office you need to know which desk (MAC) they sit at. You shout "Who is Priya?" and Priya waves.

---

## 2. Terminology

| Term | Meaning |
|---|---|
| **ARP request** | "Who has IP X? Tell IP Y (with MAC M)" |
| **ARP reply** | "IP X is at MAC N" |
| **Sender** | The one asking (source IP/MAC in the packet) |
| **Target** | The IP being resolved |
| **ARP cache / ARP table / neighbor table** | Local memory of recent IP→MAC answers |
| **Resolution** | Turning IP into MAC |

ARP (RFC 826, 1982) works only on the **local link**. It is never routed.

---

## 3. The ARP packet (28 bytes for IPv4 over Ethernet)

Carried directly in an Ethernet frame with **EtherType `0x0806`** (not inside IP).

```
 0        8        16       24       32
+---------------------+---------------------+
| Hardware type (2)   | Protocol type (2)   |   0x0001 = Ethernet | 0x0800 = IPv4
+----------+----------+---------------------+
| HLEN (1) | PLEN (1) | Operation (2)       |   6 | 4 | 1 = request, 2 = reply
+----------+----------+---------------------+
| Sender hardware address (6)               |   SHA: MAC of the sender
| Sender protocol address (4)               |   SPA: IP  of the sender
| Target hardware address (6)               |   THA: MAC being asked for (00:00:.. in a request)
| Target protocol address (4)               |   TPA: IP being asked about
+-------------------------------------------+
```

### Example: A (192.168.1.10, `aa:bb:cc:11:22:33`) asks for 192.168.1.20

**Request frame (42 bytes; the NIC pads to the 60-byte minimum before the FCS):**

```
Ethernet:  ff ff ff ff ff ff   aa bb cc 11 22 33   08 06
           └ dst: broadcast ┘  └ src: A's MAC ────┘  └ ARP
ARP:       00 01   08 00   06   04   00 01
           hw=Eth  proto=IPv4 hlen plen op=1 (request)
           aa bb cc 11 22 33  c0 a8 01 0a          ← sender MAC, sender IP 192.168.1.10
           00 00 00 00 00 00  c0 a8 01 14          ← target MAC unknown (zeros), target IP 192.168.1.20
```
**Reply (unicast to A, sent by B = 192.168.1.20 with MAC `bb:bb:bb:bb:bb:bb`):**

```
Ethernet:  aa bb cc 11 22 33   bb bb bb bb bb bb   08 06
ARP:       00 01 08 00 06 04 00 02                        ← op=2 (reply)
           bb bb bb bb bb bb  c0 a8 01 14                 ← sender = B: MAC, IP 192.168.1.20
           aa bb cc 11 22 33  c0 a8 01 0a                 ← target = A
```
The request is a **broadcast** (everyone must hear it because A doesn't know who owns the IP); the reply is **unicast** (B already learned A's MAC from the request).

---

## 4. A step-by-step journey: two hosts, one switch

Scenario: A (`192.168.1.10`) browses to B (`192.168.1.20`) on the same LAN, through a switch. ARP caches and the switch table are empty.

```
A ──┐
B ──┼── switch ── router ── (Internet)
C ──┘
```

1. **A's application** opens `http://192.168.1.20`. TCP wants to send a SYN. IP layer decides: `192.168.1.20 AND /24` = my network → **deliver directly**.
2. **A checks its ARP cache.** No entry for `.20` → the packet is **queued**, and A sends an **ARP request** (broadcast).
3. **The switch** learns `A → port 1` from the source MAC and **floods** the broadcast out ports 2, 3, 4 (B, C, router).
4. **Every host processes it**:
   - **C** (`192.168.1.30`) checks the target IP: not me → **ignores** (but many OSes still **update** an existing cache entry for the *sender* if they already had one).
   - **The router** (`192.168.1.1`): not me → ignores.
   - **B**: target IP is me → **learns A** (`192.168.1.10 → aa:bb:cc:11:22:33`) *because it will need to reply and probably talk to A*, then builds the reply.
5. **B sends the ARP reply, unicast** to A's MAC. The switch learns `B → port 2` (from the reply's source MAC) and forwards **only** to port 1.
6. **A** stores `192.168.1.20 → bb:bb:bb:bb:bb:bb` in its cache, **dequeues the SYN**, and sends the TCP frame with the right destination MAC. The switch now knows both ports, and the frame goes A→B directly.
7. Subsequent packets use the **cached** entry: **no more ARP** until it ages out.

Summary of what each device learned:

| Device | Learned |
|---|---|
| A | `192.168.1.20 → bb:bb:…` (ARP cache) |
| B | `192.168.1.10 → aa:bb:…` (from the request) |
| Switch | `aa:bb:… → port 1`, `bb:bb:… → port 2` (MAC table, from source addresses) |
| C, router | Nothing new (usually) |

### What if there's a hub instead of a switch? 
Every device (also C, and any eavesdropper) hears **all** frames, including the unicast reply. Communication works the same; there's just no privacy or efficiency (Chapter 38).

### What if the LAN has several switches/hubs joined together?
The ARP broadcast is **flooded through every switch and hub in the same broadcast domain**; the reply travels back along the learned path (each switch having learned the requester's MAC on the way). **A router in the middle is a boundary**: it does **not** forward the broadcast, so ARP only works between devices **on the same network**, never across a router.

---

## 5. Remote destinations: ARP for the gateway

A (`192.168.1.10/24`, gateway `192.168.1.1`) wants `8.8.8.8`:

```
8.8.8.8 is NOT in 192.168.1.0/24  →  next hop = default gateway 192.168.1.1
ARP request: "Who has 192.168.1.1?"     (NOT 8.8.8.8: no one on the LAN owns it)
ARP reply:   "192.168.1.1 is at 11:22:33:44:55:66"   (the router's LAN interface)
Frame:       dst MAC = 11:22:33:44:55:66   ←  IP dst stays 8.8.8.8 !
```
The **router** then needs the MAC of *its* next hop on the other side (ARP on the WAN side or a point-to-point link), and so on. At each hop: **IP dst constant; MAC dst = the next router.**

```bash
ip route get 8.8.8.8            # "8.8.8.8 via 192.168.1.1 dev eth0 src 192.168.1.10"  → ARP will be for 192.168.1.1
ip neigh show 192.168.1.1       # 192.168.1.1 dev eth0 lladdr 11:22:33:44:55:66 REACHABLE
```

---

## 6. The ARP cache

Since ARP costs a broadcast and a delay, results are cached **for a limited time** (so that moved/changed devices are eventually re-discovered).

```bash
ip neigh                         # Linux (modern);   ip -s neigh for statistics
arp -a                           # Windows, macOS, and Linux with net-tools (`arp -n`)
Get-NetNeighbor                  # Windows PowerShell
```
Typical output:

```
192.168.1.1  dev eth0 lladdr 11:22:33:44:55:66 REACHABLE
192.168.1.20 dev eth0 lladdr bb:bb:bb:bb:bb:bb STALE
192.168.1.99 dev eth0  FAILED
```

### Linux neighbor states (NUD: Neighbor Unreachability Detection)

| State | Meaning |
|---|---|
| **INCOMPLETE** | Request sent, waiting for a reply |
| **REACHABLE** | Recently confirmed working (for `base_reachable_time`, ~30 s, randomized 0.5–1.5×) |
| **STALE** | Timer expired; the entry is still used, but will be **re-verified** when traffic is sent |
| **DELAY** | Traffic sent on a stale entry; waiting 5 s for upper-layer confirmation (e.g. TCP ACK) before probing |
| **PROBE** | Sending unicast ARP probes to re-check |
| **FAILED** | No answer after retries (host down / wrong network) |
| **PERMANENT / NOARP** | Static entries |

Windows caches dynamic entries for ~15–45 s (randomized, reset on use); macOS ~20 minutes; Cisco routers 4 hours. The exact numbers vary by OS and can be tuned.

### Managing it

```bash
sudo ip neigh add 192.168.1.50 lladdr 00:11:22:33:44:55 dev eth0 nud permanent   # static entry
sudo ip neigh del 192.168.1.50 dev eth0
sudo ip neigh flush all                                                            # forget everything → next packets trigger ARP
cat /proc/sys/net/ipv4/neigh/eth0/base_reachable_time_ms                           # timing knobs
```

---

## 7. Lab: watch ARP live

Uses namespaces (nothing touches your real network). Needs `sudo`, `iproute2`, `tcpdump`, `iputils-arping`.

```bash
sudo ip link add br0 type bridge; sudo ip link set br0 up
for i in 1 2 3; do
  sudo ip netns add h$i; sudo ip link add v$i type veth peer name w$i
  sudo ip link set w$i netns h$i; sudo ip link set v$i master br0; sudo ip link set v$i up
  sudo ip netns exec h$i ip addr add 192.168.1.$((i*10))/24 dev w$i; sudo ip netns exec h$i ip link set w$i up
done
# h1 = .10, h2 = .20, h3 = .30 (the bystander)

sudo ip netns exec h3 tcpdump -nn -e -i w3 arp &          # bystander watches ARP
sleep 1
sudo ip netns exec h1 ip neigh                             # empty
sudo ip netns exec h1 ping -c 1 192.168.1.20               # triggers ARP
sudo ip netns exec h1 ip neigh                             # 192.168.1.20 lladdr <h2 MAC> REACHABLE
sudo ip netns exec h2 ip neigh                             # h2 learned h1 from the REQUEST
sudo ip netns exec h3 ip neigh                             # h3 learned nothing (it ignored the request)
sleep 1; sudo pkill tcpdump
```
In the capture you should see: `Request who-has 192.168.1.20 tell 192.168.1.10` (dst `ff:ff:ff:ff:ff:ff`) — **h3 sees it** — and the `Reply 192.168.1.20 is-at bb:...` (unicast) **which h3 does not see** (the switch sent it to h1 only).

More experiments:

```bash
# watch the states change over time
sudo ip netns exec h1 bash -c 'for i in 1 2 3 4 5 6 7; do ip neigh show 192.168.1.20; sleep 10; done'
# force re-resolution
sudo ip netns exec h1 ip neigh flush all; sudo ip netns exec h1 ping -c 1 192.168.1.20
# ask about a non-existent host → INCOMPLETE then FAILED
sudo ip netns exec h1 ping -c 1 -W 1 192.168.1.99; sudo ip netns exec h1 ip neigh show 192.168.1.99
# arping = ARP-level ping (works even if ICMP is blocked)
sudo ip netns exec h1 arping -c 2 -I w1 192.168.1.20
# detect duplicate address: -D (DAD mode); exit code shows if someone else owns it
sudo ip netns exec h1 arping -D -c 2 -I w1 192.168.1.20 ; echo "exit=$?"
```
Cleanup: `for i in 1 2 3; do sudo ip netns del h$i; done; sudo ip link del br0`.

On your real machine: `sudo tcpdump -nn -e -i <iface> arp` while you `ip neigh flush all` and browse something: you'll see your host resolve the gateway.

---

## 8. Special forms of ARP

### Gratuitous ARP (GARP)
A host announces **its own** mapping without being asked: sender IP = target IP = its own IP (request or reply form). Uses:
- **Announce a new/changed address** so caches update (after DHCP, an interface change).
- **Failover:** when a virtual IP (VRRP/HSRP/keepalived) moves to a backup machine, the new owner sends a gratuitous ARP so neighbors and switches update immediately.
- **VM/container live migration:** tells the network "this MAC is now behind a new port".
- Duplicate address detection (old style).

### ARP probe and announcement (RFC 5227)
Before using a new address, a host sends **probes** with sender IP `0.0.0.0` asking "who has X?"; no reply means the address is free (a later **announcement** follows). Using `0.0.0.0` avoids polluting others' caches with an address that may be a duplicate. This is the mechanism behind DHCP conflict checking (Chapter 37) and link-local address selection (Chapter 32).

### Proxy ARP
A router answers ARP requests **on behalf of** hosts behind it, so hosts with a wrong (too wide) mask can still reach them:

```
A (mask /16 by mistake, thinks 192.168.2.5 is local) → ARP "who has 192.168.2.5"
Router (proxy ARP enabled) → "192.168.2.5 is at <my MAC>"   → A sends to the router → router forwards
```
Handy in odd setups (old subnets, some VPNs, some Docker/cloud tricks) but it hides addressing mistakes and can grow ARP caches large; usually disabled: `sysctl net.ipv4.conf.<if>.proxy_arp`.

### Reverse ARP / InARP
RARP (MAC→IP) is obsolete (replaced by BOOTP/DHCP). Inverse ARP appears in Frame Relay/ATM.

---

## 9. Security: ARP spoofing (poisoning)

ARP has **no authentication**: any host can send a reply claiming any IP, and most hosts accept unsolicited replies for IPs they've already cached (or in some OSes even new ones).

```
Attacker M (mm:mm:..) tells A: "192.168.1.1 (the gateway) is at mm:mm:.."
Attacker tells the gateway: "192.168.1.10 (A) is at mm:mm:.."
→ A and the router now send their traffic to M, who forwards it on → man-in-the-middle
```
Impact: eavesdropping, tampering, session hijack of unencrypted traffic, denial of service. (Encrypted protocols such as HTTPS/SSH limit the damage, but metadata and DNS may still leak.)

**Signs:** the gateway's MAC in your ARP cache suddenly changes; two IPs share one MAC (`ip neigh | sort -k5`); Wireshark's "duplicate IP address configured" warnings; `arpwatch` alerts.

**Defenses:**
- **Dynamic ARP Inspection (DAI)** on managed switches, validating ARP packets against the **DHCP snooping** binding table (IP↔MAC↔port).
- **Port security** and 802.1X; **static ARP entries** for critical hosts (e.g. gateway) on servers.
- **Encrypted protocols end to end** (TLS, SSH, VPN); DNSSEC/DoH.
- **Segmentation (VLANs)** and small broadcast domains; monitoring with `arpwatch`, IDS.
- Wi-Fi client isolation.

Only test spoofing tools on **networks and machines you own or are explicitly authorized to test**.

---

## 10. IPv6 does this differently: NDP

IPv6 has **no ARP**. **Neighbor Discovery Protocol (NDP, ICMPv6)** does the same job with **Neighbor Solicitation/Advertisement** messages sent to a **solicited-node multicast** address (`ff02::1:ffXX:XXXX`, last 24 bits of the target address) instead of a broadcast, so only the few hosts that might own the address are bothered (and other nodes' CPUs aren't interrupted). NDP also does router discovery, duplicate address detection and redirects.

```bash
ip -6 neigh          # IPv6 neighbor table (same states as IPv4)
```

---

## 11. Docker, VMs and cloud

- **Docker bridge:** containers ARP each other across `docker0`; the bridge's MAC table (`bridge fdb`) and `ip neigh` inside a container (`docker exec c ip neigh`) show it. Docker gives each container a MAC derived from its IP (`02:42:ac:11:00:02` for `172.17.0.2`).
- **Traffic leaving the host** reaches the bridge IP (the gateway `172.17.0.1`): a container ARPs for the gateway, not the Internet host.
- **macvlan** networks: each container has its own MAC on your **physical** LAN, so your real switch and router ARP for them directly; the host itself often can't reach them without an extra macvlan interface.
- **VMs and live migration** use gratuitous ARP/RARP to update switches.
- **Cloud (AWS/GCP/Azure VPC):** the virtual network handles ARP itself (it's not a real Layer-2 broadcast domain); you can't ARP-spoof or sniff other tenants, and gratuitous ARP for a floating IP is often ignored: you re-assign the IP with an API call.
- **Kubernetes:** CNI plugins and kube-proxy rely on ARP/NDP inside nodes; large clusters sometimes tune ARP cache size (`gc_thresh1/2/3`) because a full neighbor table drops entries and breaks connectivity ("neighbour table overflow" in `dmesg`).

---

## 12. Troubleshooting

| Symptom | Diagnosis |
|---|---|
| `ip neigh` shows **INCOMPLETE/FAILED** for a local host | Host down, different VLAN/subnet, firewall on the host **doesn't** block ARP (it's L2), so the host is probably off/unplugged/isolated; check switch port, VLAN, wrong mask |
| Can ping by IP but MAC of gateway keeps changing | ARP spoofing, or two devices with the same IP (DHCP/static conflict) |
| Intermittent connectivity; "duplicate IP" popup | IP conflict: `arping -D -I eth0 <ip>`; find the MAC; fix static/DHCP overlap |
| First packet slow, later fast | Normal: ARP resolution delay on the first packet |
| Host reachable only after the other side pings first | Asymmetric caches, wrong mask on one side (Chapter 33), or ARP filtering (`arp_ignore`/`arp_filter`) on multi-homed hosts |
| `neighbour table overflow` in logs | Too many neighbors (big L2 networks, Kubernetes nodes); raise `net.ipv4.neigh.default.gc_thresh*` or shrink the L2 domain |
| VIP failover takes long | No/ignored gratuitous ARP; switch MAC-table aging; enable GARP on failover |
| Switch flooding traffic | MAC table entry aged out or overflow (Chapter 38) |

Tools: `ip neigh`, `arp -an`, `arping`, `tcpdump -nn -e arp`, Wireshark (`arp`, `arp.duplicate-address-detected`), `arpwatch`, switch `show mac address-table`, `show ip arp`.

---

## 13. Common misconceptions

| Misconception | Reality |
|---|---|
| "ARP finds the MAC of any IP on the Internet" | Only on the **local link**; for remote hosts you ARP for the **gateway** |
| "ARP runs over IP" | It's a separate Layer-2 protocol (EtherType `0x0806`) |
| "Every ARP packet is a broadcast" | Requests are; replies are unicast |
| "Routers forward ARP broadcasts" | No: an ARP request never leaves its broadcast domain |
| "Once resolved, always resolved" | Entries expire and are re-verified |
| "ARP is only for Ethernet" | The format supports other hardware types, but Ethernet/Wi-Fi are the common cases |
| "IPv6 uses ARP" | NDP (ICMPv6) does it, using multicast |
| "ARP is secure" | No authentication: spoofing is trivial without switch defenses |
| "Switches use ARP to build their MAC tables" | They learn from source MACs of *any* frame; they don't speak ARP (except for management) |
| "A firewall on the host blocks ARP like ping" | Ordinary host firewalls don't filter ARP; use `arptables`/`nft` bridge/ARP families |

---

## 14. Summary

- **ARP** maps a **local IP → MAC**: a **broadcast request** ("who has X?") and a **unicast reply** ("X is at MAC"), 28-byte packets carried in Ethernet type **0x0806**.
- The requester **queues** its packet, asks, **caches** the answer; the responder **learns** the requester; switches **learn** source MACs; bystanders ignore.
- For destinations **outside the subnet**, the host ARPs for the **default gateway**; IP destination stays, MAC destination becomes the next hop.
- Caches have **states and timeouts** (`ip neigh`); special forms: **gratuitous ARP**, **probes**, **proxy ARP**.
- No authentication → **ARP spoofing**; defend with **DAI + DHCP snooping**, port security, encryption.
- IPv6 uses **NDP** with multicast. Docker bridges, macvlan, VMs and Kubernetes all rely on ARP/NDP inside their L2 domains.

---

## 15. Check your understanding

1. Why does a host need ARP if it already has the destination IP?
2. Why is a request broadcast, but a reply unicast?
3. What does host A ARP for when talking to `8.8.8.8`?
4. Does the router forward A's ARP request to other networks? Why not?
5. What does the target learn from the request? What does each switch learn?
6. Describe INCOMPLETE, REACHABLE, STALE and FAILED.
7. What is a gratuitous ARP and when is it useful?
8. How does ARP spoofing work, and which two switch features defend against it?
9. What replaces ARP in IPv6?

<details>
<summary>Answers</summary>

1. The Ethernet frame needs a destination **MAC**; the switch and NIC only understand MACs.
2. The requester doesn't know who owns the IP, so everyone must hear it; the owner already learned the requester's MAC from the request and can answer directly.
3. The **default gateway** (`192.168.1.1`), because `8.8.8.8` is not on the local network.
4. No. Routers don't forward broadcasts: each interface is its own broadcast domain. That is why ARP is local only.
5. The target learns the sender's IP→MAC (from the request). Each switch learns the sender's MAC→port (source address) and floods the broadcast.
6. INCOMPLETE: request sent, awaiting reply. REACHABLE: recently confirmed. STALE: timer expired, still used but re-verified on next use. FAILED: no answer after retries.
7. A host announces its own IP↔MAC without being asked; useful after address changes, VIP failover (VRRP/keepalived) and VM migration.
8. The attacker sends forged replies mapping the gateway's IP to its own MAC (and vice versa), becoming a man-in-the-middle. Dynamic ARP Inspection (with DHCP snooping) and port security defend.
9. NDP (Neighbor Solicitation/Advertisement in ICMPv6 using solicited-node multicast).
</details>

**Practice**

1. Run the lab and produce a table of all ARP packets (src/dst MAC, op, sender/target IP) plus each host's cache afterwards.
2. Capture ARP on your real interface while flushing the neighbor table; identify your gateway's MAC and confirm it equals the router's label MAC.
3. Give two hosts the same IP in namespaces and use `arping -D` and `ip neigh` to see the conflict.
4. Set a wrong `/16` mask on one namespace and enable `proxy_arp` on the router namespace to see proxy ARP work (Chapter 33 lab).
5. On a Docker host, run two containers, show each one's `ip neigh`, and confirm the MACs against `docker inspect`.

---

**Next:** [Chapter 40 – Multiple NICs in a Single Computer](40_multiple_nics_in_single_computer_in_details.md)
