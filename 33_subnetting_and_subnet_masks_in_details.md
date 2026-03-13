# Chapter 040: Subnetting and Subnet Masks In Details

## Overview

In the previous chapter, we learned how a computer gets its first network connection: installing a NIC, getting an ARP address, connecting to a router, and receiving a proper IP address via DHCP. But we glossed over a critical question: **How does your computer know to send the DHCP request to the router when it doesn't even know the router's IP address or MAC address?**

This chapter answers that question and many more. The secret lies in understanding **subnets** and **subnet masks**—one of the most fundamental yet misunderstood concepts in networking. A subnet mask doesn't just determine how many devices can connect to a router; it fundamentally controls how devices understand network topology, make routing decisions, and communicate with each other.

When your computer wants to send data to IP address `192.168.1.50`, it must answer a critical question: "Is this device on my local network, or do I need to send this through my router?" The subnet mask provides the answer. When your computer broadcasts a DHCP DISCOVER message, the subnet mask determines who receives it. When you ping another computer, the subnet mask determines whether you use ARP (local) or routing (remote).

Without subnet masks, IP addressing would be chaos. Every device would need to know the structure of every network on the Internet. Routers couldn't aggregate routes. The Internet couldn't scale beyond a few thousand devices. Subnet masks enable the hierarchical addressing that makes the Internet possible.

This chapter systematically builds your understanding from first principles: what subnet masks are, how they work at the binary level, how they divide IP addresses into network and host portions, how they enable broadcast communication, how CIDR notation works, and how subnetting decisions affect network design. We'll use the geography analogy (country → division → district → sub-district) to make abstract concepts concrete, then dive deep into the binary mathematics that make it all work.

**By the end of this chapter, you'll understand why `255.255.255.0` isn't just a random number—it's the mathematical foundation of how local networks function.**

---

## Recap: DHCP (DORA Process)

Before diving into subnet masks, let's recap the DHCP process from the previous chapter.

### The DORA Process

**DORA: Discover, Offer, Request, Acknowledge**

```
Step 1: DISCOVER
Computer: "Is there a DHCP server on this network?"
┌─────────────────────────────────────────┐
│ DHCP DISCOVER (Broadcast)               │
│ Source IP: 0.0.0.0                      │
│ Dest IP: 255.255.255.255                │
│ Source MAC: AA:AA:AA:AA:AA:AA           │
│ Dest MAC: FF:FF:FF:FF:FF:FF (broadcast) │
│ Message: "I need an IP address!"        │
└─────────────────────────────────────────┘

Step 2: OFFER
Router: "I can give you 192.168.1.20"
┌─────────────────────────────────────────┐
│ DHCP OFFER (Unicast/Broadcast)          │
│ Source IP: 192.168.1.1 (router)         │
│ Dest IP: 192.168.1.20 (offered IP)      │
│ Message: "Here's an available IP"       │
│ - IP: 192.168.1.20                      │
│ - Subnet: 255.255.255.0                 │
│ - Gateway: 192.168.1.1                  │
│ - DNS: 8.8.8.8, 8.8.4.4                 │
│ - Lease: 86400 seconds (24 hours)      │
└─────────────────────────────────────────┘

Step 3: REQUEST
Computer: "I accept 192.168.1.20. Please assign it to me."
┌─────────────────────────────────────────┐
│ DHCP REQUEST (Broadcast)                │
│ Source IP: 0.0.0.0                      │
│ Dest IP: 255.255.255.255                │
│ Message: "I want 192.168.1.20"          │
│ Server Identifier: 192.168.1.1          │
└─────────────────────────────────────────┘

Step 4: ACKNOWLEDGE
Router: "Confirmed. 192.168.1.20 is now yours."
┌─────────────────────────────────────────┐
│ DHCP ACK (Unicast/Broadcast)            │
│ Source IP: 192.168.1.1                  │
│ Dest IP: 192.168.1.20                   │
│ Message: "Assignment confirmed"         │
│ - Your IP: 192.168.1.20                 │
│ - Subnet: 255.255.255.0                 │
│ - Gateway: 192.168.1.1                  │
│ - Lease expires: 24 hours               │
└─────────────────────────────────────────┘

Step 5: Computer configures network interface
Computer now has:
- IP: 192.168.1.20
- Subnet Mask: 255.255.255.0
- Default Gateway: 192.168.1.1
- DNS Servers: 8.8.8.8, 8.8.4.4
```

### Router's Internal State

**Router maintains DHCP lease table:**

```
┌──────────────────────────────────────────────────────┐
│ MAC Address         │ IP Address    │ Lease Expires  │
├──────────────────────────────────────────────────────┤
│ AA:AA:AA:AA:AA:AA  │ 192.168.1.20  │ 24h            │
│ BB:BB:BB:BB:BB:BB  │ 192.168.1.21  │ 24h            │
│ CC:CC:CC:CC:CC:CC  │ 192.168.1.22  │ 24h            │
└──────────────────────────────────────────────────────┘

Router tracks:
- Which MAC addresses have been assigned which IPs
- When each lease expires (so IPs can be recycled)
- Reserved IPs (static assignments)
```

---

## The Critical Question

### How Does the Computer Find the Router?

**Problem:**

```
Computer boots up with APIPA (169.254.52.143)
Computer detects router connection
Computer needs to send DHCP DISCOVER to router

But:
❌ Computer doesn't know router's IP address
❌ Computer doesn't know router's MAC address
❌ Computer hasn't received any configuration yet

How does it send DHCP DISCOVER to the router?
```

**Wrong Assumptions:**

```
❌ "Computer sends to router's default IP (192.168.1.1)"
   → Computer doesn't know this IP yet!

❌ "Computer has router's MAC from ARP"
   → ARP requires knowing IP first!

❌ "OS has router IP hardcoded"
   → Different routers use different IPs (192.168.1.1, 192.168.0.1, 10.0.0.1, etc.)
```

**The Answer:**

The computer **broadcasts** the DHCP DISCOVER message to **everyone** on the local network using:
- **IP Broadcast Address:** `255.255.255.255`
- **MAC Broadcast Address:** `FF:FF:FF:FF:FF:FF`

But how does the computer know who is "local"? How does it know which devices will receive the broadcast? The answer: **subnet masks**.

