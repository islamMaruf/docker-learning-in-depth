# Chapter 38: Hub, Switch, Router - Network Devices In Details

## Overview

You now understand DHCP. You know how a computer acquires an IP address, subnet mask, gateway, and DNS servers through the DORA handshake. You understand how packets flow through the OSI layers—Application, Transport, Network, Data Link, Physical—and how each layer adds its headers before the NIC transmits electrical signals across the wire.

But how does data actually travel from one computer to another? When you connect multiple computers together, what devices sit between them? How do those devices decide where to forward traffic?

This chapter explores the three fundamental network devices that form the infrastructure of every network:

**Hub (Layer 1 Device):** A blind repeater that floods traffic everywhere  
**Switch (Layer 2 Device):** An intelligent forwarder that learns MAC addresses  
**Router (Layer 3 Device):** A network-boundary gateway that understands IP addresses

By the end of this chapter, you'll understand:
- Why hubs are obsolete and inefficient
- How switches build MAC address tables to forward intelligently
- Why home routers contain both a switch and a router
- How your computer decides whether to send packets directly to another device or through the router
- The role subnet masks play in forwarding decisions

**This is where individual computers become networks.**

---

## The Three Devices: OSI Layer Perspective

Before diving into each device, understand their fundamental difference: **which OSI layer they operate at**.

```
┌─────────────────────────────────────────────────┐
│ OSI Model                                       │
├─────────────────────────────────────────────────┤
│ Layer 7: Application                            │
│ Layer 6: Presentation                           │
│ Layer 5: Session                                │
│ Layer 4: Transport                              │
│ Layer 3: Network      ← ROUTER works here       │
│ Layer 2: Data Link    ← SWITCH works here       │
│ Layer 1: Physical     ← HUB works here          │
└─────────────────────────────────────────────────┘
```

**Hub (L1 Device):**
- Operates at Physical Layer only
- Understands: Electrical signals (zeros and ones)
- Cannot read: MAC addresses, IP addresses, ports, application data

**Switch (L2 Device):**
- Operates at Data Link Layer
- Understands: MAC addresses, Ethernet frames
- Cannot read: IP addresses, ports, application data

**Router (L3 Device):**
- Operates at Network Layer
- Understands: IP addresses, subnets, routing
- Can read: Everything below Layer 3 (MAC addresses, physical signals)

**The higher the layer, the more intelligent the device.**

---

## Part 1: Hub - The Blind Repeater

### What is a Hub?

A hub is the simplest network device. It connects multiple computers together, allowing them to communicate. But it does so in the most primitive way possible: **blind flooding**.

```
Physical appearance:
┌───────────────────────────────────┐
│  Network Hub (Ethernet Hub)       │
│  ┌───┬───┬───┬───┬───┬───┬───┐   │
│  │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │   │  ← Ports (4-8 typical)
│  └───┴───┴───┴───┴───┴───┴───┘   │
└───────────────────────────────────┘

Ethernet cables plug into these ports
```

**Typical port count:** 4, 5, 6, 7, or 8 ports maximum

---

### Hub Topology

```
    Computer A
    MAC: AA:BB:CC:DD:EE:FF
         │
         │
    ┌────┴────┐
    │         │
    │   HUB   │  ← Layer 1 device (Physical Layer only)
    │         │
    └─┬──┬──┬─┘
      │  │  │
      │  │  └────── Computer C
      │  │          MAC: CC:CC:CC:CC:CC:CC
      │  │
      │  └────────── Computer B
      │             MAC: BB:BB:BB:BB:BB:BB
      │
      └──────────── Computer D
                    MAC: DD:DD:DD:DD:DD:DD
```

---

### How Hub Works

**Scenario:** Computer A sends a message to Computer D.

#### Step 1: Computer A Sends Message

```
Computer A creates message:
Layer 7 (Application): "Hello World"
Layer 4 (Transport): Port 12345 → Port 80
Layer 3 (Network): 192.168.1.10 → 192.168.1.13
Layer 2 (Data Link): AA:BB:CC:DD:EE:FF → DD:DD:DD:DD:DD:DD
Layer 1 (Physical): NIC converts to electrical signals

         ↓
    Electrical signals
    (zeros and ones)
         ↓
       To Hub
```

#### Step 2: Hub Receives Electrical Signals

**Hub's processing capability:**

```
Hub receives: 01010110101010101...

Hub can read: Zeros and ones (electrical voltages)

Hub CANNOT read:
✗ MAC addresses (Data Link Layer)
✗ IP addresses (Network Layer)
✗ Ports (Transport Layer)
✗ Application data (Application Layer)

Hub only understands: SIGNAL PRESENT
```

**Hub is completely blind to all higher-layer information.**

#### Step 3: Hub Forwards Blindly

**Hub's algorithm:**

```python
def hub_forward(signal, incoming_port):
    """
    Hub forwarding logic - braindead simple
    """
    for port in all_ports:
        if port != incoming_port:  # Don't send back to source
            forward(signal, port)
```

**What actually happens:**

```
Message arrives on Port 1 (from Computer A)

Hub forwards to:
- Port 2 (Computer B) ← Gets message NOT meant for it
- Port 3 (Computer C) ← Gets message NOT meant for it  
- Port 4 (Computer D) ← Gets message MEANT for it ✓

Hub floods to ALL ports except source!
```

#### Step 4: Recipients Process Message

**Computer B receives signals:**

