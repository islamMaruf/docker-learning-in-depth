# Chapter 37: DHCP REQUEST and ACKNOWLEDGE - Breaking Into Pieces In Details

## Overview

The journey is nearly complete. A computer with no IP address sent a DHCP DISCOVER broadcast (Chapter 042). The router's DHCP server responded with a generous DHCP OFFER containing an IP address, subnet mask, gateway, and DNS servers (Chapter 043). Now comes the final two-step handshake that transforms an offer into a binding commitment.

This chapter covers the final two messages of the DORA process:

**REQUEST (R):** The client formally accepts the offered IP address  
**ACKNOWLEDGE (ACK):** The router confirms and officially assigns the IP address

After these two messages, the client will have a fully configured network identity. The IP address will be bound to the client's MAC address in the router's DHCP lease table. The client can finally communicate on the network. This is the moment when a computer transitions from network outsider to network citizen.

But these final steps contain important nuances:
- Why does REQUEST still broadcast at Layer 3 even though the client knows the router's IP?
- How does the client indicate which server's offer it's accepting (important when multiple DHCP servers exist)?
- When can the router finally stop broadcasting and send unicast to the client?
- What exactly goes into the DHCP lease table, and how does it enforce IP uniqueness?

By the end of this chapter, you'll understand the complete DORA handshake from first broadcast to final confirmation, byte by byte, layer by layer, decision by decision.

**This is where network chaos gives way to network order.**

---

## Recap: The Story So Far

### DORA Progress

```
✓ D - DISCOVER (Chapter 042)
  Client → Router (broadcast)
  "I need an IP address"
  
  Layer 3: 0.0.0.0 → 255.255.255.255
  Layer 2: AA:BB:CC:DD:EE:FF → FF:FF:FF:FF:FF:FF
  Ports: 68 → 67
  Transaction ID: 0x3903F326

✓ O - OFFER (Chapter 043)
  Router → Client (broadcast)
  "I offer you 192.168.1.21"
  
  Layer 3: 192.168.1.1 → 255.255.255.255
  Layer 2: A1:B1:C1:D1:E1:F1 → AA:BB:CC:DD:EE:FF
  Ports: 67 → 68
  Transaction ID: 0x3903F326 (matched!)
  Configuration: IP, mask, gateway, DNS, lease time

⧗ R - REQUEST (This chapter - Part 1)
  Client → Router
  "I accept 192.168.1.21"

⧗ A - ACKNOWLEDGE (This chapter - Part 2)
  Router → Client
  "Confirmed! 192.168.1.21 is yours"
```

---

### Current Network State

**Client computer:**

```
┌─────────────────────────────────┐
│   Computer (Client)             │
│   ┌─────────────────────────┐   │
│   │ MAC: AA:BB:CC:DD:EE:FF  │   │
│   │ IP: None (not configured)│  │
│   │ State: OFFER received    │   │
│   │                          │   │
│   │ Offered:                 │   │
│   │   IP: 192.168.1.21       │   │
│   │   Mask: 255.255.255.0    │   │
│   │   Gateway: 192.168.1.1   │   │
│   │   DNS: 8.8.8.8, 8.8.4.4  │   │
│   │   Lease: 24 hours        │   │
│   │                          │   │
│   │ Decision: ACCEPT!        │   │
│   │ Next: Send REQUEST       │   │
│   └─────────────────────────┘   │
└─────────────────────────────────┘
```

**Router:**

```
┌─────────────────────────────────┐
│   Router                        │
│   ┌─────────────────────────┐   │
│   │ LAN MAC: A1:B1:C1:D1:E1:F1│
│   │ LAN IP: 192.168.1.1     │   │
│   │ DHCP Server: Running    │   │
│   │                          │   │
│   │ Lease Table (Pending):   │   │
│   │ AA:BB:..→192.168.1.21   │   │
│   │   State: OFFERED         │   │
│   │   Expires if no REQUEST  │   │
│   │                          │   │
│   │ Waiting for REQUEST...   │   │
│   └─────────────────────────┘   │
└─────────────────────────────────┘
```

---

## PART 1: DHCP REQUEST

### The Scenario: Client Accepts Offer

**Client's decision logic:**

```
Received OFFER from 192.168.1.1:
  Your IP: 192.168.1.21
  Subnet Mask: 255.255.255.0
  Gateway: 192.168.1.1
  DNS: 8.8.8.8, 8.8.4.4
  Lease: 86400 seconds

Validation:
✓ Transaction ID matches my DISCOVER
✓ Client MAC matches mine
✓ Offered IP is valid (not 0.0.0.0, not broadcast)
✓ Server ID provided (192.168.1.1)
✓ Essential options present (mask, gateway)

Decision: ACCEPT THIS OFFER

Create REQUEST message:
  Request IP: 192.168.1.21
  From Server: 192.168.1.1
  Same Transaction ID: 0x3903F326
```

---

### Why REQUEST is Necessary

**Why not just configure the IP immediately after OFFER?**

```
Problem scenarios:

1. Multiple DHCP servers
   Router A offers: 192.168.1.21
   Router B offers: 192.168.1.150
   Client must explicitly choose one
   REQUEST message indicates choice
   Rejected server releases offered IP back to pool

2. Race conditions
   Two clients discover simultaneously
   Server might offer same IP to both
   REQUEST allows server to detect conflict
   Server can NAK (negative acknowledge) if IP already taken

3. Network reliability
   OFFER might get lost/corrupted
   Client might not receive OFFER
   Without REQUEST, server doesn't know if client accepted
   REQUEST confirms "yes, I want this IP"

4. Lease management
   OFFER creates temporary reservation
   REQUEST converts temporary → permanent lease
   If no REQUEST received, reservation expires
   IP returns to available pool

REQUEST transforms "maybe" into "yes, please assign this IP to me"
```

---

## Layer 7: Application Layer - DHCP REQUEST Message

### REQUEST Message Structure

