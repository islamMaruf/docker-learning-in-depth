# Chapter 034: HTTP/2 In Details

## Overview

HTTP/2, standardized in 2015 (RFC 7540), represents the first major revision of HTTP in nearly 20 years since HTTP/1.1 was introduced in 1997-1998. While HTTP/1.1 achieved a stunning 90% reduction in connection overhead compared to HTTP/1.0, it carried an inherent architectural limitation that became increasingly problematic as web pages evolved to load 100+ resources: **head-of-line (HOL) blocking**.

HTTP/2 was designed with a single, focused mission: **eliminate HOL blocking while maintaining backward compatibility with HTTP/1.1 semantics**. It achieves this through a revolutionary approach called **multiplexing**—the ability to send multiple HTTP requests and responses in parallel over a single TCP connection without any ordering constraints.

This chapter provides a complete technical exploration of HTTP/2, explaining the fundamental innovation of binary framing and streams, how multiplexing works at the protocol level, the stream lifecycle, header compression (HPACK), server push capabilities, and the trade-offs that led to HTTP/3. Understanding HTTP/2 deeply is essential because it powers the modern web—every major browser and web server supports it, and protocols like gRPC are built directly on top of it.

HTTP/2 is not a new application protocol; it's a new **transport encoding** for HTTP. All the semantics you know—GET, POST, headers, status codes—remain identical. What changed is *how* this information flows over the network.

---

## The Problem HTTP/2 Solves: A Recap

### HTTP/1.0's Limitation

**Problem:** One TCP connection per request.

```
Request A → [TCP Handshake] → Send → Receive → [TCP Teardown]
Request B → [TCP Handshake] → Send → Receive → [TCP Teardown]
Request C → [TCP Handshake] → Send → Receive → [TCP Teardown]
```

**Cost:** For 100 resources, 100 TCP handshakes + 100 TCP teardowns = massive overhead.

### HTTP/1.1's Improvement

**Solution:** Persistent connections (keep-alive).

```
[TCP Handshake once]
Request A → Send → Receive
Request B → Send → Receive
Request C → Send → Receive
[TCP Teardown once]
```

**Achievement:** 90% reduction in connection overhead.

### HTTP/1.1's New Problem

**Issue:** Responses must be delivered in order (HOL blocking).

```
Request A (slow, 10s)  →  Must wait
Request B (fast, 0.1s) →  Ready but blocked behind A
Request C (fast, 0.1s) →  Ready but blocked behind A

Timeline:
t=0s:    Send A, B, C
t=0.1s:  B and C ready on server (waiting)
t=10s:   A finally ready
t=10s:   Send A
t=10.1s: Send B
t=10.2s: Send C
```

Even though B and C were ready at 0.1s, the user waited 10+ seconds.

### Browser Workaround: Connection Pooling

Browsers compensated by opening **6-8 parallel TCP connections** per domain:

```
Connection 1:  A
Connection 2:  B → Fast, arrives at 0.1s
Connection 3:  C → Fast, arrives at 0.1s
Connection 4:  D
Connection 5:  E
Connection 6:  F
```

**Problem with this workaround:**
- More memory (each TCP connection = ~100KB)
- More CPU (context switches, timers)
- Bandwidth competition (6 TCP congestion controls competing)
- Still limited to 6 parallel requests at a time

### HTTP/2's Revolutionary Solution

**Concept:** One TCP connection, **unlimited parallel streams**.

```
[Single TCP Connection]
├── Stream 1: Request A (slow)  → Chunks: [A1][A2][A3]
├── Stream 2: Request B (fast)  → Chunks: [B1][B2]
├── Stream 3: Request C (fast)  → Chunks: [C1]
└── Stream 4: Request D (fast)  → Chunks: [D1][D2]

Transmission over TCP (interleaved):
[A1][B1][C1][D1][A2][D2][B2][A3]

Client receives and reconstructs:
Stream 1: A1 + A2 + A3 = Complete response A
Stream 2: B1 + B2 = Complete response B (arrives before A completes!)
Stream 3: C1 = Complete response C (arrives before A completes!)
Stream 4: D1 + D2 = Complete response D (arrives before A completes!)
```

Now B and C can arrive independently of A's completion. **No HOL blocking at the HTTP level.**

---

## Technical Foundation: Binary Framing Layer

HTTP/2's core innovation is the **binary framing layer**—a new encoding mechanism that sits between the HTTP semantics (Layer 7) and the TCP transport (Layer 4).

### Layer Architecture

```
┌─────────────────────────────────────────┐
│  Application Layer (L7)                 │
│  HTTP Semantics: GET, POST, headers     │  ← Unchanged
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│  HTTP/2 Binary Framing Layer            │  ← NEW
│  Streams, Frames, Multiplexing          │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│  Transport Layer (L4)                   │
│  TCP: Reliable, ordered byte stream     │  ← Unchanged
└─────────────────────────────────────────┘
```

HTTP/1.1 sent plain-text messages directly over TCP:
```
GET /index.html HTTP/1.1\r\n
Host: example.com\r\n
\r\n
```

HTTP/2 converts HTTP messages into **binary frames** before sending:
```
+-----------------------------------------------+
|                 Frame Header                  |
+-----------------------------------------------+
|                 Frame Payload                 |
+-----------------------------------------------+
```

### Key Concepts

**Stream:**
A bidirectional flow of frames within a single TCP connection. Each HTTP request/response pair uses one stream.

**Frame:**
The smallest unit of communication in HTTP/2. Each frame has a type (HEADERS, DATA, SETTINGS, etc.) and belongs to a specific stream.

**Message:**
A complete HTTP request or response, consisting of one or more frames.

**Connection:**
The TCP connection that carries all streams.

### Relationship Diagram

```
Connection (TCP)
    ├── Stream 1 (Request/Response for /index.html)
    │   ├── HEADERS Frame (request headers)
    │   ├── DATA Frame (request body, if POST)
    │   ├── HEADERS Frame (response headers)
    │   └── DATA Frame (response body)
    │
    ├── Stream 3 (Request/Response for /style.css)
    │   ├── HEADERS Frame
    │   └── DATA Frame
    │
    └── Stream 5 (Request/Response for /script.js)
        ├── HEADERS Frame
        └── DATA Frame (chunk 1)
        └── DATA Frame (chunk 2)
        └── DATA Frame (chunk 3, END_STREAM flag)
```