```
1. Physical Layer: Electrical signals → Frame
2. Data Link Layer: Check destination MAC
   Destination MAC: DD:DD:DD:DD:DD:DD
   My MAC: BB:BB:BB:BB:BB:BB
   ✗ Not for me! Discard at Layer 2
   
Never reaches Layer 3, 4, 5, 6, or 7
```

**Computer C receives signals:**

```
1. Physical Layer: Electrical signals → Frame
2. Data Link Layer: Check destination MAC
   Destination MAC: DD:DD:DD:DD:DD:DD
   My MAC: CC:CC:CC:CC:CC:CC
   ✗ Not for me! Discard at Layer 2
```

**Computer D receives signals:**

```
1. Physical Layer: Electrical signals → Frame
2. Data Link Layer: Check destination MAC
   Destination MAC: DD:DD:DD:DD:DD:DD
   My MAC: DD:DD:DD:DD:DD:DD
   ✓ For me! Accept and pass to Layer 3
3. Network Layer: Check destination IP (matches)
4. Transport Layer: Check destination port (matches)
5. Application Layer: Receive "Hello World"
```

---

### Hub Characteristics

**Advantages:**
- Simple and cheap
- No configuration required
- Works immediately

**Disadvantages:**
- **Wastes bandwidth:** Every message goes to every computer
- **No intelligence:** Cannot learn or optimize
- **Security risk:** All computers receive all traffic (packet sniffing easy)
- **Collision domain:** All ports share the same bandwidth
- **Poor scalability:** Performance degrades with each added device

**Modern status:** Obsolete. Nobody uses hubs anymore.

---

### Hub Summary

```
Hub = Layer 1 Device

Capabilities:
✓ Receives electrical signals
✓ Regenerates signals (amplification)
✓ Forwards to all ports except source

Limitations:
✗ Cannot read MAC addresses
✗ Cannot read IP addresses
✗ Cannot read ports
✗ Cannot read application data
✗ Cannot make intelligent forwarding decisions

Behavior: Blind flooding
Intelligence level: Zero
```

---

## Part 2: Switch - The Intelligent Learner

### What is a Switch?

A switch is a **Layer 2 device** that operates at the Data Link Layer. Unlike a hub, a switch **can read MAC addresses** and make intelligent forwarding decisions.

```
Physical appearance:
┌────────────────────────────────────────────┐
│  Network Switch                            │
│  ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐    │
│  │1 │2 │3 │4 │5 │6 │7 │8 │9 │10│11│12│    │  ← Ports (12-48 typical)
│  └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘    │
└────────────────────────────────────────────┘

Much more ports than hub!
Can have 12, 24, 48, or even 96 ports
```

**Key difference:** Switch maintains a **MAC address table** mapping MAC addresses to port numbers.

---

### Switch Topology

```
      Computer A
      MAC: AA:AA:AA:AA:AA:AA
      IP: 192.168.1.10
           │
           │ Port 1
      ┌────┴────────┐
      │             │
      │   SWITCH    │  ← Layer 2 device (Data Link Layer)
      │             │    Maintains MAC address table
      └─┬──┬──┬──┬──┘
        │  │  │  │
   Port │  │  │  │ Port 5
        2  3  4  
        │  │  │
        │  │  └────── Computer D
        │  │          MAC: DD:DD:DD:DD:DD:DD
        │  │          IP: 192.168.1.13
        │  │
        │  └────────── Computer C
        │             MAC: CC:CC:CC:CC:CC:CC
        │             IP: 192.168.1.12
        │
        └──────────── Computer B
                      MAC: BB:BB:BB:BB:BB:BB
                      IP: 192.168.1.11
```

---

### The CAM Table (MAC Address Table)

**Switch maintains an internal table:**

```
CAM Table (Content Addressable Memory Table)
Also known as:
- MAC Address Table
- Forwarding Database
- Bridge Table

┌─────────────────┬──────────┐
│  MAC Address    │   Port   │
├─────────────────┼──────────┤
│ (empty)         │ (empty)  │  ← Initially empty!
└─────────────────┴──────────┘

Switch learns dynamically by observing traffic
```

---

### How Switch Learns: Step-by-Step Example

#### Initial State: Empty CAM Table

```
All computers connected, but switch knows nothing yet

CAM Table:
┌─────────────────┬──────────┐
│  MAC Address    │   Port   │
├─────────────────┼──────────┤
│                 │          │  ← Empty
└─────────────────┴──────────┘
```

---

#### Message 1: Computer A → Computer D

**Step 1: Computer A Sends Data**

```
Application Layer: "I love you"
Transport Layer: 51561 → 3000
Network Layer: 192.168.1.10 → 192.168.1.13
Data Link Layer: AA:AA:AA:AA:AA:AA → DD:DD:DD:DD:DD:DD
Physical Layer: Electrical signals → Switch Port 1
```

**Step 2: Switch Receives on Port 1**

```
Switch processing:
1. Physical Layer receives electrical signals
2. Data Link Layer decodes frame:

   ┌────────────────────────────────────────┐
   │ Ethernet Frame                         │
   ├────────────────────────────────────────┤
   │ Source MAC: AA:AA:AA:AA:AA:AA          │  ← Switch reads this!
   │ Dest MAC: DD:DD:DD:DD:DD:DD            │  ← Switch reads this!
   │ EtherType: 0x0800 (IPv4)               │
   │ Payload: [IP packet with data]         │
   │ FCS: 0x1234ABCD                        │
   └────────────────────────────────────────┘

3. Switch extracts Source MAC: AA:AA:AA:AA:AA:AA
4. Switch notes: "This MAC is on Port 1"
```

