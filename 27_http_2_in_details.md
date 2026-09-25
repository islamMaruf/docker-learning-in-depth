# Chapter 27: HTTP/2 in Detail

> **In one sentence:** HTTP/2 keeps the *meaning* of HTTP (methods, headers, status codes) but changes how messages are sent: as **binary frames** belonging to independent **streams** that are **multiplexed over one TCP connection**, with **compressed headers**, so slow responses no longer block fast ones at the HTTP level.

**Level:** 🟡 Intermediate → 🔴 Expert · **Reading time:** ~60 minutes

**Prerequisites:** [Chapter 26 – HTTP/1.1](26_http_1_1_in_details.md) (persistent connections, head-of-line blocking, browser connection pools) and [Chapter 23](23_tcp_in_details.md) (TCP loss and windows). TLS (Chapter 29) helps for the negotiation section, but isn't required.

---

## What you will learn

- The problems HTTP/2 solves, in the order they arose
- The **binary framing layer**: **frames**, **streams**, **messages**, **connections**, and the 9-byte frame header
- **Multiplexing** and interleaving, with stream IDs and the stream life cycle
- **HPACK** header compression (static table, dynamic table, Huffman) and why it is safe
- **Flow control** (per-stream and per-connection windows) and **SETTINGS**
- **Server push** and **priorities**: what they were, and why they faded
- How HTTP/2 starts up: **ALPN**, the connection preface, and `h2c`
- The **remaining problem** (TCP head-of-line blocking) and how **HTTP/3/QUIC** addresses it
- Security: **Rapid Reset** and other HTTP/2-specific attacks
- Performance reality, tuning, gRPC, and hands-on labs (`curl`, `nghttp`, `openssl`, Wireshark, nginx in Docker, Python)

---

## 1. The story so far

| Version | Fixed | Left behind |
|---|---|---|
| **HTTP/1.0** | Defined headers, status codes, content types | One TCP connection per request (Chapter 25) |
| **HTTP/1.1** | Persistent connections, `Host`, chunking, caching | Responses on a connection are **strictly ordered** → **head-of-line (HOL) blocking**; browsers open ~6 connections/host and use hacks (domain sharding, bundling) (Chapter 26) |
| **HTTP/2** (RFC 7540, 2015; revised as **RFC 9113**, 2022) | HOL blocking **at the HTTP level**, header redundancy, connection sprawl | **TCP-level** HOL blocking (one lost packet stalls *all* streams) |
| **HTTP/3** (RFC 9114, 2022) | Runs over QUIC/UDP, so loss on one stream doesn't block others | Newer, UDP handling in some networks |

HTTP/2 grew out of Google's experimental protocol **SPDY** (2009+). Importantly, **HTTP/2 doesn't change what HTTP means**: your `GET`, `POST`, `Content-Type`, `404` are identical. Application code usually doesn't need to change; the *encoding on the wire* does.

---

## 2. Big picture

```
HTTP/1.1  (text, ordered per connection)                    HTTP/2  (binary frames, multiplexed)

conn 1: GET /a ─► resp /a │ GET /d ─► resp /d              ONE connection, many concurrent streams:
conn 2: GET /b ─► resp /b │ GET /e ─► resp /e               stream 1: HEADERS(GET /a) …………… DATA DATA DATA
conn 3: GET /c ─► resp /c │ …                               stream 3: HEADERS(GET /b) …… DATA
   (≈ 6 connections, each request waits for a free one)      stream 5: HEADERS(GET /c) … DATA DATA
                                                             on the wire, interleaved:  H1 H3 H5 D3 D1 D5 D1 D5 D1 …
```

Layering:

```
┌─────────────────────────────────────────┐
│ HTTP semantics: methods, headers, status │   ← unchanged from HTTP/1.1
├─────────────────────────────────────────┤
│ HTTP/2 framing: streams, frames, HPACK   │   ← new (binary)
├─────────────────────────────────────────┤
│ TLS (in practice, always)                │
├─────────────────────────────────────────┤
│ TCP                                      │   ← unchanged: reliable, ordered byte stream
└─────────────────────────────────────────┘
```

---

## 3. The binary framing layer