---

## IP Address Structure Fundamentals

Before understanding subnet masks, we must deeply understand IP address structure.

### The Four Octets

**IPv4 Address:**

```
192.168.1.10

Structure:
┌────────┬────────┬────────┬────────┐
│  192   │  168   │   1    │   10   │
│ Octet 1│ Octet 2│ Octet 3│ Octet 4│
│ 8 bits │ 8 bits │ 8 bits │ 8 bits │
└────────┴────────┴────────┴────────┘
         Total: 32 bits

Dot-decimal notation: Four decimal numbers separated by dots
Each number represents 8 bits (1 octet/byte)
```

---

### Binary Representation

**Each octet is 8 bits:**

```
Bit positions (left to right):
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 7 │ 6 │ 5 │ 4 │ 3 │ 2 │ 1 │ 0 │ ← Bit position
├───┼───┼───┼───┼───┼───┼───┼───┤
│128│ 64│ 32│ 16│ 8 │ 4 │ 2 │ 1 │ ← Decimal value if bit is 1
└───┴───┴───┴───┴───┴───┴───┴───┘

Each bit can be 0 or 1
2^8 = 256 possible values per octet (0-255)
```

**Example: Converting 192 to Binary**

```
192 in binary:

128 + 64 + 0 + 0 + 0 + 0 + 0 + 0 = 192

┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 1 │ 1 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │
└───┴───┴───┴───┴───┴───┴───┴───┘
 128+ 64 = 192

192 = 11000000 (binary)
```

**Example: Converting 168 to Binary**

```
168 = 128 + 32 + 8

┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 1 │ 0 │ 1 │ 0 │ 1 │ 0 │ 0 │ 0 │
└───┴───┴───┴───┴───┴───┴───┴───┘
 128   +32  +8 = 168

168 = 10101000 (binary)
```

**Complete IP Address in Binary:**

```
192.168.1.10

192     = 11000000
168     = 10101000
1       = 00000001
10      = 00001010

Full binary representation:
11000000.10101000.00000001.00001010

Or without dots:
11000000101010000000000100001010

Total: 32 bits
```

---

### Valid Range Per Octet

**Each octet: 8 bits**

```
Minimum value (all bits 0):
00000000 = 0

Maximum value (all bits 1):
11111111 = 128+64+32+16+8+4+2+1 = 255

Valid range: 0 to 255
```

**IP Address Constraints:**

```
✅ Valid: 192.168.1.10 (all octets 0-255)
✅ Valid: 10.0.0.1 (includes 0)
✅ Valid: 255.255.255.255 (all 255)

❌ Invalid: 192.168.1.256 (256 > 255)
❌ Invalid: 192.168.1.300 (300 > 255)
❌ Invalid: 192.168.-1.10 (negative not allowed)
❌ Invalid: 192.168.1 (missing fourth octet)
```

---

## What Is a Network?

### Network Definition

**Network:** A collection of devices that can communicate directly with each other without routing.

**Example Network:**

```
                ┌──────────────────┐
                │   Router         │
                │  192.168.1.1     │
                └────────┬─────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    ┌────▼────┐     ┌────▼────┐    ┌────▼────┐
    │Computer │     │ Mobile  │    │ Printer │
    │  .10    │     │   .20   │    │   .30   │
    └─────────┘     └─────────┘    └─────────┘
         │               │               │
         └───────────────┼───────────────┘
                         │
                    ┌────▼────┐
                    │Brother's│
                    │Computer │
                    │   .40   │
                    └─────────┘

All devices connected to same router
All devices can communicate directly (same network)
```

**IP Assignments:**

```
Router:     192.168.1.1
Computer:   192.168.1.10
Mobile:     192.168.1.20
Printer:    192.168.1.30
Brother's:  192.168.1.40
```

**Questions:**

1. Are all these devices on the same network? **Not sure yet!**
2. How many devices can this router support? **Depends on subnet mask!**
3. Can device `.10` communicate directly with device `.20`? **Subnet mask determines this!**

---

## Subnet Masks: The Foundation

### What Is a Subnet Mask?

**Subnet Mask:** A 32-bit number that divides an IP address into two parts:
1. **Network Portion:** Identifies which network the device is on
2. **Host Portion:** Identifies the specific device within that network

**Common Subnet Mask:**

```
255.255.255.0

In binary:
11111111.11111111.11111111.00000000

Structure:
┌──────────────────────────┬──────────┐
│      Network Portion     │   Host   │
│    (All 1s in mask)      │ (All 0s) │
│    24 bits               │  8 bits  │
└──────────────────────────┴──────────┘
```

---

### The Geography Analogy

**Understanding Network Hierarchy:**

Think of IP addressing like geographic addressing:

```
Country Level:      Bangladesh
   ↓
Division Level:     Dhaka Division
   ↓
District Level:     Dhaka District
   ↓
Sub-District Level: Dhanmondi
   ↓
Specific Address:   House 27, Road 2
```

**Mapping to IP Addressing:**

```
Network Class:      192.168.x.x (Private network range)
   ↓
Large Network:      192.168.1.x (Specific subnet)
   ↓
Subnet:             192.168.1.0/24 (256-address block)
   ↓
Host:               192.168.1.10 (Specific device)
```

**Why Hierarchy Matters:**

```
Without hierarchy (flat addressing):
- Routers need to know location of every device individually
- Impossible to scale
- Like knowing every house address in the world without cities/districts

With hierarchy (subnetting):
- Routers know "192.168.1.x goes to Router A"
- Scalable to billions of devices
- Like knowing "Dhaka addresses go to Dhaka post office" (aggregate routing)
```

---

### Binary Subnet Mask Operation

**How Subnet Masks Work:**

The subnet mask uses binary AND operation to separate network and host portions.

**Example:**

```
IP Address:    192.168.1.10
Subnet Mask:   255.255.255.0

Binary representation:
IP:    11000000.10101000.00000001.00001010
Mask:  11111111.11111111.11111111.00000000
       ───────────────────────────────────
AND:   11000000.10101000.00000001.00000000
       └─────── Network Portion ─────┘└Host┘

Result: 192.168.1.0 (Network address)
```

**Binary AND Operation:**

