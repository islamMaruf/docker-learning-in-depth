# Chapter 41: Visualizing Multiple NICs in a Single Computer

> **In one sentence:** Your computer already has several network interfaces; this chapter teaches you to **list them on Linux, macOS and Windows**, to **decode their names** (`eth0`, `enp3s0`, `wlp2s0`, `en0`, `docker0`, `veth…`, `utun3`), and to tell **physical from virtual** interfaces.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~40 minutes

**Prerequisite:** [Chapter 40](40_multiple_nics_in_single_computer_in_details.md).

---

## What you will learn

- How interface names evolved, from `eth0` to **predictable names** like `enp3s0`, and how to **decode** them
- The **commands** to see interfaces on **Linux**, **macOS** and **Windows** (and how to read the output)
- How to recognise **physical** vs **virtual** interfaces, and what the common virtual ones are (`lo`, `docker0`, `br-…`, `veth…`, `tun0`, `wg0`, `virbr0`, `bond0`, `eth0.10`)
- How to see **which interface is used for what** (link to routing, Chapters 42–43)
- Exercises to map your own machine

---

## 1. Interface names: from `eth0` to `enp3s0`

An **interface name** is just a label the OS gives each network interface so commands and configs can refer to it (`ip addr show eth0`, `ping -I wlan0`).

### The legacy scheme (Linux ≤ ~2013)
Names were assigned **in the order the kernel found the devices**: `eth0`, `eth1`, ... for Ethernet; `wlan0`, `wlan1` for Wi-Fi.

**The problem:** the order could change between boots or after adding a card (driver load timing). Yesterday's `eth0` (LAN port) might become `eth1` today, and firewall rules or configs referring to "eth0" suddenly applied to the wrong network. On servers with several ports this was dangerous.

### Predictable Network Interface Names (systemd/udev)
Modern Linux (systemd ≥ 197) names interfaces **from the hardware's identity**, so they stay stable:

```
en  o  1          → Ethernet, onboard (firmware index) 1          eno1
en  s  3          → Ethernet, PCI hot-plug slot 3                 ens3   (common in VMs)
en  p 3 s 0       → Ethernet, PCI bus 3, slot 0                   enp3s0  (very common)
en  x aabbccddeeff→ Ethernet, from the MAC address                enxaabbccddeeff (USB adapters)
wl  p 2 s 0       → Wireless LAN, PCI bus 2, slot 0               wlp2s0
ww  ...           → Wireless WAN (mobile broadband)               wwp0s20u4
```

| Prefix | Meaning |
|---|---|
| `en` | Ethernet |
| `wl` | Wireless LAN (Wi-Fi) |
| `ww` | Wireless WAN (cellular modem) |
| `ib` | InfiniBand |

| Infix | Meaning |
|---|---|
| `o<n>` | Onboard device index provided by firmware |
| `s<n>` | PCI hot-plug slot number |
| `p<bus>s<slot>` | PCI bus and slot (physical location) |
| `x<MAC>` | Uses the MAC address (stable even if the port moves) |
| `u<port>` | USB port path |

`enp3s0` therefore reads "**en**thernet on **p**ci bus **3**, **s**lot **0**". Move the card to another slot and the name changes; but it stays the same across reboots.

You can override names (systemd `.link` files, `net.ifnames=0` kernel parameter to return to `eth0`, or custom names like `lan0`/`wan0` on routers), and cloud/VM images often use `eth0`/`ens5`/`enX0` depending on the platform.

| | Legacy | Predictable |
|---|---|---|
| Example | `eth0`, `wlan0` | `enp3s0`, `wlp2s0`, `eno1` |
| Stability | Can change between boots | Stable |
| Readability | Simple | Encodes hardware position |
| Where you'll see it | Old distros, containers, some VMs/embedded | Current desktop/server Linux |

**Containers** always name their interface `eth0` (inside their own network namespace), regardless of the host's naming.

