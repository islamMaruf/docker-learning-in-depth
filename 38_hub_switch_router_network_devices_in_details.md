# Chapter 38: Hub, Switch and Router

> **In one sentence:** A **hub** repeats every signal out of every port (Layer 1, no intelligence), a **switch** learns which MAC lives on which port and forwards frames only where needed (Layer 2), and a **router** forwards packets between different IP networks (Layer 3); a home "router" is really a router, a switch and a Wi-Fi access point in one box.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~50 minutes

**Prerequisites:** Chapters [31](31_data_link_layer_frame_in_details.md) (frames/MACs/switching basics), [32](32_first_computer_and_first_router_in_details.md) (first router) and [33](33_subnetting_and_subnet_masks_in_details.md) (masks).

---

## What you will learn

- What **collision domains** and **broadcast domains** are, and why they matter
- How a **hub** works (and why nobody sells them any more)
- How a **switch** builds its **MAC address table (CAM)** by learning, and the exact **learn / forward / flood / filter** algorithm
- How a **router** decides, and the **same-network vs different-network** rule that every host applies
- **Managed switch** features: VLANs, trunks, STP, port mirroring, PoE, port security; **Layer 3 switches**
- The **home router** = several devices in one, and where modem/firewall/AP fit
- A **hands-on lab** that builds a switch with a Linux bridge, watches it learn, and turns it into a hub
- Docker's bridge = a software switch; troubleshooting patterns

---

## 1. Two important vocabulary words

| Term | Meaning | Why care |
|---|---|---|
| **Collision domain** | The set of devices whose transmissions can **collide** if they talk at the same time (shared medium) | Smaller = faster; collisions only matter on shared/half-duplex media |
| **Broadcast domain** | The set of devices that receive each other's **broadcast** frames (`ff:ff:ff:ff:ff:ff`) | Larger = more ARP/DHCP noise; one IP subnet ≈ one broadcast domain |

| Device | OSI layer | Collision domains | Broadcast domains |
|---|---|---|---|
| **Hub** | 1 | **One** for all ports | One |
| **Switch** (unmanaged) | 2 | **One per port** (full duplex → effectively none) | **One** (all ports) |
| **Switch with VLANs** | 2 | One per port | **One per VLAN** |
| **Router** | 3 | One per interface | **One per interface** (broadcasts stop here) |

---

## 2. The hub: a "dumb" repeater

A **hub** is a multi-port repeater: whatever electrical signal arrives on one port is **copied out of every other port**. It doesn't read frames, addresses or anything else.

```
        PC-A ─┐          ┌─ PC-B
              ├─  HUB  ──┤
        PC-C ─┘          └─ PC-D

A → B: the hub sends the signal to B, C and D. C and D hear it too and discard it (wrong MAC).
```
Consequences:
- **Everyone shares the bandwidth.** A 100 Mbit/s hub with 4 PCs gives them ~100 Mbit/s *in total*, not each.
- **Half duplex only:** a device can't send and receive at the same time; two simultaneous senders **collide** and both back off (**CSMA/CD**, Chapter 31), so busy hubs slow to a crawl.
- **No privacy:** any host with a NIC in promiscuous mode can read all traffic (easy sniffing).
- **No filtering, no learning, no management.**

Hubs vanished in the 2000s when switches became cheap. You will only meet them in textbooks, old labs, and (deliberately) in some tap/monitoring setups. The lesson they teach is what *every other device improves on*.

---

## 3. The switch: learn, then forward

A **switch** (a *multi-port bridge*) reads the **destination MAC** of each frame and sends it **only out the port where that MAC lives**. To know which port, it builds a **MAC address table** (also called **CAM table**, *content-addressable memory*, or forwarding database/FDB) by watching **source MACs**.

### The learning algorithm

For every frame arriving on port *P*:

```
1. LEARN   : record (source MAC → port P, timestamp) in the table  (refresh if it exists)
2. LOOK UP : find the destination MAC in the table
3. DECIDE  :
     dst is broadcast (ff:ff:ff:ff:ff:ff) or multicast   → FLOOD out all ports except P
     dst found on port Q, Q ≠ P                           → FORWARD only to Q
     dst found on port P (same port it came from)         → FILTER (drop; sender already reached it)
     dst NOT in table (unknown unicast)                   → FLOOD out all ports except P
```
Entries **age out** (typically **300 seconds** = 5 min by default) so moved or dead devices are forgotten.

