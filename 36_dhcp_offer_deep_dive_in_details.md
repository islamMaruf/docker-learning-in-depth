# Chapter 043: DHCP OFFER - Breaking Into Pieces In Details

## Overview

In Chapter 042, we dissected the DHCP DISCOVER packet layer by layer—a computer with no IP address desperately broadcasting a request for network configuration. The packet traveled from Application Layer (L7) down to Physical Layer (L1), crossing the wire as electrical signals, reaching every device on the network segment.

Now comes the response: **DHCP OFFER**.

The router's DHCP server received the DISCOVER message. It parsed the client's MAC address, generated a Transaction ID match, selected an available IP address from its pool, and now prepares to send an offer back to the client. But here's the challenge: **the client still has no IP address**. How does the router send a unicast response to a device that has no network identity?

This chapter explores DHCP OFFER with the same layer-by-layer precision we applied to DISCOVER. But this time, we're viewing the network from the router's perspective—constructing a response packet, making critical decisions about addressing (unicast vs. broadcast), and ensuring the offer reaches the client despite the client's lack of IP configuration.

**Key questions we'll answer:**

1. How does the router construct the OFFER message?
2. What information does the OFFER contain beyond just an IP address?
3. Does the router send the OFFER as broadcast or unicast?
4. Why might broadcast be used even though the router knows the client's MAC address?
5. How do source/destination ports, IPs, and MACs change compared to DISCOVER?
6. What happens at each layer as the OFFER packet is constructed?

By the end of this chapter, you'll understand not just DHCP OFFER conceptually, but the precise byte-by-byte structure of the response packet flowing from router to client.

---

## Recap: Where We Left Off

### The DISCOVER Packet (Client → Router)

**From Chapter 042:**

```
Computer sends DHCP DISCOVER:

Layer 7 (Application):
  DHCP Message Type: DISCOVER
  Transaction ID: 0x3903F326
  Client MAC: AA:BB:CC:DD:EE:FF
  Client IP: 0.0.0.0 (none yet)
  Message: "I want an IP address"

Layer 4 (Transport):
  UDP Source Port: 68 (DHCP Client)
  UDP Dest Port: 67 (DHCP Server)

Layer 3 (Network):
  Source IP: 0.0.0.0 (no address)
  Dest IP: 255.255.255.255 (broadcast)

Layer 2 (Data Link):
  Source MAC: AA:BB:CC:DD:EE:FF (client)
  Dest MAC: FF:FF:FF:FF:FF:FF (broadcast)

Result: Broadcast received by all devices
Router's DHCP server processes the DISCOVER
```

---

### Router's DHCP Server Processing

**When router receives DISCOVER:**

```
1. Physical Layer: Electrical signals decoded
2. Data Link Layer: Ethernet frame received
   - Dest MAC: FF:FF:FF:FF:FF:FF (broadcast - accept)
   - Extract source MAC: AA:BB:CC:DD:EE:FF
3. Network Layer: IP packet processed
   - Dest IP: 255.255.255.255 (broadcast - accept)
   - Source IP: 0.0.0.0 (client has no IP)
4. Transport Layer: UDP datagram processed
   - Dest Port: 67 (DHCP server - accept)
   - Source Port: 68 (client port)
5. Application Layer: DHCP message parsed
   - Message Type: DISCOVER
   - Transaction ID: 0x3903F326
   - Client MAC: AA:BB:CC:DD:EE:FF
   - Requested parameters: IP, subnet, gateway, DNS

DHCP server decision: Send OFFER
```

---

### Router's DHCP Server State

**DHCP server configuration:**

```
Router Configuration:
├─ LAN Interface IP: 192.168.1.1
├─ Subnet Mask: 255.255.255.0
├─ DHCP Pool: 192.168.1.10 - 192.168.1.254
├─ Lease Time: 86400 seconds (24 hours)
├─ DNS Servers: 8.8.8.8, 8.8.4.4
└─ Domain: home.local

Current Lease Table:
├─ 192.168.1.10 → 11:22:33:44:55:66 (PC1) - Active
├─ 192.168.1.15 → AA:AA:BB:BB:CC:CC (Phone) - Active
└─ 192.168.1.20 → DD:EE:FF:00:11:22 (Laptop) - Active

Next available IP: 192.168.1.21
```

**DHCP server logic:**

```
Received DISCOVER from MAC: AA:BB:CC:DD:EE:FF
Transaction ID: 0x3903F326

Step 1: Check for existing lease
  → No existing lease found for this MAC

Step 2: Check for IP reservation
  → No static reservation for this MAC

Step 3: Allocate from pool
  → Pool range: 192.168.1.10 - 192.168.1.254
  → Already allocated: .10, .15, .20
  → Next available: 192.168.1.21
  → Allocate 192.168.1.21 to AA:BB:CC:DD:EE:FF

Step 4: Create lease entry (OFFERED state)
  MAC: AA:BB:CC:DD:EE:FF
  IP: 192.168.1.21
  State: OFFERED (not yet bound)
  Offered at: 10:15:23
  Transaction ID: 0x3903F326

Step 5: Prepare OFFER message
  → Include: IP, subnet mask, gateway, DNS, lease time
  → Send OFFER packet
```

---

## The Scenario: Router Sends OFFER

### Network Topology

```
┌─────────────────────────────────┐
│   Computer (Client)             │
│   ┌─────────────────────────┐   │
│   │ MAC: AA:BB:CC:DD:EE:FF  │   │
│   │ IP: None (waiting...)   │   │
│   │ State: DISCOVER sent    │   │
│   │ Waiting for OFFER       │   │
│   └─────────────────────────┘   │
└──────────────┬──────────────────┘
               │
               │ Ethernet Cable
               │
┌──────────────▼──────────────────┐
│   Router                        │
│   ┌─────────────────────────┐   │
│   │ LAN MAC: A1:B1:C1:D1:E1:F1│
│   │ LAN IP: 192.168.1.1     │   │
│   │ DHCP Server: Port 67    │   │
│   │ Allocated IP: .21       │   │
│   │ Preparing OFFER...      │   │
│   └─────────────────────────┘   │
└─────────────────────────────────┘

Router initiates OFFER response
Packet construction starts at Application Layer
```

---

## Layer 7: Application Layer - DHCP OFFER Message

### Who Initiates?

**Critical understanding:**

```
DISCOVER: Client initiates request
OFFER: Router initiates response

Direction:
DISCOVER: Computer → Router
OFFER: Router → Computer

Therefore, OFFER construction starts in router's Application Layer
```

---

### DHCP OFFER Message Structure

**DHCP OFFER message fields:**