```
DHCP REQUEST Message:
┌──────────────────────────────────────────┐
│ Op: 1 (BOOTREQUEST)                      │  1 byte
│ (Client → Server, like DISCOVER)         │
├──────────────────────────────────────────┤
│ Htype: 1 (Ethernet)                      │  1 byte
├──────────────────────────────────────────┤
│ Hlen: 6 (MAC address length)             │  1 byte
├──────────────────────────────────────────┤
│ Hops: 0 (no relays)                      │  1 byte
├──────────────────────────────────────────┤
│ Transaction ID: 0x3903F326               │  4 bytes
│ (SAME as DISCOVER and OFFER!)            │
├──────────────────────────────────────────┤
│ Seconds: 0                               │  2 bytes
├──────────────────────────────────────────┤
│ Flags: 0x8000 (broadcast)                │  2 bytes
├──────────────────────────────────────────┤
│ Client IP: 0.0.0.0                       │  4 bytes
│ (STILL no IP configured yet!)            │
├──────────────────────────────────────────┤
│ Your IP: 0.0.0.0                         │  4 bytes
│ (Empty - requesting assignment)          │
├──────────────────────────────────────────┤
│ Server IP: 0.0.0.0                       │  4 bytes
├──────────────────────────────────────────┤
│ Gateway IP: 0.0.0.0                      │  4 bytes
├──────────────────────────────────────────┤
│ Client MAC: AA:BB:CC:DD:EE:FF            │  16 bytes (6 used)
├──────────────────────────────────────────┤
│ Server Name: (empty)                     │  64 bytes
├──────────────────────────────────────────┤
│ Boot Filename: (empty)                   │  128 bytes
├──────────────────────────────────────────┤
│ Magic Cookie: 0x63825363                 │  4 bytes
├──────────────────────────────────────────┤
│ DHCP Options:                            │  Variable
│   Option 53: DHCP Message Type = 3       │  (3 = REQUEST)
│   Option 54: Server Identifier           │  (192.168.1.1)
│   Option 50: Requested IP Address        │  (192.168.1.21)
│   Option 55: Parameter Request List      │
│   Option 255: End                        │
└──────────────────────────────────────────┘

Total: ~300 bytes
```

---

### Key Fields Explained

#### Op: 1 (BOOTREQUEST)

```
Op: 1

Same as DISCOVER (client → server request)

DISCOVER: Op = 1
OFFER: Op = 2 (server reply)
REQUEST: Op = 1 (client request again)
ACK: Op = 2 (server reply again)
```

#### Client IP: Still 0.0.0.0

**Critical understanding:**

```
Client IP: 0.0.0.0

Why STILL 0.0.0.0?

Client has received OFFER but hasn't configured IP yet!

DHCP process:
1. DISCOVER: Client IP = 0.0.0.0 (no IP)
2. OFFER: Your IP = 192.168.1.21 (offered)
3. REQUEST: Client IP = 0.0.0.0 (still no IP!) ← We are here
4. ACK: Your IP = 192.168.1.21 (confirmed)
5. AFTER ACK: Client configures 192.168.1.21

Client only configures IP AFTER receiving ACK, not after OFFER!

Why wait for ACK?
- Ensures server commits to assignment
- Prevents IP conflicts
- Allows server to NAK if problem occurs
```

---

### Critical Options in REQUEST

#### Option 53: Message Type = REQUEST

```
Option 53: DHCP Message Type
Length: 1 byte
Value: 3 (DHCPREQUEST)

Message types:
1 = DISCOVER
2 = OFFER
3 = REQUEST ← This message
4 = DECLINE
5 = ACK
6 = NAK

Identifies this as REQUEST message
```

#### Option 54: Server Identifier

```
Option 54: Server Identifier
Length: 4 bytes
Value: 192.168.1.1

CRITICAL FIELD!

Purpose: Identifies which server's OFFER client is accepting

Scenario with multiple servers:
Router A (192.168.1.1) offered 192.168.1.21
Router B (192.168.2.1) offered 192.168.2.50

Client sends REQUEST with:
Option 54: 192.168.1.1
Option 50: 192.168.1.21

Router A sees: "Client chose me! Process REQUEST"
Router B sees: "Client chose A, not me. Release 192.168.2.50 back to pool"

Without Option 54:
- Both servers would try to ACK
- IP chaos!
```

#### Option 50: Requested IP Address

```
Option 50: Requested IP Address
Length: 4 bytes
Value: 192.168.1.21

THE REQUESTED IP!

Explicitly states: "I want this specific IP"

Why explicit?
- "Your IP" field might be 0.0.0.0 in some REQUEST types
- Option 50 is unambiguous
- Server checks if IP still available
- Server can NAK if IP now taken

This is the IP from OFFER's "Your IP" field
Client says: "I accept the 192.168.1.21 you offered me"
```

---

### REQUEST Message Breakdown

**Human-readable interpretation:**

```
"Hello, DHCP Server at 192.168.1.1,

This is computer with MAC address AA:BB:CC:DD:EE:FF.
Transaction ID: 0x3903F326 (the same one I used in DISCOVER)

You offered me IP address 192.168.1.21 with:
- Subnet Mask: 255.255.255.0
- Gateway: 192.168.1.1
- DNS: 8.8.8.8, 8.8.4.4
- Lease: 24 hours

I accept your offer.
I am formally requesting assignment of IP address 192.168.1.21.

Please confirm this assignment by sending ACK.

Thank you!"
```

---

### Application Layer Summary (REQUEST)

```
Application Layer creates DHCP REQUEST:

┌────────────────────────────────────────┐
│ "I ACCEPT YOUR OFFER"                  │
│                                        │
│ Message Type: REQUEST                  │
│ Transaction ID: 0x3903F326 (consistent)│
│ Server Chosen: 192.168.1.1             │
│ Requested IP: 192.168.1.21             │
│ My MAC: AA:BB:CC:DD:EE:FF              │
│ My IP: Still 0.0.0.0 (not configured)  │
│                                        │
│ Size: ~300 bytes                       │
└────────────────────────────────────────┘

This REQUEST passes to Transport Layer (L4)
```

---

## Layer 4: Transport Layer - UDP Datagram (REQUEST)

### UDP Structure

```
UDP Datagram:
┌──────────────────────────────────────────┐
│ Source Port: 68                          │  2 bytes
│ (DHCP Client - same as DISCOVER)        │
├──────────────────────────────────────────┤
│ Destination Port: 67                     │  2 bytes
│ (DHCP Server - same as DISCOVER)        │
├──────────────────────────────────────────┤
│ Length: 308 (8 header + 300 data)        │  2 bytes
├──────────────────────────────────────────┤
│ Checksum: 0x6C4E (calculated)            │  2 bytes
├──────────────────────────────────────────┤
│ Data: [DHCP REQUEST message]             │  300 bytes
└──────────────────────────────────────────┘

Total: 308 bytes

Ports same as DISCOVER: 68 → 67
Client initiating, so client port → server port
```

---

### Transport Layer Summary (REQUEST)

```
Transport Layer wraps REQUEST in UDP:

┌────────────────────────────────────────┐
│ UDP Header (8 bytes)                   │
│ ┌────────────────────────────────────┐ │
│ │ Source: 68 (client)                │ │
│ │ Dest: 67 (server)                  │ │
│ │ Length: 308                        │ │
│ │ Checksum: 0x6C4E                   │ │
│ └────────────────────────────────────┘ │
│                                        │
│ UDP Data (300 bytes)                   │
│ ┌────────────────────────────────────┐ │
│ │ [DHCP REQUEST]                     │ │
│ │ I accept 192.168.1.21              │ │
│ │ From server 192.168.1.1            │ │
│ └────────────────────────────────────┘ │
└────────────────────────────────────────┘

This UDP datagram passes to Network Layer (L3)
```

---

## Layer 3: Network Layer - IP Packet (REQUEST)

### The Addressing Decision

**The critical question: What destination IP should client use?**

