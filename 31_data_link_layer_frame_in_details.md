# Chapter 31: The Data Link Layer and Ethernet Frames

> **In one sentence:** The Data Link layer (Layer 2) moves data across **one link** (one local network) by wrapping each IP packet in a **frame** with **MAC addresses** (who on this link should receive it), a **type** (what is inside) and a **checksum** (was it damaged); switches forward frames by MAC address, and the frame is rebuilt at every router.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~55 minutes

**Prerequisites:** [Chapter 21](21_philosophy_of_osi_model.md) (encapsulation) and [Chapter 30](30_internet_protocol_ip_in_details.md) (IP, MTU, routing).

---

## What you will learn

- What Layer 2 is responsible for, and what a **frame** is (header, payload, trailer)
- The **Ethernet II frame**, byte by byte: preamble/SFD, destination and source **MAC**, **EtherType**, payload (with minimum-size **padding**), **FCS (CRC-32)**
- **MAC addresses**: format, OUI, unicast/multicast/broadcast, locally administered addresses (VMs, containers, phones)
- **VLAN tags (802.1Q)**, jumbo frames, and the frame-size numbers (64, 1518, 1522, 9000)
- **Wi-Fi (802.11)** frames and why they differ (four addresses, ACKs, CSMA/CA)
- Why we need **both** MAC and IP addresses, and how **ARP** ties them together
- How **switches** learn and forward, and how that differs from **routers** (frame rewritten at each hop)
- **Error detection** (CRC), what happens to bad frames, and how **TCP** repairs losses
- The **physical layer** below: real encodings for copper, fiber and radio (and common myths)
- Frames in **Docker** (veth, bridge, MACs) and **VMs**
- Hands-on: `ip link`, `tcpdump -e`, `ethtool`, `bridge`, Python frame-building, Wireshark

---

## 1. What Layer 2 does

IP (Chapter 30) decides **where a packet is going across the world**. It can't put bits on a wire. For each hop, the packet needs a way to get from *this* device to the *next* device on the same physical or wireless network. That is the job of the **Data Link layer**:

| Responsibility | How |
|---|---|
| **Framing**: mark where a message starts and ends | Frames with delimiters/lengths |
| **Local addressing**: identify the next device on this link | **MAC addresses** |
| **Say what's inside** | EtherType (IPv4, IPv6, ARP, ...) |
| **Detect damage** | **FCS**, a 32-bit CRC |
| **Share the medium** | Ethernet (full duplex/CSMA/CD historically), Wi-Fi **CSMA/CA** |
| **Segment traffic** | VLANs, bridges/switches |

Layer 2 does **not** route between networks, guarantee delivery, or retransmit (Ethernet silently drops bad frames; TCP or the application repairs it). Wi-Fi does add link-layer acknowledgments and retries.

### The names again
```
Application data → [TCP hdr | data]  segment → [IP hdr | segment]  packet → [Eth hdr | packet | FCS]  FRAME → bits on the wire
```
A frame's **payload is the entire IP packet**. Layer 2 treats it as opaque bytes.

---

## 2. Where frames fit in a transfer

Send 10 MB over a gigabit Ethernet LAN (typical MTU 1500, TCP MSS 1460):

| Layer | What happens | Numbers |
|---|---|---|
| TCP | Data is cut into segments of ≤ **1460** bytes | 10,485,760 ÷ 1460 ≈ **7,182 segments** |
| IP | Each segment gets a 20-byte header → packet of ≤ **1500** bytes (the MTU) | ≈ 7,182 packets |
| Ethernet | Each packet gets a 14-byte header and 4-byte FCS → frame of ≤ **1518** bytes | ≈ 7,182 frames |
| Physical | + preamble/SFD (8 bytes) + inter-frame gap (12 byte-times) per frame | **1538 byte-times** on the wire per full frame |

**Wire time at 1 Gbit/s:** each full frame occupies 1538 × 8 = 12,304 bit-times = **12.3 µs**, so 7,182 frames take about **88 ms**. That means **TCP goodput on a 1 Gbit/s Ethernet tops out around 941 Mbit/s** (1460 ÷ 1538 of the line rate), a useful number to remember. (Segments are *not* 1 MB each: they are at most one MTU minus headers.)

---

## 3. The Ethernet II frame

```
   ┌─── added by the PHY (not part of "the frame" in size counts) ───┐┌──────────── the Ethernet frame (64–1518 bytes) ────────────┐
   │ Preamble │ SFD │                                                  │ Dst MAC │ Src MAC │ EtherType │      Payload      │   FCS   │
   │  7 bytes │ 1 B │                                                  │ 6 bytes │ 6 bytes │  2 bytes  │  46 – 1500 bytes  │ 4 bytes │
   └──────────┴─────┘                                                  └─────────┴─────────┴───────────┴───────────────────┴─────────┘
                                                                        └──────────── 14-byte header ──────┘                  trailer
```