**Step 3: Switch Updates CAM Table (Learning)**

```
CAM Table after learning source:
┌─────────────────────┬──────────┐
│  MAC Address        │   Port   │
├─────────────────────┼──────────┤
│ AA:AA:AA:AA:AA:AA   │    1     │  ← LEARNED!
└─────────────────────┴──────────┘

Switch: "Aha! MAC AA:AA:... is on Port 1. I'll remember this."
```

**Step 4: Switch Checks Destination**

```
Destination MAC: DD:DD:DD:DD:DD:DD

Switch checks CAM table:
Is DD:DD:DD:DD:DD:DD in table? NO

Switch decision:
"I don't know where DD:DD:... is yet.
 I must FLOOD to all ports except Port 1."
```

**Step 5: Switch Floods to All Ports**

```
Switch forwards frame to:
- Port 2 (Computer B)
- Port 3 (Computer C)
- Port 4 (Computer D)
- Port 5 (not shown, would flood here too if connected)

Same behavior as hub at this point!
But switch is learning...
```

**Step 6: Recipients Process Frame**

```
Computer B (Port 2):
- Receives frame
- Checks Dest MAC: DD:DD:DD:DD:DD:DD
- My MAC: BB:BB:BB:BB:BB:BB
- ✗ Not for me, discard at Layer 2

Computer C (Port 3):
- Receives frame
- Checks Dest MAC: DD:DD:DD:DD:DD:DD
- My MAC: CC:CC:CC:CC:CC:CC
- ✗ Not for me, discard at Layer 2

Computer D (Port 4):
- Receives frame
- Checks Dest MAC: DD:DD:DD:DD:DD:DD
- My MAC: DD:DD:DD:DD:DD:DD
- ✓ For me! Accept and process
- Passes to Layer 3 → Layer 4 → Layer 5-7
- Application receives: "I love you"
```

---

#### Message 2: Computer D → Computer A (Reply)

**Step 1: Computer D Sends Reply**

```
Application Layer: "I love you too"
Transport Layer: 3000 → 51561 (reversed ports)
Network Layer: 192.168.1.13 → 192.168.1.10 (reversed IPs)
Data Link Layer: DD:DD:DD:DD:DD:DD → AA:AA:AA:AA:AA:AA (reversed MACs)
Physical Layer: Electrical signals → Switch Port 4
```

**Step 2: Switch Receives on Port 4**

```
Switch decodes frame:
Source MAC: DD:DD:DD:DD:DD:DD  ← On Port 4
Dest MAC: AA:AA:AA:AA:AA:AA

Switch learns: "DD:DD:... is on Port 4"
```

**Step 3: Switch Updates CAM Table**

```
CAM Table after learning:
┌─────────────────────┬──────────┐
│  MAC Address        │   Port   │
├─────────────────────┼──────────┤
│ AA:AA:AA:AA:AA:AA   │    1     │
│ DD:DD:DD:DD:DD:DD   │    4     │  ← LEARNED!
└─────────────────────┴──────────┘
```

**Step 4: Switch Checks Destination**

```
Destination MAC: AA:AA:AA:AA:AA:AA

Switch checks CAM table:
Is AA:AA:... in table? YES! Port 1!

Switch decision:
"I know where AA:AA:... is! Port 1!
 I will forward ONLY to Port 1, not flood."
```

**Step 5: Switch Forwards Intelligently**

```
Switch forwards frame to:
- Port 1 ONLY (Computer A) ✓

Does NOT forward to:
- Port 2 (Computer B) ✗
- Port 3 (Computer C) ✗
- Port 5 (if exists) ✗

MUCH more efficient than hub!
Only Computer A receives this frame
```

**Switch has learned and is now forwarding intelligently!**

---

#### Message 3: Computer A → Computer D (Again)

**Now switch knows both MACs!**

```
Computer A sends again:
Source MAC: AA:AA:AA:AA:AA:AA (Port 1)
Dest MAC: DD:DD:DD:DD:DD:DD

CAM Table lookup:
AA:AA:... → Port 1 (already known)
DD:DD:... → Port 4 (already known!)

Action: Forward ONLY to Port 4

No flooding needed!
Perfect efficiency!
```

---

### Switch Learning Algorithm

```python
def switch_process_frame(frame, incoming_port):
    """
    Switch processing logic
    """
    # Step 1: Learn source MAC
    source_mac = frame.source_mac
    cam_table[source_mac] = incoming_port
    print(f"Learned: {source_mac} is on Port {incoming_port}")
    
    # Step 2: Check destination MAC
    dest_mac = frame.destination_mac
    
    if dest_mac in cam_table:
        # Known destination - forward only to that port
        dest_port = cam_table[dest_mac]
        if dest_port != incoming_port:  # Don't send back to source
            forward(frame, dest_port)
            print(f"Forwarding to Port {dest_port} only")
    else:
        # Unknown destination - flood to all ports except source
        for port in all_ports:
            if port != incoming_port:
                forward(frame, port)
        print(f"Flooding to all ports except Port {incoming_port}")
```

---

### Switch Advantages Over Hub