```
Client knows router's IP: 192.168.1.1
(Learned from OFFER message)

Options:
A) Unicast to 192.168.1.1
B) Broadcast to 255.255.255.255

Which to choose?
```

---

### Why REQUEST Still Broadcasts

**Despite knowing router's IP, REQUEST typically broadcasts:**

```
Reason 1: Client has NO IP configured yet
- Client IP still 0.0.0.0
- Cannot reliably send/receive unicast IP packets
- Some network stacks reject sending from 0.0.0.0 to specific IP
- Broadcast is safe

Reason 2: Multiple DHCP servers scenario
- Multiple servers might have sent OFFER
- Client chose one server (Option 54)
- Broadcast ensures ALL servers receive REQUEST
- Chosen server ACKs
- Rejected servers see REQUEST wasn't for them, release IP

Example:
Router A: Offered 192.168.1.21, Server ID: 192.168.1.1
Router B: Offered 192.168.2.50, Server ID: 192.168.2.1

Client REQUESTS with Option 54: 192.168.1.1

Both routers receive broadcast REQUEST:
Router A: "Option 54 = me! Process REQUEST, send ACK"
Router B: "Option 54 ≠ me. Client chose A. Release 192.168.2.50 to pool"

Reason 3: Network topology changes
- Client might have moved to different network segment
- Original server might be unreachable
- Broadcast allows any DHCP server to respond
- Failover scenarios

RFC 2131 recommendation: Broadcast REQUEST in SELECTING state
```

---

### IP Packet Structure (REQUEST)

```
IPv4 Packet:
┌──────────────────────────────────────────────┐
│ Version: 4 (IPv4)           │ IHL: 5         │  1 byte
├──────────────────────────────────────────────┤
│ DSCP: 0  │ ECN: 0                            │  1 byte
├──────────────────────────────────────────────┤
│ Total Length: 328                            │  2 bytes
│ (20 IP + 8 UDP + 300 DHCP)                  │
├──────────────────────────────────────────────┤
│ Identification: 0x9ABC                       │  2 bytes
├──────────────────────────────────────────────┤
│ Flags: 0x4000 (Don't Fragment)               │  2 bytes
├──────────────────────────────────────────────┤
│ TTL: 64                                      │  1 byte
├──────────────────────────────────────────────┤
│ Protocol: 17 (UDP)                           │  1 byte
├──────────────────────────────────────────────┤
│ Header Checksum: 0x9D5E                      │  2 bytes
├──────────────────────────────────────────────┤
│ Source IP: 0.0.0.0                           │  4 bytes
│ (Client STILL has no IP!)                    │
├──────────────────────────────────────────────┤
│ Destination IP: 255.255.255.255              │  4 bytes
│ (Broadcast - even though client knows       │
│  router IP, still broadcasts for safety)     │
├──────────────────────────────────────────────┤
│ Payload: [UDP datagram]                      │  308 bytes
└──────────────────────────────────────────────┘

Total: 328 bytes

Addressing same as DISCOVER:
Source: 0.0.0.0 (no IP yet)
Dest: 255.255.255.255 (broadcast)
```

---

### Alternative: Unicast REQUEST

**Some implementations can unicast:**

```
Alternative (less common):
Source IP: 0.0.0.0
Dest IP: 192.168.1.1 (router's IP from OFFER)

Requirements:
- Network stack allows sending from 0.0.0.0 to specific IP
- Single DHCP server environment
- No server selection conflicts

Trade-offs:
Unicast advantages:
- Reduces broadcast traffic
- More efficient
- Router easily identifies REQUEST is for it

Broadcast advantages:
- Universal compatibility
- Handles multiple DHCP servers correctly
- RFC 2131 compliant
- Works in all scenarios

Industry practice: Most implementations broadcast REQUEST
```

---

### Network Layer Summary (REQUEST)

```
Network Layer wraps UDP in IP packet:

┌──────────────────────────────────────────────┐
│ IP Header (20 bytes)                         │
│ ┌──────────────────────────────────────────┐ │
│ │ Version: 4, IHL: 5                       │ │
│ │ Total Length: 328                        │ │
│ │ TTL: 64, Protocol: 17 (UDP)              │ │
│ │ Source IP: 0.0.0.0        ← Still none! │ │
│ │ Dest IP: 255.255.255.255  ← Broadcast!  │ │
│ └──────────────────────────────────────────┘ │
│                                              │
│ IP Payload (308 bytes)                       │
│ ┌──────────────────────────────────────────┐ │
│ │ [UDP: 68 → 67]                           │ │
│ │ [DHCP REQUEST]                           │ │
│ │ Accept 192.168.1.21 from 192.168.1.1    │ │
│ └──────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘

This IP packet passes to Data Link Layer (L2)
```

---

## Layer 2: Data Link Layer - Ethernet Frame (REQUEST)

### MAC Addressing Decision

**Client now knows router's MAC address:**

```
From OFFER frame:
Source MAC: A1:B1:C1:D1:E1:F1 (router)

Client learned: "Router's MAC is A1:B1:C1:D1:E1:F1"

Question: Unicast or broadcast at Layer 2?

Option A: Unicast to A1:B1:C1:D1:E1:F1
- Efficient, only router receives frame
- Works perfectly

Option B: Broadcast to FF:FF:FF:FF:FF:FF
- All devices receive frame
- Consistent with broadcast at Layer 3
- Handles multiple DHCP servers

Most implementations: Unicast at L2, broadcast at L3
(Best of both worlds)
```

---

### Ethernet Frame Structure (REQUEST)

```
Ethernet Frame (Typical Implementation):
┌──────────────────────────────────────────────┐
│ Preamble: 0xAA-AA-AA-AA-AA-AA-AA            │  7 bytes
├──────────────────────────────────────────────┤
│ SFD: 0xAB                                    │  1 byte
├──────────────────────────────────────────────┤
│ Destination MAC: A1:B1:C1:D1:E1:F1           │  6 bytes
│ (Unicast to router - efficient)              │
├──────────────────────────────────────────────┤
│ Source MAC: AA:BB:CC:DD:EE:FF                │  6 bytes
│ (Client NIC MAC)                             │
├──────────────────────────────────────────────┤
│ EtherType: 0x0800 (IPv4)                     │  2 bytes
├──────────────────────────────────────────────┤
│ Payload: [IP packet]                         │  328 bytes
│   0.0.0.0 → 255.255.255.255                 │
│   UDP 68 → 67                                │
│   DHCP REQUEST                               │
├──────────────────────────────────────────────┤
│ FCS: 0xABCD1234                              │  4 bytes
└──────────────────────────────────────────────┘

Total: 346 bytes

Layer 2: Unicast (A1:B1:C1:D1:E1:F1)
Layer 3: Broadcast (255.255.255.255)

Hybrid approach: Efficient + Compatible
```

---

### Data Link Layer Summary (REQUEST)

