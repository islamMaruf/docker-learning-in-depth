# Chapter 049: Routing Table In Details

## Overview

You've learned how a computer with multiple NICs can connect to multiple networks simultaneously. You understand that each NIC gets a unique IP address from its respective router via DHCP. You've seen the hardware setup—Ethernet cables, PCI slots, and multiple physical connections.

**But here's the critical problem: When your computer needs to send data, which NIC should it use?**

Consider this scenario:
- **Your computer has 3 NICs**
- **NIC 1 (en0):** Connected to Router 0, IP: 192.168.1.10
- **NIC 2 (eth-pci-bus1-slot0):** Connected to Router 1, IP: 192.168.2.12
- **NIC 3 (wlp3s0):** Connected to Router 2, IP: 192.168.3.13

You open your browser and navigate to `192.168.3.10`. **Which NIC does your operating system use to send this request?**

The answer lies in the **routing table**—one of the most fundamental data structures in networking. Every operating system maintains a routing table, and every router maintains one too. This table is the decision-making engine that determines the path for every single packet leaving your computer.

This chapter explores:
- **What is a routing table and why it exists**
- **The structure of routing tables:** Destination, gateway, flags, network interface, expiry
- **How the OS determines the correct NIC:** Binary AND operations and network address matching
- **Viewing routing tables:** Commands for Windows, macOS, and Linux
- **How routing tables are populated:** DHCP's role in adding routes
- **Network address calculation:** Understanding the relationship between IP addresses, subnet masks, and network addresses
- **The routing decision algorithm:** Step-by-step walkthrough of how packets find their path
- **Default routes and gateways:** The catch-all rule for unknown destinations

By the end of this chapter, you'll understand exactly how your computer (and routers) make intelligent routing decisions thousands of times per second.

---

## The Problem: Multiple Paths, One Decision

### Review: Computer with Multiple NICs

From the last chapter, we established this topology:

```
                Router 0                Router 1                Router 2
             (192.168.1.0/24)       (192.168.2.0/24)       (192.168.3.0/24)
                    │                       │                       │
                    │                       │                       │
            Network 192.168.1.0    Network 192.168.2.0    Network 192.168.3.0
                    │                       │                       │
                    │                       │                       │
              ┌─────┴──────┐          ┌─────┴──────┐          ┌─────┴──────┐
              │   Router 0 │          │   Router 1 │          │   Router 2 │
              │ LAN: .1.1  │          │ LAN: .2.1  │          │ LAN: .3.1  │
              │ (Gateway)  │          │ (Gateway)  │          │ (Gateway)  │
              └─────┬──────┘          └─────┬──────┘          └─────┬──────┘
                    │                       │                       │
                    │                       │                       │
                    ├───► Host Computer 1   ├───► Host Computer 3   ├───► Host Computer 5
                    │      IP: .1.12        │      IP: .2.10        │      IP: .3.10
                    │                       │                       │
                    │                       │                       │
                    └─────────────┐ ┌───────┴─────────┐ ┌───────────┴──────────┐
                                  │ │                 │ │                      │
                                  │ │                 │ │                      │
                            ┌─────▼─▼─────┐   ┌───────▼─▼──────┐   ┌─────────▼▼─────────┐
                            │     NIC 1   │   │     NIC 2       │   │       NIC 3        │
                            │ en0         │   │eth-pci-bus1-s0  │   │    wlp3s0          │
                            │192.168.1.10 │   │ 192.168.2.12    │   │   192.168.3.13     │
                            └─────────────┘   └─────────────────┘   └────────────────────┘
                                  │                   │                       │
                                  └───────────────────┴───────────────────────┘
                                                      │
                                        ┌─────────────▼──────────────┐
                                        │   YOUR COMPUTER            │
                                        │   with 3 NICs              │
                                        │   (My Computer)            │
                                        └────────────────────────────┘
```

**Your computer's network interfaces:**

| NIC Name             | IP Address      | Network            | Gateway       | Subnet Mask     |
|----------------------|-----------------|--------------------| --------------|-----------------|
| en0                  | 192.168.1.10    | 192.168.1.0/24     | 192.168.1.1   | 255.255.255.0   |
| eth-pci-bus1-slot0   | 192.168.2.12    | 192.168.2.0/24     | 192.168.2.1   | 255.255.255.0   |
| wlp3s0               | 192.168.3.13    | 192.168.3.0/24     | 192.168.3.1   | 255.255.255.0   |

**Additional computers in each network:**

**Network 1 (192.168.1.0/24):**
- Router 0 LAN: 192.168.1.1
- Your NIC 1: 192.168.1.10
- Host Computer 1: 192.168.1.12

**Network 2 (192.168.2.0/24):**
- Router 1 LAN: 192.168.2.1
- Your NIC 2: 192.168.2.12
- Host Computer 3: 192.168.2.10

**Network 3 (192.168.3.0/24):**
- Router 2 LAN: 192.168.3.1
- Your NIC 3: 192.168.3.13
- Host Computer 5: 192.168.3.10
- Host Computer 6: 192.168.3.12

---

### The Critical Question

**Scenario 1: You want to send data to `192.168.3.10`**

Your computer needs to answer: **Which NIC should send this data?**

**Possible answers:**
- A) NIC 1 (en0) with IP 192.168.1.10?
- B) NIC 2 (eth-pci-bus1-slot0) with IP 192.168.2.12?
- C) NIC 3 (wlp3s0) with IP 192.168.3.13?

**Scenario 2: You want to send data to `192.168.2.10`**

**Which NIC should send this data?**

**Scenario 3: You want to send data to `8.8.8.8` (Google DNS)**

**Which NIC should send this data?**

**The operating system cannot guess randomly!** If it sends the packet through the wrong NIC, the packet will:
1. Enter the wrong network
2. Reach the wrong router
3. The destination will never receive it
4. Communication fails

**This is where the routing table saves the day.**

---

## What is a Routing Table?

### Definition

**Routing Table:** A data structure maintained by the operating system (and routers) that maps destination addresses to network interfaces and gateways.

Think of it as a **GPS navigation system** for network packets. Just as GPS tells you which road to take to reach a destination, the routing table tells the OS which network interface to use.