### macOS names
`en0`, `en1`, … (**en**et = Ethernet, but **`en0` is usually Wi-Fi on a laptop**; the Ethernet/Thunderbolt adapter is often `en5`, `en6`); `lo0` loopback; `utunN` VPN/system tunnels; `bridge0` (Thunderbolt/sharing), `awdl0`/`llw0` (AirDrop/low-latency Wi-Fi), `anpi0` etc. (Apple internals), `gif0`/`stf0` (tunnels). Numbers differ per Mac; use `networksetup -listallhardwareports` to map "Wi-Fi" → `en0`.

### Windows names
Friendly names such as **"Ethernet"**, **"Ethernet 2"**, **"Wi-Fi"**, **"Local Area Connection\* 1"** (hidden Wi-Fi Direct), **"vEthernet (WSL)"**, **"vEthernet (Default Switch)"** (Hyper-V), **"Bluetooth Network Connection"**, **"Loopback Pseudo-Interface 1"**; also an internal GUID/interface index (`ifIndex`).

---

## 2. Seeing your interfaces

### 2.1 Linux

```bash
ip -br link                 # brief: name, state, MAC
ip -br addr                 # brief: name, state, IP addresses
ip addr                     # full: flags, MTU, all addresses
ip -s link show eth0        # + packet/byte/error counters
ls /sys/class/net           # just the names
```
Sample `ip -br addr`:

```
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp3s0           UP             192.168.1.10/24 fe80::a1b2:c3ff:fe11:2233/64
wlp2s0           DOWN
docker0          UP             172.17.0.1/16
br-2f4a9c1e7b3d  UP             172.18.0.1/16
veth9a1b2c3@if7  UP
```
Reading `ip addr` fully (one interface):

```
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether aa:bb:cc:11:22:33 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.10/24 brd 192.168.1.255 scope global dynamic noprefixroute enp3s0
       valid_lft 85231sec preferred_lft 85231sec
    inet6 fe80::a1b2:c3ff:fe11:2233/64 scope link
```

| Piece | Meaning |
|---|---|
| `2:` | Interface index |
| `enp3s0` | Name |
| `UP` / `LOWER_UP` | Administratively enabled / **the cable/link is up (carrier)**; `NO-CARRIER` = no cable/AP |
| `mtu 1500` | Maximum payload size (Chapter 30/31) |
| `state UP` | Operational state |
| `link/ether aa:bb:…` | MAC address; `brd ff:ff:…` = broadcast |
| `inet 192.168.1.10/24` | IPv4 address and prefix; `brd` = broadcast; `scope global` |
| `dynamic` / `valid_lft` | Came from DHCP; lease remaining |
| `inet6 fe80::…` | IPv6 link-local |

Other useful commands:

```bash
ip -d link show docker0       # -d details: shows "bridge", "veth", "vlan", "tun" ... i.e. the interface TYPE
ethtool enp3s0                # speed, duplex, link detected (physical NICs)
ethtool -i enp3s0             # driver, firmware, PCI bus address
lspci | grep -i -E 'ethernet|network'
nmcli device status           # NetworkManager view: type (ethernet/wifi/bridge/tun), state, connection
resolvectl status             # DNS per interface
ifconfig -a                   # legacy net-tools equivalent
```

### 2.2 macOS

```bash
ifconfig                                   # all interfaces (flags, MAC "ether", inet, status)
ifconfig en0                               # one interface
networksetup -listallhardwareports         # "Hardware Port: Wi-Fi  Device: en0  Ethernet Address: ..."
networksetup -listnetworkserviceorder      # service order = priority for default route
ipconfig getifaddr en0                     # just the IPv4 address
netstat -rn -f inet                        # routing table (Chapter 42)
scutil --nwi                               # network state summary: which interface is primary
```
GUI: **System Settings → Network** (services listed in priority order, with a green dot for active ones); **Wireless Diagnostics / Option-click Wi-Fi menu** for details.

### 2.3 Windows

```powershell
ipconfig /all                              # every adapter: description, physical address, DHCP, IPs, gateway, DNS
Get-NetAdapter                             # name, InterfaceDescription, Status, MacAddress, LinkSpeed
Get-NetIPAddress | Sort InterfaceAlias     # addresses per interface
Get-NetIPInterface                         # metric and DHCP status per interface
route print                                # routing table with interface indexes
getmac /v                                  # MAC addresses
```
GUI: **Settings → Network & Internet → Advanced network settings**, or `ncpa.cpl` (classic "Network Connections"). Device Manager → **Network adapters** shows hardware and virtual adapters.