```
Data Link Layer wraps IP in Ethernet frame:

┌────────────────────────────────────────────────┐
│ Ethernet Header (14 bytes)                     │
│ ┌────────────────────────────────────────────┐ │
│ │ Dest MAC: A1:B1:C1:D1:E1:F1  ← Router MAC │ │
│ │ Source MAC: AA:BB:CC:DD:EE:FF ← Client   │ │
│ │ EtherType: 0x0800 (IPv4)                   │ │
│ └────────────────────────────────────────────┘ │
│                                                │
│ Ethernet Payload (328 bytes)                   │
│ ┌────────────────────────────────────────────┐ │
│ │ [IP: 0.0.0.0 → 255.255.255.255]           │ │
│ │ [UDP: 68 → 67]                             │ │
│ │ [DHCP REQUEST for 192.168.1.21]           │ │
│ └────────────────────────────────────────────┘ │
│                                                │
│ FCS (4 bytes): 0xABCD1234                      │
└────────────────────────────────────────────────┘

Total: 346 bytes
Physical transmission to router
```

---

## Layer 1: Physical Layer (REQUEST)

**Same as previous messages:**

```
346 bytes = 2,768 bits
Encoding: Manchester/4B5B/8B10B
Medium: Twisted-pair copper
Voltage: Differential signaling

Signal travels from client NIC to router NIC
Decoded by router's physical layer
Passed up through router's network stack
```

---

## Router Receives REQUEST

### Router Processing

```
Step 1: Physical Layer
  Electrical signals → Digital bits

Step 2: Data Link Layer
  ├─ Dest MAC: A1:B1:C1:D1:E1:F1 (matches my MAC - accept!)
  ├─ FCS verification: Valid
  └─ Extract IP packet

Step 3: Network Layer
  ├─ Dest IP: 255.255.255.255 (broadcast - accept)
  ├─ Source IP: 0.0.0.0 (client has no IP - expected)
  ├─ Protocol: 17 (UDP)
  └─ Extract UDP datagram

Step 4: Transport Layer
  ├─ Dest Port: 67 (DHCP server - accept!)
  ├─ Source Port: 68 (DHCP client)
  └─ Extract DHCP message

Step 5: Application Layer - DHCP Server
  ├─ Message Type: REQUEST (3)
  ├─ Transaction ID: 0x3903F326
  ├─ Option 54 (Server ID): 192.168.1.1 (That's me!)
  ├─ Option 50 (Requested IP): 192.168.1.21
  ├─ Client MAC: AA:BB:CC:DD:EE:FF
  └─ DECISION: Process this REQUEST
```

---

### Router's DHCP Server Logic

```
DHCP Server receives REQUEST:

┌─────────────────────────────────────────┐
│ REQUEST Analysis                        │
│ From MAC: AA:BB:CC:DD:EE:FF             │
│ Transaction ID: 0x3903F326              │
│ Server ID (Option 54): 192.168.1.1      │
│ Requested IP (Option 50): 192.168.1.21  │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Validation Checks                       │
│ ✓ Server ID matches me (192.168.1.1)   │
│ ✓ Transaction ID matches OFFER I sent  │
│ ✓ Requested IP is what I offered       │
│ ✓ IP still available (not taken)        │
│ ✓ Client MAC matches original requester│
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Update DHCP Lease Table                 │
│                                         │
│ MAC: AA:BB:CC:DD:EE:FF                  │
│ IP: 192.168.1.21                        │
│ State: OFFERED → BOUND (committed!)     │
│ Lease Start: 2026-03-12 10:15:30       │
│ Lease Expires: 2026-03-13 10:15:30     │
│ Duration: 86400 seconds (24 hours)      │
│                                         │
│ This IP is now OFFICIALLY ASSIGNED      │
│ No other client can receive this IP     │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Prepare DHCP ACKNOWLEDGE                │
│ Confirm assignment of 192.168.1.21      │
│ Include all configuration options       │
│ Send ACK to client                      │
└─────────────────────────────────────────┘
```

---

### DHCP Lease Table Entry

**What gets stored:**

```
DHCP Lease Table:
┌──────────────────────────────────────────────────────┐
│ Entry 1:                                             │
│ ├─ MAC Address: AA:BB:CC:DD:EE:FF                    │
│ ├─ IP Address: 192.168.1.21                          │
│ ├─ Hostname: (optional, from Option 12)              │
│ ├─ State: BOUND                                      │
│ ├─ Lease Start: 2026-03-12 10:15:30 UTC             │
│ ├─ Lease Duration: 86400 seconds                     │
│ ├─ Lease Expires: 2026-03-13 10:15:30 UTC           │
│ ├─ Renewal Time (T1): 2026-03-12 22:15:30 (50%)     │
│ ├─ Rebinding Time (T2): 2026-03-13 07:15:30 (87.5%) │
│ └─ Transaction ID: 0x3903F326                        │
└──────────────────────────────────────────────────────┘

State transitions:
FREE → OFFERED (after OFFER sent)
OFFERED → BOUND (after REQUEST received)
BOUND → EXPIRED (after lease duration)
EXPIRED → FREE (IP returns to pool)
```

**Lease management:**

```
Renewal process (T1 - 50% of lease):
- At 12 hours (50% of 24h lease)
- Client sends REQUEST to same server
- Server responds with ACK, extends lease
- Lease timer resets

Rebinding process (T2 - 87.5% of lease):
- At 21 hours (87.5% of 24h lease)
- If renewal failed, client broadcasts REQUEST
- Any DHCP server can respond
- Failover mechanism

Expiration:
- At 24 hours (100% of lease)
- If no renewal/rebinding, lease expires
- Client must release IP and rediscover
- IP returns to DHCP pool
```

---

## PART 2: DHCP ACKNOWLEDGE (ACK)

### Router Sends Final Confirmation

**Router's ACK decision:**

```
✓ REQUEST validated
✓ IP still available
✓ Lease entry created in BOUND state
✓ MAC → IP mapping stored

Action: Send ACKNOWLEDGE to client
Confirm: "192.168.1.21 is officially yours!"
```

---

## Layer 7: Application Layer - DHCP ACK Message

### ACK Message Structure