```
┌───────────────────────────────────────────────────────────┐
│                    ROUTING TABLE                          │
├─────────────────┬───────────┬────────┬──────────────┬─────┤
│ Destination     │ Gateway   │ Flags  │ Interface    │ Exp │
├─────────────────┼───────────┼────────┼──────────────┼─────┤
│ 192.168.1.0/24  │ 0.0.0.0   │ U      │ en0          │  -  │
│ 192.168.2.0/24  │ 0.0.0.0   │ U      │ eth-pci-bus1 │  -  │
│ 192.168.3.0/24  │ 0.0.0.0   │ U      │ wlp3s0       │  -  │
│ 0.0.0.0/0       │ 192.168.1.1│ UG    │ en0          │  -  │
└─────────────────┴───────────┴────────┴──────────────┴─────┘
```

**Each row in the routing table answers the question:**

> "If I need to send a packet to **[Destination]**, I should send it through **[Interface]** via **[Gateway]**."

---

### Why Every Device Needs a Routing Table

**Every network-capable device maintains a routing table:**

```
┌────────────────────────────────────────────┐
│ Device Type        │ Has Routing Table?    │
├────────────────────┼───────────────────────┤
│ Personal Computer  │ YES ✓                 │
│ Laptop             │ YES ✓                 │
│ Smartphone         │ YES ✓                 │
│ Router             │ YES ✓ (more complex)  │
│ Server             │ YES ✓                 │
│ IoT Device         │ YES ✓ (simplified)    │
│ Switch (Layer 2)   │ NO ✗ (uses CAM table) │
│ Hub                │ NO ✗ (no intelligence)│
└────────────────────┴───────────────────────┘
```

**Why switches and hubs don't have routing tables:**
- **Switches:** Operate at Layer 2 (Data Link), use CAM tables that map MAC addresses to ports, don't understand IP addresses
- **Hubs:** Operate at Layer 1 (Physical), simply repeat signals to all ports, no decision-making

**Why computers and routers need routing tables:**
- They operate at Layer 3 (Network)
- They understand IP addresses
- They need to make intelligent forwarding decisions
- They may have multiple network interfaces

---

### Routing Table Structure: Column by Column

Let's dissect each column in detail.

#### 1. Destination

**Format:** Network address in CIDR notation (e.g., `192.168.1.0/24`)

**Meaning:** "This rule applies to packets destined for any IP address within this network range."

**Examples:**

```
Destination: 192.168.1.0/24
Matches: 192.168.1.0, 192.168.1.1, 192.168.1.2, ..., 192.168.1.254, 192.168.1.255
(All 256 addresses in this subnet)

Destination: 192.168.2.0/24
Matches: 192.168.2.0 through 192.168.2.255

Destination: 0.0.0.0/0
Matches: ALL IP addresses (default route)
```

**Special destinations:**

```
0.0.0.0/0        → Default route (catch-all for everything)
127.0.0.0/8      → Loopback addresses (lo interface)
169.254.0.0/16   → Link-local addresses (APIPA)
224.0.0.0/4      → Multicast addresses
```

---

#### 2. Gateway

**Format:** IP address (e.g., `192.168.1.1`) or `0.0.0.0`

**Meaning:** "To reach this destination, send the packet to this gateway (router)."

**Two possibilities:**

**Gateway = 0.0.0.0 (or "On-link"):**
- Destination is **directly reachable** on the local network
- No intermediate router needed
- Computer can ARP for the destination MAC address directly

```
Example:
Destination: 192.168.1.0/24
Gateway: 0.0.0.0
Meaning: "To reach any computer in 192.168.1.0/24, send directly through the 
         interface. No router needed."
```

**Gateway = Router IP (e.g., 192.168.1.1):**
- Destination is **NOT directly reachable**
- Must send to the router (gateway), which will forward it
- Computer ARPs for the gateway's MAC address, not the destination's

```
Example:
Destination: 0.0.0.0/0 (Default route)
Gateway: 192.168.1.1
Meaning: "For any destination not matching other rules, send to the router
         at 192.168.1.1. The router will figure out the rest."
```

---

#### 3. Flags

**Format:** Letter codes (e.g., `U`, `UG`, `UH`, `UGHS`)

**Meaning:** Metadata about the route.

**Common flags:**

```
U  (Up)             → Route is active and usable
G  (Gateway)        → Route uses a gateway (not direct)
H  (Host)           → Route is for a specific host (not a network)
D  (Dynamic)        → Route was added dynamically (by routing protocol)
M  (Modified)       → Route was modified by routing daemon
S  (Static)         → Route was manually added
!  (Reject)         → Route is rejected (packets dropped)
```

**Examples:**

```
Flags: U
Meaning: Direct route (no gateway), interface is up

Flags: UG
Meaning: Route uses a gateway, interface is up

Flags: UH
Meaning: Host-specific route (e.g., 192.168.1.100/32), interface is up
```

**Why flags matter:**
- OS checks the `U` flag to ensure route is active
- `G` flag tells OS to look up gateway MAC address, not destination MAC
- `H` flag indicates precise host route (higher priority than network routes)

---

#### 4. Network Interface

**Format:** Interface name (e.g., `en0`, `eth0`, `wlp3s0`)

**Meaning:** "Send the packet out through this physical/virtual network interface."

**Examples:**

```
Interface: en0
Meaning: Use the built-in Ethernet interface (on macOS)

Interface: eth0
Meaning: Use the first Ethernet interface (on Linux)

Interface: wlan0
Meaning: Use the wireless interface

Interface: lo
Meaning: Loopback interface (packets stay within the computer)

Interface: tun0
Meaning: VPN tunnel interface
```

**This is the final answer the routing table provides:** Which NIC to use!

---

#### 5. Expire (Optional)

**Format:** Timestamp or `-` (permanent)

**Meaning:** When this route expires.

**Use cases:**
- **Dynamic routes:** Added by DHCP or routing protocols, may expire
- **Static routes:** Manually added, typically permanent (`-`)

```
Example:
Expire: 3600s
Meaning: This route will be removed after 3600 seconds (1 hour)

Expire: -
Meaning: Permanent route (won't expire)
```

---

## Viewing the Routing Table

### Commands by Operating System

Every OS provides command-line tools to view the routing table.

#### Windows

**Command:**

```cmd
route print
```

or

```cmd
netstat -rn
```

**Example output:**