```
DHCP OFFER Message:
┌──────────────────────────────────────────┐
│ Op: 2 (BOOTREPLY)                        │  1 byte
├──────────────────────────────────────────┤
│ Htype: 1 (Ethernet)                      │  1 byte
├──────────────────────────────────────────┤
│ Hlen: 6 (MAC address length)             │  1 byte
├──────────────────────────────────────────┤
│ Hops: 0 (no relays)                      │  1 byte
├──────────────────────────────────────────┤
│ Transaction ID: 0x3903F326               │  4 bytes
│ (SAME as DISCOVER - critical!)           │
├──────────────────────────────────────────┤
│ Seconds: 0                               │  2 bytes
├──────────────────────────────────────────┤
│ Flags: 0x8000 (broadcast)                │  2 bytes
│ (Copied from DISCOVER)                   │
├──────────────────────────────────────────┤
│ Client IP: 0.0.0.0                       │  4 bytes
│ (Client still has no IP)                 │
├──────────────────────────────────────────┤
│ Your IP: 192.168.1.21                    │  4 bytes
│ (THE OFFERED IP - KEY FIELD!)            │
├──────────────────────────────────────────┤
│ Server IP: 192.168.1.1                   │  4 bytes
│ (DHCP server's IP)                       │
├──────────────────────────────────────────┤
│ Gateway IP: 0.0.0.0                      │  4 bytes
│ (No relay agent)                         │
├──────────────────────────────────────────┤
│ Client MAC: AA:BB:CC:DD:EE:FF            │  16 bytes (6 used)
│ (CHADDR - copied from DISCOVER)          │
├──────────────────────────────────────────┤
│ Server Name: (empty)                     │  64 bytes
├──────────────────────────────────────────┤
│ Boot Filename: (empty)                   │  128 bytes
├──────────────────────────────────────────┤
│ Magic Cookie: 0x63825363                 │  4 bytes
├──────────────────────────────────────────┤
│ DHCP Options:                            │  Variable
│   Option 53: DHCP Message Type = 2       │  (2 = OFFER)
│   Option 54: DHCP Server Identifier      │  (192.168.1.1)
│   Option 51: IP Address Lease Time       │  (86400 sec)
│   Option 1: Subnet Mask                  │  (255.255.255.0)
│   Option 3: Router (Gateway)             │  (192.168.1.1)
│   Option 6: DNS Servers                  │  (8.8.8.8, 8.8.4.4)
│   Option 15: Domain Name                 │  (home.local)
│   Option 255: End                        │
└──────────────────────────────────────────┘

Total: ~300 bytes
```

---

### Key Field Changes from DISCOVER

**Comparison:**

```
Field                 DISCOVER              OFFER
----------------------------------------------------------------
Op                    1 (REQUEST)           2 (REPLY)
Transaction ID        0x3903F326            0x3903F326 (SAME!)
Client IP             0.0.0.0               0.0.0.0 (still none)
Your IP               0.0.0.0               192.168.1.21 (OFFERED)
Server IP             0.0.0.0               192.168.1.1
Client MAC            AA:BB:CC:DD:EE:FF     AA:BB:CC:DD:EE:FF
Option 53             1 (DISCOVER)          2 (OFFER)
```

---

### Critical Fields Explained

#### Op: 2 (BOOTREPLY)

```
Op: 2

Values:
1 = BOOTREQUEST (client to server)
2 = BOOTREPLY (server to client)

DISCOVER uses: 1
OFFER uses: 2

This tells receiver: "This is a server response, not a client request"
```

#### Transaction ID: MUST Match

```
Transaction ID: 0x3903F326

CRITICAL REQUIREMENT:
- Client sent DISCOVER with Transaction ID 0x3903F326
- Server MUST use SAME Transaction ID in OFFER
- Client matches OFFER to its DISCOVER by Transaction ID

Why important:
- Multiple clients might send DISCOVER simultaneously
- Each has unique Transaction ID
- Client only accepts OFFER matching its Transaction ID

Example scenario:
Computer A: DISCOVER with ID 0x3903F326
Computer B: DISCOVER with ID 0xABCD1234

Router sends:
OFFER to A with ID 0x3903F326
OFFER to B with ID 0xABCD1234

Computer A receives both OFFERs:
- Checks ID 0x3903F326 → Match! Process this OFFER
- Checks ID 0xABCD1234 → No match, ignore

Without Transaction ID matching:
- Computer A might accept OFFER meant for Computer B
- IP conflicts would occur
- Network chaos!
```

#### Your IP: 192.168.1.21 (The Offered IP)

**This is the star of the OFFER message:**

```
Your IP: 192.168.1.21

Meaning: "I am offering YOU this IP address"

Client IP vs Your IP:
- Client IP: The IP client currently has (always 0.0.0.0 in OFFER)
- Your IP: The IP server is offering to client

In OFFER: Your IP = 192.168.1.21
In REQUEST: Client asks for this same IP
In ACK: Server confirms this IP

Flow:
1. DISCOVER: Your IP = 0.0.0.0 (nothing offered yet)
2. OFFER: Your IP = 192.168.1.21 ("I offer this")
3. REQUEST: Requested IP = 192.168.1.21 ("I accept")
4. ACK: Your IP = 192.168.1.21 ("Confirmed, it's yours")
```

#### Server IP: 192.168.1.1

```
Server IP: 192.168.1.1

Identifies the DHCP server making the offer

Why important:
- Multiple DHCP servers might exist on network
- Client needs to know which server offered this IP
- Used in REQUEST message to identify chosen server

Also provided in Option 54 (DHCP Server Identifier)
for redundancy
```

#### Client MAC (CHADDR): Copied from DISCOVER

```
Client MAC: AA:BB:CC:DD:EE:FF

Copied from DISCOVER message

Purpose:
- Identifies which client this OFFER is for
- Allows client to recognize "this OFFER is for me"
- Creates binding: MAC → IP

Even though all devices might receive broadcast OFFER,
only device with MAC AA:BB:CC:DD:EE:FF will process it
```

---

### DHCP Options in OFFER

**Option 53: Message Type = OFFER**

```
Option 53: DHCP Message Type
Length: 1 byte
Value: 2 (DHCPOFFER)

Distinguishes OFFER from other DHCP messages
```

**Option 54: DHCP Server Identifier**

```
Option 54: Server Identifier
Length: 4 bytes
Value: 192.168.1.1

Identifies which DHCP server sent this OFFER

Why separate from "Server IP" field:
- Option 54 was added in DHCP (RFC 2131)
- "Server IP" field inherited from BOOTP
- Option 54 is more explicit and required

Client uses this in REQUEST to indicate chosen server
```

**Option 51: IP Address Lease Time**

```
Option 51: Lease Time
Length: 4 bytes
Value: 86400 seconds (24 hours)

How long client can use this IP before renewal

Binary: 0x00015180
Seconds: 86400
Minutes: 1440
Hours: 24
Days: 1

Common lease times:
- Home routers: 24 hours (86400 sec)
- Enterprise: 8 hours (28800 sec)
- Guest networks: 1 hour (3600 sec)
- Temporary: 10 minutes (600 sec)
```

**Option 1: Subnet Mask**

