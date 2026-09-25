# Chapter 40: Multiple NICs in a Single Computer

> **In one sentence:** Almost every real computer has **more than one network interface** (Ethernet + Wi-Fi + loopback + VPN + Docker bridge + more), each connected to a different network with its own IP address, mask and possibly gateway, which raises the question every OS must answer for **every single packet**: "*which interface should this go out of?*" (the answer is the **routing table**, the topic of Chapters 42–43).

**Level:** 🟡 Intermediate · **Reading time:** ~45 minutes

**Prerequisites:** Chapters [32](32_first_computer_and_first_router_in_details.md) (NIC basics), [33](33_subnetting_and_subnet_masks_in_details.md) (masks) and [39](39_networking_inside_a_network_arp_protocol_in_details.md) (ARP).

---

## What you will learn

- How a computer gets **several NICs** (PCIe/PCI, USB, onboard, Wi-Fi, and **virtual** ones)
- What a computer looks like when it is **connected to several networks at once** ("multihomed" / "dual-homed")
- How **DHCP runs independently on each interface**
- **Why choosing the right interface is a real problem**, and what happens when the wrong one is used
- Why you can't just "use the first NIC": the OS's dilemma, previewing the routing table
- **Real-world use cases**: firewalls, servers with management networks, VPNs, laptops, Docker hosts, hypervisors
- **Hands-on:** build a two-NIC host in namespaces and watch it pick the right interface

---

## 1. Review: everything so far had one NIC

In earlier chapters our computer had **one network card, connected to one network** (`192.168.1.0/24`) with one gateway. The decision "which NIC?" never arose, because there was only one candidate:

```
Computer ── eth0 (192.168.1.10/24) ── switch ── router (192.168.1.1) ── Internet
```
That's the simple case, and it's worth knowing that the rules you learned (the four settings, ARP for the next hop, mask-based local/remote decision) are unchanged. Multiple NICs add **only one new question**: *which of my interfaces?*

---

## 2. Where do extra NICs come from?

### 2.1 Buses: how NICs connect to the computer
A NIC is a hardware device that sits on the computer's internal **bus**:

| Bus | What it is | NIC examples |
|---|---|---|
| **PCI Express (PCIe)** | The modern high-speed serial bus connecting the CPU to devices (its ancestor is **PCI**, Peripheral Component Interconnect) | Onboard Ethernet chips, add-in NICs (1/10/25/100 Gbit/s), M.2 Wi-Fi cards |
| **USB** | External plug-and-play bus | USB Ethernet adapters, USB Wi-Fi sticks, phone tethering (RNDIS/CDC-ECM) |
| **Thunderbolt** | PCIe over a cable | Fast external NICs |
| **SoC-integrated** | Ethernet/Wi-Fi inside the chip | Phones, Raspberry Pi (USB-attached internally on some models) |

Analogy (PCI as roads): the motherboard is a city; PCIe is the highway system connecting neighborhoods (CPU, memory, GPU, storage, NICs) — a NIC is a building on that road with a door (the RJ-45 port or antenna) to the outside world. You can build more buildings (add cards) as long as the road has slots (lanes).

Each PCIe device is identified by an address `bus:device.function` such as `03:00.0`:

```bash
lspci | grep -i -E 'ethernet|network'
# 00:1f.6 Ethernet controller: Intel Corporation Ethernet Connection (7) I219-V
# 02:00.0 Network controller: Intel Corporation Wi-Fi 6 AX200
lspci -k -s 02:00.0            # shows "Kernel driver in use: iwlwifi"
```
The OS then creates a **network interface** per NIC port (a "name" you can use with commands; Chapter 41).

### 2.2 Typical counts

| Machine | Typical interfaces |
|---|---|
| **Desktop** | 1 Ethernet (sometimes 2), maybe a Wi-Fi/Bluetooth card, loopback |
| **Laptop** | Wi-Fi, Ethernet (or USB dongle), loopback, VPN, virtualization/Docker bridges |
| **Server** | 2–8+ ports (management/BMC, data, storage, backup networks), often bonded |
| **Firewall / router** | 2+ dedicated ports (WAN, LAN, DMZ, ...) |
| **Docker host** | Physical NIC + `lo` + `docker0` + one `br-…` per network + one `veth…` per container |
| **Kubernetes node** | Physical NICs + CNI bridge/tunnel devices (`cni0`, `flannel.1`, `cali…`) |