---

## 3. Physical vs virtual interfaces

### 3.1 Physical
Backed by a hardware NIC and a hardware driver: Ethernet (`enp3s0`, `eno1`), Wi-Fi (`wlp2s0`), cellular (`wwan0`), USB adapters (`enx…`). They have a real **MAC**, link speed, driver (`ethtool -i`), and a PCI/USB device (`lspci`, `lsusb`).

### 3.2 Virtual
Created by software; no hardware needed:

| Name (Linux) | What it is |
|---|---|
| **`lo`** | **Loopback**: `127.0.0.1/8` (and `::1`); traffic to yourself never touches a wire (Chapter 42) |
| **`docker0`**, **`br-<id>`** | **Bridges** (software switches) for Docker networks; `172.17.0.1/16`, `172.18.0.1/16`... |
| **`veth<hash>`** | One end of a **virtual Ethernet pair**; the other end is a container's `eth0` (`@ifN` shows the peer's index) |
| **`virbr0`** | libvirt/KVM's NAT bridge (`192.168.122.1/24`) |
| **`vboxnet0`**, **`vmnet1`** | VirtualBox/VMware host-only networks |
| **`tun0`**, **`tap0`**, **`wg0`**, **`tailscale0`** | **VPN** tunnels: `tun` carries IP packets, `tap` Ethernet frames |
| **`bond0`**, **`team0`** | Aggregated NICs (bonding/LACP) |
| **`enp3s0.10`** | **VLAN** sub-interface (802.1Q tag 10) |
| **`macvlan0`**, **`ipvlan0`** | Extra MAC/IP identities on one physical NIC |
| **`vxlan0`**, **`flannel.1`**, **`cni0`**, **`cali…`** | Overlay/Kubernetes networking |
| **`dummy0`**, **`ifb0`** | Test/traffic-shaping helpers |

**Windows equivalents:** *vEthernet (WSL)*, *vEthernet (Default Switch)*, *TAP-Windows Adapter*, *Wintun*, *Loopback Pseudo-Interface*. **macOS:** `utunN`, `bridge0`, `lo0`.

Before Docker is installed you do not see `docker0`; right after installation you do (`172.17.0.1/16`), and starting a container adds a `veth` for it (removed when it stops).

### 3.3 How to tell quickly

```bash
ls -l /sys/class/net
# lrwxrwxrwx ... enp3s0 -> ../../devices/pci0000:00/0000:00:1c.2/0000:03:00.0/net/enp3s0      ← physical: under a PCI device
# lrwxrwxrwx ... docker0 -> ../../devices/virtual/net/docker0                                  ← virtual
# lrwxrwxrwx ... lo -> ../../devices/virtual/net/lo                                            ← virtual

ip -d link show | grep -E '^[0-9]+:|veth|bridge|tun|vlan|bond'    # type keywords
nmcli device status                                                # TYPE column: ethernet/wifi vs bridge/tun/veth
ethtool -i docker0 | head -3          # driver: bridge   (virtual)     vs  driver: e1000e / r8169 (physical)
```
| Clue | Physical | Virtual |
|---|---|---|
| `/sys/class/net/<if>` target | under `/devices/pci…` or `/usb…` | under `/devices/virtual/net/…` |
| `ethtool -i` driver | hardware driver (e1000e, r8169, iwlwifi) | `bridge`, `veth`, `tun`, `bonding` |
| `lspci` / `lsusb` entry | yes | no |
| Link speed | fixed (`1000Mb/s`) | often `10000Mb/s`/unknown |
| Created/destroyed | with hardware | by software (`ip link add`, Docker, VPN client) |

Virtual interfaces behave like physical ones from the routing perspective: they have addresses, routes and ARP entries (Chapter 40).

---

## 4. A realistic laptop, annotated

`ip -br addr` on a Linux laptop with Docker and a VPN:

```
lo               UNKNOWN  127.0.0.1/8 ::1/128                  ← loopback (virtual)
enp0s31f6        DOWN                                           ← Ethernet port, cable unplugged (physical)
wlp2s0           UP       192.168.1.42/24 fe80::…/64            ← Wi-Fi, connected (physical) → default route uses this
docker0          UP       172.17.0.1/16                         ← Docker's default bridge (virtual)
br-3fa2c1d09e2b  UP       172.18.0.1/16                         ← a user-defined Docker network (virtual)
veth7c1a2f0@if8  UP                                             ← a running container's host-side end (virtual, no IP)
tun0             UNKNOWN  10.8.0.6/24                           ← corporate VPN (virtual)
```
The same on macOS (`ifconfig`): `lo0`, `en0` (Wi-Fi, the active one), `en5` (Thunderbolt Ethernet, inactive), `bridge0`, `awdl0`, `utun0…utun4` (VPN/system), and on Windows `ipconfig /all`: "Wi-Fi", "Ethernet" (disconnected), "vEthernet (WSL)", "Loopback Pseudo-Interface 1", "Bluetooth Network Connection".

You now have **five or more interfaces** on a typical developer laptop, so the routing logic of Chapters 42–43 is running all the time.

---

## 5. Seeing which interface actually carries what

```bash
ip route get 1.1.1.1              # "1.1.1.1 via 192.168.1.1 dev wlp2s0 src 192.168.1.42": Internet goes via Wi-Fi
ip route get 172.17.0.2           # "172.17.0.2 dev docker0 src 172.17.0.1": containers via the bridge
ip route get 10.8.0.1             # "dev tun0": VPN network via the tunnel
ip route get 127.0.0.1            # "local 127.0.0.1 dev lo src 127.0.0.1": never leaves the machine
ip -s link show wlp2s0            # RX/TX bytes: which NIC is busy
sudo tcpdump -i wlp2s0 -nn -c 5   # watch packets on a chosen interface
sudo tcpdump -i any -nn -c 10     # watch all interfaces at once (adds the interface name)
```
macOS: `route -n get 1.1.1.1` (prints `interface: en0`). Windows: `Find-NetRoute -RemoteIPAddress 1.1.1.1` or `Test-NetConnection 1.1.1.1 -TraceRoute`.

Bandwidth per interface: `ip -s link`, `nload`, `iftop -i <if>`, `bmon`, `vnstat`, Task Manager → Performance (Windows), Activity Monitor → Network (macOS).

---

## 6. Exercises

**Exercise 1: map your machine.** Fill in the table for every interface:

| Name | Physical/virtual | Type | State | MAC | IP/prefix | Purpose |
|---|---|---|---|---|---|---|
| | | | | | | |

Linux: `ip -br link; ip -br addr; ls -l /sys/class/net; nmcli device status`. macOS: `ifconfig; networksetup -listallhardwareports`. Windows: `Get-NetAdapter; ipconfig /all`.

**Exercise 2: decode names.** Explain what `eno1`, `ens33`, `enp0s31f6`, `wlp3s0`, `enx001122334455`, `wwp0s20f0u6` mean. (Answers below.)

**Exercise 3: watch interfaces come and go.**
```bash
watch -n1 'ip -br link'         # in one terminal
docker run --rm -it alpine sh   # a new veth appears on the host; exit → it disappears
sudo ip link add dummy0 type dummy; sudo ip link set dummy0 up; sudo ip addr add 10.99.0.1/24 dev dummy0
ip route | grep 10.99           # the kernel created a connected route automatically
sudo ip link del dummy0
```
**Exercise 4: find the primary interface.** Which interface carries the default route? (`ip route show default`; macOS `route -n get default`; Windows `Get-NetRoute -DestinationPrefix 0.0.0.0/0`). If several, compare **metrics**.

**Exercise 5: find your container's twin.** In a container, run `cat /sys/class/net/eth0/iflink` (or `ip link` and read `eth0@if9`); on the host find the interface whose **index is 9**: that's the container's `veth`.

<details>
<summary>Answers to exercise 2</summary>