```
Option 1: Subnet Mask
Length: 4 bytes
Value: 255.255.255.0

Defines network vs host portion of IP

Binary: 11111111.11111111.11111111.00000000
CIDR: /24
Network: 192.168.1.0
Usable: 192.168.1.1 - 192.168.1.254
Broadcast: 192.168.1.255

Why provided:
- Client needs to know its subnet
- Determines local vs remote routing
- Essential for network communication
```

**Option 3: Router (Default Gateway)**

```
Option 3: Router (Gateway)
Length: 4 bytes
Value: 192.168.1.1

Default gateway for client

This is the router's LAN interface IP
Client sends all non-local traffic here

Why same as server IP?
- Router often runs DHCP server
- Router's LAN IP serves dual purpose:
  1. DHCP server address
  2. Default gateway for routing

Client configuration after OFFER accepted:
IP: 192.168.1.21
Mask: 255.255.255.0
Gateway: 192.168.1.1 (router)
DNS: 8.8.8.8
```

**Option 6: DNS Servers**

```
Option 6: Domain Name Server
Length: 8 bytes (2 servers × 4 bytes each)
Value: 8.8.8.8, 8.8.4.4

DNS servers client should use

Common configurations:
- Router forwards: 192.168.1.1 (router itself)
- ISP DNS: ISP-provided addresses
- Public DNS: 8.8.8.8 (Google), 1.1.1.1 (Cloudflare)

Multiple DNS servers:
Primary: 8.8.8.8
Secondary: 8.8.4.4 (fallback if primary fails)

Client uses these to resolve domain names:
google.com → 142.250.185.46
```

**Option 15: Domain Name**

```
Option 15: Domain Name
Length: Variable
Value: "home.local"

Local domain suffix

Client uses for local name resolution
Computer named "laptop" becomes "laptop.home.local"

Useful for:
- Local network DNS
- NetBIOS name resolution
- Corporate environments
```

---

### Application Layer Summary

```
Router's Application Layer creates DHCP OFFER:

┌────────────────────────────────────────┐
│ "HERE IS YOUR IP ADDRESS"              │
│                                        │
│ Message Type: OFFER                    │
│ Transaction ID: 0x3903F326 (matched!)  │
│ Your IP: 192.168.1.21                  │
│ Subnet Mask: 255.255.255.0             │
│ Gateway: 192.168.1.1                   │
│ DNS: 8.8.8.8, 8.8.4.4                  │
│ Lease: 24 hours                        │
│ For MAC: AA:BB:CC:DD:EE:FF             │
│                                        │
│ Size: ~300 bytes                       │
└────────────────────────────────────────┘

This OFFER data now passes to Transport Layer (L4)
```

---

## Layer 4: Transport Layer - UDP Datagram

### Port Direction Reversal

**DISCOVER vs OFFER:**

```
DISCOVER (Client → Server):
Source Port: 68 (client)
Dest Port: 67 (server)

OFFER (Server → Client):
Source Port: 67 (server)
Dest Port: 68 (client)

Ports are REVERSED because direction is reversed
```

---

### UDP Datagram Structure for OFFER

```
UDP Datagram:
┌──────────────────────────────────────────┐
│ Source Port: 67                          │  2 bytes
│ (DHCP Server port)                       │
├──────────────────────────────────────────┤
│ Destination Port: 68                     │  2 bytes
│ (DHCP Client port)                       │
├──────────────────────────────────────────┤
│ Length: 308 (8 header + 300 data)        │  2 bytes
├──────────────────────────────────────────┤
│ Checksum: 0x5B3F (calculated)            │  2 bytes
├──────────────────────────────────────────┤
│ Data: [DHCP OFFER message from L7]       │  300 bytes
│       (The entire OFFER with all options)│
└──────────────────────────────────────────┘

Total: 308 bytes
```

---

### Why Port 67 → Port 68?

```
Router perspective:
"I am the DHCP server running on port 67.
I am responding to a client listening on port 68.
Therefore: Source = 67, Destination = 68"

Client perspective (when receiving):
"I sent from port 68 to port 67.
Reply should come from port 67 to port 68.
Destination port 68 confirms this is for me."

Universal convention:
DHCP Server: Always port 67
DHCP Client: Always port 68

All DISCOVER/OFFER/REQUEST/ACK messages use these ports
```

---

### Checksum Calculation

```
Checksum: 0x5B3F (example)

Pseudo-header for checksum:
Source IP: 192.168.1.1
Dest IP: 255.255.255.255
Protocol: 17 (UDP)
UDP Length: 308

Checksum calculated over:
1. Pseudo-header (12 bytes)
2. UDP header (8 bytes)
3. DHCP OFFER message (300 bytes)

Total: 320 bytes used in checksum

Result: 16-bit checksum value

Client verifies:
1. Receives OFFER
2. Recalculates checksum
3. Compares with received checksum
4. Match → Accept packet
5. Mismatch → Discard (corrupted)
```

---

### Transport Layer Summary

```
Transport Layer wraps DHCP OFFER in UDP:

┌────────────────────────────────────────┐
│ UDP Header (8 bytes)                   │
│ ┌────────────────────────────────────┐ │
│ │ Source: 67 (server)                │ │
│ │ Dest: 68 (client)                  │ │
│ │ Length: 308                        │ │
│ │ Checksum: 0x5B3F                   │ │
│ └────────────────────────────────────┘ │
│                                        │
│ UDP Data (300 bytes)                   │
│ ┌────────────────────────────────────┐ │
│ │ [DHCP OFFER message]               │ │
│ │ Your IP: 192.168.1.21              │ │
│ │ From: 192.168.1.1                  │ │
│ └────────────────────────────────────┘ │
└────────────────────────────────────────┘

This UDP datagram now passes to Network Layer (L3)
```

---

## Layer 3: Network Layer - IP Packet

### The Addressing Challenge

**The fundamental problem:**

```
Client sent DISCOVER from: 0.0.0.0
Client still has no IP address
Client cannot receive unicast to any IP

Question: What destination IP should router use?

Options:
A) 192.168.1.21 (the offered IP)
B) 255.255.255.255 (broadcast)

Answer: Typically 255.255.255.255 (broadcast)
```

---

### Why Broadcast at Layer 3?

**Even though router knows the offered IP:**

```
Problem with unicast to 192.168.1.21:
1. Client doesn't have 192.168.1.21 configured yet
2. Client's network stack rejects unicast to unconfigured IP
3. Client won't see the OFFER

Solution: Broadcast to 255.255.255.255
1. All devices receive broadcast
2. Client receives OFFER despite no IP
3. Client checks OFFER's "Your IP" field
4. Client recognizes 192.168.1.21 is being offered
```

**Exception: Flag-based behavior**

```
DISCOVER Flags field: 0x8000 (broadcast bit set)

If broadcast bit SET:
Server MUST broadcast OFFER to 255.255.255.255

If broadcast bit NOT SET:
Server MAY unicast OFFER to offered IP
(But only if client can receive unicast)

Most clients set broadcast bit for safety
Most servers broadcast OFFER
```

---

### IP Packet Structure for OFFER