### 3.1 Preamble and SFD (8 bytes, added by the physical layer)
- **Preamble:** 7 bytes of `10101010` (0xAA). It lets the receiver's clock **lock onto the sender's bit timing** (like a drummer's count-in).
- **SFD (Start Frame Delimiter):** `10101011` (0xAB). The final `11` says "the frame begins right after this byte".
- Analyzers (`tcpdump`, Wireshark) don't show them, and frame sizes are counted **without** them.

### 3.2 Destination MAC (6 bytes) and Source MAC (6 bytes)
A **MAC address** (Media Access Control address, also "hardware", "physical", or "EUI-48" address) is **48 bits**, written as six hex pairs: `a4:83:e7:1c:9b:02` (or `a4-83-e7-1c-9b-02`, or `a483.e71c.9b02` on Cisco).

```
   a4 : 83 : e7    :   1c : 9b : 02
  └─────OUI─────┘    └─ device part ─┘
  vendor (assigned by IEEE)   assigned by the vendor
```

The **first byte** contains two special bits (least-significant bits of that byte):

| Bit | Name | 0 | 1 |
|---|---|---|---|
| bit 0 (LSB) | **I/G** (individual/group) | **Unicast** (one device) | **Multicast/broadcast** (a group) |
| bit 1 | **U/L** (universally/locally administered) | Assigned by the manufacturer (globally unique OUI) | **Locally administered** (set by software) |

So `a4` = `1010 0100`: I/G = 0 (unicast), U/L = 0 (universal). `02` = `0000 0010`: unicast, locally administered.

| Kind | Example | Meaning |
|---|---|---|
| **Unicast** | `a4:83:e7:1c:9b:02` | Exactly one interface |
| **Broadcast** | `ff:ff:ff:ff:ff:ff` | Every device on the local network (ARP requests, DHCP discover) |
| **Multicast** | `01:00:5e:xx:xx:xx` (IPv4 multicast), `33:33:xx:xx:xx:xx` (IPv6 multicast) | A group of interested devices |

**What a NIC does with an arriving frame:** accept it if the destination MAC is **its own**, **broadcast**, or a **multicast group it joined**; otherwise ignore it. (*Promiscuous mode*, used by `tcpdump`, accepts everything it sees.)

Truths about MAC addresses:

- They're **burned in** at manufacture, but **not truly permanent**: operating systems can change them, and **modern phones and laptops randomize** the Wi-Fi MAC per network for privacy.
- **Virtual machines and containers get generated MACs** (locally administered): Docker uses `02:42:` + the container's IPv4 in hex, so a container with `172.17.0.2` has MAC `02:42:ac:11:00:02` (172=0xac, 17=0x11, 0, 2); VirtualBox/VMware/KVM use their own OUIs.
- MAC addresses must be **unique on the same L2 network** (duplicates cause chaos), not globally.
- They are **flat** (no hierarchy), meaningful **only on the local link** and **not routable**.

### 3.3 EtherType (2 bytes)
Says what's in the payload:

| Value | Payload |
|---|---|
| `0x0800` | IPv4 |
| `0x0806` | ARP |
| `0x86DD` | IPv6 |
| `0x8100` | An **802.1Q VLAN tag** follows (section 5) |
| `0x88CC` | LLDP (neighbor discovery) |
| `0x8847` / `0x8848` | MPLS |
| `0x88A8` | 802.1ad (Q-in-Q) |

(In original IEEE 802.3 framing this field was a **length**; values ≤ 1500 mean "length", values ≥ 1536 (0x0600) mean "type", which is why the two coexist. Almost all IP traffic uses Ethernet II with EtherType.)

### 3.4 Payload: 46 to 1500 bytes
- The **1500-byte** maximum is the **MTU** of standard Ethernet (Chapter 30).
- The **46-byte minimum** exists so that the whole frame (14 header + 46 + 4 FCS) is at least **64 bytes**, a legacy of collision detection on shared cables: a sender must still be transmitting when the far end's collision reaches it. Shorter payloads are **padded with zeros** by the sender (the IP header's Total Length lets the receiver strip the padding).

### 3.5 FCS: Frame Check Sequence (4 bytes)
A **CRC-32** computed over the header and payload. The receiver recomputes it; a mismatch means the frame was damaged, so it is **silently discarded** (counted as a CRC/FCS error).

Sizes summary:

| | Bytes |
|---|---|
| Minimum frame (header + payload + FCS, no preamble) | **64** |
| Maximum standard frame | **1518** |
| With one 802.1Q tag | 64–**1522** |
| Jumbo frame | up to ~**9000-byte MTU** (every device on the path must support it) |
| On the wire per frame | + 8 (preamble+SFD) + 12 (inter-frame gap) |

---

## 4. A frame in hex: an ARP request

An **ARP request** ("who has 192.168.1.1? tell 192.168.1.50") is a perfect first frame because it's short and has a *broadcast* destination. Bytes (before FCS and padding):