---

## Stream Mechanics and Multiplexing

### Stream Identifiers

Streams are identified by **31-bit unsigned integers**.

**Allocation Rules:**
- **Client-initiated streams:** Use odd numbers (1, 3, 5, 7, ...)
- **Server-initiated streams:** Use even numbers (2, 4, 6, 8, ...)
- **Stream ID 0:** Reserved for connection-level control (SETTINGS, WINDOW_UPDATE, etc.)

**Why odd/even distinction?**
This prevents collision between client-initiated requests and server-initiated pushes. Both parties can allocate stream IDs independently without coordination.

### Stream States

Each stream progresses through a defined lifecycle:

```
                        idle
                          |
                          v
            +-------------+--------------+
            |                            |
       HEADERS sent               HEADERS received
            |                            |
            v                            v
          open ←----------------------→ open
            |                            |
       END_STREAM sent            END_STREAM received
            |                            |
            v                            v
      half-closed                  half-closed
       (local)                       (remote)
            |                            |
       END_STREAM received        END_STREAM sent
            |                            |
            +-------------+--------------+
                          |
                          v
                       closed
```

**State Descriptions:**

- **idle:** Stream ID exists but hasn't been used yet
- **open:** Both sides can send frames
- **half-closed (local):** Local endpoint sent END_STREAM, can only receive
- **half-closed (remote):** Remote endpoint sent END_STREAM, can only send
- **closed:** Stream complete, ID can be reused after a period

### Example: Complete Request/Response Lifecycle

**Scenario:** Client requests `/data.json` (Stream ID 1)

```
Time  | Client (Stream 1)              | Server (Stream 1)
------|--------------------------------|----------------------------------
t=0   | HEADERS frame                  |
      |   :method: GET                 |
      |   :path: /data.json            |
      |   :scheme: https               |
      |   :authority: api.example.com  |
      |   END_HEADERS flag set         |
      | State: open → half-closed      | State: idle → open
------|--------------------------------|----------------------------------
t=1   |                                | HEADERS frame
      |                                |   :status: 200
      |                                |   content-type: application/json
      |                                | State: open
------|--------------------------------|----------------------------------
t=2   |                                | DATA frame (chunk 1)
      |                                |   {\"users\": [
      |                                | State: open
------|--------------------------------|----------------------------------
t=3   |                                | DATA frame (chunk 2)
      |                                |   {\"id\": 1, \"name\": \"Alice\"},
      |                                | State: open
------|--------------------------------|----------------------------------
t=4   |                                | DATA frame (chunk 3, END_STREAM)
      |                                |   {\"id\": 2, \"name\": \"Bob\"}]}
      | State: half-closed → closed    | State: open → closed
```

### Multiplexing in Action: Four Concurrent Requests

Let's walk through a detailed example with timing.

**Setup:**
- Client requests 4 resources simultaneously
- Single TCP connection
- All requests sent immediately (no waiting)

**Resources:**
- A: `/index.html` (10KB, server processing: 10s)
- B: `/style.css` (4KB, server processing: 0.1s)
- C: `/script.js` (1KB, server processing: 0.1s)
- D: `/logo.png` (3KB, server processing: 0.1s)

**HTTP/2 Stream Assignment:**
- Stream 1: `/index.html`
- Stream 3: `/style.css`
- Stream 5: `/script.js`
- Stream 7: `/logo.png`

**Frame Transmission Timeline:**

```
t=0.000s  Client → Server
          [HEADERS:1 /index.html]  (Stream 1)
          [HEADERS:3 /style.css]   (Stream 3)
          [HEADERS:5 /script.js]   (Stream 5)
          [HEADERS:7 /logo.png]    (Stream 7)

t=0.100s  Server → Client
          [HEADERS:3 200 OK]       (Stream 3 response ready)
          [DATA:3 chunk1]
          [DATA:3 chunk2 END_STREAM]
          
          [HEADERS:5 200 OK]       (Stream 5 response ready)
          [DATA:5 chunk1 END_STREAM]
          
          [HEADERS:7 200 OK]       (Stream 7 response ready)
          [DATA:7 chunk1]
          [DATA:7 chunk2 END_STREAM]

t=0.150s  Client has complete responses for streams 3, 5, 7
          User sees CSS styling, JS executes, logo displays
          Still waiting for stream 1 (HTML)...

t=10.000s Server → Client
          [HEADERS:1 200 OK]       (Stream 1 finally ready)
          [DATA:1 chunk1]
          [DATA:1 chunk2]
          [DATA:1 chunk3 END_STREAM]

t=10.050s Client has complete response for stream 1
          Full page rendering complete
```

**Key Observation:**

In HTTP/1.1, the user would stare at a blank screen for 10 seconds, then everything would appear at once. In HTTP/2, the CSS, JavaScript, and images arrive at 0.15s, providing **progressive rendering** while the slow HTML is still processing.

### Data Chunking: How Large Payloads Are Split

HTTP/2 doesn't send entire responses in one DATA frame. Instead, large payloads are divided into **chunks** (frames).

**Example:** 10KB response split into frames

```
Original Request Data:
┌────────────────────────────────────────────────────┐
│  Request Body: 10KB (for POST request)             │
└────────────────────────────────────────────────────┘

HTTP/2 Chunking (assume 4KB frames):
┌────────────┐  ┌────────────┐  ┌────────────┐
│ Chunk 1    │  │ Chunk 2    │  │ Chunk 3    │
│ 4KB        │  │ 4KB        │  │ 2KB        │
│ Frame: 1   │  │ Frame: 1   │  │ Frame: 1   │
│ Data       │  │ Data       │  │ Data       │
│            │  │            │  │ END_STREAM │
└────────────┘  └────────────┘  └────────────┘

Transmission:
[HEADERS:1]
[DATA:1][DATA:1][DATA:1 END]
```

**Why chunk?**

1. **Flow Control:** Prevents one stream from monopolizing bandwidth
2. **Fairness:** Different streams can interleave frames
3. **Memory Efficiency:** Don't need to buffer entire response before sending
4. **Cancellation:** Can abort mid-transfer (RST_STREAM frame)