```
IPv4 Packet:
┌──────────────────────────────────────────────┐
│ Version: 4 (IPv4)           │ IHL: 5         │  1 byte
│ (4 bits)                    │ (4 bits)       │
├──────────────────────────────────────────────┤
│ DSCP: 0  │ ECN: 0                            │  1 byte
├──────────────────────────────────────────────┤
│ Total Length: 328                            │  2 bytes
│ (20 IP header + 8 UDP + 300 DHCP)           │
├──────────────────────────────────────────────┤
│ Identification: 0x5678                       │  2 bytes
├──────────────────────────────────────────────┤
│ Flags: 0x4000 (Don't Fragment)               │  2 bytes
│ Fragment Offset: 0                           │
├──────────────────────────────────────────────┤
│ TTL: 64                                      │  1 byte
├──────────────────────────────────────────────┤
│ Protocol: 17 (UDP)                           │  1 byte
├──────────────────────────────────────────────┤
│ Header Checksum: 0x8C4D                      │  2 bytes
├──────────────────────────────────────────────┤
│ Source IP: 192.168.1.1                       │  4 bytes
│ (Router's LAN IP)                            │
├──────────────────────────────────────────────┤
│ Destination IP: 255.255.255.255              │  4 bytes
│ (Broadcast - client has no IP yet)           │
├──────────────────────────────────────────────┤
│ Payload: [UDP datagram with DHCP OFFER]     │  308 bytes
└──────────────────────────────────────────────┘

Total: 328 bytes
```

---

### Key Field Changes from DISCOVER

**Comparison:**

```
Field                 DISCOVER              OFFER
----------------------------------------------------------------
Source IP             0.0.0.0               192.168.1.1
Destination IP        255.255.255.255       255.255.255.255
Protocol              17 (UDP)              17 (UDP)
TTL                   64                    64
Identification        0x1234                0x5678 (different)
```

---

### Source IP: 192.168.1.1

**Router has a real IP address:**

```
Source IP: 192.168.1.1

Binary: 11000000.10101000.00000001.00000001

Meaning: "This packet comes from the router at 192.168.1.1"

Why NOT 0.0.0.0?
- Router has IP address (unlike client)
- Router's LAN interface: 192.168.1.1
- Identifies DHCP server location
- Client learns gateway IP from this

Client sees OFFER:
"OFFER from 192.168.1.1
This is my DHCP server
This is also my default gateway"
```

---

### Destination IP: 255.255.255.255

**Still broadcast, even in reply:**

```
Destination IP: 255.255.255.255

Binary: 11111111.11111111.11111111.11111111

Why broadcast when router knows client MAC?
1. Client has NO IP configured yet
2. Client cannot receive unicast IP packets
3. Client's network stack would reject unicast
4. Broadcast ensures delivery

Alternative (rarely used):
Destination IP: 192.168.1.21 (offered IP)
- Only works if client accepts packets to unconfigured IP
- Not guaranteed by DHCP specification
- Most implementations use broadcast for safety
```

**Broadcast behavior:**

```
Router sends OFFER to 255.255.255.255:
┌─────────┐
│ Switch  │
└────┬────┘
     │
     ├──────────────────────────────┐
     │              │               │
┌────▼────┐    ┌────▼────┐    ┌────▼────┐
│ Client  │    │  PC2    │    │ Printer │
│ Waiting │    │         │    │         │
└─────────┘    └─────────┘    └─────────┘
   ✓              ✓              ✓
Receives      Receives       Receives
Processes     Ignores        Ignores
(MAC match)   (MAC mismatch) (Not DHCP client)

All receive Layer 3 broadcast
Only client processes (Layer 2 MAC filter actual)
```

---

### Network Layer Summary

```
Network Layer wraps UDP in IP packet:

┌──────────────────────────────────────────────┐
│ IP Header (20 bytes)                         │
│ ┌──────────────────────────────────────────┐ │
│ │ Version: 4, IHL: 5                       │ │
│ │ Total Length: 328                        │ │
│ │ TTL: 64                                  │ │
│ │ Protocol: 17 (UDP)                       │ │
│ │ Source IP: 192.168.1.1    ← Router IP   │ │
│ │ Dest IP: 255.255.255.255  ← Broadcast   │ │
│ └──────────────────────────────────────────┘ │
│                                              │
│ IP Payload (308 bytes)                       │
│ ┌──────────────────────────────────────────┐ │
│ │ [UDP datagram with DHCP OFFER]           │ │
│ │ Ports 67 → 68                            │ │
│ │ Your IP: 192.168.1.21                    │ │
│ └──────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘

This IP packet now passes to Data Link Layer (L2)
```

---

## Layer 2: Data Link Layer - Ethernet Frame

### The Critical Decision: Broadcast or Unicast?

**Unlike Layer 3 (which must broadcast), Layer 2 has a choice:**

```
Layer 3 Decision: Must use 255.255.255.255
  Reason: Client has no IP configured

Layer 2 Decision: Could use client MAC or broadcast
  Reason: Router knows client MAC from DISCOVER

Question: Which MAC address should router use as destination?

Option A: Client MAC (AA:BB:CC:DD:EE:FF)
  - Unicast directly to client
  - Efficient, no broadcast storm
  - Client NIC accepts frame (matches its MAC)

Option B: Broadcast MAC (FF:FF:FF:FF:FF:FF)
  - Broadcast to all devices
  - Less efficient, all devices process
  - Maintains consistency with Layer 3 broadcast
```

---

### Why Router Knows Client MAC

**From the DISCOVER packet:**

```
Client sent DISCOVER:
Layer 2 Source MAC: AA:BB:CC:DD:EE:FF
Layer 7 CHADDR field: AA:BB:CC:DD:EE:FF

Router extracted:
"Client MAC is AA:BB:CC:DD:EE:FF"

Router stored in lease table:
MAC: AA:BB:CC:DD:EE:FF → IP: 192.168.1.21

Router CAN send unicast to AA:BB:CC:DD:EE:FF
```

---

### Industry Practice: Mixed Approaches

**Most routers use Layer 2 unicast:**

```
Common implementation:
Layer 3: 255.255.255.255 (broadcast)
Layer 2: AA:BB:CC:DD:EE:FF (unicast)

Why this combination?
- Layer 3 broadcast: Client accepts despite no IP
- Layer 2 unicast: Efficient, only client processes

Result:
- Switch forwards only to client port (unicast)
- Client NIC accepts (matches MAC)
- Client IP stack accepts (broadcast IP)
- Other devices don't see traffic
```

**Alternative: Full broadcast:**

```
Conservative implementation:
Layer 3: 255.255.255.255 (broadcast)
Layer 2: FF:FF:FF:FF:FF:FF (broadcast)

Why?
- Maximum compatibility
- Paranoid safety approach
- Handles edge cases (NIC issues, VMs, etc.)

Trade-off:
- All devices process frame to Layer 2
- Slight CPU overhead on all devices
- More "correct" from broadcast perspective
```

---

### Ethernet Frame Structure (Unicast Destination)

**Most common implementation:**