### Worked example (four PCs on ports 1–4; the table starts empty)

| # | Frame | Learn | Decision | Result |
|---|---|---|---|---|
| 1 | A→B (A on port 1) | `A → 1` | B unknown | **Flood** to 2, 3, 4 (C and D drop it) |
| 2 | B→A (B on port 2) | `B → 2` | A known: port 1 | **Forward to 1 only** |
| 3 | A→B | (refresh A) | B known: port 2 | **Forward to 2 only**; C and D are not disturbed |
| 4 | C→D (C on 3) | `C → 3` | D unknown | Flood |
| 5 | D→C | `D → 4` | C known: 3 | Forward to 3 |
| 6 | A→ff:ff:… (ARP) | | broadcast | **Flood** always |

After step 5:

```
MAC table:   A → port 1    B → port 2    C → port 3    D → port 4
```
A↔B and C↔D can now talk **simultaneously** (each pair uses different ports, full duplex): the total capacity is the sum of the ports, not one shared wire.

### Switch vs hub

| | Hub | Switch |
|---|---|---|
| Sends a frame to | everyone | the right port (after learning) |
| Simultaneous conversations | 1 | many |
| Duplex | half | **full** |
| Collisions | frequent | none in full duplex |
| Sniffing others' traffic | trivial | needs tricks (MAC flooding, ARP spoofing, mirror port) |
| Broadcast domain | one | one (same!) |

**A switch does not reduce broadcasts**: it still floods them to all ports of the VLAN. To split broadcast domains you need VLANs or a router.

### Forwarding modes (inside the switch)
- **Store-and-forward:** receive the whole frame, check the FCS, then forward (drops corrupted frames; most common).
- **Cut-through:** start forwarding once the destination MAC (first 6 bytes) is read; lower latency, may forward bad frames (data-center/low-latency switches).

### Managed switches: what else they do

| Feature | Purpose |
|---|---|
| **VLANs (802.1Q)** and **trunk** ports | Split one physical switch into several broadcast domains; carry many VLANs over one link with tags (Chapter 31) |
| **STP / RSTP** (Spanning Tree) | Blocks redundant links so cabling loops don't create **broadcast storms** |
| **Link aggregation (LACP, 802.3ad)** | Bundle links for bandwidth/redundancy |
| **Port mirroring (SPAN)** | Copy traffic to a monitoring port for Wireshark/IDS |
| **Port security** | Limit MACs per port; block unknown devices |
| **DHCP snooping, Dynamic ARP Inspection** | Defend against rogue DHCP/ARP spoofing (Chapters 35, 39) |
| **PoE (802.3af/at/bt)** | Power over Ethernet for access points, cameras, phones |
| **QoS** | Prioritize voice/video |
| **Layer 3 switch** | A switch with routing between VLANs (switched virtual interfaces, SVIs) in hardware |

### Attacks and limits
- **MAC flooding (CAM overflow):** an attacker sends frames with thousands of fake source MACs, filling the table; the switch then floods unknown unicast like a hub, exposing traffic. Defense: port security.
- The table has finite size (thousands to hundreds of thousands of entries).
- **Loops:** without STP a loop turns broadcasts into an endless storm.

---

## 4. The router: connecting networks

A **router** has an interface in **each** network and forwards **IP packets** between them using its **routing table** (Chapter 42):

```
Network 192.168.1.0/24                  Network 10.0.0.0/24
   PC-A  PC-B   ── [switch] ── eth1:192.168.1.1  ROUTER  eth2:10.0.0.1 ── [switch] ── Server
```
Key differences from a switch:

| | Switch | Router |
|---|---|---|
| Looks at | Ethernet **MAC** | **IP** address |
| Table | MAC table (learned automatically) | Routing table (configured or learned via routing protocols) |
| Forwards broadcasts? | Yes (within the VLAN) | **No** |
| Rewrites the frame? | No | Yes: new L2 header per hop; TTL−1; recompute IP checksum |
| Separates | Collision domains | **Broadcast domains** and networks |
| Extras | VLAN, STP | NAT, firewall, DHCP relay/server, VPN, routing protocols (OSPF, BGP) |

### The home router = several devices in one