- `eno1`: Ethernet, **o**nboard, index 1.
- `ens33`: Ethernet, PCI hot-plug **s**lot 33 (typical in VMware VMs).
- `enp0s31f6`: Ethernet, PCI bus 0, slot 31, function 6 (a common onboard Intel NIC).
- `wlp3s0`: Wireless LAN, PCI bus 3, slot 0.
- `enx001122334455`: Ethernet named after the MAC `00:11:22:33:44:55` (typical USB dongle).
- `wwp0s20f0u6`: Wireless WAN (mobile broadband modem) at PCI bus 0/slot 20/function 0 on USB port 6.
</details>

---

## 7. Common misconceptions

| Misconception | Reality |
|---|---|
| "`eth0` is the first physical NIC" | Only in old/legacy naming; modern names encode hardware location; containers still use `eth0` internally |
| "`en0` on a Mac is Ethernet" | Usually **Wi-Fi** on laptops (`en` is historical) |
| "Virtual interfaces aren't used for routing" | They route exactly like physical ones |
| "`lo` is a fake and useless" | Used constantly: local servers, databases, IPC over TCP, health checks |
| "Every interface has an IP" | Bridge ports, veth host sides, and monitor-mode NICs often have none |
| "`state UP` means it has a link" | Check `LOWER_UP`/`NO-CARRIER` for the physical link |
| "More interfaces = more bandwidth" | Not without bonding or multipath |
| "If it's not in `ifconfig` it doesn't exist" | `ifconfig` hides down interfaces without `-a`, and net-tools is deprecated; use `ip` |

---

## 8. Summary

- Interfaces are named for convenience. Old **`eth0`** names depended on detection order; modern **predictable names** (`enp3s0`, `wlp2s0`, `eno1`, `enx<mac>`) encode hardware location or MAC for stability. macOS uses `enN`/`utunN`, Windows uses friendly names.
- List them with `ip -br addr` / `ip -d link` (Linux), `ifconfig` / `networksetup` (macOS), `ipconfig /all` / `Get-NetAdapter` (Windows).
- Interfaces are **physical** (PCI/USB hardware, real driver) or **virtual** (`lo`, bridges, veths, tunnels, VLANs, bonds); check `/sys/class/net`, `ethtool -i`, `nmcli`.
- A normal developer laptop has 5+ interfaces at all times; `ip route get <dest>` reveals which one a destination uses.

---

## 9. Check your understanding

1. Why did Linux move away from `eth0`/`eth1` naming?
2. Decode `enp5s0`, `wlp2s0` and `enxaabbccddeeff`.
3. Which command shows the **type** of an interface (bridge, veth, vlan…)?
4. How can you tell `docker0` is virtual, using `/sys/class/net`?
5. What do `UP` and `LOWER_UP` mean, and what does `NO-CARRIER` indicate?
6. What is the `veth…@if7` interface on a Docker host?
7. On a Mac, which interface is normally Wi-Fi?
8. Which command tells you the interface that will be used to reach `1.1.1.1`?

<details>
<summary>Answers</summary>

1. The order in which the kernel discovered devices could change between boots, so configs could hit the wrong NIC; predictable names are derived from stable hardware identity.
2. Ethernet on PCI bus 5 slot 0; Wi-Fi on PCI bus 2 slot 0; Ethernet named from MAC `aa:bb:cc:dd:ee:ff` (usually USB).
3. `ip -d link show` (or `nmcli device status`, `ethtool -i`).
4. `ls -l /sys/class/net` shows `docker0 -> ../../devices/virtual/net/docker0` (physical NICs point under `/devices/pci…`).
5. `UP`: administratively enabled; `LOWER_UP`: link/carrier present; `NO-CARRIER`: no cable/access point detected.
6. The host end of a virtual Ethernet pair whose other end is a container's `eth0` (plugged into a bridge like `docker0`).
7. `en0` (verify with `networksetup -listallhardwareports`).
8. `ip route get 1.1.1.1` (Linux), `route -n get 1.1.1.1` (macOS), `Find-NetRoute -RemoteIPAddress 1.1.1.1` (Windows).
</details>

---

**Next:** [Chapter 42 – The Routing Table](42_routing_table_in_details.md)