```
Hub vs Switch:

Initial message (unknown destination):
Hub: Floods to all ports
Switch: Floods to all ports (same as hub)

Subsequent messages (learned destination):
Hub: STILL floods to all ports (never learns)
Switch: Forwards ONLY to destination port ✓

Result:
- Switch reduces unnecessary traffic
- Switch improves security (only destination sees traffic)
- Switch scales better with more devices
- Switch eliminates collisions (dedicated bandwidth per port)
```

---

### What Switch CANNOT Do

**Switch operates at Layer 2 only:**

```
Switch can read:
✓ Source MAC address
✓ Destination MAC address
✓ Ethernet frame headers

Switch CANNOT read:
✗ IP addresses (Layer 3 - Network Layer)
✗ Ports (Layer 4 - Transport Layer)
✗ Application data (Layer 7)

Switch cannot crack open the IP packet!
Layer 3 is a black box to the switch.
```

**Example:**

```
Frame arriving at switch:
┌────────────────────────────────────────┐
│ Data Link Layer (Switch can see)      │
├────────────────────────────────────────┤
│ Source MAC: AA:AA:AA:AA:AA:AA  ← Readable
│ Dest MAC: DD:DD:DD:DD:DD:DD    ← Readable
│ EtherType: 0x0800               ← Readable
├────────────────────────────────────────┤
│ Network Layer Payload              │
│ ┌──────────────────────────────┐  │
│ │ Source IP: 192.168.1.10      │  │  ← UNREADABLE
│ │ Dest IP: 192.168.1.13        │  │  ← UNREADABLE
│ │ [Transport Layer data]       │  │  ← UNREADABLE
│ └──────────────────────────────┘  │
└────────────────────────────────────────┘

Switch: "I can only see MAC addresses. The rest is encrypted
         in a sense - it's in a protocol layer I don't understand."
```

---

### CAM Table Names

**This table has many names in the industry:**

```
Official names:
1. CAM Table (Content Addressable Memory Table)
   - Hardware-level name
   - Named after the memory type used

2. MAC Address Table
   - Most common name
   - Used in CCNA, textbooks, most documentation

3. Forwarding Database
   - IEEE 802.1D official term
   - Used in bridge/switch standards

4. Bridge Table
   - Historical name (switches evolved from bridges)
   - Still used in some contexts

All refer to the SAME table!
```

**Why multiple names?**

Historical reasons. Different organizations, different eras, different contexts. The functionality is identical—it maps MAC addresses to ports.

---

### Switch Summary

```
Switch = Layer 2 Device

Capabilities:
✓ Reads MAC addresses
✓ Maintains CAM table (MAC → Port mapping)
✓ Learns dynamically by observing traffic
✓ Forwards intelligently to specific ports
✓ Reduces unnecessary traffic

Limitations:
✗ Cannot read IP addresses (Layer 3)
✗ Cannot read ports (Layer 4)
✗ Cannot read application data (Layer 7)
✗ Cannot route between different networks

Behavior: Intelligent forwarding after learning
Intelligence level: Medium (smart, but limited to Layer 2)
```

---

## Part 3: Router - The Network Gateway

### What is a Router?

A router is a **Layer 3 device** that operates at the Network Layer. Routers **understand IP addresses** and can route traffic between different networks.

```
Physical appearance (home router):
┌───────────────────────────────────────┐
│  Home Router                          │
│  ┌──┬──┬──┬──┐                        │
│  │1 │2 │3 │4 │  ← LAN Ports          │
│  └──┴──┴──┴──┘                        │
│                                       │
│  [WAN]  ← WAN Port (Internet)         │
│  [WiFi antenna]                       │
└───────────────────────────────────────┘

You recognize this! Everyone has one at home.
```

---

### The Hidden Truth: Home Routers Contain TWO Devices

**What you call a "router" is actually TWO devices in one box:**

```
Your "Home Router" = Switch + Router combined

Physical device view:
┌─────────────────────────────────────────────────┐
│  Home Router (External View)                    │
│  ┌──┬──┬──┬──┐  [WAN]  [WiFi]                  │
│  │1 │2 │3 │4 │   Port   Antenna                │
│  └──┴──┴──┴──┘                                  │
└─────────────────────────────────────────────────┘

Internal components:
┌─────────────────────────────────────────────────┐
│  Inside the "Router"                            │
│                                                 │
│  ┌─────────────────┐      ┌─────────────────┐  │
│  │                 │      │                 │  │
│  │     SWITCH      │◄────►│     ROUTER      │  │
│  │  (Layer 2)      │      │   (Layer 3)     │  │
│  │                 │      │                 │  │
│  │  LAN Interface  │      │  WAN Interface  │  │
│  │  Ports 1-4      │      │  Internet Port  │  │
│  │  + WiFi         │      │                 │  │
│  └─────────────────┘      └─────────────────┘  │
│                                                 │
└─────────────────────────────────────────────────┘

Port 1, 2, 3, 4 connect to SWITCH component
Switch connects to ROUTER component
Router connects to WAN (Internet)
```

**When you plug an Ethernet cable into your "router," you're actually plugging into the SWITCH component!**

---

### Router Components Explained

#### LAN Interface

```
LAN Interface (Local Area Network side)
┌─────────────────────────────────────┐
│ LAN Interface                       │
│ IP: 192.168.1.1                     │  ← Private IP
│ MAC: AA:11:22:33:44:55              │  ← Router's LAN MAC
│ Subnet: 255.255.255.0 (/24)         │
│                                     │
│ Connected to internal switch        │
│ Serves as default gateway           │
│ Runs DHCP server                    │
└─────────────────────────────────────┘
```