You already have several without doing anything: `ip -br addr` on a normal Linux desktop usually lists `lo`, your Ethernet, Wi-Fi and, if Docker/VMs are installed, `docker0`/`virbr0`.

### 2.3 Virtual NICs
The OS can create **software interfaces** that look like NICs to applications: loopback `lo`, `veth` pairs, bridges, `tun/tap` (VPNs), `wg0` (WireGuard), VLAN sub-interfaces (`eth0.10`), bonds (`bond0`). They participate in routing exactly like hardware ones (Chapter 41).

---

## 3. A computer connected to several networks

Example: a workstation with **three NICs**, each attached to a *different* network:

```
                      ┌──────────────── 192.168.1.0/24 ───────── router R1 (192.168.1.1) ─── Internet
                      │
   ┌───────────┐  NIC1 ┤
   │ Computer  │  NIC2 ┼──────────────── 192.168.2.0/24 ───────── router R2 (192.168.2.1) ─── Office network
   │           │  NIC3 ┤
   └───────────┘       └──────────────── 192.168.3.0/24 ───────── router R3 (192.168.3.1) ─── Lab equipment
```
Each NIC has its **own MAC, its own IP address and its own mask**:

| | NIC1 | NIC2 | NIC3 |
|---|---|---|---|
| Network | 192.168.1.0/24 | 192.168.2.0/24 | 192.168.3.0/24 |
| MAC | `aa:aa:aa:00:00:01` | `aa:aa:aa:00:00:02` | `aa:aa:aa:00:00:03` |
| IP | 192.168.1.10 | 192.168.2.10 | 192.168.3.10 |
| Mask | 255.255.255.0 | 255.255.255.0 | 255.255.255.0 |
| Gateway | 192.168.1.1 | 192.168.2.1 | 192.168.3.1 |

A computer connected to several networks is called **multihomed** (with exactly two: **dual-homed**). Each connection is a completely independent Layer-1/2/3 attachment.

### DHCP happens once per interface
Every interface runs its own DHCP conversation (Chapters 35–37) with the DHCP server **on its own network**:

```
NIC1: Discover ─► R1's DHCP ─► Offer/Request/ACK: 192.168.1.10/24, gateway 192.168.1.1, DNS ...
NIC2: Discover ─► R2's DHCP ─► ... 192.168.2.10/24, gateway 192.168.2.1 ...
NIC3: Discover ─► R3's DHCP ─► ... 192.168.3.10/24, gateway 192.168.3.1 ...
```
The result: three addresses, three masks, and (because each DHCP server tells its client "your default gateway is me") **three default-gateway offers**, but a computer normally keeps **only one default route** in effect (the others are demoted by *metric*, Chapter 42). Similarly DNS servers from all leases may be merged or the "best" interface's used.

```bash
ip -br addr
# lo      UNKNOWN 127.0.0.1/8
# eth0    UP      192.168.1.10/24
# eth1    UP      192.168.2.10/24
# eth2    UP      192.168.3.10/24
```

---

## 4. The problem: which interface?

Now an application on this computer opens a connection to `192.168.3.20`. The kernel builds a packet: source IP `?`, destination `192.168.3.20`. It must hand the packet to **one** NIC:

```
Options:  NIC1 (net 192.168.1.0/24)   NIC2 (net 192.168.2.0/24)   NIC3 (net 192.168.3.0/24)
                    ?                          ?                          ?
```
- If the packet goes out **NIC3**: the ARP request for `192.168.3.20` (or for the gateway `192.168.3.1`) reaches the right LAN: ✅ success.
- If it goes out **NIC1**: the frame lands on the wrong network. The ARP request for `192.168.3.20` is broadcast in `192.168.1.0/24` where nobody owns that address; no answer; the packet **dies**. Or, worse, it goes to R1 (the default gateway) which either doesn't know that private network, discards it, or sends it to the Internet where private addresses are blackholed.