```
DHCP ACK Message:
┌──────────────────────────────────────────┐
│ Op: 2 (BOOTREPLY)                        │  1 byte
│ (Server → Client, like OFFER)            │
├──────────────────────────────────────────┤
│ Htype: 1 (Ethernet)                      │  1 byte
├──────────────────────────────────────────┤
│ Hlen: 6                                  │  1 byte
├──────────────────────────────────────────┤
│ Hops: 0                                  │  1 byte
├──────────────────────────────────────────┤
│ Transaction ID: 0x3903F326               │  4 bytes
│ (SAME throughout entire DORA!)           │
├──────────────────────────────────────────┤
│ Seconds: 0                               │  2 bytes
├──────────────────────────────────────────┤
│ Flags: 0x8000                            │  2 bytes
├──────────────────────────────────────────┤
│ Client IP: 0.0.0.0                       │  4 bytes
│ (Client still hasn't configured IP)      │
├──────────────────────────────────────────┤
│ Your IP: 192.168.1.21                    │  4 bytes
│ (THE CONFIRMED IP - FINAL ASSIGNMENT!)   │
├──────────────────────────────────────────┤
│ Server IP: 192.168.1.1                   │  4 bytes
├──────────────────────────────────────────┤
│ Gateway IP: 0.0.0.0                      │  4 bytes
├──────────────────────────────────────────┤
│ Client MAC: AA:BB:CC:DD:EE:FF            │  16 bytes
├──────────────────────────────────────────┤
│ Server Name: (empty)                     │  64 bytes
├──────────────────────────────────────────┤
│ Boot Filename: (empty)                   │  128 bytes
├──────────────────────────────────────────┤
│ Magic Cookie: 0x63825363                 │  4 bytes
├──────────────────────────────────────────┤
│ DHCP Options:                            │
│   Option 53: DHCP Message Type = 5       │  (5 = ACK)
│   Option 54: Server Identifier           │  (192.168.1.1)
│   Option 51: Lease Time                  │  (86400 sec)
│   Option 58: Renewal Time (T1)           │  (43200 sec)
│   Option 59: Rebinding Time (T2)         │  (75600 sec)
│   Option 1: Subnet Mask                  │  (255.255.255.0)
│   Option 3: Router                       │  (192.168.1.1)
│   Option 6: DNS                          │  (8.8.8.8, 8.8.4.4)
│   Option 15: Domain Name                 │  (home.local)
│   Option 255: End                        │
└──────────────────────────────────────────┘

Total: ~300 bytes
```

---

### ACK vs OFFER Differences

```
Field/Option           OFFER                 ACK
----------------------------------------------------------------
Op                     2 (REPLY)             2 (REPLY)
Transaction ID         0x3903F326            0x3903F326
Your IP                192.168.1.21          192.168.1.21
Option 53              2 (OFFER)             5 (ACK)
Lease State (server)   OFFERED               BOUND
Meaning                "I can offer this"    "This is yours!"
Client action          Send REQUEST          Configure IP!
```

---

### Critical Options in ACK

#### Option 53: Message Type = ACK

```
Option 53: DHCP Message Type
Value: 5 (DHCPACK)

Distinguishes ACK from OFFER (both have Op=2)

Message types:
1 = DISCOVER
2 = OFFER
3 = REQUEST
4 = DECLINE
5 = ACK ← Final confirmation!
6 = NAK (negative acknowledge)
```

#### Option 58: Renewal Time (T1)

```
Option 58: Renewal Time Value (T1)
Length: 4 bytes
Value: 43200 seconds (12 hours)

When client should attempt renewal

Calculation: 50% of lease time
Lease: 86400 seconds
T1: 43200 seconds (50%)

At T1, client sends REQUEST to same server trying to renew
```

#### Option 59: Rebinding Time (T2)

```
Option 59: Rebinding Time Value (T2)
Length: 4 bytes
Value: 75600 seconds (21 hours)

When client should attempt rebinding

Calculation: 87.5% of lease time
Lease: 86400 seconds
T2: 75600 seconds (87.5%)

At T2, if renewal failed, client broadcasts REQUEST to any server
```

---

### Application Layer Summary (ACK)

```
Application Layer creates DHCP ACK:

┌────────────────────────────────────────┐
│ "CONFIRMED! IP IS YOURS!"              │
│                                        │
│ Message Type: ACK                      │
│ Transaction ID: 0x3903F326 (final)     │
│ Your IP: 192.168.1.21 (ASSIGNED!)      │
│ Lease: 24 hours                        │
│ Renewal (T1): 12 hours                 │
│ Rebinding (T2): 21 hours               │
│ All configuration included             │
│                                        │
│ Size: ~300 bytes                       │
└────────────────────────────────────────┘

ACK passes to Transport Layer (L4)
```

---

## Layer 4: Transport Layer - UDP Datagram (ACK)

```
UDP Datagram:
┌──────────────────────────────────────────┐
│ Source Port: 67 (DHCP Server)            │  2 bytes
├──────────────────────────────────────────┤
│ Destination Port: 68 (DHCP Client)       │  2 bytes
├──────────────────────────────────────────┤
│ Length: 308                              │  2 bytes
├──────────────────────────────────────────┤
│ Checksum: 0x7D5F                         │  2 bytes
├──────────────────────────────────────────┤
│ Data: [DHCP ACK message]                 │  300 bytes
└──────────────────────────────────────────┘

Ports: 67 → 68 (server → client)
Same as OFFER
```

---

## Layer 3: Network Layer - IP Packet (ACK)

### The Final Addressing Decision

**Can router finally unicast at Layer 3?**

```
Client requested 192.168.1.21
Router is ACKing assignment of 192.168.1.21

Question: Can router use 192.168.1.21 as destination?

Answer: Depends!

Option A: Unicast to 192.168.1.21
- Client hasn't configured IP yet!
- Client won't accept packet to unconfigured IP
- FAIL

Option B: Broadcast to 255.255.255.255
- Client accepts broadcast
- Safe, guaranteed delivery
- Most implementations use this

RFC 2131: Router SHOULD broadcast ACK if client's IP was 0.0.0.0

After client configures IP (post-ACK):
- Future renewals can use unicast
- Client has functional IP stack
- Efficient communication
```

---

### IP Packet Structure (ACK)

```
IPv4 Packet:
┌──────────────────────────────────────────────┐
│ Version: 4, IHL: 5                           │  1 byte
├──────────────────────────────────────────────┤
│ DSCP: 0, ECN: 0                              │  1 byte
├──────────────────────────────────────────────┤
│ Total Length: 328                            │  2 bytes
├──────────────────────────────────────────────┤
│ Identification: 0xDEF0                       │  2 bytes
├──────────────────────────────────────────────┤
│ Flags: 0x4000, Fragment Offset: 0            │  2 bytes
├──────────────────────────────────────────────┤
│ TTL: 64                                      │  1 byte
├──────────────────────────────────────────────┤
│ Protocol: 17 (UDP)                           │  1 byte
├──────────────────────────────────────────────┤
│ Header Checksum: 0xAE6F                      │  2 bytes
├──────────────────────────────────────────────┤
│ Source IP: 192.168.1.1 (Router)              │  4 bytes
├──────────────────────────────────────────────┤
│ Destination IP: 255.255.255.255              │  4 bytes
│ (Broadcast - client hasn't configured yet)   │
├──────────────────────────────────────────────┤
│ Payload: [UDP with DHCP ACK]                 │  308 bytes
└──────────────────────────────────────────────┘

Total: 328 bytes

Still broadcast at Layer 3!
Client will only configure IP AFTER receiving this ACK
```

---

## Layer 2: Data Link Layer - Ethernet Frame (ACK)

### MAC Addressing

**Router can unicast at Layer 2:**