HTTP/1.x messages are text lines. HTTP/2 splits every message into **frames**, each a small binary unit.

| Term | Meaning |
|---|---|
| **Connection** | One TCP (+TLS) connection between client and server |
| **Stream** | An independent, bidirectional sequence of frames on that connection; **one request/response exchange = one stream** |
| **Message** | A full HTTP request or response = one `HEADERS` frame (+ optional `CONTINUATION`s) then zero or more `DATA` frames |
| **Frame** | The smallest unit; has a type, flags, a length and a **stream identifier** |

```
Connection (TCP)
 ├─ Stream 1:  HEADERS (request) ──► ◄── HEADERS (response) ◄── DATA ◄── DATA (END_STREAM)
 ├─ Stream 3:  HEADERS (request) ──► ◄── HEADERS ◄── DATA (END_STREAM)
 └─ Stream 5:  HEADERS + DATA (POST body) ──► ◄── HEADERS ◄── DATA
```

### 3.1 The frame format
Every frame starts with a **9-byte header**:

```
+-----------------------------------------------+
|                 Length (24 bits)              |   payload size in bytes (not counting these 9 bytes)
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31 bits)                 |   R = reserved, always 0
+=+=============================================================+
|                   Frame Payload (Length bytes)               ...
+---------------------------------------------------------------+
```

- **Length:** default maximum **16,384 bytes** (`SETTINGS_MAX_FRAME_SIZE`); may be raised up to 16,777,215 (2²⁴ − 1).
- **Stream ID 0** = frames that apply to the whole connection (`SETTINGS`, `PING`, `GOAWAY`, connection `WINDOW_UPDATE`).

### 3.2 Frame types (RFC 9113 defines ten)

| Type | Code | Purpose |
|---|---|---|
| **DATA** | 0x0 | Body bytes of a request/response |
| **HEADERS** | 0x1 | Header block (HPACK-compressed) that opens a stream / carries response headers or trailers |
| **PRIORITY** | 0x2 | Old dependency/weight priority signal (**deprecated** in RFC 9113) |
| **RST_STREAM** | 0x3 | Abort one stream (cancel, error) without touching others |
| **SETTINGS** | 0x4 | Connection parameters (frame size, concurrent streams, initial window...) |
| **PUSH_PROMISE** | 0x5 | Server announces a pushed response |
| **PING** | 0x6 | Liveness check / RTT measurement |
| **GOAWAY** | 0x7 | Graceful shutdown: "finish existing streams; don't start new ones" |
| **WINDOW_UPDATE** | 0x8 | Flow-control credit |
| **CONTINUATION** | 0x9 | Continues a header block that didn't fit one `HEADERS` frame |

Important **flags**: `END_STREAM` (0x1, "no more frames from me on this stream"), `END_HEADERS` (0x4, header block is complete), `PADDED` (0x8), `PRIORITY` (0x20), and `ACK` (0x1 on `SETTINGS`/`PING`).

### 3.3 Pseudo-headers
Instead of a request line and status line, HTTP/2 uses special headers that begin with `:` and come first:

```
:method: GET
:scheme: https
:authority: www.example.com      ← replaces the HTTP/1.1 Host header
:path: /index.html
```
Response: `:status: 200` followed by ordinary headers such as `content-type`. **Header names are lowercase** in HTTP/2. There is no `Connection`, `Keep-Alive`, `Upgrade` or `Transfer-Encoding: chunked` (they are connection-specific, and HTTP/2 has its own framing).

---

## 4. Streams and multiplexing

### 4.1 Stream identifiers
- 31-bit numbers. **Client-initiated streams use odd IDs** (1, 3, 5, ...); **server-initiated (push) streams use even IDs** (2, 4, ...). This lets both sides choose IDs without coordinating.
- IDs **increase** and are **never reused** on a connection. When they run out (or after many requests), the connection is simply replaced by a new one (a `GOAWAY` tells the peer).

### 4.2 Stream life cycle