```
Wrong choice:   App → kernel → NIC1 → LAN 192.168.1.x → ARP "who has 192.168.3.20?" ... silence → timeout
Right choice:   App → kernel → NIC3 → LAN 192.168.3.x → ARP → reply → delivered
```
### Why it is hard
- The **application doesn't choose**: it only knows the destination IP and port (and usually not even which interfaces exist).
- **Layers above** (TCP/UDP) don't know about interfaces either; they hand a segment to IP.
- **Every destination** could be reachable via a different interface: local networks, remote networks through one of several gateways, VPN-only ranges, the Internet.
- The choice also determines the **source IP** the packet carries (the address of the outgoing interface) and which **gateway MAC** to ARP for.
- The decision must be **fast** (millions of packets per second) and **deterministic**.

Two naive ideas fail:
| Idea | Why it fails |
|---|---|
| "Use the first NIC" | Traffic to networks behind other NICs would go the wrong way |
| "Try them all / broadcast on all NICs" | Wasteful, creates duplicates, security risk (leaks a packet into networks it should never reach), breaks ordering |

**The solution:** a table of rules, the **routing table**, that maps *destination network → interface (and next hop)*, searched with the **longest matching prefix**. A first look:

```
Destination        Gateway        Interface
192.168.1.0/24     (direct)       eth0
192.168.2.0/24     (direct)       eth1
192.168.3.0/24     (direct)       eth2
default (0.0.0.0/0) 192.168.1.1   eth0     ← "if nothing else matches, go this way"
```
Destination `192.168.3.20` → matches the third row → `eth2`. Chapter 42 explains how this table is built and read; Chapter 43 walks the whole algorithm.

---

## 5. Lab: a multihomed host in namespaces

We build a host `mh` with **two NICs**, connected to two separate networks (each with a "router"), then watch how it chooses.

```
   net1 10.1.0.0/24                       net2 10.2.0.0/24
   r1 (10.1.0.1) ── eth1 [ mh ] eth2 ── r2 (10.2.0.1)
                 10.1.0.10     10.2.0.10
   + one host on each network:  a (10.1.0.50 on net1), b (10.2.0.50 on net2)
```

```bash
# namespaces: the multihomed host, one "router/LAN" and one server for each network
for n in mh n1 n2; do sudo ip netns add $n; sudo ip netns exec $n ip link set lo up; done

# net1: mh.eth1 <-> n1.e
sudo ip link add eth1 type veth peer name n1e
sudo ip link set eth1 netns mh; sudo ip link set n1e netns n1
sudo ip netns exec mh ip addr add 10.1.0.10/24 dev eth1
sudo ip netns exec n1 ip addr add 10.1.0.50/24 dev n1e
# net2: mh.eth2 <-> n2.e
sudo ip link add eth2 type veth peer name n2e
sudo ip link set eth2 netns mh; sudo ip link set n2e netns n2
sudo ip netns exec mh ip addr add 10.2.0.10/24 dev eth2
sudo ip netns exec n2 ip addr add 10.2.0.50/24 dev n2e
for x in "mh eth1" "mh eth2" "n1 n1e" "n2 n2e"; do set -- $x; sudo ip netns exec $1 ip link set $2 up; done

# 1. Look at what the kernel built automatically from the two addresses
sudo ip netns exec mh ip -br addr
sudo ip netns exec mh ip route
# 10.1.0.0/24 dev eth1 proto kernel scope link src 10.1.0.10
# 10.2.0.0/24 dev eth2 proto kernel scope link src 10.2.0.10

# 2. Ask the kernel which interface it WOULD use, and what source address
sudo ip netns exec mh ip route get 10.1.0.50     # → dev eth1 src 10.1.0.10
sudo ip netns exec mh ip route get 10.2.0.50     # → dev eth2 src 10.2.0.10
sudo ip netns exec mh ip route get 8.8.8.8       # → RTNETLINK answers: Network is unreachable (no default route)

# 3. Prove it with packets
sudo ip netns exec mh ping -c 1 10.1.0.50
sudo ip netns exec mh ping -c 1 10.2.0.50
sudo ip netns exec mh ip neigh                   # each neighbor was learned on the CORRECT interface
```
Now **break** it to feel the problem:

```bash
sudo ip netns exec mh ip route del 10.2.0.0/24 dev eth2        # remove the route to net2
sudo ip netns exec mh ping -c 1 -W 1 10.2.0.50                 # "Network is unreachable": no route, so no NIC is chosen
# Add a WRONG route: send net2 traffic out eth1
sudo ip netns exec mh ip route add 10.2.0.0/24 dev eth1
sudo ip netns exec mh ping -c 1 -W 1 10.2.0.50                 # 100% loss
sudo ip netns exec mh ip neigh | grep 10.2.0.50                # INCOMPLETE/FAILED: it ARPed on the wrong LAN
# repair
sudo ip netns exec mh ip route del 10.2.0.0/24 dev eth1
sudo ip netns exec mh ip route add 10.2.0.0/24 dev eth2
sudo ip netns exec mh ping -c 1 10.2.0.50                      # works again
```
And a **default route** for "everything else": pretend `n1` is the gateway to the world:

```bash
sudo ip netns exec mh ip route add default via 10.1.0.50 dev eth1
sudo ip netns exec mh ip route get 8.8.8.8        # → via 10.1.0.50 dev eth1 src 10.1.0.10
sudo ip netns exec mh ip route get 10.2.0.50      # still eth2 (the more specific route wins)
```
Clean up: `for n in mh n1 n2; do sudo ip netns del $n; done`.

---

## 6. Real-world uses of multiple NICs

| Use case | How multiple NICs help |
|---|---|
| **Firewall / router** | One NIC per zone (WAN, LAN, DMZ); packets are forwarded between them under policy |
| **Server with separate networks** | Management (SSH/BMC), public traffic, **storage** (iSCSI/NFS/Ceph) and **backup** networks isolated for security and performance |
| **Laptop** | Wi-Fi + Ethernet + VPN: the OS picks per destination (corporate ranges via VPN, others direct) |
| **Bonding / teaming / LACP** | Combine NICs for **redundancy** or bandwidth; the OS sees one `bond0` |
| **Hypervisors** | Physical NICs plus virtual switches; each VM has virtual NICs (bridged, NAT or host-only) |
| **Docker/Kubernetes hosts** | Physical NIC + bridges + veth pairs + overlay tunnels (`vxlan`, `flannel.1`, `cilium_*`) |
| **Multi-WAN** | Two ISPs on two NICs for failover or load sharing (needs policy routing) |
| **Monitoring/IDS probes** | A NIC in promiscuous mode on a mirror port with no IP address at all |
| **Cross-network bridging** | Bridge/route between an isolated lab network and the office (careful: can bypass security zones!) |
| **Development** | An isolated test network (`10.99.0.0/24`) alongside the normal one |

**Security angle:** a dual-homed host can accidentally connect two networks that were meant to stay separate (e.g. a laptop on the office LAN and a guest Wi-Fi, or a jump host on prod and dev). Make sure IP forwarding stays **off** (`net.ipv4.ip_forward=0`) unless the machine is deliberately a router, and use host firewalls.

---

## 7. Docker and containers: the same problem

```bash
ip -br addr             # on a Docker host: eth0/ens…, docker0 172.17.0.1/16, br-… 172.18.0.1/16, vethXYZ (no IP: bridge ports)
ip route
# default via 192.168.1.1 dev eth0
# 172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
# 172.18.0.0/16 dev br-2f… proto kernel scope link src 172.18.0.1
docker exec <container> ip route      # inside a container: default via 172.17.0.1 dev eth0
```
- The **host** is multihomed: to reach a container at `172.17.0.5` it chooses `docker0`; to reach the Internet it chooses `eth0`, purely from the routing table.
- A **container** with one `eth0` has a single default route via the bridge IP. A container attached to **two networks** (`docker network connect`) has two interfaces and two connected routes; the default route belongs to the first/`--network` one, so watch out for "why can't my container reach the other network's subnet from outside?" surprises.
- **Overlapping subnets** are dangerous: if your VPN gives you `172.17.0.0/16` and Docker also uses `172.17.0.0/16`, the routing table can't distinguish them. Change Docker's `default-address-pools` (Chapter 34).

---

## 8. Common misconceptions