```
┌────────────────────────── the plastic box from your ISP / shop ─────────────────────────┐
│  [Modem / ONT]  ── WAN port ──►  ROUTER  ◄── internal link ──►  4-port SWITCH ── LAN 1-4 │
│  (sometimes a separate box)      NAT · firewall · DHCP · DNS forwarder                   │
│                                        └───────►  Wi-Fi ACCESS POINT (radios)            │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```
| Component | Role | Layer |
|---|---|---|
| Modem/ONT | Converts DOCSIS/DSL/fiber signals to Ethernet | 1 |
| Router | Forwards between LAN (`192.168.1.0/24`) and WAN (ISP) | 3 |
| Switch | Connects LAN ports (and the AP) into one LAN | 2 |
| Access point | Bridges 802.11 Wi-Fi to that Ethernet LAN | 1-2 |
| DHCP server, DNS forwarder | Configure/serve clients | 7 |
| NAT + stateful firewall | Share one public IP, block unsolicited inbound | 3-4 |

So the "LAN ports" are switch ports, the "WAN/Internet port" is the router's outside interface, and Wi-Fi and Ethernet devices share one subnet.

---

## 5. The rule every host follows: same network or different network?

Before sending, a host compares **destination AND mask** with **its own network** (Chapter 33):

```
Host A: 192.168.1.10/24, gateway 192.168.1.1
```
**To B = 192.168.1.20 (same network):**
```
Layer 3: src 192.168.1.10 → dst 192.168.1.20
ARP:     "who has 192.168.1.20?" → B's MAC (bb:bb:...)
Layer 2: src MAC A, dst MAC B          ← the frame goes directly, via the switch only
```
**To S = 10.0.0.5 (different network):**
```
Layer 3: src 192.168.1.10 → dst 10.0.0.5    (IP addresses stay end-to-end, apart from NAT)
ARP:     "who has 192.168.1.1?"  (the GATEWAY, not the server)
Layer 2: src MAC A, dst MAC of the ROUTER's interface   ← router receives it, re-frames it on the other side
```

| Devices involved | Same network | Different network |
|---|---|---|
| Switch(es) | Yes: forwards by MAC | Yes on each side, to reach the router |
| Router | **Never sees the traffic** | Yes: forwards, TTL−1 |
| ARP target | The destination host | The default gateway |

Golden rule: **the IP destination is the final host; the MAC destination is only the next hop.**

---

## 6. Lab: build a switch, watch it learn, and turn it into a hub

Uses Linux namespaces and a **bridge** (a software switch). Needs `sudo`, `iproute2`, `iputils-ping`, `tcpdump`. Nothing touches your real network.

```bash
# a switch and three "PCs"
sudo ip link add br0 type bridge
sudo ip link set br0 type bridge ageing_time 30000     # (in 1/100 s → 300 s; the default)
sudo ip link set br0 up
for i in 1 2 3; do
  sudo ip netns add pc$i
  sudo ip link add p$i type veth peer name e$i        # p$i stays on the "switch", e$i goes to the PC
  sudo ip link set e$i netns pc$i
  sudo ip link set p$i master br0; sudo ip link set p$i up
  sudo ip netns exec pc$i ip addr add 192.168.10.$i/24 dev e$i
  sudo ip netns exec pc$i ip link set e$i up
done

# 1. Table starts empty (apart from the bridge's own entries)
bridge fdb show br br0 | grep -v permanent
# 2. pc1 pings pc2 → both get learned
sudo ip netns exec pc1 ping -c 2 192.168.10.2
bridge fdb show br br0 | grep -v permanent          # dev p1 has pc1's MAC, dev p2 pc2's MAC
sudo ip netns exec pc1 ip -br link; sudo ip netns exec pc2 ip -br link      # compare the MACs

# 3. Prove the switch does NOT copy unicast to bystanders (pc3 is silent)
sudo ip netns exec pc3 tcpdump -nn -e -i e3 -c 10 &     # watch on pc3's wire
sleep 1
sudo ip netns exec pc1 ping -c 3 192.168.10.2
sleep 3; sudo pkill tcpdump        # pc3 saw only the first ARP broadcast (and no ICMP), because the table knows pc2's port
```
Now **make the switch behave like a hub** by forgetting everything immediately, so every frame is "unknown unicast" and gets flooded:

```bash
sudo ip link set br0 type bridge ageing_time 0           # entries expire instantly → flood always
sudo ip netns exec pc3 tcpdump -nn -e -i e3 -c 6 icmp &
sleep 1
sudo ip netns exec pc1 ping -c 3 192.168.10.2
sleep 3; sudo pkill tcpdump                              # pc3 now SEES pc1↔pc2 pings: hub behavior
sudo ip link set br0 type bridge ageing_time 30000       # restore
```
(Some kernels handle very small ageing values differently; if the effect isn't visible, disable learning per port: `sudo bridge link set dev p2 learning off` and `sudo bridge link set dev p2 flood on`.)

Now add **a router** and a second LAN (Chapter 32's lab) to see broadcasts stay inside their domain:

```bash
sudo ip netns add r; sudo ip link add r0 type veth peer name rp; sudo ip link set r0 netns r
sudo ip link set rp master br0; sudo ip link set rp up
sudo ip netns exec r ip addr add 192.168.10.254/24 dev r0; sudo ip netns exec r ip link set r0 up
sudo ip netns exec r tcpdump -nn -i r0 -c 3 arp &        # the router hears the ARP broadcasts on THIS LAN...
sleep 1; sudo ip netns exec pc1 arping -c 1 192.168.10.3; sleep 2; sudo pkill tcpdump
# ...but they never cross to any other interface of r (routers don't forward broadcasts)
```

Clean up:
```bash
for i in 1 2 3; do sudo ip netns del pc$i; done; sudo ip netns del r
sudo ip link del br0     # removes p1..p3 and rp
```

**Docker connection:** `docker0` (and each `br-<id>`) is exactly this software bridge; each container's `eth0` is one end of a veth pair whose other end is a port on the bridge. Check with `bridge link`, `bridge fdb show br docker0`, `ip -br link | grep veth`. Containers on the same bridge talk at Layer 2 (a MAC table lookup); to reach the outside world, packets go to the bridge's IP (the "router" = the host with NAT).

---

## 7. Comparison table

| | **Hub** | **Switch** | **Router** |
|---|---|---|---|
| Layer | 1 | 2 | 3 |
| Forwarding unit | bits/signals | **frames** | **packets** |
| Decision based on | nothing | destination **MAC** | destination **IP** |
| Learning | none | source MACs (automatic) | routes (static/dynamic protocols) |
| Collision domains | 1 | 1 per port | 1 per interface |
| Broadcast domains | 1 | 1 (per VLAN) | 1 per interface |
| Duplex | half | full | full |
| Security/features | none | VLAN, STP, port security… | NAT, firewall, VPN, ACLs, routing protocols |
| Typical use today | (obsolete) | connecting devices in a LAN | connecting LANs, internet gateway |
| Cost/complexity | trivial | low–medium | medium–high |

---

## 8. Design thinking: why three devices?

- **Efficiency:** a switch's learning table gives each conversation its own path; a hub wastes capacity.
- **Scalability:** a router keeps broadcast domains small: a 10,000-host flat network would drown in ARP and DHCP chatter.
- **Security and control:** routers (and firewalls) are the natural points to apply policy between networks; switches are the place for port-level controls.
- **Fault isolation:** a loop or broadcast storm is contained to its VLAN/network.

Real designs often use a **three-tier** or **leaf-spine** topology: access switches → distribution/aggregation (L3 switches) → core routers, with routing used to interconnect subnets/VLANs.

---

## 9. Troubleshooting

| Problem | Checks |
|---|---|
| **Can't reach a host on the same network** | Same subnet/mask? `ip neigh`/`arp -n` shows `FAILED`/`INCOMPLETE`? VLAN/port mismatch? Switch port up (link light, `ethtool`)? Firewall on the host? Cable/duplex |
| **Host sends local traffic to the router** (`ip route get 192.168.1.20` shows `via 192.168.1.1`) | Wrong mask (`/32` instead of `/24`, or a typo): the host thinks neighbors are remote |
| **Network slow, lights blinking constantly** | Broadcast storm/loop (missing STP), too many hosts in one VLAN, a chatty device; check switch counters, `tcpdump -nn broadcast` |
| **Random disconnects, flapping MACs** | Duplicate MACs/IPs, a loop, bad cable, PoE overload; look at the MAC table for the same MAC on two ports |
| **No internet but LAN works** | Router WAN/ISP/NAT/DNS; `ping <gateway>`, `ping 1.1.1.1`, `ping example.com` (Chapter 32) |
| **Sees others' traffic in Wireshark** | Hub/wireless/mirror port, or MAC flooding attack |
| **Wi-Fi devices can't see wired ones** | Client isolation, guest network separation, different VLANs/subnets |
| **Two "routers" chained** | Double NAT: port forwarding fails; put the second one in bridge/AP mode or connect via its LAN port, with DHCP disabled |

Commands: `ip -br link`, `ip neigh`, `bridge fdb`, `bridge link`, `ethtool -S eth0 | grep -i -E 'err|drop'`, `tcpdump -nn -e`, on switches `show mac address-table`, `show interfaces status`, `show spanning-tree`.

---

## 10. Common misconceptions

| Misconception | Reality |
|---|---|
| "A switch and a hub are basically the same" | A hub repeats to all; a switch learns and sends selectively |
| "A switch stops broadcasts" | It floods them; only routers (or VLANs) split broadcast domains |
| "A router and a switch are interchangeable" | Different layers, tables and jobs |
| "The 'router' from the ISP is only a router" | It's router + switch + Wi-Fi AP (+ modem) |
| "Switches can't be sniffed" | With MAC flooding, ARP spoofing or mirroring they can |
| "Switch ports have IP addresses" | Unmanaged switches have none; managed ones have one management IP |
| "Routers look at MAC to choose the route" | They look at the destination IP; MACs are only for the next hop |
| "More switches = slower" | Each hop adds microseconds; loops and broadcast domains are the real problem |
| "Wi-Fi is a hub" | It is a shared half-duplex medium (like a hub in spirit), so airtime is shared, but it uses access control (CSMA/CA) and encryption |

---

## 11. Summary

- **Hub**: Layer 1, repeats everything, one collision domain, half duplex. Obsolete.
- **Switch**: Layer 2, **learns source MACs**, then forwards/filters/floods by destination MAC; full duplex; one broadcast domain per VLAN; managed versions add VLAN, STP, mirroring, port security, PoE.
- **Router**: Layer 3, forwards IP packets between networks using a routing table; stops broadcasts; adds NAT/firewall/DHCP in home gear.
- Hosts use **IP + mask** to decide same-network (ARP the destination) or different-network (ARP the **gateway**). IP = final target, MAC = next hop.
- A home router bundles modem, router, switch and Wi-Fi AP. Docker's `docker0` is a software switch; the host acts as its router.

---

## 12. Check your understanding

1. What is the difference between a collision domain and a broadcast domain? How many of each does a hub have? A switch with 8 ports?
2. Describe the four things a switch can do with a frame.
3. Which MAC does the switch record, source or destination, and why?
4. What happens to unknown unicast frames? To broadcasts?
5. When PC-A talks to a server on another subnet, what are the source/destination MAC and IP on the first hop?
6. Why is a home "router" really several devices?
7. What is a MAC flooding attack and what defends against it?
8. In the lab, why did pc3 see pc1↔pc2 traffic only after `ageing_time 0`?

<details>
<summary>Answers</summary>

1. Collision domain: devices that can interfere on shared medium; broadcast domain: devices reached by a broadcast. Hub: 1 and 1. 8-port switch: 8 collision domains, 1 broadcast domain (per VLAN).
2. Learn (record the source), forward to a known port, filter (same port), flood (unknown or broadcast).
3. The **source** MAC on the ingress port, because it shows where that device is reachable.
4. Both are flooded out all ports except the ingress one (broadcasts always; unknown unicast until learned).
5. MAC: src A, dst the router's LAN-side MAC (ARP for the gateway). IP: src A, dst the server (unchanged).
6. It combines routing, switching, Wi-Fi AP, DHCP/DNS services, NAT/firewall, sometimes a modem.
7. Flooding with fake source MACs to fill the CAM table so the switch floods everything; port security (MAC limits) defends.
8. With zero ageing the table forgot pc2's port immediately, so frames to pc2 were "unknown unicast" and flooded to every port, including pc3's, just like a hub.
</details>

**Practice**

1. Run the lab; capture `bridge fdb show` before and after each ping; explain each entry.
2. Add a 4th PC on a second bridge linked by a veth "trunk" and watch how MACs are learned across two switches.
3. Enable STP on `br0` (`ip link set br0 type bridge stp_state 1`), create a loop with two veth links between two bridges, and observe the blocked port with `bridge link`.
4. Draw your home network (modem, router, switch, AP, devices) and label each device's layer and each domain.
5. On your Docker host, list the bridges, their ports (`bridge link`), and the MAC table entries for your containers.

---

**Next:** [Chapter 39 – Networking Inside a Network: ARP](39_networking_inside_a_network_arp_protocol_in_details.md)