```
                       ┌───────┐
                       │ idle  │
                       └───┬───┘
              send/recv HEADERS
                       ▼
                   ┌───────┐
        ┌─ END_STREAM sent ──►│ open  │◄── END_STREAM received ─┐
        ▼                     └───────┘                         ▼
┌────────────────┐                                   ┌─────────────────┐
│ half-closed    │    (each side finished sending    │ half-closed     │
│ (local)        │     in one direction)             │ (remote)        │
└───────┬────────┘                                   └────────┬────────┘
        └──── END_STREAM received / sent, or RST_STREAM ──────┘
                                  ▼
                              ┌────────┐
                              │ closed │
                              └────────┘
```

A typical `GET`: the client sends `HEADERS` with `END_STREAM` (no body) → the stream is **half-closed (local)**; the server sends `HEADERS` and `DATA…` with `END_STREAM` on the last one → **closed**. A `RST_STREAM` closes one stream immediately (for example, the user navigated away) while every other stream continues, something impossible in HTTP/1.1 without dropping the connection.

### 4.3 Multiplexing in practice
Imagine the client sends four requests at once: `/index.html` (server needs 10 s), `/style.css`, `/script.js`, `/logo.png` (each ready in 0.1 s):

| Time | Wire (server → client) | User sees |
|---|---|---|
| 0.0 s | Client sends `HEADERS:1`, `HEADERS:3`, `HEADERS:5`, `HEADERS:7` back-to-back | |
| 0.1 s | `HEADERS:3 200`, `DATA:3`… `END_STREAM`; same for 5 and 7 | CSS/JS/logo are complete already |
| 10 s | `HEADERS:1 200`, `DATA:1`… `END_STREAM` | HTML arrives; page completes |

In HTTP/1.1 on one connection the CSS, JS and image would have waited behind the HTML. (With 6 parallel connections the effect is similar, but at the cost of extra connections, and only 6 requests at a time.)

Large bodies are split into multiple `DATA` frames, so frames from different streams **interleave** fairly on the wire:

```
[HEADERS:1][HEADERS:3][HEADERS:5]  [D:3][D:1][D:5][D:1][D:5][D:1]  ...
```

The receiver uses the **stream ID** in each frame header to reassemble each message separately. **Concurrency limit:** each side announces `SETTINGS_MAX_CONCURRENT_STREAMS` (recommended ≥ 100; nginx default 128); the practical limit is your bandwidth and server capacity, not six.

---

## 5. HPACK: header compression

HTTP headers are large and repetitive: `User-Agent`, `Accept`, `Accept-Language`, `Cookie` (often kilobytes) are resent on **every** request. Over 100 requests, that can be tens or hundreds of KB, all on the *upload* path, which is often the slow direction.

**HPACK** (RFC 7541) compresses header lists by **remembering what was already sent on this connection**:

1. **Static table**: 61 predefined entries for very common headers/values (`:method: GET` is index 2, `:method: POST` is 3, `:path: /` is 4, `:scheme: https` is 7, `:status: 200` is 8, and so on). A whole header sent as **one byte** (for example `0x82` = "indexed header field #2").
2. **Dynamic table**: a per-connection, size-limited table both sides update as they go. The first time a header like `custom-header: my-app-v1.0` is sent, it is transmitted in full and added to the table (index 62+). Later requests just send the **index** (about 1 byte instead of ~25).
3. **Huffman coding** of literal string values (a fixed table optimized for HTTP text), typically saving ~30% on them.

Example: after the first request, a follow-up request for a different path can differ by a few bytes, because `:scheme`, `:authority`, `user-agent`, `accept`, and `cookie` are all references to table entries. Typical header cost falls from hundreds of bytes to **a handful per request** after the first, often 80–90% reduction on repeats.

**Why not just use gzip?** In 2012 the **CRIME** attack showed that compressing secret data (cookies) together with attacker-controllable data with a general compressor leaks the secret through compressed sizes. HPACK's design avoids that: it only supports exact-match references and Huffman coding (no adaptive substring matching across values), and **sensitive headers can be marked "never indexed"** so they're never added to a table.

