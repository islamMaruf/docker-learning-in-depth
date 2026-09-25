# Chapter 32: The First Computer and the First Router

> **In one sentence:** To put a computer on a network you need a **network interface** (hardware plus a driver) with a MAC address, an **IP address** (from DHCP, manual settings, or a self-assigned link-local fallback), a **subnet mask** and a **default gateway**; connecting to other networks and the internet requires a **router**, which forwards packets between networks and (at home) also does DHCP, NAT, Wi-Fi and a firewall.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~55 minutes

**Prerequisites:** [Chapter 30](30_internet_protocol_ip_in_details.md) (IP addresses, routing) and [Chapter 31](31_data_link_layer_frame_in_details.md) (MACs, frames, switches).

---

## What you will learn

- What a **network interface (NIC)** is, how drivers make it usable, and how to see yours on Linux, Windows and macOS
- The **four things** every host needs to communicate: IP address, subnet mask, default gateway, DNS
- **How a computer gets an address**: DHCP, static configuration, and the **link-local (APIPA)** fallback `169.254.0.0/16`
- **Two computers, one cable**: peer-to-peer, why cables used to be "crossover", and the `N(N−1)/2` scaling problem that led to switches
- What a **router** is, how it differs from a **switch**, and what a typical **home router** really is (router + switch + Wi-Fi AP + DHCP + NAT + firewall, sometimes + modem)
- A step-by-step journey from "no network" to "browsing the internet", including the exact packets
- **Build your own routers** in a lab with Linux network namespaces (so nothing on your real network is touched)
- Troubleshooting from the bottom up, and how Docker/VMs fit this same model

---

## 1. A computer with no network

A brand-new computer can run programs, store files and draw on a screen. What it *cannot* do is talk to other machines. A network interface on a computer is what gives it:

- a **MAC address** (Layer 2 identity), and after configuration
- an **IP address** (Layer 3 identity)

Note the wording: an IP address identifies a **network interface**, not "the computer". A laptop with Ethernet and Wi-Fi has (at least) two interfaces, each with its own MAC and IP addresses. With no interface there is nothing to address.

Historically, computers stayed standalone and moved files with floppy disks ("sneakernet") or dial-up modems. Networking hardware turned computers from islands into a connected system.

---

## 2. The Network Interface Card (NIC)

A **NIC** (network interface card/controller; also *network adapter*, *LAN card*) connects a computer to a network medium and handles Layers 1 and 2:

| Layer | What the NIC does |
|---|---|
| **1 (Physical)** | Converts digital data to electrical, optical or radio signals and back; auto-negotiates speed and duplex |
| **2 (Data Link)** | Builds and parses **frames**, adds/checks the **FCS**, filters by **MAC address**, controls access to the medium (CSMA/CA in Wi-Fi) |
| **Offloads** | Modern NICs also compute checksums, segment large TCP sends (TSO/GSO), and balance receive queues to reduce CPU load |

### Kinds of NICs
| Kind | Where it lives | Notes |
|---|---|---|
| **Integrated (onboard)** | On the motherboard/SoC | Standard today: Ethernet port (RJ-45), Wi-Fi/Bluetooth chip in laptops and phones |
| **PCIe / PCI card** | Expansion slot | Server and workstation NICs: 2.5/10/25/40/100 Gbit/s, SFP+/QSFP fiber |
| **USB adapter / dongle** | USB port | Handy for laptops without Ethernet |
| **Virtual NIC (vNIC)** | Software | VMs, containers (`veth`), Docker bridges, VPN tunnels (`wg0`, `tun0`) |

Wired Ethernet uses the **RJ-45** connector and twisted-pair cables (**Cat5e** for gigabit, **Cat6/6a** for 10 Gbit/s, max 100 m per run); wireless uses **802.11** standards on 2.4/5/6 GHz (Wi-Fi 4/5/6/6E/7).

### Drivers
Hardware needs software that knows how to talk to it: a **driver**. It initializes the chip, hands frames to/from the OS network stack, handles interrupts, manages memory buffers and reports errors. Analogy: a translator between the operating system (which speaks "generic networking") and one specific chip (which speaks its own register language).

