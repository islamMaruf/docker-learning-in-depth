# Chapter 35: DHCP Discover, Layer by Layer

> **In one sentence:** When a device joins a network with no address, it sends a **DHCP Discover**: a broadcast that walks down the whole stack: a DHCP message (Layer 7) inside a UDP datagram from port 68 to 67 (Layer 4) inside an IP packet from `0.0.0.0` to `255.255.255.255` (Layer 3) inside an Ethernet frame to `ff:ff:ff:ff:ff:ff` (Layer 2) sent as signals (Layer 1), asking "is there a DHCP server out there?".

**Level:** 🟡 Intermediate · **Reading time:** ~60 minutes

**Prerequisites:** Chapters [23](23_tcp_in_details.md)–[24](24_udp_in_details.md) (UDP), [30](30_internet_protocol_ip_in_details.md) (IP), [31](31_data_link_layer_frame_in_details.md) (Ethernet), [33](33_subnetting_and_subnet_masks_in_details.md) (masks/broadcast).

---

## What you will learn

- **What DHCP is** and why it exists, and the four-step **DORA** conversation (Discover, Offer, Request, Acknowledge)
- The **exact structure of a DHCP message** (BOOTP heritage, fixed fields, magic cookie, options)
- Why DHCP uses **UDP ports 67 and 68**, and why a client with no IP address can still send
- How **every layer wraps the message** (with a real, checksummed byte-for-byte example)
- How the **broadcast flag**, **transaction ID (xid)** and **client identifier** work
- What a **DHCP relay** does when the server is on a different network
- How to **capture and read a real Discover** with `tcpdump`/Wireshark and to run a **DHCP server + client** in a safe lab

---

## 1. The problem DHCP solves

To use a network, a host needs an **IP address, mask, gateway and DNS** (Chapter 32). Typing them by hand on every phone, laptop and printer is slow and error-prone (duplicate addresses, typos). **DHCP** (*Dynamic Host Configuration Protocol*, RFC 2131 for IPv4, options in RFC 2132) automates it:

- The client asks; a **DHCP server** replies with a **lease**: an address plus configuration valid for a limited time.
- The server keeps a pool and a table of who has what, so addresses are never duplicated.
- When you leave, the lease expires and the address returns to the pool.

DHCP grew from **BOOTP** (RFC 951, 1985), which is why the message layout has odd fields like `sname` and `file` (used for network booting) and why the ports are the BOOTP ports.

### The full conversation: DORA

```
 Client                                                    Server
   │ ── 1. DISCOVER  (broadcast)  "Any DHCP servers?" ───────►│
   │ ◄─ 2. OFFER     "You could use 192.168.1.100 ..." ────── │
   │ ── 3. REQUEST   (broadcast)  "I'll take that one." ─────►│
   │ ◄─ 4. ACK       "Confirmed. Lease 24h. Here's the config"│
```
This chapter dissects **step 1**. Chapter 36 covers Offer, and Chapter 37 covers Request and Ack.

**Why not just one round trip?** Two reasons: (1) there may be **several DHCP servers** on the network (redundancy), so the client collects offers and picks one; (2) the Request is broadcast so that the servers that *lost* learn their offered address can be released.

---

## 2. The situation before Discover

A laptop has just connected (link is up; Wi-Fi associated or cable plugged in). It has:

| Has | Doesn't have |
|---|---|
| MAC address (e.g. `aa:bb:cc:11:22:33`) | IP address |
| A working NIC and driver | Subnet mask |
| A DHCP client program (`dhclient`, `NetworkManager`, `systemd-networkd`, Windows DHCP Client service) | Gateway, DNS |
| | Any knowledge of the DHCP server's address |

The router (also the DHCP server) at `192.168.1.1` has a pool `192.168.1.100–192.168.1.200`.

So the client's problem: it must talk to **someone it can't name**, from **an address it doesn't have**. The answer: use the special addresses "this host on this network" (`0.0.0.0`) and "everyone on this link" (`255.255.255.255`).

---

## 3. Layer 7: the DHCP message