```
Router knows client MAC: AA:BB:CC:DD:EE:FF
(From entire DORA exchange)

Destination MAC: AA:BB:CC:DD:EE:FF (client)
Source MAC: A1:B1:C1:D1:E1:F1 (router)

Layer 2 unicast is safe and efficient
Client NIC accepts frames addressed to its MAC
```

---

### Ethernet Frame Structure (ACK)

```
Ethernet Frame:
┌──────────────────────────────────────────────┐
│ Preamble: 0xAA-AA-AA-AA-AA-AA-AA            │  7 bytes
├──────────────────────────────────────────────┤
│ SFD: 0xAB                                    │  1 byte
├──────────────────────────────────────────────┤
│ Destination MAC: AA:BB:CC:DD:EE:FF           │  6 bytes
│ (Client - unicast)                           │
├──────────────────────────────────────────────┤
│ Source MAC: A1:B1:C1:D1:E1:F1                │  6 bytes
│ (Router)                                     │
├──────────────────────────────────────────────┤
│ EtherType: 0x0800 (IPv4)                     │  2 bytes
├──────────────────────────────────────────────┤
│ Payload: [IP packet with ACK]                │  328 bytes
│   192.168.1.1 → 255.255.255.255             │
│   UDP 67 → 68                                │
│   DHCP ACK                                   │
├──────────────────────────────────────────────┤
│ FCS: 0x12345678                              │  4 bytes
└──────────────────────────────────────────────┘

Total: 346 bytes

Layer 2: Unicast to client MAC
Layer 3: Broadcast IP
```

---

## Layer 1: Physical Layer (ACK)

```
346 bytes transmitted as electrical signals
Router NIC → Client NIC
Decoded by client's physical layer
Passed up through client's network stack
```

---

## Client Receives ACK - Network Configuration

### Client Processing

```
Step 1-4: Frame → IP → UDP → DHCP message extraction

Step 5: Application Layer - DHCP Client
  ├─ Message Type: ACK (5)
  ├─ Transaction ID: 0x3903F326 (matches!)
  ├─ Your IP: 192.168.1.21 (ASSIGNED!)
  ├─ Subnet Mask: 255.255.255.0
  ├─ Gateway: 192.168.1.1
  ├─ DNS: 8.8.8.8, 8.8.4.4
  ├─ Lease: 86400 seconds
  ├─ T1: 43200 seconds
  ├─ T2: 75600 seconds
  └─ DECISION: CONFIGURE NETWORK INTERFACE!
```

---

### Network Interface Configuration

**Client's OS configures network interface:**

```
Linux Example:
$ sudo ip addr add 192.168.1.21/24 dev eth0
$ sudo ip route add default via 192.168.1.1
$ echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
$ echo "nameserver 8.8.4.4" | sudo tee -a /etc/resolv.conf

Windows Example:
netsh interface ip set address "Ethernet" static 192.168.1.21 255.255.255.0 192.168.1.1
netsh interface ip set dns "Ethernet" static 8.8.8.8
netsh interface ip add dns "Ethernet" 8.8.4.4 index=2

Result:
┌────────────────────────────────────────┐
│ Network Interface: eth0/Ethernet       │
│ ┌────────────────────────────────────┐ │
│ │ IP Address: 192.168.1.21           │ │
│ │ Subnet Mask: 255.255.255.0         │ │
│ │ Network: 192.168.1.0/24            │ │
│ │ Broadcast: 192.168.1.255           │ │
│ │ Gateway: 192.168.1.1               │ │
│ │ DNS: 8.8.8.8, 8.8.4.4              │ │
│ │ Lease Expires: 2026-03-13 10:15:30│ │
│ │ State: BOUND                       │ │
│ └────────────────────────────────────┘ │
│ NETWORK INTERFACE IS NOW OPERATIONAL!  │
└────────────────────────────────────────┘
```

---

### Client State Transition

```
Before ACK:
  State: INIT → SELECTING → REQUESTING
  IP: None
  Network: Inaccessible

After ACK:
  State: BOUND
  IP: 192.168.1.21 (configured)
  Network: FULLY OPERATIONAL!

Client can now:
✓ Send/receive IP packets
✓ Communicate with local network devices
✓ Route to internet via gateway
✓ Resolve domain names via DNS
✓ Participate as full network citizen
```

---

### Renewal Timer Setup

**Client schedules renewal:**

```
Lease Start: 2026-03-12 10:15:30
Lease Duration: 86400 seconds (24 hours)
Lease Expires: 2026-03-13 10:15:30

T1 (Renewal Time): 43200 seconds (50%)
  Trigger at: 2026-03-12 22:15:30
  Action: Send REQUEST to 192.168.1.1 (unicast)
  Goal: Renew lease, extend expiration

T2 (Rebinding Time): 75600 seconds (87.5%)
  Trigger at: 2026-03-13 07:15:30
  Action: Broadcast REQUEST to any DHCP server
  Goal: Rebind if original server unreachable

Expiration: 86400 seconds (100%)
  Trigger at: 2026-03-13 10:15:30
  Action: Release IP, return to INIT state
  Goal: Prevent IP conflict

Client sets timers:
- Renewal timer (T1): 12 hours from now
- Rebinding timer (T2): 21 hours from now
- Expiration timer: 24 hours from now
```

---

## The Complete DORA Flow

### All Four Messages

```
MESSAGE 1: DISCOVER (Client → Router)
┌────────────────────────────────────────────────┐
│ Layer 7: "I need an IP"                        │
│ Layer 4: 68 → 67                               │
│ Layer 3: 0.0.0.0 → 255.255.255.255 (broadcast) │
│ Layer 2: AA:BB:.. → FF:FF:FF:FF:FF:FF          │
│ Transaction ID: 0x3903F326                     │
└────────────────────────────────────────────────┘
                      ↓
MESSAGE 2: OFFER (Router → Client)
┌────────────────────────────────────────────────┐
│ Layer 7: "I offer 192.168.1.21"                │
│ Layer 4: 67 → 68                               │
│ Layer 3: 192.168.1.1 → 255.255.255.255         │
│ Layer 2: A1:B1:.. → AA:BB:.. (unicast typical)│
│ Transaction ID: 0x3903F326                     │
│ Configuration: IP, mask, gateway, DNS, lease   │
└────────────────────────────────────────────────┘
                      ↓
MESSAGE 3: REQUEST (Client → Router)
┌────────────────────────────────────────────────┐
│ Layer 7: "I accept 192.168.1.21"               │
│ Layer 4: 68 → 67                               │
│ Layer 3: 0.0.0.0 → 255.255.255.255 (broadcast) │
│ Layer 2: AA:BB:.. → A1:B1:.. (unicast typical)│
│ Transaction ID: 0x3903F326                     │
│ Option 54: Server 192.168.1.1                  │
│ Option 50: Requested 192.168.1.21              │
└────────────────────────────────────────────────┘
                      ↓
MESSAGE 4: ACKNOWLEDGE (Router → Client)
┌────────────────────────────────────────────────┐
│ Layer 7: "Confirmed! Now yours!"                │
│ Layer 4: 67 → 68                               │
│ Layer 3: 192.168.1.1 → 255.255.255.255         │
│ Layer 2: A1:B1:.. → AA:BB:.. (unicast)        │
│ Transaction ID: 0x3903F326                     │
│ Your IP: 192.168.1.21 (FINAL!)                 │
│ Lease: 24h, T1: 12h, T2: 21h                   │
└────────────────────────────────────────────────┘
                      ↓
          CLIENT CONFIGURES IP!
      NETWORK COMMUNICATION BEGINS!
```