- **Then:** you installed a driver from a CD and rebooted.
- **Now:** operating systems ship with thousands of drivers and load them automatically ("plug and play"). On **Linux** they are usually **kernel modules**; on **Windows** `.sys`/INF packages from Windows Update or the vendor; **macOS** bundles drivers for supported hardware.
- If an interface doesn't appear, the usual culprit is a **missing driver or firmware** (Wi-Fi chips commonly need `linux-firmware`).

```bash
# Linux: hardware, driver, interface
lspci -k | grep -A3 -i -E 'ethernet|network'     # PCI NICs and "Kernel driver in use: e1000e / r8169 / iwlwifi ..."
lsusb                                            # USB adapters
ip -br link                                      # interface names and MACs
ethtool -i eth0                                  # driver: e1000e, version, firmware-version, bus-info
cat /sys/class/net/eth0/address                  # MAC address
ls /sys/class/net                                # lo, eth0 (or enp3s0), wlan0 (or wlp2s0), docker0 ...
lsmod | grep -iE 'e1000|r8169|iwlwifi|ath|mt76'  # loaded driver modules
dmesg | grep -iE 'eth|wlan|firmware|link is'     # detection, firmware and link messages
```

Names like `enp3s0` (Ethernet, PCI bus 3, slot 0) or `wlp2s0` are **predictable interface names** chosen by systemd based on hardware location, replacing the older `eth0`/`wlan0`.

**Windows:** `ipconfig /all`, `Get-NetAdapter`, Device Manager → Network adapters. **macOS:** `ifconfig`, `networksetup -listallhardwareports`, System Settings → Network.

### Unique identity
Each NIC has a **MAC address** (Chapter 31), giving the interface a Layer-2 identity. Without an IP configuration it can still send and receive frames on its local link, but Layer-3 applications have nothing to bind to (except loopback).

```
No NIC → no MAC → no Layer 2 → no IP → no networking
NIC + driver → MAC available → Layer 2 works → assign IP → networking works
```

---

## 3. What a host needs to be "on the network"

| Setting | Question it answers | Example |
|---|---|---|
| **IP address** | Who am I? | `192.168.1.50` |
| **Subnet mask / prefix** | Which addresses are on **my** local network? | `255.255.255.0` (`/24`) |
| **Default gateway** | Where do I send packets for **everything else**? | `192.168.1.1` (the router) |
| **DNS server(s)** | How do I turn names into addresses? | `192.168.1.1`, `1.1.1.1` |

The subnet mask lets the host decide, for each destination: is it **local** (ARP for it, send directly) or **remote** (send to the default gateway)? Two hosts with the same address prefix on the same cable can talk directly; anything else needs a router. (Full details of masks: Chapters 33–34.)

### How the settings are obtained

| Method | How | Typical use |
|---|---|---|
| **DHCP** (dynamic) | A DHCP server hands out address, mask, gateway, DNS and a lease time | Nearly everything: homes, offices, Wi-Fi (Chapters 35–37) |
| **Static** (manual) | An administrator types them in | Servers, routers, printers, infrastructure |
| **Link-local / APIPA** | The host picks its own `169.254.x.y` address | Fallback when DHCP is absent |

---

## 4. Link-local addresses (APIPA / IPv4LL)

If DHCP doesn't answer, most desktop operating systems self-assign an address from **`169.254.0.0/16`** (RFC 3927 "IPv4 Link-Local"; Microsoft calls it **APIPA**, Automatic Private IP Addressing).

### The process
1. The interface comes up; the OS sends **DHCP Discover** broadcasts and waits (about a minute total on some systems).
2. No answer → the OS picks a **random** address in **`169.254.1.0 – 169.254.254.255`** (the first and last 256 addresses are reserved, so about **65,024** usable).
3. It checks the address isn't taken by sending **ARP probes** ("does anyone own 169.254.x.y?"); if someone answers, it picks another.
4. It configures the address with the `/16` mask and **no default gateway and no DNS**.
5. It **keeps trying DHCP** in the background and abandons the link-local address as soon as a server appears.

### What works and what doesn't
| Works | Doesn't work |
|---|---|
| Talking to other devices on the **same link** (same cable/switch/Wi-Fi) that also have link-local or that are reachable at L2, e.g. two laptops joined by a cable, printers, some IoT/mDNS discovery | Reaching the **internet** (no gateway, and these addresses are **never routed**) |
| ARP, ping, file sharing and app protocols between those devices | Talking to hosts on **other subnets**, using DNS, reaching most company resources |