#### WAN Interface

```
WAN Interface (Wide Area Network side - Internet)
┌─────────────────────────────────────┐
│ WAN Interface                       │
│ IP: 203.0.113.45                    │  ← Public IP (from ISP)
│ MAC: BB:66:77:88:99:AA              │  ← Router's WAN MAC
│                                     │
│ Connected to ISP (Internet)         │
│ Acquires IP via DHCP from ISP       │
└─────────────────────────────────────┘

Initially: No IP assigned
After connecting to ISP: Receives public IP via DHCP
```

#### Internal Switch

```
Internal Switch (Built-in)
┌─────────────────────────────────────┐
│ Switch Component                    │
│                                     │
│ Port 1: Computer A                  │
│ Port 2: Computer B                  │
│ Port 3: Computer C                  │
│ Port 4: (available)                 │
│ Port 5 (logical): Router Interface  │  ← Switch sees router
│                                     │    as another device
│ CAM Table maintained                │
└─────────────────────────────────────┘
```

**Key insight:** The switch doesn't know it's "part of" the router. It just sees the router as another connected device on a port!

---

### Home Router Scenario

```
Full topology:
                        Internet
                           │
                           │ WAN Interface
                           │ IP: 203.0.113.45 (public)
                    ┌──────┴──────┐
                    │             │
                    │   ROUTER    │  Layer 3 device
                    │             │  LAN MAC: AA:11:22:33:44:55
                    └──────┬──────┘
                           │ LAN Interface
                           │ IP: 192.168.1.1 (private)
                    ┌──────┴──────┐
                    │             │
                    │   SWITCH    │  Layer 2 device
                    │             │  (built into "router")
                    └─┬──┬──┬──┬──┘
                      │  │  │  │
              Port:   1  2  3  4
                      │  │  │  │
                      │  │  │  └─── Computer D
                      │  │  │       IP: 192.168.1.13
                      │  │  │       MAC: DD:DD:DD:DD:DD:DD
                      │  │  │
                      │  │  └─────── Computer C
                      │  │          IP: 192.168.1.12
                      │  │          MAC: CC:CC:CC:CC:CC:CC
                      │  │
                      │  └────────── Computer B
                      │             IP: 192.168.1.11
                      │             MAC: BB:BB:BB:BB:BB:BB
                      │
                      └──────────── Computer A
                                    IP: 192.168.1.10
                                    MAC: AA:AA:AA:AA:AA:AA

All computers' default gateway: 192.168.1.1 (router's LAN IP)
All computers' subnet mask: 255.255.255.0
```

---

## How Devices Decide: Same Network or Different Network?

**This is the crucial decision every computer makes before sending data.**

### The Algorithm

**Before creating the Data Link Layer frame, the sending computer calculates:**

```
Step 1: Calculate own network
Own_Network = Own_IP AND Subnet_Mask

Step 2: Calculate destination network
Dest_Network = Dest_IP AND Subnet_Mask

Step 3: Compare
IF Own_Network == Dest_Network:
    Same network!
    Send directly to destination
    Use destination's MAC address
ELSE:
    Different network!
    Send to default gateway (router)
    Use router's MAC address
```

---

### Example 1: Same Network Communication

**Computer A (192.168.1.10) sends to Computer C (192.168.1.12)**

```
Computer A's calculation:
┌─────────────────────────────────────────┐
│ Own IP: 192.168.1.10                    │
│ Subnet: 255.255.255.0                   │
│ AND operation:                           │
│   192.168.1.10                          │
│ & 255.255.255.0                         │
│ ────────────────                         │
│ = 192.168.1.0    ← Own network          │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Dest IP: 192.168.1.12                   │
│ Subnet: 255.255.255.0                   │
│ AND operation:                           │
│   192.168.1.12                          │
│ & 255.255.255.0                         │
│ ────────────────                         │
│ = 192.168.1.0    ← Dest network         │
└─────────────────────────────────────────┘

Comparison:
Own network: 192.168.1.0
Dest network: 192.168.1.0
MATCH! ✓

Decision: Same network!
Action: Send directly to Computer C
```

**Data Link Layer frame:**

```
Ethernet Frame:
┌────────────────────────────────────────┐
│ Source MAC: AA:AA:AA:AA:AA:AA          │  ← Computer A
│ Dest MAC: CC:CC:CC:CC:CC:CC            │  ← Computer C directly!
│ EtherType: 0x0800                      │
├────────────────────────────────────────┤
│ IP Packet:                             │
│   Source IP: 192.168.1.10              │
│   Dest IP: 192.168.1.12                │
│   [Rest of packet]                     │
└────────────────────────────────────────┘

Destination MAC = Computer C's MAC (not router!)
```

**Switch receives frame:**

```
Switch CAM table lookup:
Source MAC: AA:AA:AA:AA:AA:AA → Learn Port 1
Dest MAC: CC:CC:CC:CC:CC:CC → Check table

If CC:CC:... in table (Port 3): Forward to Port 3 only
If CC:CC:... not in table: Flood to all ports

Router's port: NOT included in this communication!
Router doesn't participate!
```

---

### Example 2: Different Network Communication

**Computer A (192.168.1.10) sends to Internet (8.8.8.8 - Google DNS)**