**Operational notes:** compression state is **per connection and per direction**, so a header block must be decoded in order (which is why `CONTINUATION` frames can't be interleaved with other frames), and servers cap table size and total header size (`SETTINGS_HEADER_TABLE_SIZE` default 4096 bytes, `SETTINGS_MAX_HEADER_LIST_SIZE`) to avoid memory abuse. HTTP/3 uses **QPACK**, a redesigned version that copes with out-of-order delivery.

---

## 6. Flow control and SETTINGS

TCP already has flow control, but that treats the whole connection as one pipe. HTTP/2 multiplexes many streams over it, so it needs its own **application-level, credit-based flow control** so that a stalled consumer of *one* stream (a slow download) can't clog the whole connection or consume unbounded memory.

- Each receiver advertises a **window** (bytes it can buffer) **per stream** and **per connection**. The **default initial window is 65,535 bytes** (`SETTINGS_INITIAL_WINDOW_SIZE`); real implementations raise it (for example to several MB).
- Only **`DATA`** frames are flow-controlled. Senders may not exceed the window; as the receiver consumes data it grants more credit with **`WINDOW_UPDATE`** (stream ID = a stream, or 0 = connection).

```
Stream 1 window: 65,535
server sends DATA:1 (32,768)   → window 32,767
server sends DATA:1 (32,767)   → window 0        (server must pause stream 1; other streams continue)
client processes data, sends WINDOW_UPDATE:1 (+32,768) → window 32,768   (server resumes)
```

Too-small windows are a common cause of slow HTTP/2 downloads on high-latency links (throughput ≈ window ÷ RTT, just like TCP in Chapter 23).

### `SETTINGS` (exchanged at start, changeable later, always acknowledged)
| Setting | Meaning | Typical |
|---|---|---|
| `HEADER_TABLE_SIZE` | HPACK dynamic table size | 4096 |
| `ENABLE_PUSH` | Whether the client accepts pushes | 0 in modern browsers |
| `MAX_CONCURRENT_STREAMS` | How many streams the sender of the setting will let its peer open | 100–128 |
| `INITIAL_WINDOW_SIZE` | Initial stream flow-control window | 65,535 (raised by many) |
| `MAX_FRAME_SIZE` | Largest frame payload | 16,384 |
| `MAX_HEADER_LIST_SIZE` | Advisory limit on header size | server-defined |

---

## 7. Starting an HTTP/2 connection

### 7.1 Negotiation with ALPN (the normal way)
Browsers only use HTTP/2 over **TLS** (`h2`). Note that the *spec* doesn't require TLS; browsers do. During the TLS handshake the client lists supported protocols in the **ALPN** extension (Application-Layer Protocol Negotiation):

```
ClientHello:  ALPN = ["h2", "http/1.1"]
ServerHello:  ALPN = "h2"           ← agreed; otherwise the server picks "http/1.1"
```

No extra round trip is needed. After the TLS handshake:

1. The client sends the **connection preface**: the fixed 24-byte string `PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n` followed by a `SETTINGS` frame. (The odd string deliberately looks like an invalid HTTP/1.x request, so HTTP/1.x servers reject it cleanly.)
2. The server sends its `SETTINGS`.
3. Each acknowledges the other's `SETTINGS` (`SETTINGS` with `ACK`).
4. Requests can start immediately (clients may send requests right after their own preface, without waiting).

### 7.2 Cleartext HTTP/2 (`h2c`)
Possible with **prior knowledge** (the client just starts with the preface; used between services, gRPC inside a cluster, and behind TLS-terminating proxies). The old HTTP/1.1 `Upgrade: h2c` mechanism has been **deprecated** (RFC 9113) and is not supported by browsers.

### 7.3 Connection reuse and coalescing
Browsers use **one connection per origin** (scheme+host+port). Under certain conditions (same IP and a certificate valid for both names) they may **coalesce** requests for several hostnames onto one connection. HTTP/2 idle connections are kept open, and `PING` frames or TCP keepalives detect dead peers.

### 7.4 Shutting down
`GOAWAY(last_stream_id)` says "I won't accept streams above this ID; finish the ones in flight." Servers use it during deploys, and it lets clients safely retry requests that were not processed.

---

## 8. Server push and priorities: features that faded

### Server push
With `PUSH_PROMISE`, a server could send resources it *knew* the client would need (CSS for the HTML it was returning) before being asked, saving a round trip:

```
client: HEADERS:1 GET /index.html
server: HEADERS:1 200 ...   PUSH_PROMISE:1 (promises stream 2: /style.css)   DATA:1 ...  HEADERS:2 200 / DATA:2 (style.css)
```

In practice it was hard to use well: the server can't know what the browser already has cached (so it wasted bandwidth), pushes competed with the critical HTML, and gains were small compared to alternatives. **Chrome removed support in version 106 (2022)**, other browsers followed or ignored it, nginx removed its `http2_push` directive in 1.25.1, and browsers advertise `SETTINGS_ENABLE_PUSH = 0`. Use **`<link rel="preload">`** or the **`103 Early Hints`** status code instead. (Server push technically still exists in the spec of HTTP/2 and HTTP/3, but treat it as legacy.)

### Priorities
HTTP/2's original **dependency tree + weights** scheme (`PRIORITY` frames) was complicated and inconsistently implemented by browsers and servers. **RFC 9113 deprecated it.** Its replacement is the simpler **Extensible Prioritization** scheme (**RFC 9218**): a `Priority: u=3, i` header (urgency 0–7 and an "incremental" flag) plus a `PRIORITY_UPDATE` frame, usable with both HTTP/2 and HTTP/3.

---

## 9. What HTTP/2 does *not* fix: TCP head-of-line blocking

HTTP/2 removes HOL blocking **within HTTP**. But all streams share **one TCP byte stream**, which must be delivered **in order**. If one packet is lost, TCP holds back *everything after it*, including data belonging to other, unaffected streams, until the retransmission arrives:

```
TCP segments:  [1: stream 1][2: stream 3 ✗ LOST][3: stream 5][4: stream 7]
               segments 3 and 4 arrived, but TCP can't hand them to HTTP/2 until segment 2 is repaired
```

On low-loss wired networks this rarely matters. On **lossy** networks (mobile, Wi-Fi, long distance), one lost packet can freeze the whole page, and HTTP/2 over a single connection can even **perform worse than HTTP/1.1's six independent connections**. Other issues: TCP+TLS setup needs 2–3 round trips, and connection changes (Wi-Fi ↔ cellular) break the connection.

**HTTP/3 = HTTP semantics over QUIC** (Chapter 24: a UDP-based transport). QUIC gives each stream its own loss recovery, integrates TLS 1.3 (1-RTT handshake, 0-RTT resumption), and supports connection migration. It uses the same ideas as HTTP/2 (streams, header compression via QPACK, multiplexing) with a different transport. Browsers discover it via the `Alt-Svc` header (`Alt-Svc: h3=":443"; ma=86400`) or DNS HTTPS records, and fall back to HTTP/2 if UDP is blocked. Adoption is substantial and growing among major sites and CDNs.

---

## 10. Security considerations

| Issue | Description | Mitigation |
|---|---|---|
| **Rapid Reset** (CVE-2023-44487, Oct 2023) | The client opens a stream and immediately cancels it with `RST_STREAM`, repeatedly. Cancelled streams don't count against the concurrent-stream limit, but the server has already begun work, so a tiny number of connections can generate record-breaking request rates | Patch servers/proxies; rate-limit resets per connection; close abusive connections (`GOAWAY`); use a DDoS-capable edge |
| **Other resource-exhaustion attacks** (continuation flood, settings flood, ping flood, "zero-window", HPACK bombs) | Abuse HTTP/2's flexibility to make the peer allocate memory or CPU | Keep software updated; enforce limits on frames, headers, and control-frame rates |
| **Request smuggling via HTTP/2 → HTTP/1.1 downgrade** | A front-end that translates HTTP/2 to HTTP/1.1 must correctly rebuild `Content-Length`, `Transfer-Encoding` and header syntax | Use HTTP/2 end-to-end, or a well-tested proxy; reject invalid characters (`\r\n`) in header values and pseudo-headers |
| **Cleartext exposure** | `h2c` has no encryption | Use TLS (`h2`) over untrusted networks |
| **TLS requirements** | The spec forbids weak TLS settings (old versions/ciphers) | TLS 1.2+ with modern ciphers (TLS 1.3 preferred) |
| **Compression side channels** | CRIME/BREACH-style attacks | HPACK design; avoid reflecting secrets in compressed *bodies* alongside attacker input (BREACH concerns body compression) |

---

## 11. Performance: honest expectations

- Best case (many small resources, high latency): large gains (multiple times) vs HTTP/1.1 with the same connection limits, because the requests all start immediately and the header cost drops.
- Typical real sites: **modest** but real improvement (often tens of percent). Multiplexing removes waiting for connection slots, not bandwidth limits, server think-time or render-blocking resources.
- Lossy networks: can be slower than well-tuned HTTP/1.1 because of TCP HOL blocking, while HTTP/3 helps.
- **Undo old HTTP/1.1 tricks** (domain sharding especially: it defeats connection reuse, HPACK context, and prioritization). Bundling is less essential, but still reduces overhead and improves compression; moderate bundle sizes with good cache control are a balance.
- **TLS overhead** is paid once per connection: a single connection amortizes it.
- **CDNs and reverse proxies** almost universally terminate HTTP/2 at the edge and may talk HTTP/1.1 or HTTP/2 to your origin.

### gRPC
**gRPC** uses HTTP/2 as its transport: each RPC is one stream (`HEADERS` with `:path: /pkg.Service/Method`, `content-type: application/grpc`, then `DATA` with length-prefixed Protocol Buffers messages, and **trailers** carrying `grpc-status`). It relies on multiplexing (many calls on one connection), streaming in both directions, and flow control. That's why gRPC through a proxy needs an HTTP/2-capable proxy (and why some browsers need gRPC-Web).

---

## 12. Configuration

**nginx (1.25.1 and newer):**

```nginx
server {
    listen 443 ssl;
    http2 on;                          # (older versions: listen 443 ssl http2;)
    server_name example.com;

    ssl_certificate     /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    keepalive_timeout 65;
    http2_max_concurrent_streams 128;      # default 128 (in newer versions set via `http2` block or defaults)
    location / { root /usr/share/nginx/html; }
}
```
(Older nginx: `http2_max_field_size`, `http2_max_header_size`, `http2_recv_timeout` no longer exist in current releases; check `nginx -V` and the docs for your version.)

**Apache:** `LoadModule http2_module ...`, `Protocols h2 http/1.1`, `H2MaxSessionStreams`, `H2Push off`.

**Caddy, Traefik, Envoy, HAProxy, Go's `net/http`, Node's `http2`, Jetty, Tomcat** all support it; Caddy and Go's standard library enable it automatically over TLS.

---

## 13. Hands-on labs

**Lab 1: Is a site using HTTP/2?**

```bash
curl -sI --http2 https://www.cloudflare.com | head -3          # first line: HTTP/2 200
curl -v --http2 https://example.com/ -o /dev/null 2>&1 | grep -E 'ALPN|HTTP/2|SETTINGS|stream'
openssl s_client -alpn h2 -connect example.com:443 </dev/null 2>/dev/null | grep -i 'ALPN'   # "ALPN protocol: h2"
```

**Lab 2: Browser DevTools**: Network tab → right-click the column headers → enable **Protocol** and **Connection ID**. Reload: `h2` rows share one Connection ID (multiplexing); `http/1.1` sites show many. Click a request → Headers: names are lowercase and pseudo-headers (`:method`, `:path`) appear.

**Lab 3: Your own HTTP/2 server with nginx in Docker**

```bash
mkdir h2lab && cd h2lab
openssl req -x509 -newkey rsa:2048 -nodes -keyout key.pem -out cert.pem -days 30 -subj "/CN=localhost"
cat > default.conf << 'EOF'
server {
    listen 443 ssl;
    http2 on;
    ssl_certificate     /certs/cert.pem;
    ssl_certificate_key /certs/key.pem;
    location / { root /usr/share/nginx/html; }
}
EOF
docker run -d --name h2 -p 8443:443 \
  -v "$PWD/default.conf":/etc/nginx/conf.d/default.conf:ro \
  -v "$PWD":/certs:ro nginx:1.27-alpine
curl -k -v --http2 https://localhost:8443/ -o /dev/null 2>&1 | grep -E 'ALPN|< HTTP/2|server:'
curl -k --http1.1 -sI https://localhost:8443/ | head -1        # the same server also speaks HTTP/1.1
docker rm -f h2
```
(`-k` accepts the self-signed certificate. Requires nginx ≥ 1.25.1 for `http2 on;`; use `listen 443 ssl http2;` on older images.)

**Lab 4: See the frames with `nghttp`** (`apt install nghttp2-client`, or `brew install nghttp2`):

```bash
nghttp -nv https://nghttp2.org/          # -v verbose frame log: SETTINGS, HEADERS, DATA, WINDOW_UPDATE
nghttp -nv -m 5 https://nghttp2.org/     # request the same URL 5 times on one connection: streams 1, 3, 5, 7, 9
nghttp -nas https://nghttp2.org/         # -a follow links and fetch page assets; -s show stats
```
Identify: the connection preface, `SETTINGS` and `SETTINGS ACK`, `HEADERS` with `END_HEADERS`, odd stream IDs, `DATA` with `END_STREAM`, `GOAWAY` at the end.

**Lab 5: Decrypt HTTP/2 in Wireshark**

```bash
export SSLKEYLOGFILE=$HOME/tls-keys.log
curl --http2 https://example.com/ -o /dev/null           # curl (and Chrome/Firefox launched from the same shell) write TLS secrets
# Wireshark → Preferences → Protocols → TLS → (Pre)-Master-Secret log filename = that file
# Capture on your interface, filter: http2   → expand "HyperText Transfer Protocol 2"
```

**Lab 6: Watch multiplexing and HOL blocking.** With a Python HTTP/2-capable client (`pip install httpx[http2]`) and the nginx from lab 3 (add `location /slow { proxy_pass ...; }` or use any server with a slow endpoint):

```python
import asyncio, httpx, time
async def main():
    async with httpx.AsyncClient(http2=True, verify=False) as c:
        t = time.time()
        r = await asyncio.gather(*[c.get("https://localhost:8443/") for _ in range(50)])
        print(len(r), "responses in", round(time.time()-t, 3), "s; http_version =", r[0].http_version)
asyncio.run(main())
```
`http_version` should print `HTTP/2`; all 50 requests share one connection. Compare with `http2=False` (HTTP/1.1, pooled connections).

**Lab 7: Compare with lossy networking (Linux, root).** Add `sudo tc qdisc add dev lo root netem loss 3%`, repeat lab 6 with `http2=True` and `http2=False`, then `sudo tc qdisc del dev lo root`. Observe how HTTP/2 vs HTTP/1.1 behave under loss (results vary; that's the point).

**Lab 8: HPACK in action.** Capture two requests on one connection with `nghttp -nv -m 2 https://nghttp2.org/`: the second `HEADERS` frame is far smaller than the first (look at the frame `length`).

---

## 14. Common misconceptions

| Misconception | Reality |
|---|---|
| "HTTP/2 needs TLS" | The spec allows cleartext (`h2c`); **browsers** only implement `h2` over TLS |
| "HTTP/2 is a new application protocol / changes my API" | Same semantics; only the wire format changed |
| "HTTP/2 removes all head-of-line blocking" | Only at the HTTP level. TCP HOL blocking remains (fixed by HTTP/3) |
| "HTTP/2 always makes sites much faster" | Often modestly; bandwidth, server time and render-blocking resources still dominate |
| "Server push is great, enable it" | Effectively dead: removed from Chrome and nginx. Use preload/Early Hints |
| "Stream IDs are reused" | Never reused on a connection; a new connection is opened when they run out |
| "One connection can only carry 6/…" | The concurrent-stream limit is server-announced (typically 100–128+) |
| "Headers are sent in text" | They are HPACK-encoded binary; names are lowercase |
| "HTTP/2 supports chunked encoding / `Connection` header" | No: forbidden; the framing replaces them |
| "Domain sharding still helps" | It hurts under HTTP/2 |
| "HTTP/3 replaces HTTP/2 entirely" | They coexist; clients fall back to HTTP/2 or HTTP/1.1 |

---

## 15. Summary

- **HTTP/2** = same HTTP semantics, new **binary framing**: connection → streams → frames (9-byte header with length, type, flags, stream ID).
- **Multiplexing** many requests/responses on **one** TCP+TLS connection; odd stream IDs for client requests; `RST_STREAM` cancels one stream.
- **HPACK** shrinks repeated headers using static and dynamic tables and Huffman coding, designed to resist CRIME.
- **Flow control** per stream and connection with `WINDOW_UPDATE`; `SETTINGS` tune limits.
- Negotiated via **ALPN** (`h2`) in TLS; connection preface + `SETTINGS`; `GOAWAY` for graceful shutdown.
- **Server push** and **PRIORITY** frames did not work out (removed/deprecated; replaced by preload/Early Hints and RFC 9218 priorities).
- **TCP HOL blocking** remains, addressed by **HTTP/3 over QUIC**. Watch for HTTP/2-specific attacks (**Rapid Reset**).

---

## 16. Check your understanding

1. What is the difference between a frame, a stream, and a message in HTTP/2?
2. Why do clients use odd stream IDs? Can an ID be reused?
3. Explain how HTTP/2 lets a fast response overtake a slow one on the same connection, and why HTTP/1.1 couldn't.
4. What are the static table, the dynamic table and Huffman coding in HPACK, and why is HPACK designed differently from gzip?
5. How is HTTP/2 negotiated during a TLS handshake, and what is the connection preface?
6. What does flow control protect against that TCP's flow control doesn't?
7. Why can HTTP/2 be slower than HTTP/1.1 on a lossy network?
8. Why did server push fail in practice, and what replaces its use case?
9. What is the Rapid Reset attack in one sentence?

<details>
<summary>Answers</summary>

1. A frame is the smallest unit (9-byte header + payload, tagged with a stream ID). A message is a complete request or response (a HEADERS frame plus DATA frames). A stream is the bidirectional sequence of frames carrying one request/response exchange.
2. Client-initiated streams are odd and server-initiated (push) streams are even, so neither side needs to coordinate. IDs are never reused on a connection.
3. Each response is broken into frames tagged with its own stream ID, so frames from different streams interleave and the server can send whichever is ready. HTTP/1.1 requires responses on a connection to be returned in request order.
4. Static table: 61 predefined common headers. Dynamic table: headers seen earlier on this connection, sent later as small indexes. Huffman: compact coding for literal strings. gzip-style adaptive compression leaks secrets through size changes (CRIME), whereas HPACK uses exact-match indexing plus "never indexed" for sensitive headers.
5. The client offers `h2` and `http/1.1` in the ALPN extension and the server picks `h2`. Then the client sends the 24-byte preface `PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n` and a SETTINGS frame; the server sends its SETTINGS; both acknowledge.
6. It protects individual streams and the whole connection at the application level, so one slow-consuming stream can't hog buffers or block others, while TCP's flow control sees only one byte stream.
7. All streams share a single TCP byte stream, so a lost packet delays delivery of every stream until it is retransmitted (TCP head-of-line blocking).
8. Servers couldn't know what browsers already had cached and pushes wasted bandwidth and competed with critical resources. `<link rel="preload">` and `103 Early Hints` replace it.
9. The client repeatedly opens streams and immediately cancels them with `RST_STREAM`, forcing the server to do work without hitting the concurrent-stream limit, producing enormous request floods from few connections.
</details>

**Practice**

1. Use `nghttp -nv` against a site and produce an annotated list of the frames exchanged in order (SETTINGS, HEADERS, DATA, etc.).
2. Check five well-known sites for `h2` and `h3` (`curl -sI --http3 ...` if your curl supports it, or the `alt-svc` response header).
3. Build the nginx container above, serve two large files, and use `nghttp -nv -m 2` to request them in parallel; identify the interleaved `DATA` frames.
4. Take a simple HTTP/1.1 optimization (domain sharding for images) and explain in detail why it degrades HTTP/2 performance.
5. Read RFC 9113 sections 4 (frames) and 5 (streams), and verify the frame header layout against a Wireshark capture.

---

**Next:** [Chapter 28 – DNS (Domain Name System) in Detail](28_dns_domain_name_system_in_details.md)