```
Ethernet Frame:
┌──────────────────────────────────────────────┐
│ Preamble: 0xAA-AA-AA-AA-AA-AA-AA            │  7 bytes
├──────────────────────────────────────────────┤
│ SFD: 0xAB (Start Frame Delimiter)            │  1 byte
├──────────────────────────────────────────────┤
│ Destination MAC: AA:BB:CC:DD:EE:FF           │  6 bytes
│ (Unicast to client - not broadcast!)         │
├──────────────────────────────────────────────┤
│ Source MAC: A1:B1:C1:D1:E1:F1                │  6 bytes
│ (Router's LAN interface MAC)                 │
├──────────────────────────────────────────────┤
│ EtherType: 0x0800 (IPv4)                     │  2 bytes
├──────────────────────────────────────────────┤
│ Payload: [IP packet with UDP/DHCP OFFER]    │  328 bytes
├──────────────────────────────────────────────┤
│ FCS (Frame Check Sequence): 0x9ABCDEF0       │  4 bytes
└──────────────────────────────────────────────┘

Total: 346 bytes (without preamble/SFD)
```

---

### Alternative: Broadcast Destination

**Conservative implementation:**

```
Ethernet Frame (Alternative):
┌──────────────────────────────────────────────┐
│ Destination MAC: FF:FF:FF:FF:FF:FF           │  6 bytes
│ (Broadcast - maintains consistency)          │
├──────────────────────────────────────────────┤
│ Source MAC: A1:B1:C1:D1:E1:F1                │  6 bytes
│ (Router's LAN interface MAC)                 │
├──────────────────────────────────────────────┤
│ EtherType: 0x0800 (IPv4)                     │  2 bytes
├──────────────────────────────────────────────┤
│ Payload: [IP packet with UDP/DHCP OFFER]    │  328 bytes
├──────────────────────────────────────────────┤
│ FCS: 0xABCD5678                              │  4 bytes
└──────────────────────────────────────────────┘

Total: 346 bytes
```

---

### Switch Behavior Difference

**Unicast destination MAC:**

```
Frame with Dest MAC: AA:BB:CC:DD:EE:FF

Switch receives on router port:
1. Read destination MAC: AA:BB:CC:DD:EE:FF
2. Lookup in MAC table:
   Port 1: AA:BB:CC:DD:EE:FF (Client)
   Port 2: 11:22:33:44:55:66
   Port 3: A1:B1:C1:D1:E1:F1 (Router)
3. Destination found on Port 1
4. Forward to Port 1 ONLY

┌──────────────────────┐
│      Switch          │
└──┬──────┬──────┬─────┘
   │      │      │
  Port1  Port2  Port3
   │      │      │
   ▼      X      X
Client   PC2   Router
Receives (no tx) (source)

Only client receives frame
Efficient!
```

**Broadcast destination MAC:**

```
Frame with Dest MAC: FF:FF:FF:FF:FF:FF

Switch receives on router port:
1. Read destination MAC: FF:FF:FF:FF:FF:FF
2. Broadcast address detected
3. Forward to ALL ports (except source port)

┌──────────────────────┐
│      Switch          │
└──┬──────┬──────┬─────┘
   │      │      │
  Port1  Port2  Port3
   │      │      │
   ▼      ▼      X
Client   PC2   Router
Receives Receives (source)

All devices receive frame
Process to Layer 2, check if interested
```

---

### Source MAC: Router's LAN Interface

```
Source MAC: A1:B1:C1:D1:E1:F1

This is router's LAN interface MAC address

Why important:
- Identifies sender at Layer 2
- Client learns router's MAC
- Used for future unicast replies
- Stored in client's ARP cache

Client learns:
"OFFER came from MAC A1:B1:C1:D1:E1:F1
This is my gateway at Layer 2
To reach 192.168.1.1, use MAC A1:B1:C1:D1:E1:F1"

ARP cache entry created:
192.168.1.1 → A1:B1:C1:D1:E1:F1
```

---

### Why Layer 2 Unicast is Preferred

**Technical reasoning:**

```
Advantages of Layer 2 unicast (AA:BB:CC:DD:EE:FF):
1. Efficiency: Only client processes frame
2. Switch optimization: Single port forwarding
3. Reduced broadcast domain congestion
4. Scales better in large networks
5. Client NIC accepts frame anyway (matches MAC)

Layer 3 still broadcast (255.255.255.255):
- Client accepts packet despite no IP
- IP stack processes broadcast correctly
- DHCP client receives OFFER

Result: Best of both worlds
- Layer 2 efficiency (unicast)
- Layer 3 compatibility (broadcast)
```

**Why not both broadcast?**

```
Layer 2 + Layer 3 both broadcast:
Advantages:
- Maximum paranoia/safety
- Handles weird edge cases
- Philosophically "pure" (all broadcast)

Disadvantages:
- All devices process to Layer 3
- Wasted CPU cycles on non-clients
- Increased broadcast traffic
- Less scalable

Modern routers prefer: L2 unicast + L3 broadcast
```

---

### Data Link Layer Summary

```
Data Link Layer wraps IP packet in Ethernet frame:

┌────────────────────────────────────────────────┐
│ Ethernet Header (14 bytes)                     │
│ ┌────────────────────────────────────────────┐ │
│ │ Dest MAC: AA:BB:CC:DD:EE:FF  ← Client MAC │ │
│ │ Source MAC: A1:B1:C1:D1:E1:F1 ← Router    │ │
│ │ EtherType: 0x0800 (IPv4)                   │ │
│ └────────────────────────────────────────────┘ │
│                                                │
│ Ethernet Payload (328 bytes)                   │
│ ┌────────────────────────────────────────────┐ │
│ │ [IP packet: 192.168.1.1 → 255.255.255.255]│ │
│ │ [UDP: Port 67 → 68]                        │ │
│ │ [DHCP OFFER: Your IP = 192.168.1.21]      │ │
│ └────────────────────────────────────────────┘ │
│                                                │
│ Ethernet Trailer (4 bytes)                     │
│ ┌────────────────────────────────────────────┐ │
│ │ FCS: 0x9ABCDEF0 (CRC-32)                   │ │
│ └────────────────────────────────────────────┘ │
└────────────────────────────────────────────────┘

Total frame: 346 bytes

This frame now passes to Physical Layer (L1)
```

---

## Layer 1: Physical Layer - Electrical Signals

### Signal Transmission

**Same as DISCOVER, but direction reversed:**

```
Router's NIC transmits:
346 bytes = 2,768 bits

Encoding: Manchester/4B5B/8B10B (depends on Ethernet speed)
Medium: Twisted-pair copper (Cat5e/Cat6)
Speed: 100 Mbps or 1 Gbps
Distance: Up to 100 meters

Voltage: Differential signaling
+2.5V / -2.5V (or similar, depending on standard)

Transmission time (100 Mbps):
2,768 bits / 100,000,000 bps = 27.68 microseconds

Signal travels at ~200,000 km/s in copper
(~2/3 speed of light)
```

---