```
ff ff ff ff ff ff   11 22 33 44 55 66   08 06                       ← Ethernet header (14 bytes)
00 01  08 00  06  04  00 01                                         ← ARP: hw type Ethernet, proto IPv4, hw len 6, proto len 4, opcode 1 (request)
11 22 33 44 55 66   c0 a8 01 32                                      ← sender MAC, sender IP 192.168.1.50
00 00 00 00 00 00   c0 a8 01 01                                      ← target MAC (unknown = zeros), target IP 192.168.1.1
```
That is 14 + 28 = **42 bytes**, less than the 60 bytes minimum (before the FCS), so the NIC **pads with 18 zero bytes**, then appends the 4-byte FCS = the required **64**.

**Build and check it in Python:**

```python
import struct, zlib
dst = bytes.fromhex("ffffffffffff")
src = bytes.fromhex("112233445566")
eth = dst + src + struct.pack("!H", 0x0806)
arp = struct.pack("!HHBBH", 1, 0x0800, 6, 4, 1) + src + bytes([192,168,1,50]) + bytes(6) + bytes([192,168,1,1])
frame = eth + arp
frame += bytes(60 - len(frame))                       # pad to 60 bytes (min frame without FCS)
fcs = zlib.crc32(frame) & 0xFFFFFFFF                  # CRC-32 over header+payload; hardware appends it (transmit bit/byte ordering is handled by the NIC)
print(len(frame), "bytes + 4 FCS =", len(frame)+4, "  crc32 = %08x" % fcs)
print(frame.hex(" "))
```

**Read a real one:**