### Interleaving: The Core of Multiplexing

This is where HTTP/2's power becomes evident.

**Scenario:** Streams 1, 2, 3 are all active simultaneously.

**Stream Data (simplified):**
- Stream 1: [A1][A2][A3][A4][A5]
- Stream 2: [B1][B2]
- Stream 3: [C1][C2][C3]

**Actual wire transmission (interleaved):**
```
[HEADERS:1][HEADERS:2][HEADERS:3]
[A1][B1][C1][A2][B2][C2][A3][C3][A4][A5]
```

Notice how Stream 2 completed (only 2 DATA frames) while Stream 1 was still sending. The server didn't have to wait—it could send B's response as soon as B was ready, regardless of A's state.

**Client Reassembly:**

The client receives this interleaved stream of frames and reconstructs each response:

```
Stream 1 buffer: A1 → A1+A2 → A1+A2+A3 → A1+A2+A3+A4 → A1+A2+A3+A4+A5 (complete)
Stream 2 buffer: B1 → B1+B2 (complete)
Stream 3 buffer: C1 → C1+C2 → C1+C2+C3 (complete)
```

Each stream's frames carry a **stream identifier** in the frame header, allowing the receiver to route frames to the correct stream buffer.

---

## Frame Structure: The Binary Protocol

HTTP/2 is a **binary protocol**, not text-based like HTTP/1.1. This improves parsing efficiency and reduces ambiguity.

### Frame Format

Every HTTP/2 frame has a 9-byte header followed by a payload:

```
+-----------------------------------------------+
|                 Length (24)                   |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                      |
+=+=============================================================+
|                   Frame Payload (0...)                      ...
+---------------------------------------------------------------+
```

**Field Descriptions:**

1. **Length (24 bits):**
   - Size of frame payload (not including 9-byte header)
   - Maximum: 2²⁴ - 1 = 16,777,215 bytes (~16 MB)
   - Default max (SETTINGS_MAX_FRAME_SIZE): 16,384 bytes (16 KB)

2. **Type (8 bits):**
   - Identifies frame type (HEADERS, DATA, SETTINGS, etc.)
   - Valid values: 0x0 - 0xA (and extensible)

3. **Flags (8 bits):**
   - Frame-type-specific boolean flags
   - Example: END_STREAM (0x1), END_HEADERS (0x4), PADDED (0x8)

4. **R (1 bit):**
   - Reserved bit, must be 0

5. **Stream Identifier (31 bits):**
   - Which stream this frame belongs to
   - 0 = connection-level control frame
   - 1-2³¹ = stream-specific frames

6. **Frame Payload:**
   - Variable length (0 to Length bytes)
   - Contents depend on frame type

### Frame Types

HTTP/2 defines 10 core frame types:

| Type | ID | Purpose |
|------|-----|---------|
| **DATA** | 0x0 | Carries HTTP request/response body |
| **HEADERS** | 0x1 | Carries HTTP request/response headers |
| **PRIORITY** | 0x2 | Specifies stream priority (deprecated in practice) |
| **RST_STREAM** | 0x3 | Immediately terminates a stream (error/cancel) |
| **SETTINGS** | 0x4 | Connection-level configuration parameters |
| **PUSH_PROMISE** | 0x5 | Server announces intent to push a resource |
| **PING** | 0x6 | Connection liveness check (like TCP keepalive) |
| **GOAWAY** | 0x7 | Graceful connection shutdown notification |
| **WINDOW_UPDATE** | 0x8 | Flow control: advertises available buffer space |
| **CONTINUATION** | 0x9 | Continues a sequence of HEADERS frames |

### HEADERS Frame (Detailed)

This is the most important frame type for understanding HTTP/2.

**Purpose:** Carry HTTP request/response headers.

**Structure:**
```
+---------------+
|Pad Length? (8)|  (if PADDED flag set)
+-+-------------+-----------------------------------------------+
|E|                 Stream Dependency? (31)                     |  (if PRIORITY flag set)
+-+-------------+-----------------------------------------------+
|  Weight? (8)  |                                               |  (if PRIORITY flag set)
+-+-------------+-----------------------------------------------+
|                   Header Block Fragment (*)                 ...
+---------------------------------------------------------------+
|                           Padding (*)                       ...
+---------------------------------------------------------------+
```

**Flags:**
- **END_STREAM (0x1):** No more frames for this stream (e.g., GET request with no body)
- **END_HEADERS (0x4):** Last HEADERS frame (if too large, use CONTINUATION frames)
- **PADDED (0x8):** Frame contains padding for security (prevents traffic analysis)
- **PRIORITY (0x20):** Contains stream priority information

**Example:** HTTP GET request

```
Frame Header:
+-----------------------------------------------+
| Length: 0x0032 (50 bytes)                     |
+---------------+---------------+---------------+
| Type: 0x01    | Flags: 0x05   |               |  (HEADERS, END_STREAM | END_HEADERS)
+-+-------------+---------------+---------------+
|0| Stream ID: 0x00000001 (1)                   |
+=+==============================================+

Frame Payload (Header Block, compressed via HPACK):
0x82               → :method: GET
0x86               → :scheme: https
0x44               → :authority: (literal)
  0x0f 0x777777... → www.example.com
0x84               → :path: /
0x41               → (literal header)
  0x0a 0x6163...   → accept: text/html
```

Notice the payload is **binary-encoded and compressed** using HPACK. This is radically different from HTTP/1.1's plain text.

### DATA Frame (Detailed)

**Purpose:** Carry HTTP message body (request body for POST, response body for GET/POST).

**Structure:**
```
+---------------+
|Pad Length? (8)|  (if PADDED flag set)
+---------------+-----------------------------------------------+
|                            Data (*)                         ...
+---------------------------------------------------------------+
|                           Padding (*)                       ...
+---------------------------------------------------------------+
```

**Flags:**
- **END_STREAM (0x1):** Last DATA frame for this stream
- **PADDED (0x8):** Contains padding bytes

**Example:** Response body for JSON data