`169.254.169.254` is a well-known exception in a different role: the **cloud metadata service** address in AWS/GCP/Azure.

### Recognizing it
- **Windows:** `ipconfig` shows **"Autoconfiguration IPv4 Address . . . : 169.254.x.y"** (on newer Windows just "IPv4 Address" in that range).
- **macOS:** `ifconfig en0` shows `inet 169.254.x.y netmask 0xffff0000`; System Settings says "Self-assigned IP address".
- **Linux:** not automatic by default on most server setups. Desktops using NetworkManager can use *Link-Local* mode, and `avahi-autoipd` provides it: `ip addr` shows `inet 169.254.x.y/16 scope link`.

> **In production, a `169.254.x.x` address means "I could not reach a DHCP server"**: check the cable and link lights, the switch port (VLAN, port security), whether the DHCP server is up and has free addresses in its pool, whether a relay/agent is configured (across VLANs), and try to renew.

**IPv6 does the same by design:** every IPv6 interface automatically has a **link-local `fe80::/10` address** (plus, on most networks, a global one via router advertisements: SLAAC).

---

## 5. Two computers and a cable

The simplest network: two computers, one Ethernet cable, no other equipment.

```
┌───────────┐      Ethernet cable      ┌───────────┐
│ Computer A│══════════════════════════│ Computer B│
│ 169.254.52.143/16                    169.254.100.50/16 │
└───────────┘                          └───────────┘
```

### The old crossover cable
Classic Ethernet (10/100BASE-T) used pins 1–2 to **transmit** and 3–6 to **receive** on a *computer*, but a *switch/hub* had them reversed. So:

- **Straight-through** cable: computer ↔ switch. ✅
- **Crossover** cable (transmit pair wired to the other end's receive pair): computer ↔ computer (or switch ↔ switch). ✅ (A straight cable between two computers would connect transmitters to transmitters.)

Today **Auto-MDI/MDI-X** is universal on modern NICs (it is **mandatory for Gigabit Ethernet**, which uses all four pairs in both directions): the ports detect and swap pairs themselves, so **any cable works for any connection**.

### Does it work?
With DHCP absent, both computers self-assign link-local addresses; they're in the **same `/16`**, so:

```
Is 169.254.100.50 in my network 169.254.0.0/16 ?  yes → ARP for its MAC → send the frame directly
```
```bash
ping 169.254.100.50
# 1. A broadcasts ARP:  "who has 169.254.100.50? tell 169.254.52.143"
# 2. B replies (unicast): "169.254.100.50 is at bb:bb:bb:bb:bb:bb"
# 3. A sends ICMP Echo Request in an Ethernet frame to that MAC; B replies
```
They can now share files (e.g. `python3 -m http.server 8080` on one, `http://169.254.52.143:8080` on the other) but have no internet, no DNS, and no way to add a third machine to the cable.

### Why a switch
Connecting every machine to every other needs **N(N−1)/2 cables and N−1 NICs per machine**:

| Machines | Cables |
|---|---|
| 2 | 1 |
| 3 | 3 |
| 5 | 10 |
| 10 | 45 |
| 100 | 4,950 |

A **switch** (Chapter 31) gives every machine **one** cable to a central device that forwards frames by MAC address: N cables total, the **star topology** that all LANs use.

---

## 6. Routers

### 6.1 Why a router
A switch connects devices into **one network** (one subnet, one broadcast domain). To reach **another network**, whether a second office subnet or the whole internet, you need a device that operates at **Layer 3**: a **router**. It has **an interface in each network** and forwards **IP packets** between them according to a **routing table**.

```
   Network A 192.168.1.0/24            Network B 10.0.0.0/24            Network C (ISP)
   ┌──────┐ ┌──────┐                    ┌────────┐                       
   │ PC 1 │ │ PC 2 │                    │ Server │                       
   └──┬───┘ └──┬───┘                    └───┬────┘                       
      └───┬────┘                            │                               
      [switch]                           [switch]                            
          │ .1                              │ .1                    203.0.113.2/30 (WAN)
          └──────────────── ROUTER ─────────┴────────────────────────────── ISP ─► Internet
                  LAN 192.168.1.1/24   DMZ 10.0.0.1/24
```

### 6.2 Router vs switch

| | **Switch** | **Router** |
|---|---|---|
| Layer | 2 | 3 |
| Decides by | Destination **MAC** (learned table) | Destination **IP** (routing table, longest-prefix match) |
| Connects | Devices within **one network** | **Different networks** |
| Broadcasts | Forwards them within the VLAN | **Does not forward** L2 broadcasts (each interface is its own broadcast domain) |
| Frame rewritten? | No | Yes: new L2 header per hop, TTL−1, checksum |
| Configuration | Mostly plug-and-play (unmanaged) | Needs IP addressing/routes |

### 6.3 What a "home router" actually is
The little box from your ISP is several devices in one:

| Function | Role |
|---|---|
| **Router** | Routes between your **LAN** (`192.168.1.0/24`) and the **WAN** (the ISP) |
| **Ethernet switch** | The 4 LAN ports |
| **Wireless access point** | Wi-Fi on the same LAN subnet (a bridge between 802.11 and Ethernet) |
| **DHCP server** | Hands out `192.168.1.x` addresses, mask, gateway (the router itself), DNS |
| **DNS forwarder** | Often relays DNS queries to the ISP or public resolvers |
| **NAT** | Rewrites private sources to the single public WAN address so many devices share it (Chapter 30) |
| **Stateful firewall** | Blocks unsolicited inbound traffic (NAT alone is not a firewall) |
| **Modem** (sometimes) | Cable/DSL/fiber ONT that speaks the ISP's physical protocol |

Typical configuration:

```
WAN:  address from the ISP by DHCP (or PPPoE / static)        e.g. 203.0.113.45   (or 100.64.x.x behind CGNAT)
LAN:  192.168.1.1/24, DHCP server pool 192.168.1.10 – .254, lease 24 h, DNS = the router (relays to ISP/1.1.1.1)
Wi-Fi: SSID "MyHomeNetwork", WPA3-Personal (or WPA2), channel auto → part of the same LAN
```
Good hygiene: change the router's **default admin password** (many ship with well-known defaults like `admin/admin`), keep firmware updated, disable WPS and remote administration, use WPA3/WPA2-AES with a strong passphrase, and set a guest network for visitors and IoT.

### 6.4 What the router does with one packet
`192.168.1.10` opens `https://142.250.185.206`:

1. **PC decides:** destination isn't in `192.168.1.0/24` → send to the **default gateway** `192.168.1.1`. ARP for the gateway's MAC (cached after the first time).
2. **Frame 1:** dst MAC = router LAN MAC, src MAC = PC; IP `192.168.1.10 → 142.250.185.206`, TTL 64.
3. **Router:** accept the frame, strip L2, **look up** the destination (default route → ISP), **TTL 64→63**, **NAT** rewrite: source `192.168.1.10:54321 → 203.0.113.45:60001` and record it in its table, recompute checksums, ARP for the ISP gateway MAC, build **Frame 2** and send it out the WAN interface.
4. **The reply** returns to `203.0.113.45:60001`; the router finds the NAT entry, rewrites the destination back to `192.168.1.10:54321`, decrements TTL and forwards on the LAN.

**IP addresses** (apart from NAT) stay; **MAC addresses** change at each hop.

---

## 7. From "no network" to "browsing the web"

| Step | State | What you can do |
|---|---|---|
| 1. Bare computer | No NIC/driver | Local apps only |
| 2. NIC installed + driver | MAC available, link up | Frames on the local link; no IP yet |
| 3. No DHCP present | **Link-local** `169.254.x.y/16` | Talk to devices on the same cable; no internet |
| 4. Add a **switch** | Star topology, several machines | Local file/printer sharing |
| 5. Add a **router with DHCP** | `192.168.1.x/24`, gateway, DNS | Full local network with proper addressing |
| 6. Router's **WAN** connects to the ISP | Public (or CGNAT) address + **NAT** | **Internet access** |

The first steps a host performs when you plug in a cable and open a browser (details in Chapters 35–39):

```
1. Link up (Layer 1/2)                       — NIC negotiates speed/duplex
2. DHCP Discover (broadcast)                 — "I need an address"        → Offer → Request → Ack
   result: IP 192.168.1.20/24, gateway 192.168.1.1, DNS 192.168.1.1, lease 24 h
3. ARP for the gateway                       — learns the router's MAC
4. DNS query (UDP 53) for example.com        — via the router/ISP
5. TCP handshake to the server               — via the router (NAT)
6. TLS handshake, HTTP request/response
```

---

## 8. Configuring addresses by hand

Only the *temporary* forms are shown (they vanish at reboot). Use your distribution's network manager/netplan/`nmcli` for persistent settings.

**Linux (`iproute2`)**

```bash
ip -br addr                                         # what do I have now?
sudo ip addr flush dev eth0
sudo ip addr add 192.168.1.100/24 dev eth0          # static address
sudo ip link set eth0 up
sudo ip route add default via 192.168.1.1 dev eth0  # default gateway
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf   # (systemd-resolved systems: resolvectl dns eth0 1.1.1.1)
ping -c 2 192.168.1.1     # gateway reachable? (L2/L3 works)
ping -c 2 1.1.1.1         # internet by IP? (routing/NAT works)
ping -c 2 example.com     # DNS works?
```

Persistent examples: **Ubuntu netplan** (`/etc/netplan/01-net.yaml`):

```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [192.168.1.100/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```
then `sudo netplan apply`. With **NetworkManager**: `nmcli con mod "Wired connection 1" ipv4.method manual ipv4.addresses 192.168.1.100/24 ipv4.gateway 192.168.1.1 ipv4.dns 1.1.1.1 && nmcli con up "Wired connection 1"`. (The old `/etc/network/interfaces` with `ifupdown` still works on Debian.)

**Windows:** Settings → Network → adapter → Edit IP assignment; or

```cmd
netsh interface ip set address "Ethernet" static 192.168.1.100 255.255.255.0 192.168.1.1
netsh interface ip set dns "Ethernet" static 1.1.1.1
ipconfig /all
ipconfig /release & ipconfig /renew        # DHCP
ipconfig /flushdns
```

**macOS:** `networksetup -setmanual "Wi-Fi" 192.168.1.100 255.255.255.0 192.168.1.1`; `networksetup -setdnsservers Wi-Fi 1.1.1.1`; `networksetup -setdhcp Wi-Fi`.

---

## 9. Lab: build your own two-network topology with a router (Linux namespaces)

Network namespaces are like tiny isolated computers on your machine; `veth` pairs are virtual cables. This lab builds **two LANs joined by a router**, then gives the router a **WAN and NAT**. Nothing touches your real network. (Needs `sudo`; install `iproute2`, `iputils-ping`, `traceroute`, `tcpdump`, `iptables`.)

```
  h1 10.0.1.10/24 ──┐                           ┌── h2 10.0.2.10/24
       gw 10.0.1.1  │                           │  gw 10.0.2.1
                    └── r1 (10.0.1.1 | 10.0.2.1) ┘
                         router namespace "r"
                              │ 203.0.113.1/24  (WAN)
                              └────────── "inet" 203.0.113.2/24  (a pretend internet host)
```

### 9.1 Two LANs and one router

```bash
# namespaces
for n in h1 h2 r inet; do sudo ip netns add $n; sudo ip netns exec $n ip link set lo up; done

# cables (veth pairs)
sudo ip link add h1-r type veth peer name r-h1
sudo ip link add h2-r type veth peer name r-h2
sudo ip link add r-inet type veth peer name inet-r
sudo ip link set h1-r netns h1;   sudo ip link set r-h1 netns r
sudo ip link set h2-r netns h2;   sudo ip link set r-h2 netns r
sudo ip link set r-inet netns r;  sudo ip link set inet-r netns inet

# addresses (like the NIC configuration of each machine)
sudo ip netns exec h1 ip addr add 10.0.1.10/24 dev h1-r
sudo ip netns exec h2 ip addr add 10.0.2.10/24 dev h2-r
sudo ip netns exec r  ip addr add 10.0.1.1/24  dev r-h1
sudo ip netns exec r  ip addr add 10.0.2.1/24  dev r-h2
sudo ip netns exec r  ip addr add 203.0.113.1/24 dev r-inet
sudo ip netns exec inet ip addr add 203.0.113.2/24 dev inet-r
for x in "h1 h1-r" "h2 h2-r" "r r-h1" "r r-h2" "r r-inet" "inet inet-r"; do set -- $x; sudo ip netns exec $1 ip link set $2 up; done

# default gateways for the hosts
sudo ip netns exec h1 ip route add default via 10.0.1.1
sudo ip netns exec h2 ip route add default via 10.0.2.1
```

**Test before the router forwards:** the router namespace doesn't forward packets between interfaces by default.

```bash
sudo ip netns exec h1 ping -c 2 10.0.1.1      # OK: router is on h1's own network (L2 delivery)
sudo ip netns exec h1 ping -c 2 10.0.2.10     # FAILS: h1 → router → (router not forwarding)
sudo ip netns exec r sysctl -w net.ipv4.ip_forward=1     # turn the machine into a ROUTER
sudo ip netns exec h1 ping -c 2 10.0.2.10     # NOW works!
```
That single sysctl is what distinguishes a router from an ordinary host.

**Observe what a router does**

```bash
sudo ip netns exec h1 ping -c 1 10.0.2.10 | grep ttl          # "ttl=63": started at 64, decremented once by the router
sudo ip netns exec h1 traceroute -n 10.0.2.10                 # hop 1: 10.0.1.1 (the router), hop 2: 10.0.2.10
sudo ip netns exec h1 ip neigh                                # h1 knows the ROUTER's MAC (10.0.1.1), never h2's
sudo ip netns exec h2 ip neigh                                # h2 knows the router's other-side MAC
sudo ip netns exec r tcpdump -nn -e -i r-h1 -c 4 icmp &       # frames on side 1: dst MAC = the router
sleep 1; sudo ip netns exec h1 ping -c 1 10.0.2.10
sudo ip netns exec r tcpdump -nn -e -i r-h2 -c 2 icmp         # frames on side 2: different MACs, SAME IP addresses
```
You have just seen the central rule live: **IPs unchanged, MACs rewritten at the router, TTL decremented**.

### 9.2 Give it an "internet" and NAT

```bash
sudo ip netns exec inet python3 -m http.server 80 --bind 203.0.113.2 &      # a pretend web server on the "internet"
sudo ip netns exec r ip route add default via 203.0.113.2                   # router's default route: toward the ISP
sudo ip netns exec h1 curl -s --max-time 3 http://203.0.113.2/ | head -3    # request reaches inet...
sudo ip netns exec inet ip route                                            # ...but inet has no route back to 10.0.1.0/24 → the reply is lost
```
This is exactly why NAT exists: the "internet" knows nothing about private networks. Add **masquerading** on the router's WAN interface:

```bash
sudo ip netns exec r iptables -t nat -A POSTROUTING -o r-inet -j MASQUERADE
sudo ip netns exec inet tcpdump -nn -i inet-r -c 6 'tcp port 80' &
sleep 1
sudo ip netns exec h1 curl -s http://203.0.113.2/ | head -3                # works; tcpdump shows the SOURCE as 203.0.113.1 (the router), not 10.0.1.10
sudo ip netns exec h2 curl -s http://203.0.113.2/ | head -3                # both LANs share the router's single "public" address
sudo ip netns exec r conntrack -L 2>/dev/null | head                       # the NAT table (if conntrack-tools is installed)
```

### 9.3 Try link-local (APIPA-style) on a "cable"

```bash
sudo ip netns add a; sudo ip netns add b
sudo ip link add a-b type veth peer name b-a
sudo ip link set a-b netns a; sudo ip link set b-a netns b
sudo ip netns exec a ip addr add 169.254.52.143/16 dev a-b; sudo ip netns exec a ip link set a-b up
sudo ip netns exec b ip addr add 169.254.100.50/16 dev b-a; sudo ip netns exec b ip link set b-a up
sudo ip netns exec a ping -c 2 169.254.100.50     # works: same link, same /16, ARP resolves directly
sudo ip netns exec a ping -c 1 8.8.8.8            # "Network is unreachable": no gateway
sudo ip netns del a; sudo ip netns del b
```

### 9.4 Clean up

```bash
sudo pkill -f "http.server 80" 2>/dev/null
for n in h1 h2 r inet; do sudo ip netns del $n; done      # deleting the namespaces removes their veth ends
```

**Ideas to extend:** add a third LAN; add a **DHCP server** (`dnsmasq` in namespace `r` with `--interface=r-h1 --dhcp-range=10.0.1.100,10.0.1.200`) and use `dhclient` in `h1`; block h1 from h2 with an `iptables -A FORWARD` rule on the router; add a static route and a second router; use `tcpdump` to watch DHCP (`udp port 67 or 68`) and ARP.

---

## 10. Docker and VMs are the same model

| Home network concept | Docker equivalent |
|---|---|
| Computer + NIC | Container + `eth0` (one end of a **veth pair**) |
| Ethernet switch | Linux **bridge** `docker0` / `br-<id>` |
| Router + default gateway | The **host**, at the bridge address (`172.17.0.1`), forwarding with `ip_forward=1` |
| NAT (masquerade) | iptables `MASQUERADE` rule on outbound traffic |
| Port forwarding on the router | `docker run -p 8080:80` (DNAT) |
| DHCP server | Docker's IPAM assigns addresses when containers start (or `--ip`) |
| DNS forwarder | Embedded DNS at `127.0.0.11` |

Check them: `docker network inspect bridge`, `ip route`, `sudo iptables -t nat -S | grep -i -E 'MASQ|DNAT'`, `sysctl net.ipv4.ip_forward`. **VMs:** a *NAT'd* vNIC (VirtualBox/VMware "NAT") is a private network behind a host-side router; a *bridged* vNIC joins your real LAN like a physical machine; "host-only" is a private switch between host and VMs.

---

## 11. Troubleshooting: bottom-up

| Step | Check | Fails? |
|---|---|---|
| 1. **Interface exists and is up** | `ip -br link` (`UP`, `LOWER_UP`), driver loaded (`lspci -k`, `dmesg`) | Driver/firmware, cable, disabled adapter, airplane mode |
| 2. **Link** | Link lights, `ethtool eth0` (`Link detected: yes`, speed) | Cable/port/switch |
| 3. **Address** | `ip -br addr`: DHCP address or **`169.254.x.x`** (= DHCP failed) | DHCP server/pool, VLAN, cable; `dhclient -r && dhclient`, `ipconfig /renew` |
| 4. **Gateway** | `ip route` has `default via …`; `ping <gateway>` | Wrong mask/subnet, no default route, ARP failing |
| 5. **Internet by IP** | `ping 1.1.1.1` | Router WAN down, ISP, NAT/firewall, no default route on the router |
| 6. **DNS** | `ping example.com` fails but `ping 1.1.1.1` works; `dig example.com` | Wrong DNS servers; fix DHCP/`resolv.conf` |
| 7. **Application** | `curl -v https://example.com`, firewall, proxy | Proxy/firewall/app problem |

Specific patterns:

| Symptom | Likely cause |
|---|---|
| **`169.254.x.x` address** | DHCP unreachable or none exists |
| Local devices reachable, internet not | No/incorrect default gateway; router WAN down; ISP outage; NAT disabled |
| Can ping IPs but not names | DNS |
| Can't ping a neighbor on the same subnet | **Mask mismatch** (e.g. one host `/24`, the other `/16`), firewall blocking ICMP, wrong VLAN, isolation ("client isolation" on Wi-Fi), duplicate IP |
| Slow or flaky | Wi-Fi interference, cable fault (check counters), overloaded router, duplex mismatch |
| Two devices fight/random drops | **IP conflict** (a static address inside the DHCP pool): `arping -D -I eth0 192.168.1.20` |
| Router login page unreachable | Wrong address (try `ip route | grep default`), or you're on the guest network |
| After router reset nothing works | Default IP range/DHCP changed; renew leases |

---

## 12. Common misconceptions

| Misconception | Reality |
|---|---|
| "A computer has an IP address" | Each **interface** has one (or several) |
| "APIPA gives internet access" | It's link-local only: no gateway, never routed |
| "`169.254.x.x` is normal" | On a real network it signals DHCP failure |
| "You need a crossover cable for two PCs" | Not anymore (Auto-MDI-X) |
| "A router and a switch are the same" | Different layers; a home 'router' contains both |
| "NAT is a firewall" | NAT rewrites addresses; the firewall is a separate (stateful) function |
| "The router forwards Ethernet broadcasts" | It doesn't; each interface is its own broadcast domain |
| "The modem, the router and the access point are one thing" | Often combined in one box, but they are distinct functions |
| "My router's public IP is unique to me" | May be shared via **CGNAT** (100.64.0.0/10 on the WAN side) |
| "Any device with a driver-less NIC will work" | Without a driver, the OS can't use the hardware |
| "IP addresses are burned into hardware" | MACs are (mostly); IPs are configured |
| "A default gateway is optional" | Without one, a host can't reach anything off its own subnet |

---

## 13. Summary

- A host needs a **NIC + driver** (MAC address), then an **IP address, subnet mask, default gateway and DNS**, from **DHCP**, **manual** config, or the **link-local `169.254.0.0/16`** fallback (no gateway, never routed; seeing it means DHCP failed).
- **Two machines + a cable** form the smallest network; **switches** avoid the N(N−1)/2 cabling problem (star topology); **Auto-MDI-X** ended the crossover-cable era.
- A **router** connects different networks at Layer 3, forwarding by routing table: destination IP unchanged, **MAC rewritten**, **TTL decremented**; `net.ipv4.ip_forward=1` is what makes a Linux host a router.
- A **home router** = router + switch + Wi-Fi AP + DHCP + DNS forwarder + NAT + firewall (+ maybe modem).
- Build the journey in layers (NIC → address → gateway → internet → DNS → app) and **troubleshoot bottom-up**.
- Docker bridges, veths, NAT and `-p` are the same building blocks in software.

---

## 14. Check your understanding

1. Why does an IP address belong to an interface, not a computer?
2. A Windows PC shows `169.254.17.203`. What does it mean, what works, what doesn't, and what would you check first?
3. What are the four settings a host needs to communicate beyond its own subnet?
4. How does a host decide whether to send a packet directly or to the default gateway?
5. Why did two-computer links need crossover cables, and why don't they now?
6. How many cables would a full mesh of 12 computers need? What's the star alternative?
7. In the namespace lab, `h1` can ping the router but not `h2` until one command is run on the router. Which, and why?
8. `ping` from `h1` to `h2` shows `ttl=63`. Explain.
9. Why does `curl` from `h1` to the "internet" host reach it but never get a reply until NAT is enabled?
10. List the functions of a typical home router.

<details>
<summary>Answers</summary>

1. A computer may have several interfaces (Ethernet, Wi-Fi, virtual ones), each connected to a network and each needing its own address; the address identifies the attachment point on a network.
2. A link-local (APIPA) address: DHCP failed. It works with other devices on the same link, but not for the internet, other subnets or DNS (no gateway). Check the cable/link lights, whether the router/DHCP server is on and has free leases, the switch port/VLAN, then renew (`ipconfig /renew`).
3. IP address, subnet mask (prefix), default gateway, DNS server(s).
4. It ANDs the destination with its mask: if the destination is inside its own subnet it ARPs and sends directly, otherwise it sends the packet to the default gateway.
5. Transmit and receive pairs were on different pins on hosts vs switches, so a computer-to-computer link needed the pairs crossed. Auto-MDI/MDI-X (mandatory on Gigabit) lets ports swap pairs automatically.
6. 12 × 11 / 2 = 66 cables (and 11 NICs per computer). A star with a switch needs 12 cables and one NIC per computer.
7. `sudo ip netns exec r sysctl -w net.ipv4.ip_forward=1`. Enabling IP forwarding lets the machine forward packets between its interfaces (acting as a router); otherwise it only accepts packets addressed to itself.
8. The initial TTL of 64 was decremented once by the router (hosts on the same subnet would show 64).
9. The pretend internet host has no route to the private `10.0.1.0/24` network, so replies to the private source address are lost. NAT rewrites the source to the router's public-side address, which the server can reply to.
10. Routing, Ethernet switching, Wi-Fi access point, DHCP server, DNS forwarding, NAT, stateful firewall, sometimes a modem.
</details>

**Practice**

1. On your machine, document every interface: name, MAC, IPv4/IPv6 addresses, driver (`ethtool -i`), and which routes use it. Identify the default gateway and confirm its MAC in `ip neigh`.
2. Unplug the cable (or disconnect from Wi-Fi) and reconnect while running `sudo tcpdump -i <if> -nn -e 'udp port 67 or 68 or arp'`; annotate the DHCP and ARP packets you see.
3. Run the namespace lab; then add a third LAN (`10.0.3.0/24`) and prove `h3` reaches `h1` and `h2` through the router. Add a firewall rule on the router that blocks h1 → h3 but allows h2 → h3.
4. Log in to your home router (from your default-gateway address), list its DHCP leases, and match a device's MAC (from `ip neigh`) to a lease entry.
5. Create the static/DHCP configurations in section 8 on a VM and deliberately break each element (wrong mask, missing gateway, wrong DNS) to see the exact symptom.

---

**Next:** [Chapter 33 – Subnetting and Subnet Masks](33_subnetting_and_subnet_masks_in_details.md)