### Physical Layer Summary

```
Physical Layer converts frame to electrical signals:

Ethernet Frame (346 bytes)
         ↓
Binary representation (2,768 bits)
         ↓
Encoding (Manchester, 4B/5B, etc.)
         ↓
Electrical signals
         ↓
Transmitted on copper cable
         ↓
Received by client's NIC
         ↓
Decoded back to binary
         ↓
Passed up through client's network stack
```

---

## The Complete DHCP OFFER Packet

### Full Stack View

```
┌────────────────────────────────────────────────────────┐
│ Layer 1 (Physical)                                     │
│ Electrical signals from router to client               │
│ ~2,768 bits transmitted                                │
└────────────────────┬───────────────────────────────────┘
                     │
┌────────────────────▼───────────────────────────────────┐
│ Layer 2 (Data Link) - Ethernet Frame                   │
│ ┌────────────────────────────────────────────────────┐ │
│ │ Dest MAC: AA:BB:CC:DD:EE:FF (Client)               │ │
│ │ Source MAC: A1:B1:C1:D1:E1:F1 (Router)             │ │
│ │ EtherType: 0x0800 (IPv4)                           │ │
│ │ FCS: 0x9ABCDEF0                                    │ │
│ └────────────────────────────────────────────────────┘ │
│ Total: 346 bytes                                       │
└────────────────────┬───────────────────────────────────┘
                     │
┌────────────────────▼───────────────────────────────────┐
│ Layer 3 (Network) - IP Packet                          │
│ ┌────────────────────────────────────────────────────┐ │
│ │ Version: 4, IHL: 5, TTL: 64                        │ │
│ │ Protocol: 17 (UDP)                                 │ │
│ │ Source IP: 192.168.1.1 (Router)                    │ │
│ │ Dest IP: 255.255.255.255 (Broadcast)               │ │
│ │ Total Length: 328 bytes                            │ │
│ └────────────────────────────────────────────────────┘ │
└────────────────────┬───────────────────────────────────┘
                     │
┌────────────────────▼───────────────────────────────────┐
│ Layer 4 (Transport) - UDP Datagram                     │
│ ┌────────────────────────────────────────────────────┐ │
│ │ Source Port: 67 (DHCP Server)                      │ │
│ │ Dest Port: 68 (DHCP Client)                        │ │
│ │ Length: 308 bytes                                  │ │
│ │ Checksum: 0x5B3F                                   │ │
│ └────────────────────────────────────────────────────┘ │
└────────────────────┬───────────────────────────────────┘
                     │
┌────────────────────▼───────────────────────────────────┐
│ Layer 7 (Application) - DHCP OFFER Message             │
│ ┌────────────────────────────────────────────────────┐ │
│ │ Op: 2 (BOOTREPLY)                                  │ │
│ │ Transaction ID: 0x3903F326 (matched!)              │ │
│ │ Client MAC: AA:BB:CC:DD:EE:FF                      │ │
│ │ Your IP: 192.168.1.21 ← OFFERED IP!               │ │
│ │ Server IP: 192.168.1.1                             │ │
│ │ Options:                                           │ │
│ │   Message Type: OFFER (2)                          │ │
│ │   Server ID: 192.168.1.1                           │ │
│ │   Lease Time: 86400 sec (24 hours)                 │ │
│ │   Subnet Mask: 255.255.255.0                       │ │
│ │   Gateway: 192.168.1.1                             │ │
│ │   DNS: 8.8.8.8, 8.8.4.4                            │ │
│ │ Message: "I offer you IP 192.168.1.21"            │ │
│ └────────────────────────────────────────────────────┘ │
│ Total: ~300 bytes                                      │
└────────────────────────────────────────────────────────┘
```

---

## Client Reception and Processing

### How Client Receives DHCP OFFER

```
Step 1: Physical Layer receives electrical signals
└─> Decodes to binary frame (2,768 bits)

Step 2: Data Link Layer processes Ethernet frame
├─> Checks destination MAC: AA:BB:CC:DD:EE:FF
│   (Matches my MAC - accept!)
├─> Checks FCS: Recalculate CRC-32
│   (Match - frame valid)
└─> Strips Ethernet header/trailer
    Passes IP packet to Network Layer

Step 3: Network Layer processes IP packet
├─> Checks destination IP: 255.255.255.255
│   (Broadcast - accept)
│   Note: Even though I don't have IP yet, I accept broadcasts
├─> Checks protocol: 17 (UDP)
├─> Verifies checksum
└─> Strips IP header
    Passes UDP datagram to Transport Layer

Step 4: Transport Layer processes UDP datagram
├─> Checks destination port: 68
│   (DHCP client port - accept)
├─> Verifies checksum
└─> Strips UDP header
    Passes DHCP message to Application Layer

Step 5: Application Layer processes DHCP OFFER
├─> DHCP client receives message
├─> Reads Op: 2 (BOOTREPLY)
├─> Reads Transaction ID: 0x3903F326
│   (Matches my DISCOVER Transaction ID - accept!)
├─> Reads Client MAC: AA:BB:CC:DD:EE:FF
│   (Matches my MAC - this OFFER is for me!)
├─> Reads Your IP: 192.168.1.21
│   (This is the offered IP address!)
├─> Reads options: Subnet mask, gateway, DNS, lease time
└─> DECISION: Accept offer? Send REQUEST

Client state transition:
INIT → SELECTING → (received OFFER) → Prepare REQUEST
```

---

### Client's Decision Logic

```
Client received OFFER:
┌─────────────────────────────────────────┐
│ Offered IP: 192.168.1.21                │
│ From Server: 192.168.1.1                │
│ Subnet: 255.255.255.0                   │
│ Gateway: 192.168.1.1                    │
│ DNS: 8.8.8.8, 8.8.4.4                   │
│ Lease: 24 hours                         │
│ Transaction ID: 0x3903F326 (matches!)   │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Validation checks:                      │
│ ✓ Transaction ID matches my DISCOVER    │
│ ✓ Client MAC matches mine               │
│ ✓ Offered IP is valid (not 0.0.0.0)     │
│ ✓ Server ID provided (192.168.1.1)     │
│ ✓ Essential options included (mask, gw) │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Multiple OFFERs?                        │
│ - Wait 1-2 seconds for more OFFERs     │
│ - If multiple servers respond:          │
│   - Choose first received (typical)     │
│   - Or choose by policy (prefer server) │
│ - Most networks: only one DHCP server   │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Decision: Accept this OFFER             │
│ Next step: Send DHCP REQUEST            │
│ Requesting: 192.168.1.21                │
│ From Server: 192.168.1.1                │
│ Same Transaction ID: 0x3903F326         │
└─────────────────────────────────────────┘
```

---

## DISCOVER vs OFFER Comparison

### Side-by-Side Layer Comparison