```
Frame Header:
+-----------------------------------------------+
| Length: 0x0400 (1024 bytes)                   |
+---------------+---------------+---------------+
| Type: 0x00    | Flags: 0x00   |               |  (DATA, no flags yet)
+-+-------------+---------------+---------------+
|0| Stream ID: 0x00000001 (1)                   |
+=+==============================================+

Frame Payload (actual HTTP body):
{"users": [{"id": 1, "name": "Alice"}, ... (1024 bytes total)

[... more DATA frames ...]

Final DATA Frame:
+-----------------------------------------------+
| Length: 0x0100 (256 bytes)                    |
+---------------+---------------+---------------+
| Type: 0x00    | Flags: 0x01   |               |  (DATA, END_STREAM)
+-+-------------+---------------+---------------+
|0| Stream ID: 0x00000001 (1)                   |
+=+==============================================+

Frame Payload:
{"id": 100, "name": "Zack"}]}
```

The final DATA frame has the `END_STREAM` flag set, signaling completion.

---

## Header Compression: HPACK

HTTP headers are repetitive and verbose. HTTP/1.1 sent them as plain text:

```
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 ...
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Upgrade-Insecure-Requests: 1
```

This is ~300 bytes per request. For 100 requests, that's **30 KB of redundant headers**.

HTTP/2 uses **HPACK** (Header Compression for HTTP/2, RFC 7541) to compress headers.

### HPACK Core Concepts

**1. Static Table (61 entries):**

A predefined lookup table of common HTTP headers:

| Index | Header Name          | Header Value |
|-------|----------------------|--------------|
| 1     | :authority           |              |
| 2     | :method              | GET          |
| 3     | :method              | POST         |
| 4     | :path                | /            |
| 5     | :path                | /index.html  |
| 6     | :scheme              | http         |
| 7     | :scheme              | https        |
| 8     | :status              | 200          |
| 9     | :status              | 204          |
| ...   | ...                  | ...          |
| 31    | content-type         | application/json |
| ...   | ...                  | ...          |
| 61    | www-authenticate     |              |

**2. Dynamic Table:**

A connection-specific table that learns headers during the session. When a new header appears, it's added to the dynamic table for future reference.

**Example:**
First request sends: `custom-header: my-app-v1.0`
This gets added to dynamic table at index 62.
Second request can reference: `62` (1 byte instead of 25 bytes)

**3. Huffman Encoding:**

String values are Huffman-encoded for additional compression.

### HPACK Encoding Example

**HTTP/1.1 Request:**
```
GET /resource HTTP/1.1
Host: www.example.com
```

**HTTP/2 HEADERS Frame (HPACK-encoded):**

```
:method: GET         → 0x82 (index 2 in static table)
:scheme: https       → 0x87 (index 7 in static table)
:path: /resource     → 0x04 0x09 2f7265736f75726365 (literal, Huffman)
:authority: www.example.com → 0x01 0x0f 777777772e6578616d706c652e636f6d (literal, Huffman)
```

**Size comparison:**
- HTTP/1.1: ~40 bytes (plain text)
- HTTP/2: ~25 bytes (HPACK binary)

**40% reduction** on this small example. For real-world headers with cookies, authentication tokens, and custom headers, the savings approach **70-80%**.

### Security Consideration: HPACK and CRIME

HPACK was carefully designed to prevent **CRIME** (Compression Ratio Info-leak Made Easy) attacks. CRIME exploits compression to leak sensitive data by observing compressed size variations.

**HPACK Protection:**
- Never compresses header values from different contexts together
- Uses a static dictionary that doesn't leak information
- Optional: Sensitive headers (like cookies) can be marked as non-indexed

---

## Server Push: Proactive Resource Delivery

HTTP/2 introduces an optional feature: **server push**. The server can send resources to the client *before* the client requests them.

### Motivation

**Typical HTTP/1.1 Flow:**
```
1. Client requests /index.html
2. Server sends /index.html
3. Client parses HTML, discovers <link rel="stylesheet" href="/style.css">
4. Client requests /style.css
5. Server sends /style.css
```

Total time: 2 RTT (round trips) + processing

**HTTP/2 Server Push:**
```
1. Client requests /index.html
2. Server sends /index.html
3. Server ALSO sends /style.css (proactively, without request)
```

Total time: 1 RTT + processing

The server "pushes" resources it knows the client will need, eliminating the extra round trip.

### PUSH_PROMISE Frame

Before pushing a resource, the server sends a **PUSH_PROMISE** frame:

```
+---------------+
|Pad Length? (8)|
+-+-------------+-----------------------------------------------+
|R|                  Promised Stream ID (31)                    |
+-+------------------------------------------------------------+
|                   Header Block Fragment (*)                 ...
+---------------------------------------------------------------+
|                           Padding (*)                       ...
+---------------------------------------------------------------+
```

**Purpose:** Notify the client that "I'm about to push /style.css as Stream 4."

**Client Options:**
- Accept the push (let it arrive)
- Reject the push (send RST_STREAM if already cached)

**Example Timeline:**

```
Client → Server:  HEADERS:1 GET /index.html

Server → Client:  HEADERS:1 200 OK
                  PUSH_PROMISE:1 (will push Stream 2: /style.css)
                  PUSH_PROMISE:1 (will push Stream 4: /script.js)
                  DATA:1 (body of /index.html)
                  
Server → Client:  HEADERS:2 200 OK (pushed /style.css)
                  DATA:2 (body of /style.css, END_STREAM)
                  
Server → Client:  HEADERS:4 200 OK (pushed /script.js)
                  DATA:4 (body of /script.js, END_STREAM)
```

Client now has `/index.html`, `/style.css`, and `/script.js` without requesting the latter two.

### Server Push: Trade-offs

**Benefits:**
- Eliminates request RTT for known dependencies
- Useful for critical render-path resources (CSS, JS, fonts)
- Great for resources with cache misses

**Drawbacks:**
- Server might push resources the client already has cached
- Wastes bandwidth if client would have cached hit
- Complicates client-side caching logic
- Difficult to predict what client needs

**Reality:** Server push is **rarely used** in practice (as of 2025). Most sites achieve better results with:
- HTTP/2 multiplexing (fast enough)
- Resource hints: `<link rel="preload">`
- Service workers for cache management

HTTP/3 **removed** server push entirely, replaced by `PRIORITY_UPDATE` and external push mechanisms.

---

## Flow Control and Backpressure