---

### Transaction ID Consistency

**Critical throughout:**

```
DISCOVER: Transaction ID = 0x3903F326
OFFER:    Transaction ID = 0x3903F326 ✓
REQUEST:  Transaction ID = 0x3903F326 ✓
ACK:      Transaction ID = 0x3903F326 ✓

All four messages use SAME Transaction ID

Why critical:
- Links messages in DORA sequence
- Allows client to match OFFER/ACK to its DISCOVER/REQUEST
- Handles multiple simultaneous DHCP transactions
- Prevents message confusion

Without Transaction ID:
- Client receives ACK meant for different computer
- IP conflicts
- Network chaos
```

---

### Addressing Strategy Summary

```
Layer 3 (IP) Addressing:
┌──────────┬─────────────┬─────────────────┬─────────────┐
│ Message  │ Source IP   │ Dest IP         │ Why         │
├──────────┼─────────────┼─────────────────┼─────────────┤
│ DISCOVER │ 0.0.0.0     │ 255.255.255.255 │ No IP yet   │
│ OFFER    │ 192.168.1.1 │ 255.255.255.255 │ Client no IP│
│ REQUEST  │ 0.0.0.0     │ 255.255.255.255 │ Still no IP │
│ ACK      │ 192.168.1.1 │ 255.255.255.255 │ Still no IP!│
└──────────┴─────────────┴─────────────────┴─────────────┘

Layer 2 (MAC) Addressing:
┌──────────┬─────────────┬─────────────────┬─────────────┐
│ Message  │ Source MAC  │ Dest MAC        │ Why         │
├──────────┼─────────────┼─────────────────┼─────────────┤
│ DISCOVER │ AA:BB:..    │ FF:FF:FF:FF:FF:FF│ Find server│
│ OFFER    │ A1:B1:..    │ AA:BB:.. *      │ To client   │
│ REQUEST  │ AA:BB:..    │ A1:B1:.. *      │ To router   │
│ ACK      │ A1:B1:..    │ AA:BB:..        │ To client   │
└──────────┴─────────────┴─────────────────┴─────────────┘

* Some implementations use broadcast FF:FF:FF:FF:FF:FF

Pattern: Layer 3 always broadcast during DORA,
        Layer 2 can unicast (more efficient)
```

---

## Packet Capture: Complete DORA

### tcpdump Captures

```bash
$ sudo tcpdump -i eth0 -vvv -n port 67 or port 68

# DISCOVER
10:15:20.123 IP 0.0.0.0.68 > 255.255.255.255.67: 
  BOOTP/DHCP, Request, length 300, xid 0x3903f326
  Client-Ethernet-Address aa:bb:cc:dd:ee:ff
  DHCP-Message Option 53: Discover

# OFFER
10:15:20.145 IP 192.168.1.1.67 > 255.255.255.255.68:
  BOOTP/DHCP, Reply, length 300, xid 0x3903f326
  Your-IP 192.168.1.21
  Server-IP 192.168.1.1
  DHCP-Message Option 53: Offer
  Server-ID Option 54: 192.168.1.1
  Subnet-Mask Option 1: 255.255.255.0
  Default-Gateway Option 3: 192.168.1.1
  DNS Option 6: 8.8.8.8, 8.8.4.4
  Lease-Time Option 51: 86400

# REQUEST
10:15:20.167 IP 0.0.0.0.68 > 255.255.255.255.67:
  BOOTP/DHCP, Request, length 300, xid 0x3903f326
  Client-Ethernet-Address aa:bb:cc:dd:ee:ff
  DHCP-Message Option 53: Request
  Server-ID Option 54: 192.168.1.1
  Requested-IP Option 50: 192.168.1.21

# ACK
10:15:20.189 IP 192.168.1.1.67 > 255.255.255.255.68:
  BOOTP/DHCP, Reply, length 300, xid 0x3903f326
  Your-IP 192.168.1.21
  DHCP-Message Option 53: ACK
  Server-ID Option 54: 192.168.1.1
  Lease-Time Option 51: 86400
  Renewal-Time Option 58: 43200
  Rebinding-Time Option 59: 75600

Total DORA time: 66 milliseconds
Client now has IP: 192.168.1.21
```

---

### Wireshark Analysis

```
Wireshark Display Filter: bootp

Frame 1: DHCP DISCOVER
  ├─ Time: 0.000000 (baseline)
  ├─ Src: 0.0.0.0, Dst: 255.255.255.255
  ├─ Transaction ID: 0x3903f326
  └─ Message: Discover

Frame 2: DHCP OFFER  
  ├─ Time: 0.022000 (22ms after DISCOVER)
  ├─ Src: 192.168.1.1, Dst: 255.255.255.255
  ├─ Transaction ID: 0x3903f326
  ├─ Your IP: 192.168.1.21
  └─ Message: Offer

Frame 3: DHCP REQUEST
  ├─ Time: 0.044000 (44ms, 22ms after OFFER)
  ├─ Src: 0.0.0.0, Dst: 255.255.255.255
  ├─ Transaction ID: 0x3903f326
  ├─ Server ID: 192.168.1.1
  ├─ Requested IP: 192.168.1.21
  └─ Message: Request

Frame 4: DHCP ACK
  ├─ Time: 0.066000 (66ms, 22ms after REQUEST)
  ├─ Src: 192.168.1.1, Dst: 255.255.255.255
  ├─ Transaction ID: 0x3903f326
  ├─ Your IP: 192.168.1.21
  └─ Message: ACK

Follow DHCP stream:
All four messages share Transaction ID: 0x3903f326
Complete handshake in 66 milliseconds
```

---

## Troubleshooting DORA

### Problem 1: No IP Configuration After ACK

```
Symptoms:
- ACK received
- But `ip addr show` shows no IP configured

Possible causes:

1. DHCP client service not running
   Check (Linux): systemctl status NetworkManager
   Check (Windows): services.msc → DHCP Client
   Fix: Start/enable service

2. Network manager conflict
   dhclient vs systemd-networkd vs NetworkManager
   Multiple services fighting
   Fix: Disable conflicting services

3. Manual configuration present
   Static IP configured manually
   DHCP client refuses to override
   Fix: Remove static configuration

4. ACK validation failed
   Client rejected ACK (bad options, conflicts)
   Check client logs: journalctl -u NetworkManager
   Fix: Correct server configuration
```