```
Rules:
1 AND 1 = 1
1 AND 0 = 0
0 AND 1 = 0
0 AND 0 = 0

Simple: Output is 1 only if BOTH inputs are 1
```

**Step-by-Step Example:**

```
IP Address: 192.168.1.10

Octet 1: 192
Binary: 11000000
Mask:   11111111 (255)
────────────────
AND:    11000000 = 192 ✓ (Network part)

Octet 2: 168
Binary: 10101000
Mask:   11111111 (255)
────────────────
AND:    10101000 = 168 ✓ (Network part)

Octet 3: 1
Binary: 00000001
Mask:   11111111 (255)
────────────────
AND:    00000001 = 1 ✓ (Network part)

Octet 4: 10
Binary: 00001010
Mask:   00000000 (0)
────────────────
AND:    00000000 = 0 ✓ (Host part zeroed out)

Network Address: 192.168.1.0
```

---

### Determining Network Membership

**Key Insight:** Two devices are on the same network if their network portions match.

**Example 1: Same Network**

```
Device A:
IP:    192.168.1.10
Mask:  255.255.255.0
AND:   192.168.1.0 (Network address)

Device B:
IP:    192.168.1.20
Mask:  255.255.255.0
AND:   192.168.1.0 (Network address)

Network addresses match: 192.168.1.0 = 192.168.1.0 ✓
Result: Same network, can communicate directly
```

**Example 2: Different Networks**

```
Device A:
IP:    192.168.1.10
Mask:  255.255.255.0
AND:   192.168.1.0 (Network address)

Device C:
IP:    192.168.2.10
Mask:  255.255.255.0
AND:   192.168.2.0 (Network address)

Network addresses differ: 192.168.1.0 ≠ 192.168.2.0 ✗
Result: Different networks, must route through gateway
```

---

## Common Subnet Masks

### Class-Based Subnet Masks

**Historical classification (obsolete but still referenced):**

```
Class A: 255.0.0.0
- First octet: Network
- Last 3 octets: Hosts
- Example: 10.5.10.50 (Network: 10.0.0.0)
- Hosts per network: 16,777,214

Class B: 255.255.0.0
- First 2 octets: Network
- Last 2 octets: Hosts
- Example: 172.16.10.50 (Network: 172.16.0.0)
- Hosts per network: 65,534

Class C: 255.255.255.0
- First 3 octets: Network
- Last octet: Hosts
- Example: 192.168.1.50 (Network: 192.168.1.0)
- Hosts per network: 254
```

---

### Subnet Mask: 255.255.255.0 (/24)

**Most common home network subnet mask**

```
Subnet Mask: 255.255.255.0
Binary: 11111111.11111111.11111111.00000000

Network portion: First 24 bits (3 octets)
Host portion: Last 8 bits (1 octet)

Example Network: 192.168.1.0/24

Valid IP Range:
- Network address: 192.168.1.0 (all host bits 0)
- First usable: 192.168.1.1
- Last usable: 192.168.1.254
- Broadcast: 192.168.1.255 (all host bits 1)

Total addresses: 2^8 = 256
Usable addresses: 256 - 2 = 254
(Subtract network address and broadcast address)
```

**Why 254 Usable Hosts?**

```
Host portion: 8 bits (last octet)
Possible values: 0-255 (256 total)

Reserved addresses:
- .0 (00000000): Network address
- .255 (11111111): Broadcast address

Usable range: .1 to .254
Count: 254 devices
```

**Typical Home Router Configuration:**

```
Network: 192.168.1.0/24
Router IP: 192.168.1.1
DHCP Pool: 192.168.1.10 to 192.168.1.254
Static IPs: 192.168.1.2 to 192.168.1.9 (reserved for servers)
```

---

### Subnet Mask: 255.255.0.0 (/16)

**Larger networks**

```
Subnet Mask: 255.255.0.0
Binary: 11111111.11111111.00000000.00000000

Network portion: First 16 bits (2 octets)
Host portion: Last 16 bits (2 octets)

Example Network: 192.168.0.0/16

Valid IP Range:
- Network address: 192.168.0.0
- First usable: 192.168.0.1
- Last usable: 192.168.255.254
- Broadcast: 192.168.255.255

Total addresses: 2^16 = 65,536
Usable addresses: 65,534
```

**When Used:**

```
- Large enterprise networks
- Campus networks (universities)
- Medium-sized organizations
- Need to support thousands of devices
```

---

### Subnet Mask: 255.0.0.0 (/8)

**Very large networks**

```
Subnet Mask: 255.0.0.0
Binary: 11111111.00000000.00000000.00000000

Network portion: First 8 bits (1 octet)
Host portion: Last 24 bits (3 octets)

Example Network: 10.0.0.0/8

Valid IP Range:
- Network address: 10.0.0.0
- First usable: 10.0.0.1
- Last usable: 10.255.255.254
- Broadcast: 10.255.255.255

Total addresses: 2^24 = 16,777,216
Usable addresses: 16,777,214
```

**When Used:**

```
- Massive enterprise networks
- Cloud providers (AWS VPC, etc.)
- ISPs internal networks
- Organizations with tens of thousands of devices
```

---

## CIDR Notation

### What Is CIDR?

**CIDR: Classless Inter-Domain Routing**

**Definition:** Notation that specifies how many bits are in the network portion.

**Format:** `IP_Address/Prefix_Length`

Examples:
```
192.168.1.0/24
10.0.0.0/8
172.16.0.0/16
192.168.1.128/25
```

---

### Understanding the Prefix Length

**The number after the slash (/) indicates how many bits are "1" in the subnet mask.**

```
/24 means first 24 bits are network portion

Binary:
11111111.11111111.11111111.00000000
├──────── 24 bits ─────────┤└─ 8 ─┘
        Network              Host

Decimal: 255.255.255.0
```

**Common CIDR Prefixes:**

```
/8  = 255.0.0.0         → Network: 1 octet,  Host: 3 octets
/16 = 255.255.0.0       → Network: 2 octets, Host: 2 octets
/24 = 255.255.255.0     → Network: 3 octets, Host: 1 octet
/32 = 255.255.255.255   → Network: 4 octets, Host: 0 octets (single IP)
```

---

### CIDR Calculation Examples