```
===========================================================================
Interface List
 12...00 ff 4c a5 b2 17 ......Intel(R) Ethernet Connection
 18...ac de 48 00 11 22 ......TP-Link USB WiFi Adapter
  1...........................Software Loopback Interface 1
===========================================================================

IPv4 Route Table
===========================================================================
Active Routes:
Network Destination        Netmask          Gateway       Interface  Metric
          0.0.0.0          0.0.0.0      192.168.1.1    192.168.1.10      25
        127.0.0.0        255.0.0.0         On-link         127.0.0.1     331
      192.168.1.0    255.255.255.0         On-link      192.168.1.10     281
     192.168.1.10  255.255.255.255         On-link      192.168.1.10     281
    192.168.1.255  255.255.255.255         On-link      192.168.1.10     281
      192.168.2.0    255.255.255.0         On-link      192.168.2.12     281
     192.168.2.12  255.255.255.255         On-link      192.168.2.12     281
    192.168.2.255  255.255.255.255         On-link      192.168.2.12     281
      192.168.3.0    255.255.255.0         On-link      192.168.3.13     281
     192.168.3.13  255.255.255.255         On-link      192.168.3.13     281
    192.168.3.255  255.255.255.255         On-link      192.168.3.13     281
===========================================================================
```

---

#### macOS

**Command:**

```bash
netstat -rn
```

**Example output:**

```
Routing tables

Internet:
Destination        Gateway            Flags        Netif Expire
default            192.168.1.1        UGSc           en0
127.0.0.1          127.0.0.1          UH             lo0
192.168.1.0/24     link#4             UCS            en0      !
192.168.1.1        0:50:56:ff:1a:b2   UHLWIir        en0   1168
192.168.1.10       127.0.0.1          UHS            lo0
192.168.2.0/24     link#8             UCS     eth-pci-bus1  !
192.168.2.12       127.0.0.1          UHS            lo0
192.168.3.0/24     link#12            UCS         wlp3s0      !
192.168.3.13       127.0.0.1          UHS            lo0
```

**Explanation:**
- `default` = `0.0.0.0/0` (all destinations)
- `link#4` = Direct connection (no gateway needed)
- `UGSc` = Up, Gateway, Static, Cloning (can create host-specific routes)

---

#### Linux

**Command:**

```bash
ip route
```

or

```bash
route -n
```

**Example output (`ip route`):**

```
default via 192.168.1.1 dev en0 proto dhcp metric 100
192.168.1.0/24 dev en0 proto kernel scope link src 192.168.1.10 metric 100
192.168.2.0/24 dev eth-pci-bus1-slot0 proto kernel scope link src 192.168.2.12 metric 200
192.168.3.0/24 dev wlp3s0 proto kernel scope link src 192.168.3.13 metric 300
```

**Explanation:**
- `default via X` = Default route through gateway X
- `dev en0` = Use device (interface) en0
- `proto kernel` = Route added by kernel
- `scope link` = Directly connected network
- `src X` = Use X as source IP when sending

**Example output (`route -n`):**

```
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    100    0        0 en0
192.168.1.0     0.0.0.0         255.255.255.0   U     100    0        0 en0
192.168.2.0     0.0.0.0         255.255.255.0   U     200    0        0 eth-pci-bus1-slot0
192.168.3.0     0.0.0.0         255.255.255.0   U     300    0        0 wlp3s0
```

---

### Understanding Your Computer's Routing Table

Let's analyze your computer's routing table in detail:

```
┌─────────────────┬───────────────┬────────┬──────────────────┐
│ Destination     │ Gateway       │ Flags  │ Interface        │
├─────────────────┼───────────────┼────────┼──────────────────┤
│ 192.168.1.0/24  │ 0.0.0.0       │ U      │ en0              │
│ 192.168.2.0/24  │ 0.0.0.0       │ U      │ eth-pci-bus1-s0  │
│ 192.168.3.0/24  │ 0.0.0.0       │ U      │ wlp3s0           │
│ 0.0.0.0/0       │ 192.168.1.1   │ UG     │ en0              │
└─────────────────┴───────────────┴────────┴──────────────────┘
```

#### Row 1: Direct route to Network 1

```
Destination: 192.168.1.0/24
Gateway: 0.0.0.0
Interface: en0
```

**Meaning:**

> "To reach any IP in 192.168.1.0-255, send directly through `en0`. No gateway needed because this network is directly connected to `en0`."

**When this rule is used:**
- Destination IP: `192.168.1.12` → Matches! Use `en0`.
- Destination IP: `192.168.1.100` → Matches! Use `en0`.
- Destination IP: `192.168.1.1` (router) → Matches! Use `en0`.

---

#### Row 2: Direct route to Network 2

```
Destination: 192.168.2.0/24
Gateway: 0.0.0.0
Interface: eth-pci-bus1-slot0
```

**Meaning:**

> "To reach any IP in 192.168.2.0-255, send directly through `eth-pci-bus1-slot0`. No gateway needed."

**When this rule is used:**
- Destination IP: `192.168.2.10` → Matches! Use `eth-pci-bus1-slot0`.
- Destination IP: `192.168.2.1` (router) → Matches! Use `eth-pci-bus1-slot0`.

---

#### Row 3: Direct route to Network 3

```
Destination: 192.168.3.0/24
Gateway: 0.0.0.0
Interface: wlp3s0
```

**Meaning:**

> "To reach any IP in 192.168.3.0-255, send directly through `wlp3s0`. No gateway needed."

**When this rule is used:**
- Destination IP: `192.168.3.10` → Matches! Use `wlp3s0`.
- Destination IP: `192.168.3.12` → Matches! Use `wlp3s0`.

---

#### Row 4: Default route (catch-all)

```
Destination: 0.0.0.0/0
Gateway: 192.168.1.1
Interface: en0
```

**Meaning:**

> "For any destination that doesn't match the above rules (like Internet addresses), send to gateway `192.168.1.1` through `en0`."

**When this rule is used:**
- Destination IP: `8.8.8.8` (Google DNS) → Doesn't match Networks 1, 2, or 3 → Use default route!
- Destination IP: `1.1.1.1` (Cloudflare DNS) → Use default route!
- Destination IP: `93.184.216.34` (example.com) → Use default route!

**Why only one default route?**

Your computer **could** have multiple default routes with different metrics (priorities), but typically:
- One primary internet connection is chosen
- Usually the first NIC configured or the one with the best connection
- In this example, `en0` (Network 1) is the primary internet gateway

---

## Network Address Calculation: The Foundation

### Why Network Addresses Matter

The routing table uses **network addresses** (e.g., `192.168.1.0/24`), not individual host IPs. But how is the network address derived?

**Answer: Binary AND operation between IP address and subnet mask.**