The DHCP client builds the message. All multi-byte fields are **big-endian (network order)**.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+---------------+---------------+---------------+---------------+
|     op (1)    |  htype (1)    |   hlen (1)    |   hops (1)    |
+---------------+---------------+---------------+---------------+
|                         xid (4)                               |
+-------------------------------+-------------------------------+
|           secs (2)            |           flags (2)           |
+-------------------------------+-------------------------------+
|                        ciaddr (4)                             |
+---------------------------------------------------------------+
|                        yiaddr (4)                             |
+---------------------------------------------------------------+
|                        siaddr (4)                             |
+---------------------------------------------------------------+
|                        giaddr (4)                             |
+---------------------------------------------------------------+
|                        chaddr (16)                            |
|                                                               |
+---------------------------------------------------------------+
|                        sname (64)                             |
+---------------------------------------------------------------+
|                        file (128)                             |
+---------------------------------------------------------------+
|                 magic cookie 0x63825363 (4)                   |
+---------------------------------------------------------------+
|                        options (variable)                     |
+---------------------------------------------------------------+
```
Fixed part: 236 bytes + 4-byte cookie = **240 bytes**, then options.

| Field | Size | Discover value | Meaning |
|---|---|---|---|
| **op** | 1 | `1` (BOOTREQUEST) | `1` client→server, `2` server→client |
| **htype** | 1 | `1` | Hardware type: Ethernet |
| **hlen** | 1 | `6` | Hardware address length (MAC = 6 bytes) |
| **hops** | 1 | `0` | Incremented by **relays**; client sets 0 |
| **xid** | 4 | random, e.g. `0x3903F326` | **Transaction ID**: matches replies to this request |
| **secs** | 2 | `0` (grows on retries) | Seconds since the client began acquiring |
| **flags** | 2 | `0x8000` or `0x0000` | Top bit = **BROADCAST flag** (see §7) |
| **ciaddr** | 4 | `0.0.0.0` | Client's current IP (only when it already has one, e.g. renewing) |
| **yiaddr** | 4 | `0.0.0.0` | "Your" IP: filled by the **server** in Offer/Ack |
| **siaddr** | 4 | `0.0.0.0` | Next server (e.g. TFTP for PXE) |
| **giaddr** | 4 | `0.0.0.0` | **Relay agent** address (0 when no relay) |
| **chaddr** | 16 | client MAC + padding | Client hardware address |
| **sname / file** | 64 / 128 | zeros | Server host name / boot file (PXE) |
| **magic cookie** | 4 | `63 82 53 63` | Marks the start of DHCP options |

### Options (TLV: code, length, value)

Each option: `code (1 byte)`, `length (1 byte)`, `value (length bytes)`. Two are special: `0` = pad, **`255` = end**.

| Code | Name | Typical Discover content |
|---|---|---|
| **53** | DHCP Message Type | `1` = Discover (**required**; the *only* thing that says which message this is) |
| **61** | Client Identifier | `01` + MAC (type 1 = Ethernet); identifies the client for leases |
| **12** | Host Name | `laptop` |
| **55** | Parameter Request List | "please tell me: 1 subnet mask, 3 router, 6 DNS, 15 domain name, 28 broadcast address, 51 lease time…" |
| **50** | Requested IP | Only if the client wants a specific address (e.g. its previous lease) |
| **60** | Vendor Class ID | e.g. `MSFT 5.0`, `android-dhcp-14`, `PXEClient:…` |
| **57** | Maximum DHCP message size | Optional |
| **255** | End | Terminator (then padding to reach 300 bytes) |

Message types (option 53): `1` Discover, `2` Offer, `3` Request, `4` Decline, `5` ACK, `6` NAK, `7` Release, `8` Inform.

### The Discover in words
> "I am `aa:bb:cc:11:22:33` (host name *laptop*). I have no address. Is there a DHCP server? If you answer, please include: subnet mask, router, DNS servers, domain name, broadcast address and lease time."

Notice what is **absent**: a real source address and any server address. The client can only shout.

---

## 4. Layer 4: UDP

```
Source port:       68  (DHCP client)
Destination port:  67  (DHCP server)
Length:            8 + DHCP message size
Checksum:          over pseudo-header + UDP header + data
```
### Why UDP, not TCP?
- **TCP needs an established connection**, a handshake between two known addresses. The client has neither an address nor a known peer.
- **TCP can't broadcast.** UDP can.
- A DHCP exchange is a tiny request/response; the app handles retransmission itself (with random backoff: about 4, 8, 16, 32 seconds, with jitter, per RFC 2131).

### Why the strange fixed ports (67 and 68)?
Both sides use **well-known ports** (unlike normal client/server where the client picks a random port). This is a BOOTP legacy: a server's reply may be *broadcast* to the whole subnet, and a broadcast reply can only be delivered to the right program if the client's port is predictable. So **client = 68, server = 67**.

---

## 5. Layer 3: IP

| Field | Value | Why |
|---|---|---|
| Version/IHL | `4` / `5` (20-byte header) | |
| TTL | 64 (Linux default; 128 on Windows) | |
| Protocol | `17` (UDP) | |
| **Source IP** | **`0.0.0.0`** | "This host on this network": I have no address yet (RFC 1122 permits this specifically for DHCP) |
| **Destination IP** | **`255.255.255.255`** | *Limited broadcast*: every host on my local link. Routers **do not forward** it |

Why not `192.168.1.255` (the subnet broadcast)? The client doesn't know the subnet yet. `255.255.255.255` requires no knowledge at all.

---

## 6. Layer 2: Ethernet

```
Destination MAC:  ff:ff:ff:ff:ff:ff   (broadcast; the IP broadcast is delivered to this MAC)
Source MAC:       aa:bb:cc:11:22:33   (the client's real MAC; this one it does have)
EtherType:        0x0800   (IPv4)
```
- **No ARP needed**: broadcast IP maps to broadcast MAC by rule.
- A **switch** floods this frame out of every port in the VLAN (unknown/broadcast destination). Every NIC receives it and the OS decides whether a DHCP server process is listening. Most hosts (which aren't DHCP servers) silently drop it, since nothing listens on UDP 67.
- Length: the DHCP message is padded to **at least 300 bytes** (BOOTP minimum), so no Ethernet padding is needed.

## 7. The broadcast flag and how the reply comes back

The Offer will be addressed to `yiaddr` (the offered address), which the client doesn't own yet. Two ways to deliver it:

| Client sets flags | Server sends the reply as | Notes |
|---|---|---|
| `0x0000` (unicast OK) | **Unicast** frame to the client's **MAC** (`chaddr`), IP dst = `yiaddr`, **without ARP** (the server inserts the pair directly) | Most Linux/Windows clients |
| `0x8000` (broadcast) | Broadcast `255.255.255.255` / `ff:ff:ff:ff:ff:ff` | For clients whose IP stack **can't accept unicast IP before being configured** (some embedded/old stacks) |

Many servers ignore the flag or default to broadcast; behavior varies (Chapter 36 shows both).

## 8. Layer 1

The frame is serialized to bits and sent out: preamble and start delimiter first (7 + 1 bytes), then the frame, then the FCS, then an inter-frame gap of 12 bytes' time. On Wi-Fi the same frame is wrapped in an 802.11 data frame, sent after a successful association, using CSMA/CA. The line coding depends on the medium (e.g. PAM-5 for 1000BASE-T, MLT-3 with 4B/5B for 100BASE-TX, Manchester for 10BASE-T); you never need to know this to use DHCP, only that Layers 1–2 deliver bits to everyone on the link.

---

## 9. A complete Discover, byte by byte

Built with the fields above and real checksums (you can reproduce it with the script in §14):

**Ethernet header (14 bytes)**
```
ff ff ff ff ff ff   aa bb cc 11 22 33   08 00
└ dst broadcast ┘   └── src MAC ─────┘   └ IPv4
```
**IPv4 header (20 bytes)**
```
45 00 01 48 00 00 00 00 40 11 79 a6 00 00 00 00 ff ff ff ff
│  │  │   │  │   │   │  │  │   │   └ src 0.0.0.0 ┘ └ dst 255.255.255.255 ┘
│  │  │   │  │   │   │  │  └ header checksum 0x79a6
│  │  │   │  │   │   │  └ protocol 0x11 = UDP
│  │  │   │  │   │   └ TTL 0x40 = 64
│  │  │   │  └ flags/fragment offset 0 (DF not set)
│  │  │   └ identification 0
│  │  └ total length 0x0148 = 328
│  └ DSCP/ECN 0
└ version 4, IHL 5
```
**UDP header (8 bytes)**
```
00 44   00 43   01 34   72 15
 68      67      308     checksum 0x7215
```
(308 = 8 + 300)

**DHCP (300 bytes; first 44 shown)**
```
01 01 06 00  39 03 f3 26  00 00 80 00
op htype hlen hops   xid = 0x3903F326   secs=0   flags=0x8000
00 00 00 00 (ciaddr) 00 00 00 00 (yiaddr) 00 00 00 00 (siaddr) 00 00 00 00 (giaddr)
aa bb cc 11 22 33 00 00 00 00 00 00 00 00 00 00   ← chaddr (MAC + 10 zero bytes)
... 64 zero bytes (sname) + 128 zero bytes (file) ...
63 82 53 63                                        ← magic cookie
35 01 01                    ← option 53, length 1, value 1 = DISCOVER
3d 07 01 aa bb cc 11 22 33  ← option 61 (0x3d), length 7, type 01 + MAC
0c 06 6c 61 70 74 6f 70     ← option 12, length 6, "laptop"
37 06 01 03 06 0f 1c 33     ← option 55 (0x37): request 1,3,6,15,28,51
ff                          ← option 255 (end)
00 00 ...                   ← padding up to 300 bytes
```
Sizes: DHCP 300 → UDP 308 → IP 328 → **frame 342 bytes** (+ 4 FCS on the wire).

Layer-by-layer summary (encapsulation; each layer only looks at its own header):

```
Ethernet: dst ff:ff:ff:ff:ff:ff  src aa:bb:cc:11:22:33   type IPv4
  IPv4:   src 0.0.0.0            dst 255.255.255.255      proto UDP   TTL 64
    UDP:  68 → 67
      DHCP: op=BOOTREQUEST xid=0x3903f326  chaddr=aa:bb:cc:11:22:33  options: [53=Discover, 61, 12, 55 …]
```

---

## 10. What happens at the server (or relay)

1. The router's NIC accepts the broadcast frame; the IP layer sees protocol UDP, destination port **67**; the DHCP server process (listening on it) receives the payload.
2. It checks: message type Discover; which **interface** it arrived on (→ which subnet's pool to use); has this client (by client-ID / MAC) had a lease before? Any static reservation? Is the pool exhausted?
3. It **reserves** a candidate address temporarily (so it isn't offered to someone else in the same moment), possibly after **checking it isn't in use** (an ICMP echo or ARP probe; ISC dhcpd does this by default), then sends the **Offer** (Chapter 36).
4. If there is **no** matching pool or the pool is exhausted, it stays silent, and the client retries with backoff; after repeated failure Windows and macOS fall back to link-local `169.254.x.x` (Chapter 32).

### DHCP relay (server on another network)
Broadcasts don't cross routers. Enterprise networks have one central DHCP server; each router interface facing clients runs a **relay agent** (`ip helper-address` on Cisco, `dhcrelay` on Linux):

```
Client ─Discover (broadcast)─► Relay/router ─unicast─► Central DHCP server
       hops=1, giaddr = relay's IP on the client's subnet ←(the server picks the pool matching giaddr)
```
The relay sets **giaddr** (its address on the client's network) so the server knows which subnet to allocate from, and it adds **option 82** (relay agent information: circuit/remote ID) in some deployments.

---

## 11. Retransmission, timing and robustness

- Discover retransmissions use exponential backoff with randomization (**~4 s, 8 s, 16 s, 32 s, 64 s**; RFC 2131). The randomness prevents everyone in a building from retrying at once after a power cut (a "DHCP storm").
- Windows waits roughly a minute before falling back to APIPA and keeps trying every 5 minutes.
- **`secs`** grows with each retry so servers can prioritize.
- **`xid`** is random per transaction, so a client ignores replies meant for others.

---

## 12. Security notes

DHCP has **no authentication** in its basic form:

| Threat | What happens | Defense |
|---|---|---|
| **Rogue DHCP server** | An attacker or a misplugged home router answers faster and hands out a wrong gateway/DNS (man-in-the-middle) | **DHCP snooping** on managed switches (trusted vs untrusted ports) |
| **DHCP starvation** | Flood of Discovers with fake MACs exhausts the pool | Port security, rate limits, snooping |
| **Spoofed release/requests** | Forcing clients off | Snooping bindings, dynamic ARP inspection |

A classic home/office incident: someone plugs an old router's **LAN** port into the wall by mistake; its DHCP server answers and half the office receives bogus addresses.

---

## 13. Docker and virtualization connection

- **Docker does not use DHCP on its bridge**: the daemon's IPAM assigns container addresses directly; there is no Discover on `docker0` (unless you use `macvlan` with an external DHCP plugin).
- **VMs**: VirtualBox/VMware NAT networks and libvirt's `virbr0` run a built-in **dnsmasq** DHCP server; a bridged VM sends a Discover onto your physical LAN and gets an address from your real router.
- **Cloud:** VMs in AWS/Azure/GCP get their private address, gateway and DNS from the provider's DHCP service on boot, just like a laptop on a LAN.
- **PXE boot:** a diskless machine's firmware sends a Discover with option 60 `PXEClient`, and the response carries `siaddr`/`file` (a boot server and boot file): the reason `sname`/`file` still exist.

---

## 14. Lab

### 14.1 Watch a real Discover on your machine (safe: it only observes)

```bash
sudo tcpdump -i any -nn -vv -e 'udp port 67 or udp port 68'
# in another terminal, force your machine to renew (disconnect and reconnect Wi-Fi, or):
sudo dhclient -r eth0 && sudo dhclient -v eth0        # release, then a fresh DORA  (name your interface)
# NetworkManager systems: nmcli device reapply eth0   or   nmcli con down "<name>" && nmcli con up "<name>"
```
Look for `Discover`, and identify: source `0.0.0.0.68`, dest `255.255.255.255.67`, `Client-Ethernet-Address`, `xid`, options 53/55/61/12. Save with `-w dhcp.pcap` and open in **Wireshark** (`bootp` display filter).

### 14.2 A private DHCP server and client in namespaces

Needs `dnsmasq`, `isc-dhcp-client` (dhclient), `tcpdump`. Nothing touches your real network. (Use `-sf /bin/true` so the client does not run its script and change your host's DNS settings.)

```bash
sudo ip netns add srv; sudo ip netns add cli
sudo ip link add s0 type veth peer name c0
sudo ip link set s0 netns srv; sudo ip link set c0 netns cli
sudo ip netns exec srv ip addr add 192.168.50.1/24 dev s0
sudo ip netns exec srv ip link set s0 up; sudo ip netns exec cli ip link set c0 up

# capture in the background (in the server namespace)
sudo ip netns exec srv tcpdump -nn -vv -e -i s0 -w /tmp/dhcp.pcap 'udp port 67 or udp port 68' &
sleep 1

# DHCP server: no DNS (port=0), pool .100–.150, 12 h lease, advertise the router, log every message
sudo ip netns exec srv dnsmasq --no-daemon --port=0 --interface=s0 --bind-interfaces \
     --dhcp-range=192.168.50.100,192.168.50.150,12h \
     --dhcp-option=option:router,192.168.50.1 --dhcp-option=option:dns-server,1.1.1.1 \
     --dhcp-leasefile=/tmp/lab.leases --log-dhcp &
sleep 1

# DHCP client, one attempt, verbose, script disabled
sudo ip netns exec cli dhclient -1 -v -sf /bin/true -lf /tmp/cli.lease -pf /tmp/cli.pid c0
cat /tmp/cli.lease /tmp/lab.leases                                     # the lease the client stored / the server table
sudo pkill tcpdump; sudo tcpdump -nn -vv -e -r /tmp/dhcp.pcap | head -60   # read the DORA
```
You should see four packets: DHCPDISCOVER, DHCPOFFER, DHCPREQUEST, DHCPACK, and the server's `--log-dhcp` output describing each.

Clean up:
```bash
sudo pkill dnsmasq; sudo pkill dhclient
sudo ip netns del srv; sudo ip netns del cli; rm -f /tmp/dhcp.pcap /tmp/cli.* /tmp/lab.leases
```

### 14.3 Rebuild the packet yourself (Python, no libraries)

```python
import struct
mac = bytes.fromhex("aabbcc112233")
opts = bytes([53,1,1]) + bytes([61,7,1]) + mac + bytes([12,6]) + b"laptop" + bytes([55,6,1,3,6,15,28,51])
dhcp = struct.pack("!BBBBIHH4s4s4s4s", 1,1,6,0, 0x3903F326, 0, 0x8000, b"\0"*4,b"\0"*4,b"\0"*4,b"\0"*4)
dhcp += mac + b"\0"*10 + b"\0"*64 + b"\0"*128 + bytes.fromhex("63825363") + opts + b"\xff"
dhcp = dhcp.ljust(300, b"\0")
print(len(dhcp))        # 300
```
Extend it with the UDP/IP headers (the checksum method is in Chapters 23–24 and 30) and compare against the bytes in §9. To *send* one on a lab network, use `scapy` (`Ether()/IP()/UDP()/BOOTP()/DHCP()`).

---

## 15. Troubleshooting Discover problems

| Symptom | Where to look |
|---|---|
| Client stuck at "obtaining IP address" / gets `169.254.x.x` | Capture on the client: are Discovers leaving? Capture on the server side: do they arrive? Answered? |
| Discovers leave, none arrive at the server | Switch/VLAN/port issue, wrong VLAN, DHCP snooping dropping frames, missing **relay** (`ip helper-address`) |
| Discovers arrive, no Offer | Server not listening on that interface, no pool for that subnet, **pool exhausted**, reservation/ACL blocks the MAC, firewall blocks UDP 67/68 |
| Offers received, from an unexpected server | **Rogue DHCP** |
| Works on cable, not on Wi-Fi | Wi-Fi client isolation, broadcast filtering, AP/controller DHCP proxy settings |
| Works after a while | Slow Offer, pool nearly full, server does ICMP conflict checks (added delay) |
| Container can't get DHCP | Docker bridge doesn't run DHCP; use `docker network` IPAM or macvlan with a DHCP plugin |

Useful: `journalctl -u NetworkManager | grep -i dhcp`, `journalctl -u systemd-networkd`, `sudo dhclient -v`, `nmap --script broadcast-dhcp-discover` (lists DHCP servers on your segment), `dhcpdump`.

---

## 16. Common misconceptions

| Misconception | Reality |
|---|---|
| "DHCP only assigns IPs" | It also delivers mask, gateway, DNS, lease time, NTP, domain, PXE boot info, and more |
| "DHCP uses TCP" | UDP, ports 67 (server) and 68 (client) |
| "Discover is sent to the server's IP" | It's a broadcast; the client doesn't know the server |
| "The source IP is 255.255.255.255" | Source `0.0.0.0`, destination `255.255.255.255` |
| "Routers forward DHCP broadcasts" | Not unless a **relay agent** converts them |
| "Every message is a broadcast" | Discover/Request broadcast at first; replies may be unicast; renewals are unicast |
| "The Discover message identifies itself by port" | The **option 53** value identifies the message type |
| "DHCP is secure" | No authentication; use snooping |
| "Docker containers use DHCP" | Docker's IPAM assigns statically from the network's pool |

---

## 17. Summary

- **DHCP** automates address configuration through a four-message conversation: **D**iscover, **O**ffer, **R**equest, **A**cknowledge.
- **Discover** is a **broadcast**: DHCP message (op=1, xid, chaddr, options 53/61/55…) → **UDP 68→67** → IP **`0.0.0.0` → `255.255.255.255`** → Ethernet **`ff:ff:ff:ff:ff:ff`**.
- The message layout is inherited from **BOOTP** (240 fixed bytes with the magic cookie `63 82 53 63`, then TLV options ending in `255`).
- **Broadcasts don't cross routers**; **relay agents** (giaddr) carry them to a central server.
- DHCP is unauthenticated: use **DHCP snooping** against rogue servers.

---

## 18. Check your understanding

1. Why does Discover use source `0.0.0.0` and destination `255.255.255.255`?
2. Why does DHCP use UDP, and why fixed ports 67/68?
3. Which option actually says "this is a Discover"?
4. What is `xid` used for? What is `chaddr`?
5. What does the broadcast flag change?
6. How do the sizes work out: DHCP 300 bytes → frame ___ bytes?
7. The DHCP server is on another subnet. What must exist and what does `giaddr` do?
8. What happens on Windows if no server ever answers?

<details>
<summary>Answers</summary>

1. The client has no address and doesn't know the server or subnet; `0.0.0.0` means "this host, this network" and `255.255.255.255` reaches everyone on the local link.
2. TCP needs an established connection between known addresses and can't broadcast; the fixed ports let broadcast replies reach the client's DHCP process (BOOTP legacy).
3. Option 53 (DHCP Message Type) with value 1.
4. `xid` matches replies to the request; `chaddr` carries the client's hardware address so the server can identify it and reply.
5. It tells the server whether to reply by broadcast (`0x8000`) or by unicast to the client's MAC before it has an IP.
6. 300 + 8 (UDP) + 20 (IP) + 14 (Ethernet) = 342 bytes (+4 FCS).
7. A relay agent on the client's router interface forwards the broadcast as unicast to the server; `giaddr` lets the server choose the pool for the client's subnet.
8. After retries it self-assigns a `169.254.x.x` link-local address and keeps trying DHCP periodically.
</details>

**Practice**

1. Capture your own Discover/Offer/Request/Ack (§14.1) and fill in a table of xid, options and addresses for each packet.
2. Run the namespace lab; change the pool and lease time; watch how the packets change.
3. Use `nmap --script broadcast-dhcp-discover` on a network you own and identify every DHCP server (any surprises?).
4. Reproduce the 342-byte frame in Python/scapy and check it with `tcpdump -r`.
5. Explain how the exchange would change with a relay in the path (draw the addresses in each hop).

---

**Next:** [Chapter 36 – DHCP Offer Deep Dive](36_dhcp_offer_deep_dive_in_details.md)