**Example 1: /24**

```
192.168.1.0/24

Subnet Mask: 255.255.255.0
Binary: 11111111.11111111.11111111.00000000
        └────────── 24 ones ──────────┘

Network: 192.168.1.0
Host bits: 32 - 24 = 8 bits
Total IPs: 2^8 = 256
Usable IPs: 254
Range: 192.168.1.1 to 192.168.1.254
```

**Example 2: /16**

```
172.16.0.0/16

Subnet Mask: 255.255.0.0
Binary: 11111111.11111111.00000000.00000000
        └───── 16 ones ────┘

Network: 172.16.0.0
Host bits: 32 - 16 = 16 bits
Total IPs: 2^16 = 65,536
Usable IPs: 65,534
Range: 172.16.0.1 to 172.16.255.254
```

**Example 3: /25 (Non-standard)**

```
192.168.1.0/25

Subnet Mask: 255.255.255.128
Binary: 11111111.11111111.11111111.10000000
        └────────── 25 ones ──────────┘└─7─┘

Network: 192.168.1.0
Host bits: 32 - 25 = 7 bits
Total IPs: 2^7 = 128
Usable IPs: 126
Range: 192.168.1.1 to 192.168.1.126
Broadcast: 192.168.1.127
```

---

### Converting CIDR to Subnet Mask

**Algorithm:**

1. Create 32-bit binary number
2. Set first N bits to 1 (where N is the prefix length)
3. Set remaining bits to 0
4. Convert to decimal

**Example: /20**

```
Step 1: Need 20 ones, then 12 zeros
11111111.11111111.11110000.00000000
├──────── 20 ones ─────────┤└─12 0s─┘

Step 2: Convert each octet to decimal
11111111 = 255
11111111 = 255
11110000 = 128+64+32+16 = 240
00000000 = 0

Result: 255.255.240.0
```

**Verification:**

```
192.168.16.0/20

Subnet Mask: 255.255.240.0
Host bits: 12
Total IPs: 2^12 = 4096
Usable: 4094
Range: 192.168.16.0 to 192.168.31.255
```

---

## Broadcast Addresses

### What Is a Broadcast Address?

**Broadcast Address:** A special IP address that sends packets to **all devices** on a local network.

**Characteristics:**
- All host bits set to 1
- Packets sent to broadcast address are delivered to every device on the network
- Does not cross routers (local network only)

---

### Types of Broadcast

#### 1. Limited Broadcast

```
Address: 255.255.255.255
Binary: 11111111.11111111.11111111.11111111

Purpose: Broadcast to all devices on the local network
Scope: Never forwarded by routers

Used by:
- DHCP DISCOVER (computer doesn't have IP yet)
- Initial network discovery

Example:
Computer (no IP) sends DHCP DISCOVER:
Source IP: 0.0.0.0
Dest IP: 255.255.255.255 (limited broadcast)

All devices on local network receive this packet
```

#### 2. Directed Broadcast

```
Network: 192.168.1.0/24
Broadcast Address: 192.168.1.255

Binary:
IP:   11000000.10101000.00000001.11111111
      └────── Network ─────┘└─ All 1s ─┘

Purpose: Broadcast to all devices on specific network
Scope: Can be routed (but often disabled for security)

Example:
Computer on 192.168.2.0/24 sends to 192.168.1.255
Router forwards to network 192.168.1.0/24
All devices on 192.168.1.0/24 receive packet
```

---

### Calculating Broadcast Address

**Algorithm:**

1. Take network address
2. Set all host bits to 1

**Example 1: 192.168.1.0/24**

```
Network: 192.168.1.0
Mask: 255.255.255.0 (24 bits network, 8 bits host)

Network binary: 11000000.10101000.00000001.00000000
Set host bits to 1: 11000000.10101000.00000001.11111111

Broadcast: 192.168.1.255
```

**Example 2: 10.0.0.0/8**

```
Network: 10.0.0.0
Mask: 255.0.0.0 (8 bits network, 24 bits host)

Network binary: 00001010.00000000.00000000.00000000
Set host bits to 1: 00001010.11111111.11111111.11111111

Broadcast: 10.255.255.255
```

**Example 3: 192.168.1.128/25**

```
Network: 192.168.1.128
Mask: 255.255.255.128 (25 bits network, 7 bits host)

Last octet:
Network: 10000000 (128)
Broadcast: 10111111 (127 more: 128+127=255... wait, 10111111 = 191)

Let me recalculate:
10000000 = 128
Set last 7 bits to 1: 1 0111111 = 128+63 = 191

Broadcast: 192.168.1.191 + wait...

Actually:
10000000 (network bit = 1)
Set remaining 7 bits to 1: 1 1111111 = 255

Broadcast: 192.168.1.255
```

Let me recalculate /25 correctly:

```
/25 = First 25 bits are network

Binary: 11111111.11111111.11111111.10000000
Decimal: 255.255.255.128

If network is 192.168.1.128:
Last octet binary: 10000000
                   └┬┘└──┬──┘
                    │   │
                 Net(1) Host(7 bits)

Network: 192.168.1.128 (10000000)
Broadcast: Set host bits to 1
           1 0000000 (net bit) + 1111111 (host bits all 1)
           = 11111111 = 255

Broadcast: 192.168.1.255
```

Actually, let me reconsider. With /25:

```
First 25 bits = network
Last 7 bits = host

Network: 192.168.1.0/25
Binary last octet: 0 0000000
                   └┬┘└──┬──┘
                    │   │
                 Net(1) Host(7)

Two /25 networks in 192.168.1.0/24:
1. 192.168.1.0/25: .0 to .127 (broadcast .127)
2. 192.168.1.128/25: .128 to .255 (broadcast .255)

For 192.168.1.0/25:
Network: 00000000
Broadcast: 01111111 = 127

For 192.168.1.128/25:
Network: 10000000 = 128
Broadcast: 11111111 = 255
```

**Correct Example 3: 192.168.1.0/25**

```
Network: 192.168.1.0/25
Mask: 255.255.255.128 (25 bits network, 7 bits host)

Last octet:
Network: 00000000 (0)
Set last 7 bits to 1: 0 1111111 = 127

Broadcast: 192.168.1.127
Range: 192.168.1.1 to 192.168.1.126
```