```
Computer A's calculation:
┌─────────────────────────────────────────┐
│ Own IP: 192.168.1.10                    │
│ Subnet: 255.255.255.0                   │
│ AND operation:                           │
│   192.168.1.10                          │
│ & 255.255.255.0                         │
│ ────────────────                         │
│ = 192.168.1.0    ← Own network          │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Dest IP: 8.8.8.8                        │
│ Subnet: 255.255.255.0                   │
│ AND operation:                           │
│   8.8.8.8                               │
│ & 255.255.255.0                         │
│ ────────────────                         │
│ = 8.8.8.0        ← Dest network         │
└─────────────────────────────────────────┘

Comparison:
Own network: 192.168.1.0
Dest network: 8.8.8.0
NO MATCH! ✗

Decision: Different network!
Action: Send to default gateway (router)
```

**Data Link Layer frame:**

```
Ethernet Frame:
┌────────────────────────────────────────┐
│ Source MAC: AA:AA:AA:AA:AA:AA          │  ← Computer A
│ Dest MAC: AA:11:22:33:44:55            │  ← ROUTER's MAC!
│ EtherType: 0x0800                      │
├────────────────────────────────────────┤
│ IP Packet:                             │
│   Source IP: 192.168.1.10              │  ← Still Computer A
│   Dest IP: 8.8.8.8                     │  ← Still Internet
│   [Rest of packet]                     │
└────────────────────────────────────────┘

Layer 2 destination: Router's MAC!
Layer 3 destination: Still 8.8.8.8!

This is critical:
- MAC address = next hop (router)
- IP address = final destination (Internet)
```

**Switch receives frame:**

```
Switch CAM table lookup:
Source MAC: AA:AA:AA:AA:AA:AA → Learn Port 1
Dest MAC: AA:11:22:33:44:55 → Check table

AA:11:22... is router's MAC!
Router's MAC mapped to Port 5 (internal)

Action: Forward to Port 5 (router's interface)

This is how traffic reaches the router!
```

**Router receives frame:**

```
Router processing (Layer 3 device):
1. Data Link Layer: Check Dest MAC
   Dest MAC: AA:11:22:33:44:55 (my MAC!)
   ✓ Accept frame
   
2. Network Layer: Check Dest IP
   Dest IP: 8.8.8.8
   My IP: 192.168.1.1 (LAN), 203.0.113.45 (WAN)
   ✗ Not for me - must ROUTE to Internet
   
3. Routing decision:
   - Look up 8.8.8.8 in routing table
   - Next hop: ISP gateway
   - Outgoing interface: WAN
   
4. Create new frame for WAN side:
   Source MAC: BB:66:77:88:99:AA (my WAN MAC)
   Dest MAC: [ISP gateway MAC]
   [Same IP packet inside]
   
5. Forward to Internet via WAN port
```

**Router changes MAC addresses but keeps IP addresses unchanged (NAT aside)!**

---

### The Subnet Mask AND Operation Explained

**Why does this work?**

```
Example IP: 192.168.1.10
Binary:     11000000.10101000.00000001.00001010

Subnet: 255.255.255.0
Binary: 11111111.11111111.11111111.00000000
        └────────────┬─────────────┘└───┬────┘
           Network portion         Host portion

AND operation (bit-by-bit):
1 AND 1 = 1
1 AND 0 = 0
0 AND 1 = 0
0 AND 0 = 0

Result:
  11000000.10101000.00000001.00001010  (192.168.1.10)
& 11111111.11111111.11111111.00000000  (255.255.255.0)
  ──────────────────────────────────
= 11000000.10101000.00000001.00000000  (192.168.1.0)

The host bits (last octet) become 0
Only network bits remain!
```

**Different IPs, same network:**

```
192.168.1.10 & 255.255.255.0 = 192.168.1.0
192.168.1.11 & 255.255.255.0 = 192.168.1.0
192.168.1.12 & 255.255.255.0 = 192.168.1.0
192.168.1.13 & 255.255.255.0 = 192.168.1.0

All produce 192.168.1.0 → Same network!
```

**Different IPs, different networks:**

```
192.168.1.10 & 255.255.255.0 = 192.168.1.0
8.8.8.8      & 255.255.255.0 = 8.8.8.0

Different results → Different networks!
Must route through gateway!
```

---

## Complete Communication Flow: Same Network

**Scenario:** Computer A sends "Hello" to Computer C (both on 192.168.1.0/24 network)

```
Step 1: Computer A creates message
Application: "Hello"
Transport: 12345 → 80
Network: 192.168.1.10 → 192.168.1.12
        ↓
Decision: Check if same network
Own: 192.168.1.10 & 255.255.255.0 = 192.168.1.0
Dest: 192.168.1.12 & 255.255.255.0 = 192.168.1.0
MATCH! Same network!
        ↓
Data Link: AA:AA:... → CC:CC:... (direct!)
Physical: Electrical signals → Switch Port 1

Step 2: Switch receives on Port 1
Learn: AA:AA:... → Port 1 (update CAM table)
Destination: CC:CC:...
CAM lookup: CC:CC:... → Port 3
Forward: Only to Port 3

Step 3: Computer C receives on Port 3
Check MAC: CC:CC:... (mine!) ✓
Check IP: 192.168.1.12 (mine!) ✓
Check Port: 80 (HTTP server) ✓
Pass to application: Receive "Hello"

Router was never involved!
Internal network traffic stays internal!
```

---

## Complete Communication Flow: Different Network

**Scenario:** Computer A sends HTTP request to 8.8.8.8 (Google DNS, Internet)