| Misconception | Reality |
|---|---|
| "A computer has one IP address" | One **per interface** (and often several per interface, IPv4 and IPv6) |
| "The OS tries every NIC" | It selects exactly one egress interface per packet using the routing table |
| "An app picks the NIC" | It usually doesn't; it can *bind* to a source address or interface (`SO_BINDTODEVICE`, `curl --interface`), which influences but doesn't replace routing |
| "Two NICs on the same network double the speed automatically" | Not without bonding/LACP or multipath; the OS uses one at a time per flow |
| "Two default gateways is fine" | Only the best-metric default route is used; a second one is a backup at best, and causes asymmetric routing if misconfigured |
| "A multihomed host forwards traffic between networks" | Only if IP forwarding is on |
| "Each NIC needs a different subnet" | Yes for normal routing; two NICs in the same subnet cause confusing behavior (ARP flux: which NIC answers?) |
| "Virtual interfaces aren't real" | They route, ARP and filter exactly like hardware ones |

---

## 9. Summary

- Computers commonly have **several interfaces** (PCIe/USB/onboard/Wi-Fi + virtual ones); each has its own **MAC, IP, mask**, and runs **its own DHCP**.
- A host attached to several networks is **multihomed**. For every outgoing packet the OS must choose **one interface (and next hop)**; the wrong choice means unreachable or leaked traffic.
- Applications don't decide; a **routing table** maps destination networks to interfaces/gateways using **longest-prefix match** (Chapters 42–43). The kernel builds "connected" routes automatically from each interface's address and mask.
- Only one **default route** is normally active. Watch for overlapping subnets, IP forwarding, and multi-NIC security implications.
- Docker hosts, VPN laptops, servers and firewalls all rely on this same mechanism.

---

## 10. Check your understanding

1. Name three ways a computer can get an extra network interface, including one virtual.
2. A computer has NICs on `192.168.1.0/24`, `192.168.2.0/24`, `192.168.3.0/24`. Which addresses/masks does it have? How many DHCP conversations happened?
3. Why can't the kernel simply "use the first NIC"?
4. What is the source IP of a packet sent out NIC2?
5. What happens if a packet for `192.168.3.20` leaves via NIC1?
6. Which routes does the kernel create automatically when you add `10.2.0.10/24` to `eth2`?
7. Why is it risky for a dual-homed server to have `ip_forward=1`?
8. Why do overlapping subnets (VPN vs Docker) break routing?

<details>
<summary>Answers</summary>

1. PCIe add-in card or onboard chip; USB adapter (or tethering); virtual (bridge, veth, tun/tap, VPN, bond, VLAN interface).
2. One address and one mask per NIC (e.g. `.1.10`, `.2.10`, `.3.10`, all `/24`), and three separate DHCP DORA exchanges, one per network.
3. Destinations live on different networks behind different NICs; using the wrong NIC means the ARP/packet never reaches the right LAN.
4. The IP address assigned to NIC2 (e.g. `192.168.2.10`), typically the interface address chosen for the route.
5. It's ARPed/sent onto the wrong LAN (or to the wrong gateway) and is lost, causing timeouts, or leaks into a network it shouldn't reach.
6. A connected route `10.2.0.0/24 dev eth2 proto kernel scope link src 10.2.0.10`.
7. The host would forward traffic between the networks, possibly bypassing firewalls/segmentation between zones.
8. The table can hold only one best route for a given prefix, so traffic meant for one network goes to the other (or is unreachable).
</details>

**Practice**

1. On your machine list all interfaces, classify each as physical or virtual, and note its IP and mask (`ip -br addr`, `ls -l /sys/class/net`).
2. Run the lab; then add a third namespace/NIC and confirm routes appear.
3. Run `ip route get` for five destinations (your LAN, a Docker container, `8.8.8.8`, `127.0.0.1`, a VPN range) and explain each answer.
4. With `nmcli`, connect Ethernet and Wi-Fi at the same time; compare the `default` routes and their **metrics**.
5. Create a Docker container attached to two networks (`docker network connect`) and inspect its routes.

---

**Next:** [Chapter 41 – Visualizing Multiple NICs in a Single Computer](41_visualizing_multiple_nics_in_single_computer_in_details.md)