HTTP/2 includes sophisticated **flow control** to prevent fast senders from overwhelming slow receivers.

### Problem Without Flow Control

```
Fast Server                            Slow Client (mobile, limited RAM)
    |                                         |
    |--- DATA:1 (10MB file, send fast) ----->| (buffer overflows!)
    |                                         | Client can't process fast enough
    |--- DATA:1 (more data)              --->| CRASH or connection drop
```

### HTTP/2 Flow Control Mechanism

**Concept:** Credit-based system. Sender cannot send more than receiver has advertised available buffer space.

**WINDOW_UPDATE Frame:**

```
+-+-------------------------------------------------------------+
|R|              Window Size Increment (31)                     |
+-+-------------------------------------------------------------+
```

**Stream ID:**
- 0 → Connection-level window
- N → Stream-specific window

**Example: Stream-Level Flow Control**

```
Initial state:
  Stream 1 window size: 65,535 bytes (default)

Server → Client:  DATA:1 (32,768 bytes)
  Stream 1 window: 65,535 - 32,768 = 32,767 remaining

Server → Client:  DATA:1 (32,767 bytes)
  Stream 1 window: 32,767 - 32,767 = 0 remaining
  [Server must stop sending on Stream 1]

Client (processes data, frees buffer):
Client → Server:  WINDOW_UPDATE:1 (+32,768)
  Stream 1 window: 0 + 32,768 = 32,768

Server can now resume sending on Stream 1.
```

**Connection-Level Flow Control:**

Same mechanism, but controls total data across all streams:

```
Client → Server:  WINDOW_UPDATE:0 (+65,536)
  (All streams combined can receive 65,536 more bytes)
```

**Why both levels?**
- **Connection level:** Prevents total memory exhaustion
- **Stream level:** Fairly allocates bandwidth across streams (prevent one stream from monopolizing)

---

## Stream Prioritization (Deprecated)

HTTP/2 originally included a **stream prioritization** mechanism using dependency trees and weights.

**Concept:** Mark some streams as higher priority:

```
Stream 1 (HTML, weight=128, priority=high)
├── Stream 3 (CSS, weight=64, depends on 1)
└── Stream 5 (JS, weight=64, depends on 1)
        └── Stream 7 (image, weight=32, depends on 5)
```

This would tell the server: "Send HTML first, then CSS and JS in parallel, then images."

**Reality:** This was **rarely implemented correctly** in browsers or servers. The model was too complex and didn't match real-world needs.

**HTTP/2 RFC 7540 Errata (2022):** Formally **deprecated** stream priorities due to lack of adoption.

**HTTP/3 Replacement:** Uses **PRIORITY_UPDATE frames** with simpler "urgency" and "incremental" hints, designed based on lessons learned from HTTP/2's failed priority model.

---

## HTTP/2 vs HTTP/1.1: Complete Comparison

| Feature | HTTP/1.1 | HTTP/2 |
|---------|----------|--------|
| **Encoding** | Text-based (ASCII) | Binary frames |
| **Multiplexing** | No (sequential responses) | Yes (unlimited concurrent streams) |
| **Header Compression** | None | HPACK (30-80% reduction) |
| **Server Push** | No | Yes (rarely used) |
| **Request Prioritization** | None | Yes (deprecated due to complexity) |
| **Flow Control** | TCP-level only | Per-stream + connection-level |
| **Streams per Connection** | 1 request/response at a time | Unlimited (limited by SETTINGS) |
| **Connections per Domain** | 6-8 (browser workaround) | 1 (sufficient due to multiplexing) |
| **HOL Blocking (HTTP-level)** | Yes (responses ordered) | No (responses interleaved) |
| **HOL Blocking (TCP-level)** | Yes | Yes (still uses TCP) |
| **Connection Overhead** | ~3.5 RTT per request (HTTP/1.0) / ~1 RTT (HTTP/1.1 keep-alive) | ~1.5 RTT total for all requests |
| **Header Size (100 requests)** | ~30 KB (uncompressed) | ~6-9 KB (HPACK compressed) |
| **Browser Support** | Universal | All modern browsers (2015+) |
| **Server Support** | Universal | Nginx, Apache, Caddy, Node.js, Go, etc. |

---

## Implementation Deep Dive

### Client-Side Behavior

**HTTP/2 Connection Establishment:**

```
1. TCP Handshake (SYN, SYN-ACK, ACK)
2. TLS Handshake (if HTTPS)
   - Client Hello includes ALPN extension: ["h2"]
   - Server Hello agrees: ALPN protocol = "h2"
3. HTTP/2 Connection Preface
   Client → Server: "PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n" (24 bytes magic string)
   Client → Server: SETTINGS frame (connection parameters)
4. Server → Client: SETTINGS frame (server parameters)
5. Both send: SETTINGS ACK
6. Connection ready, can send HEADERS + DATA frames
```

**ALPN (Application-Layer Protocol Negotiation):**

HTTP/2 requires TLS's ALPN extension to negotiate the protocol. The client says "I support h2 (HTTP/2)" in the TLS handshake, and the server agrees (or falls back to http/1.1).

**HTTP/2 over plaintext (h2c):**

Possible but rare. Requires HTTP/1.1 Upgrade mechanism:

```
GET / HTTP/1.1
Host: example.com
Connection: Upgrade, HTTP2-Settings
Upgrade: h2c
HTTP2-Settings: <base64-encoded SETTINGS frame>

HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: h2c

[HTTP/2 framing begins]
```

### Sending a Request (Pseudocode)

```python
import http2  # Hypothetical HTTP/2 library

# Establish connection
conn = http2.connect("https://api.example.com")

# Send multiple requests concurrently
stream1 = conn.request(
    method="GET",
    path="/users",
    headers={
        "accept": "application/json"
    }
)

stream2 = conn.request(
    method="POST",
    path="/users",
    headers={
        "content-type": "application/json"
    },
    body='{"name": "Alice"}'
)

stream3 = conn.request(
    method="GET",
    path="/products",
    headers={
        "accept": "application/json"
    }
)

# All three requests are sent immediately, no waiting

# Receive responses (may arrive in any order)
response1 = stream1.get_response()  # Might arrive 3rd
response2 = stream2.get_response()  # Might arrive 1st
response3 = stream3.get_response()  # Might arrive 2nd

print(f"User list: {response1.body}")
print(f"Created user: {response2.body}")
print(f"Products: {response3.body}")
```