---

### Problem 2: IP Conflict Detected

```
Symptoms:
- Client configured IP 192.168.1.21
- But communication fails
- ARP shows conflict

Scenario:
┌──────────────┐     ┌──────────────┐
│  Computer A  │     │  Computer B  │
│  192.168.1.21│     │  192.168.1.21│
│  (DHCP)      │     │  (Static!)   │
└──────────────┘     └──────────────┘
        Both have same IP!

Cause:
- Computer B has static IP 192.168.1.21
- DHCP server didn't know (no lease entry)
- DHCP server assigned same IP to Computer A
- IP conflict!

Detection (client side):
$ ip addr show eth0
  inet 192.168.1.21/24 brd 192.168.1.255 scope global eth0
  duplicate address detection in progress

Solutions:
1. Remove static IP from Computer B
2. Exclude 192.168.1.21 from DHCP pool
3. Use DHCP reservation for Computer B
4. Expand DHCP pool range
```

---

### Problem 3: Lease Not Renewing

```
Symptoms:
- Initial DORA successful
- IP works for hours
- At T1 (12h), renewal fails
- At lease expiration, IP released
- Client becomes disconnected

Diagnosis:

T1 Renewal (50%):
$ sudo tcpdump -i eth0 port 67 or port 68
# At 12 hours, no REQUEST seen
# Or REQUEST sent but no ACK received

Possible causes:
1. Router/DHCP server offline
2. Network connectivity lost
3. Firewall blocking renewal
4. DHCP server lease table full
5. Server crashed, lost lease table

Solutions:
- Check router uptime
- Verify network cable connected
- Check firewall rules
- Restart DHCP server
- Increase lease time (longer leases)
```

---

### Problem 4: Multiple ACKs (Multiple DHCP Servers)

```
Scenario:
- Two routers both running DHCP
- Both send OFFER
- Client sends REQUEST (broadcasts)
- BOTH routers send ACK!
- Client receives two ACKs with different IPs

Example:
Router A ACK: Your IP = 192.168.1.21
Router B ACK: Your IP = 192.168.1.150

Client behavior:
- Typically accepts first received ACK
- Configures IP from first ACK
- Ignores second ACK

Problem:
- Both routers think client has their offered IP
- Routing confusion
- IP appears in two lease tables

Solution:
- Only one DHCP server per network segment
- Disable DHCP on one router
- Or use DHCP relay properly
```

---

## Summary and Key Takeaways

### DORA Complete

```
Four-message handshake:

1. DISCOVER: "I need an IP" (broadcast)
2. OFFER: "Here's 192.168.1.21" (broadcast)
3. REQUEST: "I accept 192.168.1.21" (broadcast)
4. ACK: "Confirmed, it's yours" (broadcast)

Result:
- Client: Has IP 192.168.1.21
- Router: Lease table entry created
- Network: Operational communication
```

---

### Why REQUEST Broadcasts

```
Even though client knows router IP:

Reasons:
1. Client still has no IP configured (0.0.0.0)
2. Multiple DHCP servers need to see choice
3. Maximum compatibility
4. RFC 2131 compliant

Result:
- Layer 3: Broadcast (255.255.255.255)
- Layer 2: Unicast (router MAC) - efficiency

Best of both worlds
```

---

### Critical Options

```
Option 53: Message Type
- 1=DISCOVER, 2=OFFER, 3=REQUEST, 5=ACK

Option 54: Server Identifier
- Tells router "I chose you"
- Critical for multiple server scenarios

Option 50: Requested IP Address
- Explicit IP request
- Allows server to validate availability

Option 51: Lease Time
- How long IP is valid
- Client must renew before expiration

Options 58/59: T1/T2
- When to renew (50%, 87.5%)
- Automatic lease management
```

---

### Transaction ID

```
0x3903F326 throughout entire DORA

Consistency is CRITICAL:
- Links all four messages
- Allows proper matching
- Prevents confusion
- Handles simultaneous DORAs

If Transaction ID doesn't match:
- Client ignores OFFER/ACK
- DHCP fails
- Must retry DISCOVER
```

---

### DHCP Lease Table

```
Router stores:
MAC: AA:BB:CC:DD:EE:FF
IP: 192.168.1.21
State: BOUND
Expires: 2026-03-13 10:15:30

Purpose:
- Prevents IP conflicts
- Enforces uniqueness
- Tracks usage
- Manages renewals
```

---

### State Transitions

```
Client States:
INIT → SELECTING → REQUESTING → BOUND

INIT: No configuration, send DISCOVER
SELECTING: Received OFFER(s), choose one
REQUESTING: Sent REQUEST, waiting ACK
BOUND: Received ACK, IP configured, operational

Lease Management:
BOUND → RENEWING (at T1, 50%)
RENEWING → REBINDING (at T2, 87.5%)
REBINDING → INIT (at expiration, 100%)
```

---

## Conclusion

DHCP DORA is now complete. What began as a desperate broadcast—a computer with no IP address pleading for network identity—has culminated in a fully configured network interface.

The REQUEST message formalized acceptance: "Yes, I want 192.168.1.21 from server 192.168.1.1." Even though the client knew the router's IP, it broadcast the REQUEST to handle edge cases: multiple DHCP servers, network topology changes, RFC compliance. The Option 54 (Server Identifier) field explicitly indicated which server's offer was accepted, allowing rejected servers to release their offered IPs back to their pools.

The ACKNOWLEDGE message sealed the deal. The router's DHCP server validated the request, created a BOUND lease entry in its lease table, and sent final confirmation. The client received the ACK and, for the first time, configured its network interface with a real IP address.

**From four simple messages—DISCOVER, OFFER, REQUEST, ACKNOWLEDGE—a computer transforms from network outsider to network citizen.**

The next time your laptop connects to WiFi and instantly works, remember: underneath that seamless experience, a precise four-way handshake occurred. Source IPs of 0.0.0.0, destination IPs of 255.255.255.255, Transaction IDs matching across messages, UDP ports 67 and 68, MAC address learning, lease tables updating, renewal timers setting.

**This is DHCP. This is how computers join networks. This is how chaos gives way to order.**

---

## Further Reading

- **RFC 2131:** Dynamic Host Configuration Protocol (complete DHCP specification)
- **RFC 2132:** DHCP Options and BOOTP Vendor Extensions
- **"TCP/IP Illustrated, Volume 1" by W. Richard Stevens:** DHCP chapter
- **ISC DHCP Server:** Open-source DHCP server implementation and documentation
- **Wireshark:** DHCP packet analysis tutorials
- **RFC 3046:** DHCP Relay Agent Information Option
- **RFC 4361:** Node-specific Client Identifiers for DHCPv4
- **RFC 3927:** Dynamic Configuration of IPv4 Link-Local Addresses (APIPA/AutoIP)
- **"Computer Networks" by Andrew S. Tanenbaum:** Network configuration protocols