---

### Broadcast at Different Layers

**Layer 3 (IP Broadcast):**

```
IP: 255.255.255.255 (limited broadcast)
IP: 192.168.1.255 (directed broadcast for 192.168.1.0/24)

Layer 3 broadcast → Sent to all devices on IP network
```

**Layer 2 (MAC Broadcast):**

```
MAC: FF:FF:FF:FF:FF:FF

Layer 2 broadcast → Sent to all devices on Ethernet segment
```

**DHCP DISCOVER uses BOTH:**

```
DHCP DISCOVER Frame:
┌─────────────────────────────────────┐
│ Ethernet Header:                    │
│   Dest MAC: FF:FF:FF:FF:FF:FF       │ ← L2 broadcast
│   Source MAC: AA:AA:AA:AA:AA:AA     │
│   EtherType: 0x0800 (IPv4)          │
├─────────────────────────────────────┤
│ IP Header:                          │
│   Source IP: 0.0.0.0                │
│   Dest IP: 255.255.255.255          │ ← L3 broadcast
│   Protocol: UDP (17)                │
├─────────────────────────────────────┤
│ UDP Header:                         │
│   Source Port: 68 (DHCP client)     │
│   Dest Port: 67 (DHCP server)       │
├─────────────────────────────────────┤
│ DHCP Payload:                       │
│   Message Type: DISCOVER            │
│   Client MAC: AA:AA:AA:AA:AA:AA     │
└─────────────────────────────────────┘

Result: Every device on local network receives this frame
Only DHCP servers respond
```

---

## Network Address

### What Is a Network Address?

**Network Address:** The first address in a subnet where all host bits are 0.

**Purpose:**
- Identifies the network itself (not a specific device)
- Cannot be assigned to a device
- Used in routing tables

**Characteristics:**
- All host bits = 0
- Not usable for devices
- Represents the entire network

---

### Calculating Network Address

**Algorithm:**

1. Take any IP in the range
2. Apply subnet mask using AND operation
3. Result is network address

**Example 1:**

```
IP: 192.168.1.50
Mask: 255.255.255.0

Binary AND:
IP:   11000000.10101000.00000001.00110010
Mask: 11111111.11111111.11111111.00000000
────────────────────────────────────────────
Net:  11000000.10101000.00000001.00000000

Network Address: 192.168.1.0
```

**Example 2:**

```
IP: 10.50.100.200
Mask: 255.0.0.0

Binary AND:
IP:   00001010.00110010.01100100.11001000
Mask: 11111111.00000000.00000000.00000000
────────────────────────────────────────────
Net:  00001010.00000000.00000000.00000000

Network Address: 10.0.0.0
```

---

### Network, Broadcast, and Usable Range

**Complete Example: 192.168.1.0/24**

```
Network: 192.168.1.0/24
Subnet Mask: 255.255.255.0

Breakdown:
┌─────────────────────────────────────────┐
│ Network Address: 192.168.1.0            │ ← All host bits = 0
│                                         │    (Not usable)
├─────────────────────────────────────────┤
│ First Usable: 192.168.1.1               │ ← Typically router/gateway
├─────────────────────────────────────────┤
│ Usable Range: 192.168.1.2 to           │
│               192.168.1.254             │ ← Devices (computers, printers, etc.)
│ (253 addresses)                         │
├─────────────────────────────────────────┤
│ Broadcast Address: 192.168.1.255        │ ← All host bits = 1
│                                         │    (Not usable)
└─────────────────────────────────────────┘

Total: 256 addresses
Usable: 254 addresses
```

---

## How Computers Use Subnet Masks

### The Routing Decision

**Every time a computer sends a packet, it must answer:**

**"Is the destination on my local network, or should I send it to my gateway?"**

**Algorithm:**

```
1. Apply subnet mask to my IP → My network address
2. Apply subnet mask to destination IP → Destination network address
3. Compare:
   - If same: Destination is local → Use ARP, send directly
   - If different: Destination is remote → Send to default gateway
```

---

### Example 1: Local Destination

**Computer Configuration:**

```
My IP: 192.168.1.10
My Subnet: 255.255.255.0
My Gateway: 192.168.1.1
```

**Sending to 192.168.1.20:**

```
Step 1: Calculate my network
192.168.1.10 AND 255.255.255.0 = 192.168.1.0

Step 2: Calculate destination network
192.168.1.20 AND 255.255.255.0 = 192.168.1.0

Step 3: Compare
192.168.1.0 = 192.168.1.0 ✓ Same network!

Decision: Destination is local
Action:
1. Use ARP to find MAC address of 192.168.1.20
2. Send frame directly to that MAC address
3. No router involved
```

**ARP Process:**

```
Computer broadcasts ARP request:
"Who has 192.168.1.20? Tell 192.168.1.10"

Device at 192.168.1.20 replies:
"192.168.1.20 is at BB:BB:BB:BB:BB:BB"

Computer caches this, sends frame:
Dest MAC: BB:BB:BB:BB:BB:BB
Source MAC: AA:AA:AA:AA:AA:AA
Payload: IP packet to 192.168.1.20
```

---

### Example 2: Remote Destination

**Computer Configuration:**

```
My IP: 192.168.1.10
My Subnet: 255.255.255.0
My Gateway: 192.168.1.1
```

**Sending to 8.8.8.8 (Google DNS):**

```
Step 1: Calculate my network
192.168.1.10 AND 255.255.255.0 = 192.168.1.0

Step 2: Calculate destination network
8.8.8.8 AND 255.255.255.0 = 8.8.8.0

Step 3: Compare
192.168.1.0 ≠ 8.8.8.0 ✗ Different networks!

Decision: Destination is remote
Action:
1. IP packet dest still 8.8.8.8 (doesn't change)
2. But Ethernet frame dest MAC = gateway's MAC
3. Send frame to gateway (192.168.1.1)
4. Gateway routes packet toward destination
```

**Frame Details:**