### Server-Side Implementation (Node.js Example)

```javascript
const http2 = require('http2');
const fs = require('fs');

// Create HTTP/2 server
const server = http2.createSecureServer({
    key: fs.readFileSync('server-key.pem'),
    cert: fs.readFileSync('server-cert.pem')
});

server.on('stream', (stream, headers) => {
    const method = headers[':method'];
    const path = headers[':path'];
    
    console.log(`${method} ${path} (Stream ${stream.id})`);
    
    // Different streams handled concurrently
    if (path === '/slow') {
        // Simulate slow database query
        setTimeout(() => {
            stream.respond({
                ':status': 200,
                'content-type': 'text/plain'
            });
            stream.end('Slow response after 10 seconds');
        }, 10000);
    } else if (path === '/fast') {
        // Fast response
        stream.respond({
            ':status': 200,
            'content-type': 'application/json'
        });
        stream.end(JSON.stringify({ message: 'Fast!' }));
    } else {
        stream.respond({
            ':status': 404
        });
        stream.end('Not Found');
    }
    
    // Note: /fast responds immediately even if /slow is still processing
    // This is the power of multiplexing
});

server.listen(8443, () => {
    console.log('HTTP/2 server listening on https://localhost:8443');
});
```

**Key Observation:** The server doesn't need explicit threading or async/await for multiplexing. The HTTP/2 framing layer handles interleaving automatically.

### Frame Construction (Low-Level Example)

**Creating a HEADERS frame manually:**

```python
import struct

# HEADERS frame for: GET /api/data HTTP/2
stream_id = 1

# HPACK-encoded headers (simplified, not real HPACK)
headers_payload = b'\x82'  # :method: GET (index 2 from static table)
headers_payload += b'\x87'  # :scheme: https (index 7)
headers_payload += b'\x84'  # :path: / (index 4)
# ... more encoded headers ...

length = len(headers_payload)
frame_type = 0x01  # HEADERS
flags = 0x05  # END_STREAM | END_HEADERS
stream_id = 1

# Construct 9-byte frame header
frame_header = struct.pack(
    '!IBBBI',
    length >> 8,         # Length (upper 16 bits)
    length & 0xFF,       # Length (lower 8 bits)
    frame_type,          # Type
    flags,               # Flags
    stream_id & 0x7FFFFFFF  # Stream ID (31 bits, R bit = 0)
)

# Complete frame
frame = frame_header + headers_payload

# Send over TCP connection
tcp_socket.send(frame)
```

**Receiving and parsing frames:**

```python
def read_frame(tcp_socket):
    # Read 9-byte header
    header = tcp_socket.recv(9)
    if len(header) < 9:
        raise ConnectionError("Incomplete frame header")
    
    # Parse header
    length = (header[0] << 16) | (header[1] << 8) | header[2]
    frame_type = header[3]
    flags = header[4]
    stream_id = int.from_bytes(header[5:9], 'big') & 0x7FFFFFFF
    
    # Read payload
    payload = b""
    while len(payload) < length:
        chunk = tcp_socket.recv(length - len(payload))
        if not chunk:
            raise ConnectionError("Connection closed mid-frame")
        payload += chunk
    
    return {
        'length': length,
        'type': frame_type,
        'flags': flags,
        'stream_id': stream_id,
        'payload': payload
    }

# Usage
while True:
    frame = read_frame(tcp_socket)
    if frame['type'] == 0x01:  # HEADERS
        print(f"Received HEADERS on stream {frame['stream_id']}")
        # Decode HPACK payload...
    elif frame['type'] == 0x00:  # DATA
        print(f"Received DATA on stream {frame['stream_id']}: {len(frame['payload'])} bytes")
```

---

## Real-World Performance Characteristics

### Benchmark: HTTP/1.1 vs HTTP/2

**Test Setup:**
- 100 resources (HTML, CSS, JS, 97 images)
- Each resource: 10 KB average
- Network: 100ms RTT, 10 Mbps bandwidth
- Server processing: 10ms per resource (average)

**HTTP/1.1 (6 connections):**
```
Connection 1-6 each handle ~17 resources
Each resource: 1 RTT (request/response) = 100ms
Total time per batch: 100ms
Batches needed: 17 batches
Total: 17 × 100ms = 1,700ms = 1.7 seconds
```

**HTTP/2 (1 connection, full multiplexing):**
```
All 100 requests sent immediately (pipelined)
Server processes in parallel (limited by CPU)
Responses arrive as ready (interleaved)
Total: 100ms (initial RTT) + 10ms (processing) = 110ms
```

**Result:** HTTP/2 is **15× faster** for this workload.