```
Layer 7 (Application):
┌────────────────────────┬────────────────────────┐
│ DISCOVER               │ OFFER                  │
├────────────────────────┼────────────────────────┤
│ Op: 1 (REQUEST)        │ Op: 2 (REPLY)          │
│ Transaction: 0x3903F326│ Transaction: 0x3903F326│
│ Client IP: 0.0.0.0     │ Client IP: 0.0.0.0     │
│ Your IP: 0.0.0.0       │ Your IP: 192.168.1.21  │
│ Server IP: 0.0.0.0     │ Server IP: 192.168.1.1 │
│ Client MAC: AA:BB:...  │ Client MAC: AA:BB:...  │
│ Msg Type: DISCOVER (1) │ Msg Type: OFFER (2)    │
│ No options provided    │ Mask, Gateway, DNS, etc│
└────────────────────────┴────────────────────────┘

Layer 4 (Transport):
┌────────────────────────┬────────────────────────┐
│ DISCOVER               │ OFFER                  │
├────────────────────────┼────────────────────────┤
│ Source Port: 68        │ Source Port: 67        │
│ Dest Port: 67          │ Dest Port: 68          │
│ Length: 308            │ Length: 308            │
│ Checksum: 0x4F2A       │ Checksum: 0x5B3F       │
└────────────────────────┴────────────────────────┘

Layer 3 (Network):
┌────────────────────────┬────────────────────────┐
│ DISCOVER               │ OFFER                  │
├────────────────────────┼────────────────────────┤
│ Source: 0.0.0.0        │ Source: 192.168.1.1    │
│ Dest: 255.255.255.255  │ Dest: 255.255.255.255  │
│ Protocol: UDP (17)     │ Protocol: UDP (17)     │
│ TTL: 64                │ TTL: 64                │
└────────────────────────┴────────────────────────┘

Layer 2 (Data Link):
┌────────────────────────┬────────────────────────┐
│ DISCOVER               │ OFFER                  │
├────────────────────────┼────────────────────────┤
│ Source: AA:BB:CC:...   │ Source: A1:B1:C1:...   │
│ Dest: FF:FF:FF:FF:FF:FF│ Dest: AA:BB:CC:... *   │
│ EtherType: IPv4        │ EtherType: IPv4        │
│ FCS: 0x12345678        │ FCS: 0x9ABCDEF0        │
└────────────────────────┴────────────────────────┘

* Layer 2 destination can be unicast (AA:BB:...) 
  or broadcast (FF:FF:FF:FF:FF:FF) depending on router
```

---

### Key Differences Summary

```
Initiator:
DISCOVER: Client initiates
OFFER: Router responds

Transaction ID:
Both: SAME (0x3903F326) - critical for matching

Addressing (MAC):
DISCOVER: Client MAC → Broadcast
OFFER: Router MAC → Client MAC (typically)

Addressing (IP):
DISCOVER: 0.0.0.0 → 255.255.255.255
OFFER: 192.168.1.1 → 255.255.255.255

Ports:
DISCOVER: 68 → 67
OFFER: 67 → 68

Critical Field:
DISCOVER: "I need an IP"
OFFER: "Your IP = 192.168.1.21"

Information:
DISCOVER: Minimal (just request)
OFFER: Rich (IP, mask, gateway, DNS, lease time)
```

---

## Troubleshooting DHCP OFFER Issues

### Common Problems

**Problem 1: Client never receives OFFER**

```
Symptoms:
- DISCOVER sent successfully
- No OFFER received
- Client remains in SELECTING state
- Eventually timeout and retry DISCOVER

Possible causes:

1. DHCP server not running
   Check: Router DHCP service status
   Fix: Enable/start DHCP server

2. DHCP pool exhausted
   Check: Router lease table full
   Fix: Expand pool, reduce lease time, release old leases

3. Firewall blocking UDP port 67
   Check: Router firewall rules
   Fix: Allow UDP 67/68 (DHCP ports)

4. Network cable issue
   Check: Link lights, cable continuity
   Fix: Replace cable, reseat connectors

5. Switch blocking broadcasts
   Check: Switch broadcast storm protection
   Fix: Adjust switch VLAN/broadcast settings

Diagnosis:
$ sudo tcpdump -i eth0 -n port 67 or port 68
# Look for DISCOVER but no following OFFER
```

---

**Problem 2: OFFER received but ignored by client**

```
Symptoms:
- Wireshark shows OFFER arriving
- Client doesn't accept OFFER
- Client retransmits DISCOVER

Possible causes:

1. Transaction ID mismatch
   DISCOVER: 0x3903F326
   OFFER: 0xABCD1234 (different!)
   Result: Client ignores OFFER (not for me)
   Fix: Router DHCP server bug, restart server

2. Client MAC mismatch in OFFER
   DISCOVER from: AA:BB:CC:DD:EE:FF
   OFFER for: AA:BB:CC:DD:EE:FE (typo!)
   Result: Client ignores (not my MAC)
   Fix: Router parsing bug

3. Malformed OFFER packet
   Missing required options (subnet mask, etc.)
   Invalid IP address (0.0.0.0)
   Corrupted packet (checksum fail)
   Fix: Update router firmware

4. Client DHCP service issue
   DHCP client not listening on port 68
   DHCP client crashed
   Fix: Restart network service

Diagnosis:
$ sudo tcpdump -i eth0 -vvv -n port 68
# Verify OFFER reaches client, check fields
```

---

**Problem 3: Multiple OFFERs confusing client**

```
Symptoms:
- Client receives multiple OFFERs
- Different IPs offered
- Client behavior unpredictable

Scenario:
Router A offers: 192.168.1.21
Router B offers: 192.168.1.150
Client receives both OFFERs

Client behavior:
- Typically accepts first received
- Sends REQUEST for chosen IP
- Other server ignores REQUEST (not for me)

Problem if both servers assign IP:
- IP conflict
- Network communication fails

Solution:
- Only one DHCP server per network segment
- Disable extra DHCP servers
- Use DHCP relay if multiple subnets needed

Diagnosis:
$ sudo tcpdump -i eth0 -n port 67 or port 68
# Look for multiple OFFER packets from different sources
```

---

**Problem 4: OFFER has wrong configuration**

```
Symptoms:
- Client accepts OFFER
- Client configures IP
- But client can't reach internet

Possible wrong configurations:

1. Wrong subnet mask
   Offered: 255.255.0.0
   Should be: 255.255.255.0
   Result: Client thinks entire /16 is local, routing breaks

2. Wrong gateway
   Offered: 192.168.1.254 (non-existent)
   Should be: 192.168.1.1
   Result: Client can't reach internet

3. Wrong DNS servers
   Offered: 0.0.0.0
   Should be: 8.8.8.8
   Result: Client can't resolve domain names

4. IP already in use
   Offered: 192.168.1.21 (but PC2 has this IP)
   Result: IP conflict, communication fails for both

Fix: Correct router DHCP configuration

Diagnosis:
Client side:
$ ip addr show
$ ip route show
$ cat /etc/resolv.conf

Verify offered configuration matches network topology
```

---

### Packet Capture Example

**Capturing DHCP OFFER:**