```
Ethernet Frame:
┌─────────────────────────────────────┐
│ Dest MAC: RR:RR:RR:RR:RR:RR         │ ← Gateway's MAC
│ Source MAC: AA:AA:AA:AA:AA:AA       │ ← My MAC
├─────────────────────────────────────┤
│ IP Packet:                          │
│   Source IP: 192.168.1.10           │ ← My IP
│   Dest IP: 8.8.8.8                  │ ← Still Google's IP!
└─────────────────────────────────────┘

Key: MAC addresses change at each hop (L2)
     IP addresses stay the same end-to-end (L3)
```

---

### Example 3: Different Subnet Mask Changes Decision

**Configuration A: /24**

```
My IP: 192.168.1.10
Subnet: 255.255.255.0 (/24)
Gateway: 192.168.1.1

Destination: 192.168.2.10

My network: 192.168.1.10 AND 255.255.255.0 = 192.168.1.0
Dest network: 192.168.2.10 AND 255.255.255.0 = 192.168.2.0

192.168.1.0 ≠ 192.168.2.0

Decision: Remote (send to gateway)
```

**Configuration B: /16 (Different Subnet Mask)**

```
My IP: 192.168.1.10
Subnet: 255.255.0.0 (/16)  ← Different!
Gateway: 192.168.1.1

Destination: 192.168.2.10

My network: 192.168.1.10 AND 255.255.0.0 = 192.168.0.0
Dest network: 192.168.2.10 AND 255.255.0.0 = 192.168.0.0

192.168.0.0 = 192.168.0.0 ✓

Decision: Local (use ARP, send directly)
```

**Key Insight:**

The subnet mask fundamentally changes network topology! With /16, 192.168.1.10 and 192.168.2.10 are on the same network. With /24, they're on different networks. Same IPs, different behavior!

---

## Answering the Original Question

### How Does DHCP DISCOVER Work?

**The Question Revisited:**

```
Computer (169.254.52.143) needs IP from router
But:
- Doesn't know router's IP
- Doesn't know router's MAC
- Hasn't been configured yet

How does it send DHCP DISCOVER?
```

**The Answer:**

```
Step 1: Computer knows it needs network configuration
        OS has subnet mask awareness built-in

Step 2: Computer broadcasts DHCP DISCOVER
        IP: 0.0.0.0 → 255.255.255.255 (limited broadcast)
        MAC: My MAC → FF:FF:FF:FF:FF:FF (MAC broadcast)

Step 3: Frame sent to all devices on local network segment
        (All devices connected to same switch/router)

Step 4: All devices receive frame
        - Computers ignore (not DHCP servers)
        - Printers ignore
        - Router receives, sees DHCP DISCOVER
        - Router's DHCP server process responds

Step 5: Router sends DHCP OFFER back
        Includes subnet mask: 255.255.255.0

Step 6: Computer configures with received settings
        IP: 192.168.1.20
        Subnet: 255.255.255.0
        Gateway: 192.168.1.1

Step 7: Now computer knows network topology!
        Can determine local vs remote destinations
        Knows to send remote traffic to 192.168.1.1
```

---

### Why Broadcast Works

**Physical Topology:**

```
            ┌──────────────┐
            │   Router     │
            │  (DHCP)      │
            └───────┬──────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
    ┌───▼───┐   ┌───▼───┐   ┌───▼───┐
    │  PC   │   │Mobile │   │Printer│
    │(DHCP  │   │       │   │       │
    │Client)│   │       │   │       │
    └───────┘   └───────┘   └───────┘

All devices connected to same Ethernet segment
Broadcast frame reaches all devices physically
Only router's DHCP server responds
```

**Broadcast Scope:**

```
┌─────────────────────────────────────┐
│       Local Network Segment         │
│  (All devices receive broadcast)    │
│                                     │
│  ┌─────┐  ┌─────┐  ┌─────┐        │
│  │ PC  │  │Mobile│  │Router│       │
│  └─────┘  └─────┘  └─────┘        │
│                                     │
└─────────────────────────────────────┘
         ↕ Broadcast reaches all
         
         ❌ Broadcast does NOT cross router
         
┌─────────────────────────────────────┐
│      Different Network Segment      │
│ (Does NOT receive broadcast)        │
│                                     │
│  ┌─────┐  ┌─────┐                  │
│  │Server│  │ PC2 │                  │
│  └─────┘  └─────┘                  │
└─────────────────────────────────────┘
```

---

## Practical Subnet Calculations

### Determining Network Information

**Given: IP and Subnet Mask, find everything**

**Example: 192.168.1.75 / 255.255.255.0**

```
Step 1: Network Address
192.168.1.75 AND 255.255.255.0
= 192.168.1.0

Step 2: Count host bits
255.255.255.0 = /24
32 - 24 = 8 host bits

Step 3: Total addresses
2^8 = 256

Step 4: Usable addresses
256 - 2 = 254

Step 5: First usable
Network + 1 = 192.168.1.1

Step 6: Last usable
Broadcast - 1 = 192.168.1.254

Step 7: Broadcast
All host bits 1 = 192.168.1.255

Summary:
Network: 192.168.1.0/24
First: 192.168.1.1 (usually gateway)
Last: 192.168.1.254
Broadcast: 192.168.1.255
Usable: 254 addresses
CIDR: /24
```

---

### Complex Example: 172.16.50.75 / 255.255.240.0

```
Step 1: Network Address

172.16.50.75 in binary:
10101100.00010000.00110010.01001011

255.255.240.0 in binary:
11111111.11111111.11110000.00000000

AND operation:
10101100.00010000.00110000.00000000
= 172.16.48.0

Network: 172.16.48.0

Step 2: Count host bits
240 = 11110000 (4 network bits + 4 host bits in this octet)
Total network bits: 16 + 4 = 20
Host bits: 32 - 20 = 12

Or: Count zero bits in 255.255.240.0
Last octet: 0 = 8 zeros
Third octet: 240 = 11110000 = 4 zeros
Total host bits: 4 + 8 = 12

Step 3: Calculate range
2^12 = 4096 total addresses
Usable: 4094

Step 4: Broadcast address
Network: 172.16.48.0 = 10101100.00010000.00110000.00000000
Set 12 host bits to 1: 10101100.00010000.00111111.11111111
= 172.16.63.255

Step 5: First and last usable
First: 172.16.48.1
Last: 172.16.63.254

Summary:
Network: 172.16.48.0/20
Range: 172.16.48.0 to 172.16.63.255
First usable: 172.16.48.1
Last usable: 172.16.63.254
Broadcast: 172.16.63.255
Usable addresses: 4094
CIDR: /20
```