---

### Review: Binary AND Operation

**AND Truth Table:**

```
0 AND 0 = 0
0 AND 1 = 0
1 AND 0 = 0
1 AND 1 = 1
```

**Rule:** Result is `1` only if both inputs are `1`.

---

### Calculating Network Address: Step-by-Step

**Example: Network 1**

```
Router 0 LAN IP:    192.168.1.1
Subnet Mask:        255.255.255.0
```

**Step 1: Convert IP to binary**

```
192.168.1.1 in binary:

192 = 11000000
168 = 10101000
  1 = 00000001
  1 = 00000001

Full: 11000000.10101000.00000001.00000001
```

**Step 2: Convert subnet mask to binary**

```
255.255.255.0 in binary:

255 = 11111111
255 = 11111111
255 = 11111111
  0 = 00000000

Full: 11111111.11111111.11111111.00000000
```

**Step 3: Perform Binary AND**

```
IP:          11000000.10101000.00000001.00000001
Subnet Mask: 11111111.11111111.11111111.00000000
-----------------------------------------------  (AND)
Result:      11000000.10101000.00000001.00000000
```

**Step 4: Convert result to decimal**

```
11000000 = 192
10101000 = 168
00000001 = 1
00000000 = 0

Network Address: 192.168.1.0
```

**Conclusion:** The network address for Router 0's LAN (and all devices on it) is **192.168.1.0**.

---

### Calculating All Three Networks

**Network 1:**

```
Any IP in 192.168.1.0/24 (like 192.168.1.10, 192.168.1.12)
AND
Subnet Mask 255.255.255.0
=
Network Address: 192.168.1.0
```

**Network 2:**

```
Any IP in 192.168.2.0/24 (like 192.168.2.12, 192.168.2.10)
AND
Subnet Mask 255.255.255.0
=
Network Address: 192.168.2.0
```

**Network 3:**

```
Any IP in 192.168.3.0/24 (like 192.168.3.13, 192.168.3.10)
AND
Subnet Mask 255.255.255.0
=
Network Address: 192.168.3.0
```

**These network addresses populate the routing table's "Destination" column.**

---

### Understanding /24 Notation (CIDR)

**CIDR = Classless Inter-Domain Routing**

**Format:** `IP/prefix_length`

**Example:** `192.168.1.0/24`

**What /24 means:**

```
/24 = First 24 bits are the network portion
    = 255.255.255.0 subnet mask

Binary subnet mask:
11111111.11111111.11111111.00000000
↑ 24 bits of 1s ↑        ↑ 8 bits of 0s ↑
```

**Common CIDR notations:**

```
/8  = 255.0.0.0          = 16,777,216 hosts (Class A)
/16 = 255.255.0.0        =    65,536 hosts (Class B)
/24 = 255.255.255.0      =       256 hosts (Class C)
/30 = 255.255.255.252    =         4 hosts (Point-to-point links)
/32 = 255.255.255.255    =         1 host  (Single host route)
```

**In our example:**

```
192.168.1.0/24 means:
- Network: 192.168.1.0
- Netmask: 255.255.255.0
- Range: 192.168.1.0 to 192.168.1.255
- Usable hosts: 192.168.1.1 to 192.168.1.254 (254 hosts)
```

---

## The Routing Decision Algorithm

### How the OS Chooses the Correct Route

When your application (browser, game, app) sends data to a destination IP, the operating system follows this algorithm:

```
┌─────────────────────────────────────────────────────────┐
│          ROUTING DECISION ALGORITHM                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ Input: Destination IP address (e.g., 192.168.3.10)     │
│                                                         │
│ Step 1: Calculate network address for destination      │
│         (using each route's subnet mask)                │
│                                                         │
│ Step 2: Find matching routes in routing table          │
│         - Compare destination network with route entry  │
│         - Multiple routes may match                     │
│                                                         │
│ Step 3: Select most specific route                     │
│         - Longest prefix match (more 1s in netmask)     │
│         - /32 > /24 > /16 > /8 > /0                     │
│                                                         │
│ Step 4: Extract information from selected route        │
│         - Gateway IP (if any)                           │
│         - Network interface                             │
│                                                         │
│ Step 5: Send packet through chosen interface           │
│         - If Gateway = 0.0.0.0: Direct send (ARP for   │
│           destination MAC)                              │
│         - If Gateway = Router IP: Send to router (ARP  │
│           for gateway MAC)                              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

### Example Walkthrough 1: Sending to 192.168.3.10

**Scenario:** Your computer's browser sends HTTP request to `http://192.168.3.10:3000`

**Step 1: OS receives destination IP from application layer**

```
Application Layer: "I need to send HTTP GET to 192.168.3.10:3000"
Network Layer (OS): "Let me check my routing table to find the correct interface..."
```

**Step 2: OS checks each route**

```
Routing Table:
1. 192.168.1.0/24 → en0
2. 192.168.2.0/24 → eth-pci-bus1-slot0
3. 192.168.3.0/24 → wlp3s0
4. 0.0.0.0/0 → en0 (via 192.168.1.1)
```

**Step 3: Calculate network address for each route**

**Route 1: Does 192.168.3.10 belong to 192.168.1.0/24?**

```
Destination IP:       192.168.3.10      = 11000000.10101000.00000011.00001010
Route 1 Netmask:      255.255.255.0     = 11111111.11111111.11111111.00000000
                                          ---------------------------------------- AND
Result:                                   11000000.10101000.00000011.00000000
                                          = 192.168.3.0

Route 1 Network:      192.168.1.0

192.168.3.0 ≠ 192.168.1.0 → NO MATCH
```

**Route 2: Does 192.168.3.10 belong to 192.168.2.0/24?**

```
Destination IP:       192.168.3.10      = 11000000.10101000.00000011.00001010
Route 2 Netmask:      255.255.255.0     = 11111111.11111111.11111111.00000000
                                          ---------------------------------------- AND
Result:                                   11000000.10101000.00000011.00000000
                                          = 192.168.3.0

Route 2 Network:      192.168.2.0

192.168.3.0 ≠ 192.168.2.0 → NO MATCH
```

**Route 3: Does 192.168.3.10 belong to 192.168.3.0/24?**