**Note:** This is a synthetic best-case. Real-world improvements are typically **2-3× faster** due to:
- Bandwidth limitations (multiplexing doesn't add bandwidth)
- Server processing bottlenecks
- Not all resources ready simultaneously

### CDN and HTTP/2

Major CDNs (Cloudflare, Fastly, Akamai, AWS CloudFront) all support HTTP/2:

**Benefits:**
- Reduced origin server load (fewer connections)
- Faster edge-to-client delivery (multiplexing)
- Lower bandwidth costs (header compression)

**Cloudflare Reports (2016):**
- 30% reduction in page load time (average)
- 50% reduction in connection count
- 40% reduction in bandwidth (HPACK + multiplexing efficiency)

### gRPC: HTTP/2's Killer Application

**gRPC** (Google Remote Procedure Call) is built **exclusively** on HTTP/2.

**Why gRPC Uses HTTP/2:**

1. **Bidirectional Streaming:** gRPC supports streaming RPCs (client sends stream, server responds with stream). HTTP/2's full-duplex streams enable this natively.

2. **Multiplexing:** Single connection for all RPC calls. No connection pool management.

3. **Flow Control:** Per-stream backpressure prevents overwhelming services.

4. **Binary Encoding:** gRPC uses Protocol Buffers (binary). HTTP/2's binary framing is a perfect match.

**Example gRPC Call (HTTP/2 frames):**

```
Client → Server:  HEADERS:1 (RPC: /users.UserService/GetUser)
                  DATA:1 (Protobuf: {user_id: 42}, END_STREAM)

Server → Client:  HEADERS:1 (gRPC status: OK)
                  DATA:1 (Protobuf: {id: 42, name: "Alice"}, END_STREAM)
```

This is just HTTP/2 HEADERS + DATA frames with gRPC-specific semantics in the headers.

**gRPC Streaming Example:**

```
Client → Server:  HEADERS:1 (RPC: /chat.ChatService/StreamMessages)

Server → Client:  HEADERS:1 (gRPC status: OK)
                  DATA:1 (message 1)
                  DATA:1 (message 2)
                  ...
                  DATA:1 (message 100, END_STREAM)
```

Server sends 100 messages over the same stream. HTTP/1.1 couldn't do this—it would need 100 requests or chunked encoding hacks.

---

## HTTP/2 Limitations and HTTP/3's Motivation

Despite HTTP/2's strengths, it has one critical flaw: **TCP head-of-line blocking**.

### The TCP HOL Problem

HTTP/2 eliminates HTTP-level HOL blocking, but **TCP-level HOL blocking** remains.

**Scenario: Packet Loss**

```
TCP Connection (single byte stream):
[Packet 1][Packet 2][Packet 3][Packet 4][Packet 5]

Transmission:
[Packet 1] → Arrives ✓
[Packet 2] → LOST ✗
[Packet 3] → Arrives (buffered, waiting for 2)
[Packet 4] → Arrives (buffered, waiting for 2)
[Packet 5] → Arrives (buffered, waiting for 2)

TCP must wait for Packet 2 retransmit before delivering 3, 4, 5.
```

**Impact on HTTP/2 Streams:**

```
Stream 1 data in Packet 2 (lost)
Stream 3 data in Packet 3 (arrived but blocked)
Stream 5 data in Packet 4 (arrived but blocked)
Stream 7 data in Packet 5 (arrived but blocked)

All streams are blocked by Stream 1's lost packet!
```

Even though Streams 3, 5, 7 have received all their data, TCP withholds delivery because Packet 2 is missing. This is **TCP head-of-line blocking**.

**HTTP/1.1 with 6 connections:**

If one connection loses a packet, only that connection's stream blocks. The other 5 connections continue unaffected. Counterintuitively, HTTP/1.1's multiple connections provide resilience against TCP HOL.

### When TCP HOL Matters

**Mobile Networks:**
- Packet loss rate: 1-5% (vs 0.1% on wired)
- HTTP/2 can perform **worse** than HTTP/1.1 on high-loss networks

**High-Latency Networks:**
- Satellite (500ms RTT): Retransmit takes 500ms, blocking all streams

**Congested Networks:**
- Coffee shop WiFi: Packet loss spikes to 10%+

### HTTP/3's Solution: QUIC

HTTP/3 replaces TCP with **QUIC** (Quick UDP Internet Connections):

**QUIC Characteristics:**
- Built on UDP (not TCP)
- Per-stream reliability (not per-connection)
- Stream 1 loss doesn't block Stream 3
- Integrated TLS 1.3 (1-RTT handshake)
- Connection migration (switch from WiFi to cellular seamlessly)

**HTTP/3 Benefits:**
- No TCP HOL blocking
- Faster connection establishment (0-RTT in some cases)
- Better mobile performance

**HTTP/3 Adoption (2025):**
- Supported by all major browsers
- Supported by Cloudflare, Google, Facebook, etc.
- ~30% of top 1000 websites use HTTP/3

---

## Browser Developer Tools and HTTP/2

### Inspecting HTTP/2 Traffic

**Chrome DevTools:**

1. Open DevTools (F12)
2. Network tab
3. Right-click column header → Enable "Protocol"
4. Reload page

You'll see:
```
Name           Status  Type       Protocol    Time
index.html     200     document   h2          245ms
style.css      200     stylesheet h2          12ms
script.js      200     script     h2          18ms
logo.png       200     image      h2          25ms
```

"h2" indicates HTTP/2. "http/1.1" indicates HTTP/1.1.

**Connection ID:**

Right-click → Enable "Connection ID"

```
Name           Protocol  Connection ID
index.html     h2        127
style.css      h2        127
script.js      h2        127
logo.png       h2        127
```

All using the same connection (127), demonstrating multiplexing.

**Timing Breakdown:**

Click a resource → Timing tab:

```
Queueing:           0.5ms   (waiting for available connection slot)
Stalled:            0.0ms   (HTTP/2 = no stalling!)
DNS Lookup:         0.0ms   (cached)
Initial Connection: 0.0ms   (reusing existing)
SSL:                0.0ms   (reusing existing)
Request sent:       0.2ms
Waiting (TTFB):     10.0ms  (server processing)
Content Download:   2.0ms
```

Notice "Stalled" is 0.0ms—HTTP/2 never waits for connection slots.

### Firefox Developer Tools

Similar to Chrome:
1. DevTools → Network
2. Click gear icon → Show "HTTP Version"
3. Look for "HTTP/2.0" in the Protocol column

### curl and HTTP/2

```bash
# Test HTTP/2 support
curl -I --http2 https://www.google.com

# Verbose output showing HTTP/2 frames
curl -v --http2 https://www.google.com

# Force HTTP/2 (fail if not supported)
curl --http2-prior-knowledge https://www.google.com
```

**Sample output:**
```
* ALPN, offering h2
* ALPN, offering http/1.1
* TLSv1.3 ...
* ALPN, server accepted to use h2
> GET / HTTP/2
> Host: www.google.com
> user-agent: curl/7.68.0
> accept: */*
>
< HTTP/2 200
< content-type: text/html; charset=ISO-8859-1
< date: Mon, 11 Mar 2024 10:30:00 GMT
```

---

## HTTP/2 Configuration and Tuning

### Nginx Configuration

```nginx
server {
    listen 443 ssl http2;  # Enable HTTP/2
    server_name example.com;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    # HTTP/2 specific settings
    http2_max_concurrent_streams 128;   # Max parallel streams (default: 128)
    http2_max_field_size 16k;           # Max header field size
    http2_max_header_size 32k;          # Max total headers size
    
    # Recommended: Enable gzip (works with HTTP/2)
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    
    location / {
        root /var/www/html;
    }
}
```

### Apache Configuration

```apache
# Enable HTTP/2 module
LoadModule http2_module modules/mod_http2.so

<VirtualHost *:443>
    ServerName example.com
    
    # Enable HTTP/2
    Protocols h2 http/1.1
    
    # HTTP/2 tuning
    H2MaxSessionStreams 100
    H2ModernTLSOnly on
    H2Push on
    
    SSLEngine on
    SSLCertificateFile /path/to/cert.pem
    SSLCertificateKeyFile /path/to/key.pem
    
    DocumentRoot /var/www/html
</VirtualHost>
```

### Performance Tuning Guidelines

**1. Increase Max Concurrent Streams:**

Default is often 100-128. For resource-heavy pages:
```
http2_max_concurrent_streams 256;
```

**2. Optimize Initial Window Size:**

Larger windows reduce round trips for large responses:
```nginx
http2_recv_timeout 30s;
http2_chunk_size 8k;
```

**3. Enable Server Push (Carefully):**

Only push critical resources:
```nginx
location = /index.html {
    http2_push /style.css;
    http2_push /script.js;
}
```

**4. Avoid Domain Sharding:**

HTTP/2 makes domain sharding counterproductive. Consolidate:
```
BAD:  cdn1.example.com, cdn2.example.com, cdn3.example.com
GOOD: cdn.example.com (single domain, HTTP/2 multiplexing)
```

**5. Monitor Connection Reuse:**

Check server logs for connection persist rates:
```
HTTP/1.1: ~6 connections per client
HTTP/2: ~1 connection per client
```

---

## Key Takeaways

### What HTTP/2 Achieved

1. **Eliminated HTTP-level HOL blocking** via multiplexing
2. **Reduced header overhead by 30-80%** via HPACK compression
3. **Single connection** replaces 6-8 connections (less memory, less CPU, less network overhead)
4. **Binary framing** enables efficient parsing and reduces ambiguity
5. **Backward compatible** with HTTP/1.1 semantics (same methods, headers, status codes)

### What HTTP/2 Couldn't Solve

1. **TCP-level HOL blocking** remains (one lost packet blocks all streams)
2. **Server push** proved impractical and is rarely used
3. **Stream prioritization** was too complex and poorly adopted (deprecated)
4. **TLS requirement** effectively mandates HTTPS (good for security, adds latency for initial handshake)

### When to Use HTTP/2

**Always prefer HTTP/2 over HTTP/1.1** for:
- Web applications (browsers fully support it)
- APIs (REST or gRPC)
- High-resource pages (100+ resources)
- Mobile applications (reduces connection overhead)
- CDN-delivered content (all major CDNs support it)

**HTTP/2 is the default** for modern web infrastructure. Unless you're maintaining legacy systems or debugging HTTP/1.1-specific issues, you should be using HTTP/2.

### The Road to HTTP/3

HTTP/2 dominated 2015-2020, but by 2022, **HTTP/3** (using QUIC over UDP) began widespread adoption:

**HTTP/3 Advantages:**
- No TCP HOL blocking (QUIC provides per-stream reliability)
- Faster handshakes (0-RTT possible with QUIC)
- Better mobile performance (connection migration)
- Simplified protocol stack (QUIC integrates transport + TLS)

**HTTP/3 Adoption (2025):**
- ~40% of top websites support it
- All major browsers support it
- Growing CDN support (Cloudflare leads adoption)

**Recommendation:** Understand HTTP/2 deeply, because:
1. HTTP/3 is HTTP/2-over-QUIC (same framing, different transport)
2. HTTP/2 is still dominant and will be for years
3. Many HTTP/2 concepts (streams, frames, multiplexing) carry forward to HTTP/3

---

## Conclusion

HTTP/2 represents a masterclass in protocol evolution. It took the fundamental semantics of HTTP—requests, responses, headers, status codes—and completely reimagined how they flow over the network. By introducing binary framing, multiplexing, and stream independence, HTTP/2 eliminated the head-of-line blocking that plagued HTTP/1.1 for nearly two decades.

The elegance of HTTP/2 lies in its **transparency**: to application developers, it's still just HTTP. You send GET requests, receive 200 OK responses, and work with familiar headers. But under the hood, the binary framing layer orchestrates an intricate dance of interleaved frames across dozens of concurrent streams over a single TCP connection.

From 2015 to 2025, HTTP/2 transformed the web:
- Page load times dropped by 30-50% on average
- Server connection counts decreased by 80-90%
- Mobile experience improved dramatically (fewer handshakes, less battery drain)
- New application models emerged (gRPC's bidirectional streaming, real-time APIs)

Yet HTTP/2's reliance on TCP ultimately revealed its Achilles' heel: TCP-level head-of-line blocking. On lossy mobile networks, HTTP/2 could perform **worse** than HTTP/1.1's multiple connections. This limitation directly motivated HTTP/3's adoption of QUIC.

Understanding HTTP/2 comprehensively—streams, frames, multiplexing, HPACK, flow control—is essential not just for working with HTTP/2 itself, but for understanding HTTP/3, gRPC, WebTransport, and the future of application protocols. The binary framing mental model HTTP/2 introduced is here to stay.

The web's evolution from HTTP/1.0's one-connection-per-request (1996) → HTTP/1.1's persistent connections (1998) → HTTP/2's multiplexing (2015) → HTTP/3's per-stream reliability (2022) tells a story of continuous innovation driven by emerging bottlenecks. Each protocol didn't fail; rather, the web's demands outgrew the constraints they were optimized for.

HTTP/2 will remain a cornerstone of web infrastructure for years to come. Master it, and you'll understand the foundation upon which the modern Internet is built.

---

## Further Reading

- **RFC 7540:** HTTP/2 Specification (2015)
- **RFC 7541:** HPACK - Header Compression for HTTP/2
- **RFC 9113:** HTTP/2 (Updated specification, 2022)
- **RFC 9114:** HTTP/3 (to understand HTTP/2's evolution)
- **High Performance Browser Networking** by Ilya Grigorik - Chapter on HTTP/2
- **gRPC Documentation:** Real-world HTTP/2 usage examples
- **HTTP/2 Explained** by Daniel Stenberg (curl author)
- **Cloudflare Blog:** HTTP/2 performance studies and real-world data