---

## Subnet Planning

### Choosing the Right Subnet Mask

**Factors to Consider:**

1. **Number of devices needed**
2. **Room for growth**
3. **Network segmentation requirements**
4. **IP address conservation**

---

### Small Network (Home/Small Office)

**Requirement:** 10-50 devices

```
Recommended: /24 (255.255.255.0)

Provides: 254 usable addresses
Example: 192.168.1.0/24

Pros:
✓ More than enough addresses
✓ Simple to understand (last octet is host)
✓ Standard for home networks
✓ Easy mental math

Cons:
- "Wastes" addresses if only need 10 devices
- But IP addresses are not scarce in private networks
```

---

### Medium Network (Department/Floor)

**Requirement:** 100-500 devices

```
Option 1: Multiple /24 networks
- 192.168.1.0/24 (254 hosts)
- 192.168.2.0/24 (254 hosts)
- 192.168.3.0/24 (254 hosts)
Total: 762 hosts across 3 subnets

Option 2: Single /23 network
- 192.168.1.0/23
- Provides: 510 usable addresses
- Range: 192.168.0.0 to 192.168.1.255

Option 3: Single /22 network
- 192.168.0.0/22
- Provides: 1022 usable addresses
- Range: 192.168.0.0 to 192.168.3.255
```

---

### Large Network (Campus/Enterprise)

**Requirement:** 1000-10000 devices

```
Recommended: /16 (255.255.0.0)

Example: 10.50.0.0/16
Provides: 65,534 usable addresses
Range: 10.50.0.1 to 10.50.255.254

Or subdivide with VLANs:
10.50.0.0/24 - VLAN 10 (Engineering)
10.50.1.0/24 - VLAN 20 (Sales)
10.50.2.0/24 - VLAN 30 (Marketing)
...
10.50.255.0/24 - VLAN 265

256 possible /24 subnets within /16
```

---

### Point-to-Point Links

**Requirement:** 2 devices (router-to-router link)

```
Recommended: /30 (255.255.255.252)

Provides: 2 usable addresses (perfect!)

Example: 10.0.0.0/30
Binary subnet: 11111111.11111111.11111111.11111100
Host bits: 2

Addresses:
- Network: 10.0.0.0
- Router A: 10.0.0.1
- Router B: 10.0.0.2
- Broadcast: 10.0.0.3

Efficient: No wasted IPs
```

---

## Subnetting a Network

### Dividing 192.168.1.0/24 into Smaller Subnets

**Scenario:** You have 192.168.1.0/24 and need 4 separate networks

**Solution: Use /26 (255.255.255.192)**

```
Original: 192.168.1.0/24 (254 hosts)

Divide into 4 subnets: /26 each (62 hosts each)

Subnet 1: 192.168.1.0/26
- Range: 192.168.1.0 to 192.168.1.63
- Usable: 192.168.1.1 to 192.168.1.62
- Broadcast: 192.168.1.63

Subnet 2: 192.168.1.64/26
- Range: 192.168.1.64 to 192.168.1.127
- Usable: 192.168.1.65 to 192.168.1.126
- Broadcast: 192.168.1.127

Subnet 3: 192.168.1.128/26
- Range: 192.168.1.128 to 192.168.1.191
- Usable: 192.168.1.129 to 192.168.1.190
- Broadcast: 192.168.1.191

Subnet 4: 192.168.1.192/26
- Range: 192.168.1.192 to 192.168.1.255
- Usable: 192.168.1.193 to 192.168.1.254
- Broadcast: 192.168.1.255
```

**Why This Works:**

```
/24: 24 network bits, 8 host bits
/26: 26 network bits, 6 host bits

Borrowed 2 bits from host portion for subnetting:
2^2 = 4 subnets
Each subnet has 2^6 = 64 addresses (62 usable)
```

---

### Variable Length Subnet Masking (VLSM)

**Scenario:** Need different-sized networks from 192.168.1.0/24

**Requirements:**
- Department A: 100 hosts
- Department B: 50 hosts
- Department C: 25 hosts
- Point-to-point link: 2 hosts

**Solution:**

```
Department A: 192.168.1.0/25 (126 hosts)
- Range: .0 to .127
- Usable: .1 to .126

Department B: 192.168.1.128/26 (62 hosts)
- Range: .128 to .191
- Usable: .129 to .190

Department C: 192.168.1.192/27 (30 hosts)
- Range: .192 to .223
- Usable: .193 to .222

Point-to-point: 192.168.1.224/30 (2 hosts)
- Range: .224 to .227
- Usable: .225 to .226

Remaining space: 192.168.1.228/28 to 192.168.1.255
(Available for future growth)
```

---

## Troubleshooting with Subnet Masks

### Common Subnet Mask Problems

#### Problem 1: Mismatched Subnet Masks

```
Computer A:
IP: 192.168.1.10
Subnet: 255.255.255.0 (/24)

Computer B:
IP: 192.168.1.20
Subnet: 255.255.0.0 (/16)  ← Misconfigured!

Computer A attempts to ping Computer B:

Computer A calculation:
My network: 192.168.1.10 AND 255.255.255.0 = 192.168.1.0
Dest network: 192.168.1.20 AND 255.255.255.0 = 192.168.1.0
Same network → Send directly via ARP

Computer B receives ping:
Reply source: 192.168.1.10
Computer B calculation:
My network: 192.168.1.20 AND 255.255.0.0 = 192.168.0.0
Dest network: 192.168.1.10 AND 255.255.0.0 = 192.168.0.0
Same network → Send directly

Result: Communication works BUT asymmetric routing issues possible
```

**Fix:** Ensure all devices on same network have same subnet mask.

---

#### Problem 2: IP Outside Subnet Range