```
Destination IP:       192.168.3.10      = 11000000.10101000.00000011.00001010
Route 3 Netmask:      255.255.255.0     = 11111111.11111111.11111111.00000000
                                          ---------------------------------------- AND
Result:                                   11000000.10101000.00000011.00000000
                                          = 192.168.3.0

Route 3 Network:      192.168.3.0

192.168.3.0 = 192.168.3.0 → MATCH! ✓
```

**Route 4: Does 192.168.3.10 belong to 0.0.0.0/0?**

```
Destination IP:       192.168.3.10      = 11000000.10101000.00000011.00001010
Route 4 Netmask:      0.0.0.0           = 00000000.00000000.00000000.00000000
                                          ---------------------------------------- AND
Result:                                   00000000.00000000.00000000.00000000
                                          = 0.0.0.0

Route 4 Network:      0.0.0.0

0.0.0.0 = 0.0.0.0 → MATCH! ✓
```

**Step 4: Multiple matches found! Which one to use?**

```
Matching routes:
- Route 3: 192.168.3.0/24 → wlp3s0
- Route 4: 0.0.0.0/0 → en0 (default)
```

**Step 5: Select most specific route (longest prefix match)**

```
Route 3: /24 = 24 bits of network portion
Route 4: /0  = 0 bits of network portion

24 > 0 → Route 3 is more specific

WINNER: Route 3
```

**Step 6: Extract information from Route 3**

```
Route 3:
- Destination: 192.168.3.0/24
- Gateway: 0.0.0.0 (direct connection)
- Interface: wlp3s0
```

**Step 7: Send packet**

```
OS Decision: "Use wlp3s0 interface. Destination is directly reachable."

Next steps:
1. OS constructs IP packet with source IP 192.168.3.13 (wlp3s0's IP)
2. OS needs destination MAC address
3. OS checks ARP table for 192.168.3.10's MAC
4. If not found, OS sends ARP request: "Who has 192.168.3.10? Tell 192.168.3.13"
5. Host Computer 5 responds: "I am 192.168.3.10, my MAC is XX:XX:XX:XX:XX:XX"
6. OS constructs Ethernet frame with destination MAC
7. OS sends frame through wlp3s0
8. NIC converts to electrical/electromagnetic signals
9. Packet travels through Router 2's switch to Host Computer 5
```

**Result: Successful routing! ✓**

---

### Example Walkthrough 2: Sending to 8.8.8.8 (Google DNS)

**Scenario:** Your computer runs `ping 8.8.8.8`

**Step 1-3: Check all routes**

**Route 1: Does 8.8.8.8 belong to 192.168.1.0/24?**

```
8.8.8.8 AND 255.255.255.0 = 8.8.8.0
8.8.8.0 ≠ 192.168.1.0 → NO MATCH
```

**Route 2: Does 8.8.8.8 belong to 192.168.2.0/24?**

```
8.8.8.8 AND 255.255.255.0 = 8.8.8.0
8.8.8.0 ≠ 192.168.2.0 → NO MATCH
```

**Route 3: Does 8.8.8.8 belong to 192.168.3.0/24?**

```
8.8.8.8 AND 255.255.255.0 = 8.8.8.0
8.8.8.0 ≠ 192.168.3.0 → NO MATCH
```

**Route 4: Does 8.8.8.8 belong to 0.0.0.0/0?**

```
8.8.8.8 AND 0.0.0.0 = 0.0.0.0
0.0.0.0 = 0.0.0.0 → MATCH! ✓
```

**Step 4: Only one match (default route)**

```
WINNER: Route 4 (default route)
```

**Step 5: Extract information**

```
Route 4:
- Destination: 0.0.0.0/0 (default)
- Gateway: 192.168.1.1
- Interface: en0
```

**Step 6: Send packet**

```
OS Decision: "Use en0 interface. Send to gateway 192.168.1.1."

Next steps:
1. OS constructs IP packet with:
   - Source IP: 192.168.1.10 (en0's IP)
   - Destination IP: 8.8.8.8
2. OS needs gateway's MAC address (NOT destination's MAC!)
3. OS checks ARP table for 192.168.1.1's MAC
4. If found, use it; if not, send ARP request
5. OS constructs Ethernet frame with:
   - Source MAC: en0's MAC
   - Destination MAC: Router 0's MAC (gateway)
6. OS sends frame through en0
7. Frame reaches Router 0
8. Router 0 examines IP packet, sees destination 8.8.8.8
9. Router 0 checks ITS routing table
10. Router 0 forwards to WAN interface (Internet)
11. Packet travels through ISP network to Google's servers
```

**Result: Successful routing through gateway! ✓**

---

### Example Walkthrough 3: Sending to 192.168.2.10

**Quick analysis:**

```
Destination: 192.168.2.10

Check Route 2: 192.168.2.0/24
192.168.2.10 AND 255.255.255.0 = 192.168.2.0
192.168.2.0 = 192.168.2.0 → MATCH!

Also matches Route 4 (default), but Route 2 is more specific (/24 > /0)

WINNER: Route 2
Interface: eth-pci-bus1-slot0
Gateway: 0.0.0.0 (direct)

Result: OS uses eth-pci-bus1-slot0, ARPs for 192.168.2.10's MAC, sends directly.
```

---

## How Routing Tables Are Populated

### Automatic Routes: DHCP's Role

When your computer's NIC receives an IP address via DHCP, the operating system **automatically** adds routes to the routing table.

**DHCP Discover/Offer/Request/Acknowledge Process (Review):**

```
Step 1: Computer → DHCP Server (Router)
"DHCP Discover: I need network configuration!"

Step 2: DHCP Server → Computer
"DHCP Offer: I offer you:
- IP Address: 192.168.1.10
- Subnet Mask: 255.255.255.0
- Gateway: 192.168.1.1
- DNS: 192.168.1.1"

Step 3: Computer → DHCP Server
"DHCP Request: I accept your offer!"

Step 4: DHCP Server → Computer
"DHCP Acknowledge: Configuration confirmed!"
```

**What happens after DHCP Acknowledge:**

```
1. OS assigns IP 192.168.1.10 to interface en0
2. OS calculates network address:
   192.168.1.10 AND 255.255.255.0 = 192.168.1.0
3. OS automatically adds route:
   Destination: 192.168.1.0/24
   Gateway: 0.0.0.0
   Interface: en0
4. OS automatically adds default route using gateway from DHCP:
   Destination: 0.0.0.0/0
   Gateway: 192.168.1.1
   Interface: en0
```

**This happens for every NIC:**