```
Step 1: Computer A creates message
Application: HTTP GET request
Transport: 45678 → 80
Network: 192.168.1.10 → 8.8.8.8
        ↓
Decision: Check if same network
Own: 192.168.1.10 & 255.255.255.0 = 192.168.1.0
Dest: 8.8.8.8 & 255.255.255.0 = 8.8.8.0
NO MATCH! Different network!
        ↓
Data Link: AA:AA:... → AA:11:22:... (router MAC!)
Physical: Electrical signals → Switch Port 1

Step 2: Switch receives on Port 1
Learn: AA:AA:... → Port 1
Destination: AA:11:22:... (router MAC)
CAM lookup: AA:11:22:... → Port 5 (internal router port)
Forward: Only to Port 5 (to router)

Step 3: Router receives
Data Link: Check MAC: AA:11:22:... (mine!) ✓
Network: Check IP: 8.8.8.8 (not mine)
Routing decision: Look up 8.8.8.8 in routing table
Next hop: ISP gateway (via WAN interface)
Create new frame:
  Source MAC: BB:66:77:... (WAN MAC)
  Dest MAC: [ISP gateway MAC]
  [Same IP packet inside: 192.168.1.10 → 8.8.8.8]
Forward to WAN port → Internet

Router participated because destination was outside local network!
```

---

## Why This Design Matters

### Efficiency

```
Same network traffic:
Hub: All 4 computers receive every message (wasteful)
Switch: Only sender and receiver communicate (efficient)
Router: Not involved at all (most efficient)

Different network traffic:
Only router processes and forwards
Local traffic stays local
Internet-bound traffic goes through router
```

### Security

```
Same network:
Computers can eavesdrop only if switch floods (first packet)
After learning, only destination receives traffic

Different network:
Router acts as gateway/firewall
Can inspect, block, or allow traffic
Provides NAT (Network Address Translation)
Hides internal network from Internet
```

### Scalability

```
Hubs: Max 8 devices (collision domain limitations)
Switches: 12-48 devices typical, 96+ possible
Routers: Connect multiple networks, unlimited scale
```

---

## Device Comparison Table

```
┌──────────────┬──────────┬─────────┬─────────┐
│ Feature      │   Hub    │ Switch  │ Router  │
├──────────────┼──────────┼─────────┼─────────┤
│ OSI Layer    │    L1    │   L2    │   L3    │
├──────────────┼──────────┼─────────┼─────────┤
│ Reads MAC    │    ✗     │   ✓     │   ✓     │
│ Reads IP     │    ✗     │   ✗     │   ✓     │
│ Reads Ports  │    ✗     │   ✗     │   ✓     │
├──────────────┼──────────┼─────────┼─────────┤
│ Intelligence │  None    │ Medium  │  High   │
├──────────────┼──────────┼─────────┼─────────┤
│ Forwarding   │  Flood   │ Learned │ Routing │
│              │  always  │ table   │ table   │
├──────────────┼──────────┼─────────┼─────────┤
│ Typical Use  │ Obsolete │ LAN     │ Gateway │
├──────────────┼──────────┼─────────┼─────────┤
│ Connects     │ Devices  │ Devices │ Networks│
│              │ (poorly) │ (LAN)   │ (WAN)   │
└──────────────┴──────────┴─────────┴─────────┘
```

---

## Key Takeaways

### Hub
- Layer 1 device (Physical Layer)
- Blind flooding to all ports
- No intelligence, no learning
- Obsolete technology
- Cannot read anything except electrical signals

### Switch
- Layer 2 device (Data Link Layer)
- Reads MAC addresses
- Maintains CAM table (MAC → Port mapping)
- Learns dynamically
- Forwards intelligently to specific ports
- Cannot read IP addresses or higher-layer data

### Router
- Layer 3 device (Network Layer)
- Reads IP addresses
- Routes between different networks
- Home "routers" contain BOTH switch + router
- Makes forwarding decisions based on IP, not MAC

### Forwarding Decision Logic

**Computer decides before sending:**
```
IF (Source_IP & Subnet_Mask) == (Dest_IP & Subnet_Mask):
    Same network
    Use dest_MAC = destination computer's MAC
ELSE:
    Different network
    Use dest_MAC = router's MAC (default gateway)
```

**The destination IP never changes, but destination MAC changes based on whether routing is needed!**

---

## Troubleshooting Common Issues

### Problem 1: Can't Reach Other Computer on Same Network

**Symptoms:**
- Computer A can't ping Computer B
- Both on same network (192.168.1.0/24)
- Router seems fine

**Diagnosis:**

```bash
# Check if switch has learned MACs
# (Requires switch CLI access - not always possible on home switches)

# On Computer A:
ping 192.168.1.11  # Computer B's IP
# Fails

# Check ARP table
arp -a
# No entry for 192.168.1.11

# Try forcing ARP
arping -c 5 192.168.1.11
```

**Possible causes:**
1. **Switch port failure:** Physical connection issue
2. **CAM table full:** Switch table overflow (rare on modern switches)
3. **VLAN mismatch:** Computers on different VLANs (advanced topic)
4. **Firewall on Computer B:** Blocking ICMP (ping)

**Solutions:**
- Check cable connections
- Restart switch
- Verify subnet mask matches (should be 255.255.255.0)
- Disable firewall temporarily for testing

---

### Problem 2: Computer Sends to Router MAC Instead of Direct

**Symptoms:**
- Computer A sends to Computer C
- Both on 192.168.1.0/24
- But packets go through router unnecessarily