```bash
$ sudo tcpdump -i eth0 -vvv -n port 67 or port 68 -w dhcp.pcap

# Wait for DHCP transaction
# Ctrl+C to stop

$ tcpdump -r dhcp.pcap -vvv -n

Output:
10:15:23.500000 IP (tos 0x0, ttl 64, id 22136, offset 0, flags [none], proto UDP (17), length 328)
    192.168.1.1.67 > 255.255.255.255.68: [udp sum ok] BOOTP/DHCP, Reply, length 300, xid 0x3903f326, Flags [none] (0x0000)
      Your-IP 192.168.1.21
      Server-IP 192.168.1.1
      Client-Ethernet-Address aa:bb:cc:dd:ee:ff
      Vendor-rfc1048 Extensions
        Magic Cookie 0x63825363
        DHCP-Message Option 53, length 1: Offer
        Server-ID Option 54, length 4: 192.168.1.1
        Lease-Time Option 51, length 4: 86400
        Subnet-Mask Option 1, length 4: 255.255.255.0
        Default-Gateway Option 3, length 4: 192.168.1.1
        Domain-Name-Server Option 6, length 8: 8.8.8.8, 8.8.4.4

Analysis:
✓ Source: 192.168.1.1 (router)
✓ Dest: 255.255.255.255 (broadcast)
✓ Ports: 67 → 68
✓ DHCP Message: OFFER
✓ Your IP: 192.168.1.21 (offered)
✓ Transaction ID: 0x3903f326 (matches DISCOVER)
✓ All essential options present
```

---

## Summary and Key Takeaways

### The OFFER Construction Process

```
Router constructs DHCP OFFER:

1. Application Layer (L7):
   Creates OFFER message with:
   - Your IP: 192.168.1.21 (allocated from pool)
   - Subnet Mask: 255.255.255.0
   - Gateway: 192.168.1.1
   - DNS: 8.8.8.8, 8.8.4.4
   - Lease: 24 hours
   - Transaction ID: 0x3903F326 (matched from DISCOVER)
   → ~300 bytes DHCP OFFER

2. Transport Layer (L4):
   Wraps in UDP: ports 67 → 68 (reversed from DISCOVER)
   → 308 bytes UDP datagram

3. Network Layer (L3):
   Wraps in IP: 192.168.1.1 → 255.255.255.255
   (Broadcast - client has no IP yet)
   → 328 bytes IP packet

4. Data Link Layer (L2):
   Wraps in Ethernet: A1:B1... → AA:BB:... (unicast typical)
   (Or broadcast FF:FF:... in conservative implementations)
   → 346 bytes Ethernet frame

5. Physical Layer (L1):
   Converts to electrical signals
   → 2,768 bits on wire
```

---

### Critical Concepts

**1. Broadcast vs Unicast Decision**

```
Layer 3 (IP): MUST broadcast (255.255.255.255)
  Reason: Client has no IP configured

Layer 2 (Ethernet): CAN unicast (AA:BB:CC:DD:EE:FF)
  Reason: Router knows client MAC from DISCOVER
  
Common: Layer 3 broadcast + Layer 2 unicast
  Result: Efficient and compatible
```

**2. Transaction ID Matching**

```
DISCOVER: Transaction ID = 0x3903F326
OFFER: Transaction ID = 0x3903F326 (MUST MATCH)

Without matching:
- Client can't identify its OFFER
- Multiple clients would conflict
- DHCP would break
```

**3. Port Reversal**

```
Request direction (DISCOVER):
Client port 68 → Server port 67

Response direction (OFFER):
Server port 67 → Client port 68

All DHCP communication uses these two ports
```

**4. The "Your IP" Field**

```
This is the key payload of OFFER:
Your IP = 192.168.1.21

Tells client: "This is the IP I'm offering you"

Client will REQUEST this IP in next message
Server will ACKNOWLEDGE this IP in final message
```

---

### OFFER in DORA Context

```
Current status: 2 of 4 complete

✓ D - DISCOVER (Chapter 042)
  Client: "I need an IP address"
  0.0.0.0 → 255.255.255.255 (broadcast)

✓ O - OFFER (Chapter 043 - THIS CHAPTER)
  Router: "I offer you 192.168.1.21"
  192.168.1.1 → 255.255.255.255 (broadcast)

⧗ R - REQUEST (Next chapter)
  Client: "I accept 192.168.1.21"
  0.0.0.0 → 255.255.255.255 (broadcast again!)

⧗ A - ACKNOWLEDGE (Future chapter)
  Router: "Confirmed, 192.168.1.21 is yours"
  192.168.1.1 → 192.168.1.21 (can be unicast now!)
```

---

### Why OFFER Matters

**Before OFFER:**
- Client has no IP address
- Client can't communicate on network
- Client waiting in limbo

**After OFFER:**
- Client knows an IP is available
- Client knows network configuration (mask, gateway, DNS)
- Client can proceed to REQUEST
- Almost ready for network communication

**OFFER is the first glimpse of network identity** for a new computer. The router says "I see you, I recognize your MAC address, and I have network resources available for you." This is the moment when a computer transitions from complete unknown to potential network member.

---

## Conclusion

DHCP OFFER is the gracious response to a desperate plea. A computer with no IP address broadcast its DISCOVER, and the router—having heard that broadcast—now replies with an OFFER.

The OFFER packet travels from router to client, carrying not just an IP address, but complete network configuration: subnet mask, gateway, DNS servers, domain name, and lease duration. The router carefully matches the Transaction ID from the DISCOVER, ensuring this OFFER reaches the correct client even in networks with multiple simultaneous DHCP requests.

The addressing strategy reveals networking wisdom: Layer 3 broadcasts (because the client has no IP), while Layer 2 typically unicasts (because the router knows the client's MAC). This hybrid approach balances compatibility with efficiency—ensuring the OFFER reaches the client while minimizing unnecessary network traffic.

By breaking down the OFFER packet layer by layer, you now understand:
- How routers construct responses to broadcast requests
- Why Transaction IDs are absolutely critical for message matching
- How broadcast and unicast can coexist across different layers
- What information a DHCP server provides beyond just an IP address
- The precise byte structure of the response flowing router to client

**DHCP OFFER: The router's generous response, offering network citizenship to a computer asking for identity.**

The client now has an offer. In the next chapter, we'll see the client formally REQUEST that offered IP address, completing the handshake.

---

## Further Reading

- **RFC 2131:** Dynamic Host Configuration Protocol (DHCP OFFER message specification)
- **RFC 2132:** DHCP Options and BOOTP Vendor Extensions
- **"TCP/IP Illustrated, Volume 1" by W. Richard Stevens:** DHCP protocol flow
- **"Computer Networks" by Andrew S. Tanenbaum:** DHCP request/response patterns
- **Wireshark DHCP analysis:** Filter: `bootp` or `dhcp`
- **ISC DHCP Server documentation:** Open-source DHCP server implementation
- **RFC 3046:** DHCP Relay Agent Information Option
- **RFC 4361:** Node-specific Client Identifiers for DHCPv4