```
en0 receives DHCP:
  → Adds route for 192.168.1.0/24

eth-pci-bus1-slot0 receives DHCP:
  → Adds route for 192.168.2.0/24

wlp3s0 receives DHCP:
  → Adds route for 192.168.3.0/24
```

**You don't manually create these routes—DHCP does it automatically!**

---

### Manual Routes: Network Administrators

**Scenarios where routes are manually added:**

1. **Static IP configuration** (no DHCP)
2. **Custom routing** (force traffic through specific gateways)
3. **VPN connections** (add routes for VPN subnets)
4. **Complex network topologies** (multi-homed hosts)

**Commands to add routes manually:**

**Windows:**

```cmd
route add 192.168.4.0 mask 255.255.255.0 192.168.1.1
```

**macOS:**

```bash
sudo route add -net 192.168.4.0/24 192.168.1.1
```

**Linux:**

```bash
sudo ip route add 192.168.4.0/24 via 192.168.1.1 dev en0
```

---

### Loopback Routes

**Every system has a loopback route:**

```
Destination: 127.0.0.0/8
Interface: lo (loopback)
Gateway: 127.0.0.1
```

**Purpose:** Packets sent to any `127.x.x.x` address stay within the computer—they never leave the network interface.

**Use cases:**
- **Local servers:** `localhost` or `127.0.0.1` for local development
- **Testing:** Applications can test network code without external network
- **IPC (Inter-Process Communication):** Processes communicate via localhost

**Example:**

```bash
ping 127.0.0.1

# Packet never leaves the computer!
# OS routes through 'lo' interface
# Immediately loops back to the same computer
```

---

## Default Routes and Default Gateways

### What is a Default Route?

**Default Route:** A catch-all route that matches any destination not matched by more specific routes.

```
Destination: 0.0.0.0/0
```

**Why 0.0.0.0/0 matches everything:**

```
Any IP AND 0.0.0.0 = 0.0.0.0

Example:
8.8.8.8 AND 0.0.0.0 = 0.0.0.0 → Matches!
1.1.1.1 AND 0.0.0.0 = 0.0.0.0 → Matches!
93.184.216.34 AND 0.0.0.0 = 0.0.0.0 → Matches!
```

**Because /0 = 0 bits of network portion**, it matches every possible IP address.

---

### What is a Default Gateway?

**Default Gateway:** The router IP specified in the default route.

```
Default Route:
Destination: 0.0.0.0/0
Gateway: 192.168.1.1 ← This is the default gateway
Interface: en0
```

**Purpose:** When the OS doesn't know how to reach a destination (no specific route exists), it sends the packet to the default gateway, trusting the router to figure it out.

**Analogy:**

```
You're in a small town with these roads:
- Main Street → Your neighborhood
- Oak Avenue → Another neighborhood
- Pine Road → Third neighborhood

You need to go to New York City, but there's no direct road listed.

Default gateway = The highway entrance

You drive to the highway entrance (default gateway), and from there, the highway 
system (Internet routers) takes you to New York.
```

---

### Why Only One Default Gateway (Usually)

**Your computer could have multiple default gateways:**

```
Default route 1: 0.0.0.0/0 via 192.168.1.1 dev en0 metric 100
Default route 2: 0.0.0.0/0 via 192.168.2.1 dev eth-pci-bus1-slot0 metric 200
Default route 3: 0.0.0.0/0 via 192.168.3.1 dev wlp3s0 metric 300
```

**Metric = Priority (lower = higher priority)**

**But typically:**
- One primary internet connection exists
- Only one default route is configured
- Other interfaces are for local networks only

**In our example:**
- Router 0 (192.168.1.1) is the primary internet gateway
- Router 1 and Router 2 might be:
  - Internal networks (no internet access)
  - Backup connections (higher metric)
  - Isolated test networks

---

## Real-World Command Line Examples

### Checking Your Routing Table (macOS)

```bash
$ netstat -rn
Routing tables

Internet:
Destination        Gateway            Flags        Netif Expire
default            192.168.1.1        UGSc           en0
127.0.0.1          127.0.0.1          UH             lo0
192.168.1.0/24     link#4             UCS            en0      !
192.168.1.10       127.0.0.1          UHS            lo0
192.168.2.0/24     link#8             UCS     eth-pci-bus1  !
192.168.3.0/24     link#12            UCS         wlp3s0      !
```

**Analysis:**

```
Row 1: default → 192.168.1.1 → en0
"For unknown destinations, send to 192.168.1.1 through en0"

Row 2: 127.0.0.1 → lo0
"Loopback address stays in the computer"

Row 3: 192.168.1.0/24 → en0
"Network 1 is directly connected via en0"

Row 4: 192.168.1.10 → lo0
"My own IP on en0 (packets to myself loop back)"

Row 5: 192.168.2.0/24 → eth-pci-bus1
"Network 2 is directly connected via eth-pci-bus1-slot0"

Row 6: 192.168.3.0/24 → wlp3s0
"Network 3 is directly connected via wlp3s0"
```

---

### Checking IP Configuration (macOS)

```bash
$ ifconfig

lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> mtu 16384
	inet 127.0.0.1 netmask 0xff000000

en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	ether a4:83:e7:2f:5c:d1
	inet 192.168.1.10 netmask 0xffffff00 broadcast 192.168.1.255

eth-pci-bus1-slot0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	ether 00:0c:29:5a:b3:f2
	inet 192.168.2.12 netmask 0xffffff00 broadcast 192.168.2.255

wlp3s0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	ether ac:de:48:00:11:22
	inet 192.168.3.13 netmask 0xffffff00 broadcast 192.168.3.255
```

**What you see:**
- **lo0:** Loopback interface (`127.0.0.1`)
- **en0:** First Ethernet, IP `192.168.1.10`, subnet mask `255.255.255.0` (0xffffff00 in hex)
- **eth-pci-bus1-slot0:** PCI Ethernet card, IP `192.168.2.12`
- **wlp3s0:** Wireless interface, IP `192.168.3.13`

---

### Testing Routing (All Platforms)

**Ping a device in Network 3:**

```bash
$ ping 192.168.3.10
PING 192.168.3.10 (192.168.3.10): 56 data bytes
64 bytes from 192.168.3.10: icmp_seq=0 ttl=64 time=1.234 ms
64 bytes from 192.168.3.10: icmp_seq=1 ttl=64 time=0.987 ms

# OS used wlp3s0 interface (you can verify with tcpdump)
```

**Ping Google DNS (uses default route):**