```bash
sudo tcpdump -i any -nn -e -XX -c 1 arp          # in another terminal: ping a local device you haven't contacted (or: sudo ip neigh flush all)
# ... 11:22:33:44:55:66 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.1.1 tell 192.168.1.50
```
The `-e` flag prints the link-layer (Ethernet) header. `length 42` matches our hand-built frame (the capture doesn't show padding or FCS, because NIC hardware handles them).

A second frame, an **IPv4/ICMP echo**, has EtherType `08 00`, then `45 00 00 54 ...` (the IP header from Chapter 30), then the ICMP message. For the 84-byte IP packet of a default ping the frame is 14 + 84 + 4 = **102 bytes** on the wire, since the payload already exceeds 46 bytes and needs no padding.

---

## 5. VLANs (802.1Q), jumbo frames and other variations

### 5.1 VLAN tagging
A **VLAN** (Virtual LAN) splits one physical switch (or a network of switches) into several **separate broadcast domains**, so departments, guests and servers stay isolated without separate hardware. The frame carries a **4-byte 802.1Q tag** between the source MAC and the EtherType:

```
Dst MAC │ Src MAC │ 0x8100 (TPID) │ PCP(3) DEI(1) VLAN-ID(12) │ real EtherType │ payload │ FCS
                     └───────── 4-byte tag ──────────────────┘
```
- **VLAN ID** is 12 bits → **1–4094** usable VLANs. **PCP** is a 3-bit priority (QoS) field.
- A switch port is either an **access port** (untagged, belongs to one VLAN, for end devices) or a **trunk port** (carries tagged frames of many VLANs between switches or to a router/hypervisor).
- Different VLANs can talk only through a **router** (L3), which is why "inter-VLAN routing" exists. Tagging makes the maximum frame 1522 bytes.
- **Q-in-Q (802.1ad)** stacks two tags (providers).

```bash
ip -d link show          # look for "vlan protocol 802.1Q id 100"
sudo ip link add link eth0 name eth0.100 type vlan id 100     # create a VLAN sub-interface (needs a tagged switch port to be useful)
```

### 5.2 Jumbo frames
A larger MTU (up to ~9000) reduces per-frame overhead for bulk traffic (storage, backups). **Every** device on the path (NICs, switches, VLANs, tunnels) must be configured for it, or you get silent drops and PMTU problems (Chapter 30). Enable with `ip link set eth0 mtu 9000` where supported.

### 5.3 Other Layer-2 protocols
PPP/PPPoE (DSL), HDLC, Frame Relay/ATM (legacy), Wi-Fi 802.11, Bluetooth, cellular data links, and virtual L2 such as **VXLAN** (an Ethernet frame inside a UDP packet: overlay networks in Docker Swarm/Kubernetes, Chapter 40s).

---

## 6. Wi-Fi (802.11) frames

Radio is harder than a cable: signals fade, interfere, and a device **can't listen while it transmits**. So Wi-Fi's link layer is more elaborate:

| | Ethernet (802.3) | Wi-Fi (802.11) |
|---|---|---|
| Medium | Copper/fiber, point-to-point with a switch | Shared radio channel |
| Addresses in header | 2 (dst, src) | **Up to 4** (receiver, transmitter, plus the "real" source/destination when an **access point** relays) |
| Media access | Full duplex on switched links (no collisions) | **CSMA/CA**: listen, wait a random back-off, transmit, expect an **ACK** |
| Delivery | No link ACK; drops silently on error | **Link-layer ACK** for each unicast frame and **retransmissions** |
| Frame types | Data (plus control like pause) | **Management** (beacon, probe, authenticate, associate), **control** (RTS/CTS, ACK), **data** |
| Max payload | 1500 (MTU) | Up to 2304 (MSDU); aggregation (A-MPDU) for efficiency |
| Security | Physical access (802.1X optional) | **Encryption in the link layer**: WPA2 (AES-CCMP), WPA3 |
| Error rate | Very low | Much higher; adaptive modulation lowers rate when the signal is poor |

Address usage (infrastructure mode): a laptop sending to a server via the AP puts the **AP's MAC** as receiver, its own as transmitter, and the **final destination MAC** as the third address; the AP then builds an Ethernet frame for the wired side (the AP is a **bridge** between 802.11 and 802.3).

Extras: frequency bands **2.4 / 5 / 6 GHz**, channel width 20–160 MHz, **OFDM/OFDMA + QAM (up to 1024-QAM in Wi-Fi 6/7)**, **MIMO/MU-MIMO**, the **hidden node problem**, **roaming** between APs, and **airtime fairness**. Wi-Fi throughput is shared by everyone on the channel and drops with distance and interference.

---

## 7. MAC address + IP address: why both, and how ARP glues them

| | IP address | MAC address |
|---|---|---|
| Layer | 3 | 2 |
| Scope | Whole internet (**end-to-end**) | One local link (**hop-by-hop**) |
| Structure | Hierarchical (network + host) → routable/aggregatable | Flat (vendor + serial) |
| Assigned by | DHCP/admin/ISP | Manufacturer (or software) |
| Changes during a packet's trip? | No (except NAT) | **At every router** |

If we only had MACs, routing the internet would need every router to know every device in the world (no hierarchy). If we only had IPs, Layer 2 wouldn't know which physical port, radio, or cable to use. Separation of concerns solves both.

### ARP (Address Resolution Protocol)
Your PC knows the **next hop's IP** (the default gateway `192.168.1.1`), but a frame needs the **MAC**. **ARP** resolves it on the local network (Chapter 39 in depth):

1. Check the **ARP cache** (`ip neigh`).
2. If missing, **broadcast** an ARP request (`dst ff:ff:ff:ff:ff:ff`, EtherType 0x0806): "Who has 192.168.1.1? Tell 192.168.1.50."
3. The owner replies **unicast** with its MAC ("192.168.1.1 is at aa:bb:cc:dd:ee:ff").
4. The sender caches the mapping for a few minutes and sends the frame.

IPv6 uses **Neighbor Discovery** (ICMPv6) instead of ARP.

```bash
ip neigh                     # the neighbor (ARP) cache: IP, dev, MAC, state (REACHABLE, STALE, DELAY, FAILED)
arp -a                       # older tool
sudo ip neigh flush all      # clear it, then ping and capture the ARP exchange
```

---

## 8. Switches and routers handle frames differently

### 8.1 A hub, a bridge and a switch
- **Hub (L1, obsolete):** repeats every bit out of every other port, so one big **collision domain** (half duplex, CSMA/CD).
- **Bridge/Switch (L2):** looks at **destination MAC** and forwards **only where needed**; each port is its own collision domain; full duplex; multiple conversations at once.

### 8.2 How a switch learns and forwards
A switch keeps a **MAC address table** (CAM table): `MAC → port (+ VLAN)`.

```
1. Frame arrives on port 1 with src MAC A → LEARN: "A is on port 1" (entries age out, typically ~5 minutes).
2. Look up the destination MAC B:
     • known, on port 3          → forward ONLY out of port 3 (unless port 3 == arrival port: drop)
     • unknown unicast           → FLOOD out of all ports in the same VLAN except the arrival port
     • broadcast/multicast       → flood (multicast may be pruned by IGMP snooping)
3. When B answers, the switch learns B's port, and later frames go straight there.
```

Switches use fast hardware (ASICs and TCAM) and operate at "wire speed". Managed switches add VLANs, **STP** (below), port security, **mirroring (SPAN)**, link aggregation (LACP), and PoE.

**Broadcast domain vs collision domain:** all ports of a switch (in one VLAN) share one **broadcast domain** (a broadcast reaches all of them); a **router** (or a VLAN boundary) separates broadcast domains.

**Loops and STP:** with redundant links, a broadcast can circle forever, causing a **broadcast storm** that takes down the network. **Spanning Tree Protocol (802.1D, RSTP 802.1w)** blocks redundant ports so the topology is loop-free.

### 8.3 A router rebuilds the frame at every hop

```
Frame 1 (PC → home router):   dst MAC = router's LAN MAC, src MAC = PC's MAC,        IP: 192.168.1.50 → 142.250.185.206, TTL 64
   router: check FCS → strip Ethernet header/trailer → route by IP → TTL-1 → NEW frame:
Frame 2 (router → ISP):       dst MAC = ISP gateway's MAC, src MAC = router's WAN MAC, IP: 203.0.113.50 → 142.250.185.206 (NAT), TTL 63
```
IP header (dest) constant; MACs new at each hop. A **switch never changes the MACs** and never touches the IP header; it forwards the *same* frame.

| | Switch | Router |
|---|---|---|
| Layer / looks at | 2 / Ethernet header (dst MAC, VLAN) | 3 / IP header (dst IP, TTL) |
| Table | MAC table (learned) | Routing table (configured/learned by protocols) |
| Rewrites the frame? | No (may add/remove VLAN tag) | Yes (new L2 header per hop; TTL, checksum) |
| Broadcast handling | Floods within the VLAN | **Doesn't forward** L2 broadcasts |
| Typical speed | Wire speed in hardware | Hardware (ASIC) on core routers; slower in software |

(Modern "Layer 3 switches" do both in hardware.)

---

## 9. Error detection and what happens to bad frames

**Types of errors:** single-bit flips, **burst errors** (interference), truncated frames (cable unplugged mid-frame), and frames that are too short/long ("runts"/"giants").

**CRC-32** (polynomial `0x04C11DB7`, computed in hardware) catches all single-bit errors, all bursts up to 32 bits and virtually every longer error (undetected probability ≈ 2⁻³²). A bad frame is **discarded by the receiving NIC**; nothing is sent back. Counters increase:

```bash
ip -s link show eth0             # RX: errors, dropped, overrun, frame (CRC/alignment errors)
ethtool -S eth0 | grep -iE 'crc|err|drop|collis|fcs|align'    # NIC-specific counters
ethtool eth0                     # speed, duplex, link detected, auto-negotiation
```

**Recovery** happens above: **TCP** notices a missing segment (no ACK or duplicate ACKs) and retransmits (Chapter 23); UDP applications tolerate or handle loss themselves. **Wi-Fi** retransmits at the link layer itself.

**Error counters as diagnostics**

| Symptom (counter) | Typical cause | Fix |
|---|---|---|
| Rising **CRC/FCS/frame errors** | Bad/damaged/too-long cable, bad connector, EMI, failing NIC/SFP, **duplex mismatch** | Replace cable, check length (≤100 m copper), re-terminate, swap ports/NIC, set both ends to auto-negotiate |
| **Late collisions / collisions** on a "switched" link | **Duplex mismatch** (one end forced full, the other half) or a hub | Fix duplex settings |
| **Runts** (<64 bytes) / **giants** (>1518 B, or >1522 with VLAN) | Collisions, faulty NIC, or MTU mismatch (jumbo vs standard) | Align MTUs |
| **Broadcast storm** (huge broadcast counts, switch CPU high, network unusable) | L2 **loop** without STP | Enable STP/RSTP, remove the loop, storm control |
| **MAC flapping** (a MAC keeps moving between ports) | Loop, duplicate MAC, misconfigured VM/NIC teaming | Investigate topology; unique MACs |
| **Link flapping** | Bad cable/port, power saving (EEE) issues | Replace cable/port |
| Packets drop only when large | MTU mismatch or PMTU black hole (Chapter 30) | Check MTU end-to-end |

---

## 10. The physical layer below the frame

Layer 2 hands a frame to **Layer 1**, which turns bits into signals. The details depend on the medium (and are often misdescribed as "high voltage = 1, low = 0"):

| Medium | How bits are sent |
|---|---|
| **10BASE-T** (10 Mbit/s, old) | **Manchester** encoding: every bit has a mid-bit voltage transition (self-clocking). Each bit's *transition direction* encodes 0 or 1 |
| **100BASE-TX** (Fast Ethernet) | 4B/5B block coding + **MLT-3** (three voltage levels) over 2 twisted pairs |
| **1000BASE-T** (Gigabit) | **PAM-5** (five voltage levels) using **all four pairs simultaneously in both directions** with echo cancellation |
| **2.5/5/10GBASE-T** | PAM-16 and advanced signal processing; needs Cat5e/6/6a cabling and shorter runs for 10G |
| **Fiber** (SFP/SFP+/QSFP) | Light: LEDs or **lasers**, on/off keying or advanced modulation; **multimode** (short, up to a few hundred meters) vs **single-mode** (kilometers, 10 km–80+ km); multiple wavelengths on one fiber (WDM) |
| **Wi-Fi** | Radio: **OFDM/OFDMA** subcarriers, each modulated with **BPSK/QPSK/16-/64-/256-/1024-QAM** (more bits per symbol at higher signal quality), 2.4/5/6 GHz, MIMO antennas |
| **Cellular** | Licensed spectrum, OFDM variants (4G/5G) |

Practical rules: copper Ethernet is limited to **100 m** per segment; use **Cat5e or better** for gigabit, **Cat6/6a** for 10G; fiber for long distance or noise immunity; wireless shares airtime. The **preamble** and **inter-frame gap** are physical-layer artifacts that don't appear in captures.

---

## 11. Frames in Docker, containers and VMs

A container's network interface is one end of a **veth pair** (a virtual Ethernet cable) whose other end plugs into a **Linux bridge** (`docker0` or `br-<id>`), a **software switch** doing exactly the learn/flood/forward behavior of section 8.

```
 container A               container B
   eth0 (02:42:ac:11:00:02)   eth0 (02:42:ac:11:00:03)
     │ veth pair                │ veth pair
   vethabc ────┐         ┌──── vethdef
               └── docker0 (bridge = virtual switch, 172.17.0.1) ──── host routing/NAT ──── physical NIC
```

- **Same bridge → pure Layer 2:** A→B frames go through the bridge; ARP resolves B's MAC.
- **To the outside:** frames go to the bridge's own MAC (the gateway), and the **host routes and NATs** the IP packet (Chapter 30).
- **MACs** are generated from IPs (`02:42:` + hex IP).
- **Other Docker network drivers:** **macvlan** gives each container its own MAC directly on your physical LAN (visible to your switch; some Wi-Fi and cloud networks reject multiple MACs on one port), **ipvlan** shares the host's MAC, **overlay** wraps L2 frames in **VXLAN/UDP** between hosts, **host** shares the host's interfaces.
- **VMs:** each VM NIC has a virtual MAC; a **bridged** vNIC appears as a separate device on your LAN; hypervisor "vSwitches" work like switches.
- **Security:** MAC spoofing and **ARP spoofing** attacks work on shared L2 (Chapter 39); enable port security, Dynamic ARP Inspection, and isolation (`icc=false`, separate networks).

---

## 12. Hands-on labs

**Lab 1: See your L2 identity and neighbors**

```bash
ip -br link            # interfaces, state, MAC (e.g. eth0 UP a4:83:e7:1c:9b:02), flags
ip link show eth0      # mtu 1500, "link/ether <MAC> brd ff:ff:ff:ff:ff:ff"
ip neigh               # who you've talked to recently: IP → MAC, state
ethtool eth0 | grep -E 'Speed|Duplex|Link detected|Auto-negotiation'      # (may need sudo; not for Wi-Fi)
iw dev wlan0 link 2>/dev/null                                             # Wi-Fi: SSID, signal, bitrate
```
Decode the first byte of your MAC: is it unicast? universally or locally administered?

**Lab 2: Capture frames with MAC addresses**

```bash
sudo tcpdump -i any -nn -e -c 10                     # -e shows "src MAC > dst MAC, ethertype IPv4 (0x0800), length N"
sudo tcpdump -i eth0 -nn -e -XX -c 1 icmp            # full hex: the first 14 bytes are the Ethernet header
ping -c 1 <gateway>
sudo tcpdump -nn -e -i eth0 broadcast                # only broadcast frames (ff:ff:ff:ff:ff:ff)
sudo tcpdump -nn -e -i eth0 ether multicast          # multicast frames
```

**Lab 3: Watch ARP**

```bash
sudo ip neigh flush all
sudo tcpdump -i eth0 -nn -e arp &
ping -c 1 <gateway-ip>
ip neigh              # the entry is back (REACHABLE)
```
You'll see the broadcast "who-has" and the unicast "is-at" with their MACs, and the frame lengths (42 bytes captured).

**Lab 4: Docker's bridge and MAC scheme**

```bash
docker run -d --name a alpine:3.20 sleep 3600
docker run -d --name b alpine:3.20 sleep 3600
docker inspect -f '{{.NetworkSettings.IPAddress}} {{.NetworkSettings.MacAddress}}' a b       # 172.17.0.2 02:42:ac:11:00:02 ...
ip -br link | grep -E 'docker0|veth'         # the bridge and one veth per container
bridge link                                  # ports attached to docker0
bridge fdb show br docker0 | head            # the bridge's MAC table (learned addresses)
docker exec a sh -c 'ip neigh; ping -c1 172.17.0.3; ip neigh'      # ARP entry for B appears: its MAC is 02:42:ac:11:00:03
sudo tcpdump -i docker0 -nn -e -c 6 &        # then ping again to see the frames (ARP + ICMP)
docker rm -f a b
```

**Lab 5: Build a tiny virtual switch** (Linux, root; uses namespaces so it doesn't touch your real network)

```bash
sudo ip link add br0 type bridge && sudo ip link set br0 up
for n in 1 2 3; do
  sudo ip netns add h$n
  sudo ip link add v$n type veth peer name p$n
  sudo ip link set p$n netns h$n
  sudo ip link set v$n master br0 up
  sudo ip netns exec h$n ip addr add 10.5.0.$n/24 dev p$n
  sudo ip netns exec h$n ip link set p$n up
done
sudo ip netns exec h1 ping -c 2 10.5.0.2          # frames flow through the bridge (a learning switch)
bridge fdb show br br0 | grep -v permanent        # MACs learned per port
sudo ip netns exec h3 tcpdump -nn -e -c 3 -i p3 &  # h3 sees ARP broadcasts but NOT unicast between h1 and h2
sudo ip netns exec h1 ping -c 2 10.5.0.2
# clean up
for n in 1 2 3; do sudo ip netns del h$n; done; sudo ip link del br0
```
Observe: the first frame (ARP broadcast) is **flooded** to everyone; after learning, unicast frames go **only** to the right port, so h3 stops seeing the h1↔h2 traffic. That's how a switch works.

**Lab 6: VLANs in a namespace lab**

```bash
sudo ip netns add v && sudo ip link add vv type veth peer name pv && sudo ip link set pv netns v
sudo ip link set vv up && sudo ip netns exec v ip link set pv up
sudo ip link add link vv name vv.100 type vlan id 100 && sudo ip link set vv.100 up
sudo ip netns exec v ip link add link pv name pv.100 type vlan id 100 && sudo ip netns exec v ip link set pv.100 up
sudo ip addr add 10.100.0.1/24 dev vv.100; sudo ip netns exec v ip addr add 10.100.0.2/24 dev pv.100
sudo tcpdump -i vv -nn -e -c 3 &
ping -c 2 10.100.0.2               # tcpdump shows "vlan 100, p 0, ethertype IPv4" (the 802.1Q tag)
sudo ip netns del v; sudo ip link del vv 2>/dev/null
```

**Lab 7: Frame counters and errors**

```bash
ip -s link show eth0
ethtool -S eth0 2>/dev/null | head -30
watch -n1 "cat /proc/net/dev"              # per-interface bytes/packets/errs/drops
```
Find `errs`, `drop`, `frame`, `colls`; on a healthy wired link they're zero.

**Lab 8: MTU and giant frames**

```bash
ip link show docker0 | grep mtu
ping -c 2 -M do -s 1472 <gateway>        # fits in a 1500-byte MTU
ping -c 2 -M do -s 1473 <gateway>        # too big → error
```

**Lab 9: Wireshark**: capture on your interface; add columns `eth.src`, `eth.dst`, `eth.type`; expand a frame: **Frame → Ethernet II → IPv4 → TCP**; compare `frame.len` with the sum of headers and payload; look at the OUI resolution of vendor names (View → Name Resolution → Resolve physical addresses).

---

## 13. Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| No link ("NO-CARRIER", `Link detected: no`) | Cable, port, switch off, NIC down | `ip link`, `ethtool`, swap cable/port |
| Link up but slow (100 Mbit/s instead of 1 Gbit/s) | Bad cable pairs, negotiation problem | `ethtool eth0 | grep Speed`; replace cable |
| Errors/CRC counters climbing, slow transfers | Bad cable/connector/interference, duplex mismatch | `ethtool -S`, replace cable, auto-negotiate |
| A device is reachable by IP only sometimes | Duplicate IP or duplicate MAC, ARP conflicts | `arping -D -I eth0 <ip>`, `ip neigh` (MAC changes), switch logs |
| Whole network crawls, switch lights blink madly | **Broadcast storm** from a loop | Check for looped cables; enable STP |
| Can reach devices in one VLAN but not another | VLAN misconfig (access/trunk, tagging), no inter-VLAN router | Switch port config; `tcpdump -e` for tags |
| Docker containers on macvlan can't reach the host | Known macvlan limitation (host can't talk to its own macvlan children) | Add a macvlan interface on the host |
| Wi-Fi connects but is slow/unstable | Interference, weak signal, too many devices, DFS channels | `iw dev wlan0 link`, change channel/band |
| "Destination host unreachable" from the local subnet | ARP not resolving (device off, firewall, wrong VLAN) | `ip neigh` shows FAILED/INCOMPLETE |
| Large packets fail, small ones work | MTU mismatch (jumbo, tunnels), PMTU black hole | Lab 8, Chapter 30 |

---

## 14. Common misconceptions

| Misconception | Reality |
|---|---|
| "MAC addresses are used across the internet" | They matter only on one link and are rewritten at every router |
| "A MAC address is permanent and unique forever" | Usually assigned once, but changeable/randomized; VMs/containers use generated ones |
| "The preamble is part of the frame's 64–1518 bytes" | It (and the inter-frame gap) are added by the PHY and not counted |
| "Payload min is 46 bytes because Ethernet needs it" | It ensures the 64-byte minimum frame; shorter data is padded |
| "An Ethernet switch looks at IP addresses" | Plain L2 switches look only at MACs (and VLAN tags) |
| "Switches change the frame" | They forward it unchanged; routers rebuild it |
| "Ethernet retransmits lost frames" | It doesn't; upper layers (TCP) do (Wi-Fi does retransmit at L2) |
| "Ethernet '1' is high voltage, '0' is low voltage" | Real encodings are Manchester, MLT-3, PAM-5 and more |
| "Gigabit Ethernet gives 1000 Mbit/s of TCP data" | About 941 Mbit/s after headers, preamble and gaps |
| "Wi-Fi is just wireless Ethernet" | Different framing, ACKs, up to four addresses, encryption at L2, shared airtime |
| "Frames get bigger than the MTU" | Payload ≤ MTU (1500); the frame with headers is up to 1518 (1522 tagged) |
| "Collisions are a normal part of modern Ethernet" | Not on switched full-duplex links; collisions indicate a hub or duplex mismatch |

---

## 15. Summary

- **Layer 2** delivers frames across **one link** using **MAC addresses**; a **frame** = header (dst MAC, src MAC, EtherType) + payload (an IP packet) + trailer (**FCS/CRC-32**).
- **Ethernet II frame:** 14-byte header, 46–1500-byte payload (padded), 4-byte FCS → **64–1518 bytes** (1522 with a VLAN tag); preamble/SFD added by the PHY.
- **MAC addresses:** 48 bits, OUI + device part, **I/G** and **U/L** bits, `ff:ff:ff:ff:ff:ff` = broadcast; local-only, not routable, spoofable/randomized.
- **802.1Q VLANs** split broadcast domains with a 4-byte tag; trunks carry multiple VLANs; inter-VLAN traffic needs a router.
- **Switches** learn source MACs and forward/flood by destination MAC without altering frames; **routers** rebuild the L2 header at every hop; **STP** prevents loops.
- **Bad frames** are silently dropped; **TCP** (or Wi-Fi's link-layer retries) recovers. Watch error counters.
- **ARP** maps next-hop IPs to MACs; Docker bridges and veths are software switches with generated MACs.
- Goodput on gigabit Ethernet is about **941 Mbit/s**; Wi-Fi is a shared, more complex medium.

---

## 16. Check your understanding

1. List the fields of an Ethernet II frame with their sizes, and give the minimum and maximum frame sizes.
2. A NIC receives a frame addressed to `01:00:5e:00:00:fb`. Under what condition does it accept it?
3. What does the first byte `02` of a MAC address tell you?
4. A frame carries a 28-byte ARP payload. How many bytes of padding are added, and how long is the frame with FCS?
5. What happens to a frame whose FCS doesn't match? Who repairs the loss?
6. Why do the MAC addresses change at each router, but the destination IP doesn't?
7. A switch receives a frame for a MAC it has never seen. What does it do? Later it sees a frame from that MAC. What changes?
8. Why must a Wi-Fi frame have up to four address fields?
9. What MAC would Docker assign to a container with IP `172.18.0.5`?
10. TCP over 1 Gbit/s Ethernet reaches only ~941 Mbit/s. Explain where the missing ~6% goes.

<details>
<summary>Answers</summary>

1. Preamble 7 + SFD 1 (PHY), dst MAC 6, src MAC 6, EtherType 2, payload 46–1500, FCS 4. Minimum frame 64 bytes, maximum 1518 (1522 with an 802.1Q tag), excluding preamble/SFD.
2. It's a multicast address (I/G bit = 1, IPv4 mDNS group 224.0.0.251); the NIC accepts it only if it has joined that multicast group (or is in promiscuous mode).
3. `02` = `00000010`: I/G = 0 (unicast) and U/L = 1 (locally administered, i.e. set by software such as Docker, a VM, or MAC randomization).
4. 14 + 28 = 42 bytes; padded with 60 − 42 = 18 zero bytes; with the 4-byte FCS the frame is 64 bytes.
5. It is discarded by the receiving NIC with no notification. TCP (or the application) notices the missing data and retransmits; in Wi-Fi the link layer retries.
6. MAC addresses are hop-by-hop: each router builds a new frame for the next link. The destination IP identifies the end device, so it stays constant (except when NAT rewrites it).
7. It floods the frame out of all ports of that VLAN except the ingress; when it later sees that MAC as a source, it learns which port it is on and afterwards forwards frames to it out of that single port.
8. In infrastructure mode a frame may involve a sender, a receiver, an access point relaying it, and a final destination/source: receiver, transmitter, and the real source and destination addresses are needed (address 4 only in special wireless-bridge/mesh modes).
9. `02:42:ac:12:00:05` (172 = 0xac, 18 = 0x12, 0, 5).
10. Per full frame the wire carries 1538 byte-times (1518 frame + 8 preamble/SFD + 12 inter-frame gap), while TCP data is only 1460 bytes: 1460 ÷ 1538 ≈ 0.949 of line rate; TCP ACKs and options reduce it slightly further to ~941 Mbit/s.
</details>

**Practice**

1. Capture 20 frames with `tcpdump -e` on your network; make a table of source MAC, destination MAC, EtherType and length; identify unicast, broadcast and multicast frames and the vendor (OUI) of each source.
2. Hand-build the Ethernet + IPv4 + ICMP echo request bytes for a ping between two made-up hosts; compute the IP header checksum (Chapter 30) and the frame length, then compare with a real capture.
3. Run lab 5 and record the bridge's MAC table before and after the first ping; explain each change.
4. Configure two VLAN sub-interfaces with different IDs in namespaces and prove that hosts in different VLANs can't reach each other without a router.
5. On a wired link, read `ethtool -S` and `ip -s link` counters before and after a large download, and explain each counter that changed.

---

**Next:** [Chapter 32 – The First Computer and the First Router](32_first_computer_and_first_router_in_details.md)