```
Network: 192.168.1.0/24
Router: 192.168.1.1

Computer misconfigured:
IP: 192.168.2.50
Subnet: 255.255.255.0
Gateway: 192.168.1.1

Computer tries to communicate:

To local device (192.168.2.100):
My network: 192.168.2.50 AND 255.255.255.0 = 192.168.2.0
Dest network: 192.168.2.100 AND 255.255.255.0 = 192.168.2.0
Same network → ARP for 192.168.2.100
❌ No response (no device at 192.168.2.100)

To Internet:
Gateway: 192.168.1.1
My network: 192.168.2.0
Gateway network: 192.168.1.0
Different networks! → ARP for gateway
❌ No response (gateway on different subnet)

Result: Total communication failure
```

**Fix:** Ensure IP address is within the subnet range defined by network/mask.

---

#### Problem 3: Using Network or Broadcast Address

```
Network: 192.168.1.0/24

Computer misconfigured:
IP: 192.168.1.0  ← Network address!

Result:
- Some OSes reject this configuration
- Communication fails (packets to self)
- Network stack malfunction

Or:

IP: 192.168.1.255  ← Broadcast address!

Result:
- Computer receives all broadcast packets
- Routing failures
- Network stack confused
```

**Fix:** Only use addresses in usable range (.1 to .254 for /24).

---

### Diagnostic Commands

**Linux:**

```bash
# Show IP configuration
$ ip addr show eth0
inet 192.168.1.10/24 brd 192.168.1.255 scope global eth0

# Show routing table
$ ip route show
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.10

# Calculate network from IP/mask
$ ipcalc 192.168.1.75/24
Network:   192.168.1.0/24
Broadcast: 192.168.1.255
HostMin:   192.168.1.1
HostMax:   192.168.1.254
Hosts/Net: 254
```

**Windows:**

```cmd
REM Show IP configuration
C:\> ipconfig /all
IPv4 Address: 192.168.1.10
Subnet Mask: 255.255.255.0
Default Gateway: 192.168.1.1

REM Show routing table
C:\> route print
Network Destination    Netmask          Gateway       Interface
0.0.0.0                0.0.0.0          192.168.1.1   192.168.1.10
192.168.1.0            255.255.255.0    On-link       192.168.1.10
```

---

## Summary and Key Takeaways

### Essential Concepts

**1. Subnet Mask Purpose:**
- Divides IP address into network and host portions
- Determines which devices are "local" (same network)
- Controls routing decisions (local vs remote)
- Enables hierarchical IP addressing

**2. Binary AND Operation:**
- IP AND Mask = Network Address
- Devices with same network address are on same network
- Local destinations: Use ARP, send directly
- Remote destinations: Send to default gateway

**3. CIDR Notation:**
- `/24` = 24 network bits, 8 host bits = 255.255.255.0
- `/16` = 16 network bits, 16 host bits = 255.255.0.0
- `/8` = 8 network bits, 24 host bits = 255.0.0.0
- Shorter prefix = larger network

**4. Special Addresses:**
- **Network Address:** All host bits = 0 (identifies network)
- **Broadcast Address:** All host bits = 1 (sends to all devices)
- **Usable Range:** Network + 1 to Broadcast - 1

**5. DHCP Broadcast Mystery Solved:**
- Computer sends DHCP DISCOVER to 255.255.255.255
- Uses MAC broadcast FF:FF:FF:FF:FF:FF
- All devices on local physical network receive it
- Only DHCP server (router) responds
- No need to know router IP/MAC beforehand

**6. Subnet Mask Examples:**
```
255.255.255.0 (/24): 254 hosts - Home networks
255.255.0.0 (/16): 65,534 hosts - Enterprise networks
255.255.255.252 (/30): 2 hosts - Point-to-point links
255.255.255.128 (/25): 126 hosts - Small subnets
```

---

### Practical Applications

**Home Network:**
```
Network: 192.168.1.0/24
Router: 192.168.1.1
Devices: .10, .20, .30, .40
All can communicate directly
Subnet mask: 255.255.255.0
```

**Office with VLANs:**
```
Engineering: 10.1.10.0/24
Sales: 10.1.20.0/24
Guest WiFi: 10.1.100.0/24
Each department isolated by subnet
Router connects subnets
```

**Data Center:**
```
Servers: 172.16.0.0/16 (65K addresses)
Management: 10.0.0.0/24 (254 addresses)
Storage: 192.168.10.0/24
Different functions, different subnets
```

---

## Conclusion

Subnet masks are the invisible foundation of IP networking. Every routing decision, every ARP request, every broadcast, every DHCP transaction depends on subnet masks working correctly. When your computer determines whether to send a packet to the local network or to the gateway, it's using the subnet mask. When a router decides whether to forward a broadcast or block it, it's using subnet boundaries defined by subnet masks.

Understanding subnet masks at the binary level transforms networking from memorized rules to logical conclusions. You now know why `255.255.255.0` produces 254 usable hosts (8 host bits = 2^8 - 2). You know why `192.168.1.10` and `192.168.1.20` can communicate directly with `/24` masks. You know exactly how DHCP DISCOVER works—broadcasting to `255.255.255.255` and `FF:FF:FF:FF:FF:FF` ensures every device on the local network segment receives it, including the router's DHCP server.

The geography analogy makes subnetting intuitive: just as countries divide into divisions, divisions into districts, and districts into sub-districts, IP networks divide into subnets, subnets into smaller subnets, enabling hierarchical addressing that scales from home networks to the global Internet.

From configuring your home router to designing enterprise networks to troubleshooting connectivity issues to understanding cloud networking—subnet masks are fundamental. Master them, and you've mastered one of the core pillars of internetworking.

**Subnet masks: the mathematical foundation that makes local networks possible and the Internet scalable.**

---

## Further Reading

- **RFC 950:** Internet Standard Subnetting Procedure
- **RFC 1918:** Address Allocation for Private Internets
- **RFC 3021:** Using 31-Bit Prefixes on IPv4 Point-to-Point Links
- **RFC 4632:** Classless Inter-domain Routing (CIDR)
- **"TCP/IP Illustrated, Volume 1" by W. Richard Stevens:** Chapters on IP addressing and subnetting
- **Cisco CCNA Study Guides:** Comprehensive subnetting practice
- **"Computer Networks" by Andrew S. Tanenbaum:** Network layer addressing
- **Online subnet calculators:** Practice converting between CIDR, decimal, and binary
- **Variable Length Subnet Masking (VLSM) tutorials**
- **Subnetting practice problems and exercises**