```bash
$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=117 time=12.456 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=11.234 ms

# OS used en0 interface and gateway 192.168.1.1
```

**Trace route to see path:**

```bash
$ traceroute 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 64 hops max, 52 byte packets
 1  192.168.1.1 (192.168.1.1)  1.234 ms  0.987 ms  1.123 ms  ← Default gateway
 2  10.0.0.1 (10.0.0.1)  8.234 ms  7.654 ms  8.012 ms      ← ISP router
 3  172.16.5.1 (172.16.5.1)  10.123 ms  9.876 ms  10.234 ms ← ISP backbone
 ...
 12  8.8.8.8 (8.8.8.8)  12.456 ms  11.987 ms  12.123 ms     ← Google DNS
```

---

## Deep Dive: Routing Table Edge Cases

### Host-Specific Routes

**Sometimes you need a route for a single IP, not a network:**

```
Destination: 192.168.1.100/32
Gateway: 192.168.2.1
Interface: eth-pci-bus1-slot0
```

**What /32 means:**

```
/32 = All 32 bits are network portion
    = 255.255.255.255 subnet mask
    = Only ONE IP address

This route matches ONLY 192.168.1.100, nothing else.
```

**Use case:**

```
You want all traffic to 192.168.1.100 to go through a specific gateway,
even though 192.168.1.0/24 has a different route.

Longest prefix match: /32 > /24
OS will use the /32 route for 192.168.1.100
OS will use the /24 route for all other 192.168.1.x addresses
```

---

### Blackhole Routes

**Purpose:** Drop packets to specific destinations (security, blocking)

```
Destination: 192.168.99.0/24
Gateway: 127.0.0.1
Interface: lo
Flags: R (Reject)
```

**What happens:**

```
Any packet to 192.168.99.0/24 is sent to loopback (127.0.0.1), effectively 
dropping it. OS may return "Network unreachable" error to application.
```

**Use case:**
- Block access to malicious networks
- Prevent accidental routing to test networks
- Security policies

---

### Multiple Routes with Metrics

**When multiple routes match, metrics determine priority:**

```
Route 1: 0.0.0.0/0 via 192.168.1.1 metric 100  ← Primary (lower metric)
Route 2: 0.0.0.0/0 via 192.168.2.1 metric 200  ← Backup (higher metric)
```

**Normal operation:**
- OS uses Route 1 (lower metric = higher priority)

**If Route 1 fails:**
- OS automatically switches to Route 2
- This provides failover redundancy

---

## Routing in Routers vs. Computers

### Similarities

```
Both maintain routing tables
Both use longest prefix match
Both populate routes via DHCP (routers use DHCP on WAN interface)
Both calculate network addresses with binary AND
```

### Differences

**Computer routing table:**
- Simple (usually 5-20 entries)
- Mostly direct routes (directly connected networks)
- One default route (to home router)
- Static or DHCP-learned
- Purpose: "How do I send packets from this computer?"

**Router routing table:**
- Complex (thousands to millions of entries in backbone routers)
- Many indirect routes (remote networks)
- Learned via routing protocols (BGP, OSPF, RIP, EIGRP)
- Dynamic updates constantly
- Purpose: "How do I forward packets from one network to another?"

**Example router table (simplified):**

```
┌──────────────────┬────────────────┬────────┬──────────────┐
│ Destination      │ Gateway        │ Flags  │ Interface    │
├──────────────────┼────────────────┼────────┼──────────────┤
│ 192.168.1.0/24   │ 0.0.0.0        │ U      │ LAN          │
│ 0.0.0.0/0        │ ISP_Router_IP  │ UG     │ WAN          │
└──────────────────┴────────────────┴────────┴──────────────┘
```

**Enterprise router table (more complex):**

```
┌──────────────────┬────────────────┬────────┬──────────────┐
│ Destination      │ Gateway        │ Flags  │ Interface    │
├──────────────────┼────────────────┼────────┼──────────────┤
│ 10.0.0.0/8       │ 0.0.0.0        │ U      │ eth0         │
│ 172.16.0.0/12    │ 10.0.1.1       │ UG     │ eth0         │
│ 192.168.0.0/16   │ 10.0.2.1       │ UG     │ eth0         │
│ 8.8.8.0/24       │ ISP_Router     │ UG     │ WAN          │
│ 0.0.0.0/0        │ ISP_Router     │ UG     │ WAN          │
└──────────────────┴────────────────┴────────┴──────────────┘
```

---

## Common Confusions Clarified

### Confusion 1: "Why can't I just use the first NIC for everything?"

**Answer:**

If you only use one NIC, you can't reach devices on the other networks. Networks are isolated by routers.

```
If you send to 192.168.3.10 through en0 (Network 1):
- Packet enters Network 1 (192.168.1.0/24)
- Router 0 sees destination 192.168.3.10
- Router 0 doesn't know about Network 3 (they're separate routers!)
- Router 0 might use its default route (Internet), sending packet away
- Packet never reaches 192.168.3.10

Correct way:
- Use wlp3s0 (which is directly connected to Network 3)
- Packet enters Network 3 (192.168.3.0/24)
- 192.168.3.10 is directly reachable
- Success!
```

---

### Confusion 2: "How does the OS know each NIC's IP address?"

**Answer:**

Each NIC maintains its own IP configuration.

```
OS Data Structure (simplified):

Interface en0:
  - Hardware: Ethernet adapter
  - IP: 192.168.1.10
  - Netmask: 255.255.255.0
  - MAC: a4:83:e7:2f:5c:d1
  - Status: UP
  - Gateway: 192.168.1.1

Interface eth-pci-bus1-slot0:
  - Hardware: PCI Ethernet card in slot 0
  - IP: 192.168.2.12
  - Netmask: 255.255.255.0
  - MAC: 00:0c:29:5a:b3:f2
  - Status: UP
  - Gateway: 192.168.2.1

Interface wlp3s0:
  - Hardware: WiFi adapter
  - IP: 192.168.3.13
  - Netmask: 255.255.255.0
  - MAC: ac:de:48:00:11:22
  - Status: UP
  - Gateway: 192.168.3.1
```

When the OS decides to use `wlp3s0`, it automatically uses `192.168.3.13` as the source IP.

---

### Confusion 3: "Can I have the same IP on multiple NICs?"

**Technical answer: Yes, but it's almost always wrong.**

```
en0: 192.168.1.10
eth-pci-bus1-slot0: 192.168.1.10  ← Same IP!
```