**Diagnosis:**

```bash
# On Computer A:
ip route get 192.168.1.12
# Shows: via 192.168.1.1 (WRONG! Should be direct)

# Check subnet mask
ip addr show
# eth0: inet 192.168.1.10/32  ← PROBLEM! /32 instead of /24
```

**Cause:** Incorrect subnet mask

```
Subnet mask: 255.255.255.255 (/32)
Means: Only 192.168.1.10 is "local", everything else is "remote"

Result:
192.168.1.10 & 255.255.255.255 = 192.168.1.10
192.168.1.12 & 255.255.255.255 = 192.168.1.12
NO MATCH → Sends to router

Should be:
Subnet mask: 255.255.255.0 (/24)
192.168.1.10 & 255.255.255.0 = 192.168.1.0
192.168.1.12 & 255.255.255.0 = 192.168.1.0
MATCH → Sends directly
```

**Solution:**

```bash
# Linux:
sudo ip addr add 192.168.1.10/24 dev eth0

# Windows:
netsh interface ip set address "Ethernet" static 192.168.1.10 255.255.255.0 192.168.1.1
```

---

### Problem 3: Switch Flooding Too Much

**Symptoms:**
- Network slow
- All computers receiving traffic not meant for them
- Like hub behavior

**Diagnosis:**

```
Cause: CAM table not learning (or aging out too quickly)

Possible reasons:
1. Rapidly changing MAC addresses (MAC spoofing attack)
2. CAM table aging time too short
3. Switch memory issue
4. Switch overload
```

**Solution:**
- Restart switch
- Check for MAC spoofing attacks
- Upgrade switch firmware
- Replace with higher-capacity switch

---

### Problem 4: Router Not Forwarding to Internet

**Symptoms:**
- Can ping other computers on LAN (192.168.1.0/24)
- Cannot ping Internet (8.8.8.8)

**Diagnosis:**

```bash
# Check default gateway configuration
ip route show default
# default via 192.168.1.1 dev eth0

# Ping gateway
ping 192.168.1.1
# Success (can reach router)

# Ping Internet
ping 8.8.8.8
# Fails

# Check router's WAN interface
# (Access router admin page: http://192.168.1.1)
```

**Possible causes:**
1. **Router WAN interface down:** No Internet connection
2. **ISP issue:** Modem offline
3. **Router not configured:** WAN IP not assigned
4. **Routing table missing:** Default route to ISP not configured

**Solution:**
- Check ISP connection
- Restart modem and router
- Verify WAN interface has public IP
- Check router logs for errors

---

## Advanced Topics Preview

### VLANs (Virtual LANs)

**What you'll learn later:**

```
One physical switch can create multiple logical networks:

VLAN 10: Engineering (192.168.10.0/24)
VLAN 20: Sales (192.168.20.0/24)

Physical Port 1 → VLAN 10
Physical Port 2 → VLAN 10
Physical Port 3 → VLAN 20
Physical Port 4 → VLAN 20

Ports in different VLANs cannot communicate
(even though same physical switch!)

Requires router for inter-VLAN communication
```

### Layer 3 Switches

```
Hybrid device: Switch + Router combined

Can perform:
- Layer 2 switching (MAC forwarding)
- Layer 3 routing (IP routing)

Faster than separate switch + router
Used in enterprise networks
```

### Spanning Tree Protocol (STP)

```
Problem: Switch loops cause broadcast storms
Solution: STP disables redundant paths

Prevents:
- Infinite packet loops
- CAM table thrashing
- Network meltdown

Creates loop-free topology
```

---

## Conclusion

You now understand the three fundamental network devices:

**Hub:** The obsolete blind repeater that floods everything everywhere. Layer 1 device. No intelligence. Never learns. Wastes bandwidth.

**Switch:** The intelligent Layer 2 forwarder. Reads MAC addresses. Maintains CAM table. Learns dynamically. Forwards only to destination port after learning. Cannot read IP addresses.

**Router:** The Layer 3 gateway between networks. Reads IP addresses. Routes traffic between different networks. Your home "router" is actually a switch + router combined. Devices decide whether to send to router or directly based on subnet mask AND operation.

**The critical insight:** Your computer performs the AND operation **before** creating the Data Link Layer frame. The subnet mask determines whether the destination MAC will be the actual destination's MAC (same network) or the router's MAC (different network). The IP header's destination IP never changes—only the Ethernet frame's destination MAC changes based on routing needs.

Next, you'll learn how these devices handle more complex scenarios: multiple routers, NAT, port forwarding, firewalls, and how Docker networking leverages these concepts to create container networks.

**This is where networking theory becomes networking reality.**

---

## Further Reading

- **IEEE 802.3:** Ethernet standard (hub/switch behavior)
- **IEEE 802.1D:** Spanning Tree Protocol (switch loops)
- **"Computer Networks" by Andrew S. Tanenbaum:** Bridges, switches, routers chapter
- **RFC 826:** ARP (Address Resolution Protocol) - how computers learn MACs
- **CCNA Study Guide:** Switch CAM table, router fundamentals
- **Cisco Switch Configuration Guide:** CAM table management, port configuration
- **"TCP/IP Illustrated, Volume 1":** Routing fundamentals chapter
- **Wireshark tutorials:** Capturing and analyzing switch/router traffic
- **"Network Warrior" by Gary A. Donahue:** Practical switch and router configuration