**Problems:**
1. **Routing ambiguity:** Which interface should receive packets to 192.168.1.10?
2. **ARP conflicts:** Two NICs with the same IP would respond to ARP requests, causing chaos
3. **Network confusion:** Other devices don't know which MAC to use

**Correct approach:**
- Each NIC must have a unique IP (at least on different networks)
- Same IP is only acceptable for load balancing/bonding (advanced configurations where NICs work together as one logical interface)

---

### Confusion 4: "What if two routes have the same prefix length?"

**Example:**

```
Route 1: 192.168.1.0/24 via 192.168.2.1 metric 100
Route 2: 192.168.1.0/24 via 192.168.3.1 metric 200
```

**Answer: Use metric (lower = higher priority)**

```
OS will use Route 1 (metric 100) by default.
If Route 1 becomes unavailable, OS will fail over to Route 2 (metric 200).
```

---

## Practical Exercise: Analyzing Your Own Routing Table

### Step 1: View your routing table

**macOS/Linux:**

```bash
netstat -rn
```

or

```bash
ip route    # Linux only
```

**Windows:**

```cmd
route print
```

---

### Step 2: Identify key routes

Look for:

1. **Default route** (`0.0.0.0` or `default`)
   - What is the gateway IP?
   - Which interface is used?

2. **Direct routes** (gateway `0.0.0.0` or `link`)
   - What local networks are directly connected?

3. **Loopback route** (`127.0.0.0/8` or `127.0.1.1`)

---

### Step 3: Test routing decisions

**Pick a local IP in your network:**

```bash
ping 192.168.1.5
```

Expected: Uses direct route (no gateway), fast response.

**Pick an Internet IP:**

```bash
ping 8.8.8.8
```

Expected: Uses default route (through gateway), slightly slower.

**Trace the path:**

```bash
traceroute 8.8.8.8
```

First hop should be your default gateway.

---

### Step 4: Verify interface IPs

```bash
# macOS/Linux
ifconfig

# Linux (modern)
ip addr

# Windows
ipconfig
```

Match interface names in routing table with interface IPs.

---

## Summary and Key Takeaways

### What We Learned

1. **Routing tables are decision-making engines** that map destinations to interfaces and gateways.

2. **Every operating system maintains a routing table**, populated automatically by DHCP or manually by administrators.

3. **Routing table structure:**
   - Destination: Network address in CIDR notation
   - Gateway: Next-hop router (or 0.0.0.0 for direct)
   - Flags: Metadata (U, G, H, etc.)
   - Interface: Which NIC to use
   - Expire: Route lifetime

4. **Binary AND operation** determines network addresses by applying subnet mask to IP.

5. **Routing algorithm:**
   - Match destination IP against all routes
   - Select most specific (longest prefix match)
   - Extract gateway and interface
   - Send packet

6. **Default route (0.0.0.0/0)** is the catch-all for unknown destinations, pointing to the default gateway.

7. **Multiple NICs require multiple routes**, one for each directly connected network.

8. **Longest prefix match** ensures specific routes override general routes.

---

### Critical Insights

**Insight 1: The OS never guesses**

```
Every packet's path is determined by routing table lookup.
No randomness, no guessing—only deterministic algorithm.
```

**Insight 2: Direct vs. gateway routes**

```
Gateway 0.0.0.0: "I can reach this network directly. ARP for destination MAC."
Gateway X.X.X.X: "I cannot reach this network directly. ARP for gateway MAC,
                  let gateway forward it."
```

**Insight 3: Default route is your internet doorway**

```
Without a default route, your computer cannot access the Internet.
Default gateway = Your router = Your connection to the outside world.
```

**Insight 4: Routing tables are hierarchical**

```
More specific routes are preferred:
/32 > /24 > /16 > /8 > /0

This allows fine-grained control per host while having broader network routes.
```

---

### Visual Summary

```
┌────────────────────────────────────────────────────────────────┐
│                     ROUTING TABLE FLOW                         │
└────────────────────────────────────────────────────────────────┘

Application Layer
      │
      │ "Send to 192.168.3.10"
      ▼
Network Layer (OS)
      │
      ├─► Check Routing Table
      │   ├─ 192.168.1.0/24? No
      │   ├─ 192.168.2.0/24? No
      │   ├─ 192.168.3.0/24? YES! ✓
      │   └─ Use wlp3s0, Gateway 0.0.0.0 (direct)
      │
      ├─► Construct IP Packet
      │   ├─ Source IP: 192.168.3.13 (wlp3s0's IP)
      │   ├─ Destination IP: 192.168.3.10
      │   └─ Protocol: TCP/UDP/ICMP
      │
      ├─► Need Destination MAC
      │   └─► Check ARP Table
      │       ├─ Found? Use it
      │       └─ Not found? Send ARP request
      │
      ├─► Construct Ethernet Frame
      │   ├─ Source MAC: wlp3s0's MAC
      │   ├─ Destination MAC: 192.168.3.10's MAC
      │   └─ Payload: IP packet
      │
      ▼
Data Link Layer
      │
      ▼
Physical Layer (NIC wlp3s0)
      │
      └─► Convert to electromagnetic waves (WiFi)
          └─► Transmit through Router 2
              └─► Reach 192.168.3.10
```

---

## Connection to Previous and Next Chapters

### Previous Chapter (048: Visualizing Multiple NICs)

**What we learned:**
- How to physically connect multiple NICs
- How each NIC gets its own IP via DHCP
- The problem: "Which NIC to use?"

**This chapter answers:**
- The **routing table** is the solution
- OS uses network addresses, subnet masks, and binary AND to make decisions
- Every packet's path is deterministic, not random

---

### Next Chapter (050: [Next Topic])

**What we'll explore:**
- How routers maintain their own, more complex routing tables
- Routing protocols (RIP, OSPF, BGP) that routers use to learn routes dynamically
- How Internet routing works at scale
- Traceroute deep dive: Visualizing packet paths through multiple routers

---

## Final Thought

**The routing table is the invisible GPS of the Internet.**

Every packet, every millisecond, is routed through complex network topologies using the simple rules you've learned in this chapter. From your computer to Google's servers, from a smart bulb to the cloud, from a phone call to WhatsApp—every single packet follows its path according to routing tables.

**There is no magic. Only routers, routing tables, and binary AND operations.**

And now, you understand exactly how it works.

---

**End of Chapter 049**
