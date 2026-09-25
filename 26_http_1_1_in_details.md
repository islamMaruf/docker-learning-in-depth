# Chapter 26: HTTP/1.1 in Detail

> **In one sentence:** HTTP/1.1 fixed HTTP/1.0's biggest waste by making TCP connections **persistent** (reused for many requests), and added the **`Host` header** (virtual hosting), **chunked transfer**, **better caching**, **range requests** and more methods, but it still delivers responses one at a time per connection, which is why browsers open several connections and why HTTP/2 was invented.

**Level:** 🟡 Intermediate · **Reading time:** ~55 minutes

**Prerequisites:** [Chapter 25 – HTTP/1.0](25_http_1_0_in_details.md) and the TCP basics in [Chapter 23](23_tcp_in_details.md).

---

## What you will learn

- What HTTP/1.1 changed compared to 1.0, and *why*
- **Persistent connections** (keep-alive): how they work, how the end of each response is detected, the arithmetic of the savings
- **Message framing:** `Content-Length` vs **chunked transfer encoding**
- The mandatory **`Host` header** and virtual hosting
- **Methods** (`GET`, `HEAD`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`, ...), idempotency and safety
- **Caching:** `Cache-Control`, `ETag`, conditional requests (`304`)
- **Range requests**, content negotiation, compression, cookies, redirects, `100 Continue`
- **Pipelining** and **head-of-line blocking**, and why browsers open ~6 connections per host
- Domain sharding and other old workarounds (and why they're obsolete)
- Security issues (slow-request DoS, request smuggling), tuning, and practical Docker/nginx examples
- Labs with `curl`, `nc`, `tcpdump` and Python

---

## 1. What problem did HTTP/1.1 solve?

HTTP/1.0 opened a new TCP connection for **every** resource (Chapter 25). A page with 40 resources meant 40 handshakes, 40 slow-starts and 40 teardowns. HTTP/1.1 (RFC 2068 in 1997; RFC 2616 in 1999; rewritten as RFC 7230–7235 in 2014 and RFC 9110–9112 in 2022) is the version that dominated the web from the late 1990s to the mid 2010s and is still everywhere today.

```
HTTP/1.0:   [connect] req1 resp1 [close]   [connect] req2 resp2 [close]   [connect] req3 resp3 [close]
HTTP/1.1:   [connect] req1 resp1  req2 resp2  req3 resp3 ...           (idle timeout or Connection: close) [close]
```

### The main changes

| Change | Benefit |
|---|---|
| **Persistent connections by default** | Skip repeated handshakes, TLS setups and slow-start |
| **Mandatory `Host` header** | Many websites can share one IP address and port (**virtual hosting**) |
| **Chunked transfer encoding** | Send a body of *unknown length* while keeping the connection reusable (streaming, generated content) |
| **Better caching**: `Cache-Control`, `ETag`, `If-None-Match`, `Vary` | Fewer and cheaper repeat downloads |
| **Range requests** (`Range`, `206 Partial Content`) | Resume downloads, video seeking, parallel chunks |
| **More methods**: `PUT`, `DELETE`, `OPTIONS`, `TRACE`, `CONNECT` (and later `PATCH`) | Enough vocabulary for REST-style APIs |
| **`100 Continue`** (`Expect: 100-continue`) | Check that the server will accept a large upload *before* sending it |
| **Content negotiation** (`Accept-*`, `Vary`) | Language, encoding and type selection |
| **Pipelining** (optional) | Send several requests without waiting (rarely usable in practice) |
| **Many more status codes** (e.g. 100, 206, 303, 307, 409, 410, 413, 415, 429 [added later], 505) | More precise semantics |

---

## 2. Persistent connections

### 2.1 How it works
In HTTP/1.1, **the connection stays open after a response** unless someone says otherwise. Either side can request closing with a header:

```
Connection: close
```

The old HTTP/1.0 extension `Connection: keep-alive` is **unnecessary** in 1.1 (still sent by many clients, and harmless). The `Keep-Alive: timeout=5, max=100` header is a non-standard *hint* from some servers about how long or how many requests the connection will be kept.

A persistent connection ends when: a peer sends `Connection: close`, the **idle timeout** expires (nginx defaults to 75 s, Apache to 5 s), a maximum number of requests is reached (`keepalive_requests`), an error occurs, or the peer simply closes the socket.

### 2.2 The key question: where does one response end?
On a reused connection the client can no longer rely on "the server closes when it's done". Every response must therefore be **self-delimiting**. HTTP/1.1 defines how (message framing):

1. Responses to `HEAD` and status `204`/`304` have **no body**.
2. `Transfer-Encoding: chunked` → the body arrives in chunks, ending with a zero-length chunk (section 3).
3. `Content-Length: N` → exactly **N** bytes of body follow.
4. Otherwise, the body runs until the server closes the connection (which makes the connection non-reusable).

Getting this right is critical: if `Content-Length` is wrong, the next response gets misparsed. (This is also the root of *request smuggling* attacks, section 10.)

### 2.3 The savings, with correct arithmetic
Let **RTT** = 100 ms, and ignore data transfer time. Fetch **N = 100 small resources**, one after another on one connection:

| | HTTP/1.0 (new connection each) | HTTP/1.1 (one reused connection) |
|---|---|---|
| Per resource | 1 RTT (handshake) + 1 RTT (request/response) = **2 RTT** | **1 RTT** (request/response) |
| Setup | (included above) | 1 RTT once |
| Total for 100 | 100 × 2 = **200 RTT = 20 s** | 1 + 100 × 1 = **101 RTT = 10.1 s** |

So persistence roughly **halves** the latency of sequential fetching (and with HTTPS it saves far more, since each new connection also pays 1–2 extra RTT for TLS). Add the removal of slow-start restarts and CPU cost, and typical gains are large, but the often-quoted "90%" figure isn't universal; it depends on the mix of resources. (Teardown isn't counted: it happens off the critical path.)

Browsers then reduce the remaining cost by fetching things **in parallel over several connections** (section 6).

---

## 3. Message framing: `Content-Length` vs chunked

### `Content-Length`
The server must know the size before sending the headers. Fine for files, awkward for content that is generated on the fly.

### Chunked transfer encoding
```
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

5\r\n
Hello\r\n
7\r\n
, World\r\n
0\r\n
\r\n
```

Each chunk is `size-in-hex CRLF data CRLF`; a chunk of size `0` ends the body (optionally followed by *trailer* headers). This lets a server start sending immediately (streaming logs, server-rendered pages, `Server-Sent Events`, large database exports) and still reuse the connection.

Try it: `curl -v https://httpbin.org/stream/3` shows `Transfer-Encoding: chunked`. To see the raw chunks: `curl --raw -i http://httpbin.org/stream/2`.

(Requests can be chunked as well, e.g. streaming uploads.) HTTP/2 doesn't use chunked encoding (it has its own framing).

---

## 4. The `Host` header and virtual hosting

In HTTP/1.0 the request line has only the **path**, so a server couldn't know *which site* was asked for if several share an IP address. HTTP/1.1 makes this header **mandatory**:

```
GET /index.html HTTP/1.1
Host: www.example.org
```

A single server (or reverse proxy) at one IP:port can now host `www.example.org`, `blog.example.net`, `shop.example.com`, ... and choose by `Host`. That is **name-based virtual hosting**, and it is how shared hosting, CDNs and Kubernetes **Ingress** controllers route traffic. (For HTTPS, the *TLS* handshake needs the name too, via SNI, Chapter 29.)

A server must answer a 1.1 request without `Host` with `400 Bad Request`. Try:

```bash
printf 'GET / HTTP/1.1\r\n\r\n' | nc example.com 80          # likely 400
printf 'GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n' | nc example.com 80
```

Without `Connection: close`, `nc` will just sit there because the server keeps the connection open, which is persistence in action (press Ctrl+C).

---

## 5. Methods and their semantics

| Method | Purpose | Has request body | Safe | Idempotent | Typical response |
|---|---|---|---|---|---|
| **GET** | Read a resource | no | ✅ | ✅ | `200` + body |
| **HEAD** | `GET` without the body | no | ✅ | ✅ | headers only |
| **POST** | Submit data / trigger processing / create a subordinate resource | yes | ❌ | ❌ | `200`, `201` + `Location`, `303` |
| **PUT** | Create or **replace** the resource at this URL | yes | ❌ | ✅ | `200`, `201`, `204` |
| **PATCH** (RFC 5789) | **Partially** modify a resource | yes | ❌ | not guaranteed | `200`, `204` |
| **DELETE** | Remove the resource | usually no | ❌ | ✅ | `200`, `202`, `204` |
| **OPTIONS** | Ask what is allowed (used by browsers for **CORS preflight**) | no | ✅ | ✅ | `204` + `Allow`/`Access-Control-*` |
| **TRACE** | Echo the request back (diagnostics; usually disabled: security) | no | ✅ | ✅ | |
| **CONNECT** | Ask a proxy to open a TCP tunnel (used for HTTPS through proxies) | no | ❌ | ❌ | `200` |

- **Safe** = doesn't change server state, so crawlers and prefetchers may call it freely. **Idempotent** = repeating the request has the same effect as doing it once, so clients and proxies can safely **retry** it after a network failure. `POST` isn't idempotent (double-clicking "Pay" may charge twice), which is why APIs add **idempotency keys**.
- **Never change data with `GET`.** A `GET /delete?id=5` link will be deleted by a browser prefetch or a web crawler.
- REST style: `GET /users/5`, `PUT /users/5`, `DELETE /users/5`, `POST /users`.

### Status codes added or emphasized in 1.1

| Code | Meaning |
|---|---|
| **100 Continue** | "Go ahead and send the body" (after `Expect: 100-continue`) |
| **101 Switching Protocols** | Upgrade to WebSocket or HTTP/2 cleartext |
| **206 Partial Content** | Response to a `Range` request |
| **303 See Other** | After a `POST`, go `GET` this URL (Post/Redirect/Get) |
| **307 / 308** | Temporary / permanent redirect that **keeps the method** (`302`/`301` historically let clients turn `POST` into `GET`) |
| **304 Not Modified** | Your cached copy is still valid |
| **400, 401, 403, 404** | Bad request; authenticate; forbidden; not found |
| **405 Method Not Allowed** (+ `Allow` header), **406**, **409 Conflict**, **410 Gone**, **411 Length Required**, **412**, **413 Content Too Large**, **415 Unsupported Media Type**, **416 Range Not Satisfiable** | Precise client errors |
| **429 Too Many Requests** (RFC 6585) | Rate-limited (`Retry-After`) |
| **500, 502, 503, 504, 505** | Server error; bad gateway; unavailable; gateway timeout; HTTP version not supported |

---

## 6. Head-of-line blocking, pipelining and browser connection pools

### 6.1 Ordering rule
On **one** HTTP/1.1 connection, requests and responses form a strict sequence: **response N must be returned before response N+1**, and a client generally sends the next request only after receiving the previous response.

```
one connection:   req A ─────────► [ server works 10 s ] ─► resp A │ req B ─► resp B │ req C ─► resp C
                                    ↑ everything behind A waits ("head-of-line blocking")
```

### 6.2 Pipelining (allowed, rarely used)
HTTP/1.1 allows a client to send several requests **without waiting** for the earlier responses:

```
client:  GET /a   GET /b   GET /c   ──►
server:                              ◄── resp /a  resp /b  resp /c     (must be in this order!)
```

Pipelining saves time when responses are quick. But because responses must stay in order, a slow first response blocks all the others, and many proxies and servers handled pipelined requests badly (and unsafe/non-idempotent requests can't be safely retried). As a result **browsers disabled or removed pipelining** (Firefox turned it off, Chrome never shipped it enabled, Chrome later removed it). You can still try it by hand:

```bash
printf 'GET /a HTTP/1.1\r\nHost: localhost:8080\r\n\r\nGET /b HTTP/1.1\r\nHost: localhost:8080\r\nConnection: close\r\n\r\n' | nc localhost 8080
```

### 6.3 What browsers actually do
Browsers work around head-of-line blocking by opening **several parallel connections to each host**, typically **6** (Chrome, Firefox, Safari, Edge) and using each one for one request at a time:

```
connection 1: req1 resp1 | req7  resp7  | ...        ← each connection handles ONE request at a time
connection 2: req2 resp2 | req8  resp8  | ...
   ...
connection 6: req6 resp6 | req12 resp12 | ...
```

RFC 2616 recommended *at most 2* connections per server; browsers ignored it as pages grew. A slow response now only blocks the other requests on *its* connection.

### 6.4 The price of many connections
- Each connection has TCP state and send/receive buffers on both ends; with TLS, extra per-connection crypto state and a **handshake per connection**.
- Each starts in **slow start**, competing with its siblings (and sometimes causing congestion). Six connections also gets a bigger share of a bottleneck link than a client using one (fairness).
- Servers must handle *6 × users* concurrent connections (e.g. 10,000 users ⇒ up to 60,000 connections). Event-driven servers (nginx) cope; thread-per-connection servers struggle.

### 6.5 Old workarounds (and why they're now obsolete)
| Trick | Idea | Problem today |
|---|---|---|
| **Domain sharding** (`cdn1.`, `cdn2.` hostnames) | Circumvent the 6-per-host limit → 12–24 connections | More DNS lookups, TLS handshakes, slow starts; **harmful with HTTP/2** |
| **Concatenating** CSS/JS files, **image sprites**, inlining | Fewer requests | Poor caching granularity (one change invalidates the bundle); unnecessary with multiplexing (though bundling still helps for other reasons) |
| **Cookieless static domains** | Avoid sending cookies on every image request | Still fine with a CDN, but less important with header compression |
| **Preconnect/DNS-prefetch** hints (`<link rel="preconnect">`) | Do handshakes early | Still useful |

HTTP/2 (Chapter 27) removes the need for most of these by **multiplexing** many requests over a single connection.

---

## 7. Caching

Caching lets a client (or a CDN/proxy) reuse a previous response without contacting the origin, or only revalidating cheaply. Two questions: **Is my copy still fresh?** and, if not, **has it changed?**

### 7.1 Freshness: `Cache-Control`
```
Cache-Control: max-age=3600            # fresh for 1 hour, reusable without asking
Cache-Control: public, max-age=31536000, immutable   # static hashed assets: cache "forever"
Cache-Control: no-cache                # may store, but MUST revalidate before every reuse
Cache-Control: no-store                # never store (sensitive data)
Cache-Control: private                 # only the browser may cache (not shared caches like CDNs)
```
(Older `Expires: <date>` and `Pragma: no-cache` are superseded.) `Age:` tells how long a response has been in a shared cache.

### 7.2 Validation: conditional requests
When a stored response is stale, the client asks *"changed since?"* instead of downloading again:

```
# first response
HTTP/1.1 200 OK
ETag: "33a64df5"
Last-Modified: Wed, 21 Oct 2015 07:28:00 GMT
Cache-Control: max-age=60

# later (after 60 s), the browser revalidates
GET /style.css HTTP/1.1
Host: example.com
If-None-Match: "33a64df5"
If-Modified-Since: Wed, 21 Oct 2015 07:28:00 GMT

# unchanged → tiny response, no body
HTTP/1.1 304 Not Modified
ETag: "33a64df5"
```

An **ETag** is an opaque version identifier (a hash or version number); it's more precise than `Last-Modified` (1-second resolution).

### 7.3 `Vary`
`Vary: Accept-Encoding` (or `Accept-Language`, `Cookie`) tells caches that the response depends on those request headers, so they must store separate variants.

**Best practice:** hashed static assets (`app.3f9a1c.js`) → `max-age=31536000, immutable`; HTML → `no-cache` (revalidate) or a short `max-age`; APIs with personal data → `private, no-store`.

---

## 8. More HTTP/1.1 features

### 8.1 Range requests
```
GET /video.mp4 HTTP/1.1
Range: bytes=0-999

HTTP/1.1 206 Partial Content
Content-Range: bytes 0-999/104857600
Content-Length: 1000
Accept-Ranges: bytes
```
Uses: resume a download, video seeking, parallel downloads. Try `curl -r 0-99 -i https://example.com/` (works if the server supports ranges; otherwise you'll get `200` and the whole file). A server advertises support with `Accept-Ranges: bytes`.

### 8.2 Compression and content negotiation
```
Accept-Encoding: gzip, br          →   Content-Encoding: gzip   (compressed body; Vary: Accept-Encoding)
Accept-Language: en-US,en;q=0.8     →   Content-Language: en
Accept: application/json           →   Content-Type: application/json
```
Compressing text (HTML/CSS/JS/JSON) with gzip or Brotli typically cuts it by 70–90%. `curl --compressed -v ...` asks for it. Don't compress already-compressed data (JPEG, MP4).

### 8.3 Cookies and state
HTTP is stateless; sessions are layered on with cookies (RFC 6265): the server sends `Set-Cookie: sid=abc123; HttpOnly; Secure; SameSite=Lax; Max-Age=3600`, and the browser sends `Cookie: sid=abc123` with every matching request. Set `HttpOnly` (hide from JavaScript), `Secure` (HTTPS only) and `SameSite`.

### 8.4 Redirects
`301`/`308` permanent; `302`/`307` temporary; `303` "see other" (after POST). The target is in `Location:`. `curl -L` follows them.

### 8.5 `Expect: 100-continue`
For a large upload, the client sends headers with `Expect: 100-continue`; the server answers `100 Continue` (send it) or an error such as `401`/`413` (don't waste bandwidth).

### 8.6 Proxies and `CONNECT`, `Via`, `X-Forwarded-For`
A reverse proxy or load balancer terminates the client connection and opens its own to the backend; it adds `X-Forwarded-For: <client-ip>` / `Forwarded:` and `X-Forwarded-Proto` so the app can see the original client (**trust these only from your own proxy**). `CONNECT host:443` builds a TCP tunnel through a forward proxy (that's how HTTPS is proxied).

### 8.7 Connection upgrade
`Connection: Upgrade` + `Upgrade: websocket` (→ `101 Switching Protocols`) turns the HTTP/1.1 connection into a **WebSocket**, a persistent, full-duplex channel.

---

## 9. Code: an HTTP/1.1 keep-alive server and client (Python)

### Server: two-line change from the basic one
```python
# http11_server.py
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
import time

class Handler(BaseHTTPRequestHandler):
    protocol_version = "HTTP/1.1"            # ← enables persistent connections (default is HTTP/1.0!)

    def do_GET(self):
        if self.path == "/slow":
            time.sleep(3)
        body = f"you asked for {self.path}\n".encode()
        self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.send_header("Content-Length", str(len(body)))     # required to keep the connection reusable
        self.end_headers()
        self.wfile.write(body)

ThreadingHTTPServer(("0.0.0.0", 8080), Handler).serve_forever()
```

### Client: reuse one connection
```python
# http11_client.py
import http.client, time
conn = http.client.HTTPConnection("127.0.0.1", 8080)        # ONE TCP connection
for path in ["/a", "/b", "/c"]:
    t = time.time()
    conn.request("GET", path)                                 # Host: header added automatically
    resp = conn.getresponse()
    print(resp.status, resp.getheader("Content-Length"), resp.read(), f"{(time.time()-t)*1000:.1f} ms")
conn.close()
```

Run the server, then the client under `sudo tcpdump -i lo -nn 'tcp port 8080'`: you'll see **one** SYN handshake and three request/response pairs before a single FIN. Remove `protocol_version = "HTTP/1.1"` and each request gets a new connection again.

**Head-of-line blocking demo** with `curl` (HTTP/1.1, one connection vs several):

```bash
# sequential on one connection: /slow (3 s) then /fast
time curl -s localhost:8080/slow localhost:8080/fast --next        # curl reuses the connection, requests run one after the other → ~3 s + tiny
# parallel on separate connections (like a browser):
time (curl -s localhost:8080/slow & curl -s localhost:8080/fast & wait)     # /fast doesn't wait for /slow
```

---

## 10. Security considerations

| Issue | Description | Mitigation |
|---|---|---|
| **Slow-request DoS ("Slowloris")** | Persistent connections + a server that waits for requests: an attacker opens thousands of connections and dribbles partial headers to hold them open | Aggressive **header/body read timeouts**, per-IP connection limits, a reverse proxy/CDN (nginx, HAProxy) buffering requests, event-driven servers |
| **Request smuggling** | A proxy and a backend disagree on where a request ends (conflicting `Content-Length`/`Transfer-Encoding`), so an attacker "smuggles" a hidden second request | Reject ambiguous messages; use HTTP/2 end-to-end or consistent, updated parsers; normalize at the edge |
| **Response splitting / header injection** | Unsanitized `\r\n` in header values inserted from user input | Never put raw user input into headers; use framework helpers |
| **Cleartext exposure** | Everything, including cookies and passwords, is visible | Use **HTTPS**; `Strict-Transport-Security` |
| **Method abuse** | `TRACE`, `PUT`/`DELETE` enabled unintentionally | Disable unneeded methods; restrict with authorization |
| **Cache poisoning / wrong `Vary`** | A shared cache stores a response for the wrong audience | `Cache-Control: private`/`no-store` for user-specific data; correct `Vary` |
| **Host header attacks** | Apps that trust `Host` for links (password reset poisoning) | Validate against an allowlist |

---

## 11. Configuration and tuning

**nginx**

```nginx
http {
    keepalive_timeout  15s;        # idle time before the server closes a client connection (default 75s)
    keepalive_requests 1000;       # max requests per connection

    upstream backend {
        server app:3000;
        keepalive 32;              # keep idle connections to the backend for reuse
    }
    server {
        location / {
            proxy_pass http://backend;
            proxy_http_version 1.1;          # REQUIRED for upstream keep-alive (nginx defaults to 1.0 upstream!)
            proxy_set_header Connection "";  # remove "Connection: close"
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
        gzip on;
        gzip_types text/plain text/css application/json application/javascript;
    }
}
```

**Apache:** `KeepAlive On`, `MaxKeepAliveRequests 100`, `KeepAliveTimeout 5`.

Guidance: keep-alive timeouts of a few seconds to a minute balance reuse against idle memory; set **client-side pool idle timeouts shorter than the server's/LB's** to avoid reusing a connection the other side just closed ("connection reset" surprises); use HTTP/2 for browsers, and HTTP/1.1 with connection pools between services; compress text; set correct cache headers; put slow work behind a queue instead of holding requests open.

---

## 12. Hands-on lab

**1. Watch keep-alive with curl**

```bash
curl -v http://example.com/ http://example.com/ 2>&1 | grep -E 'Connected|Re-using|Connection #|left intact'
# "Connected to ... (#0)" once; the second request says "Re-using existing connection"
```

**2. Send `Connection: close`** and see it honored: `curl -v -H 'Connection: close' http://example.com/` (response includes `Connection: close` and the server closes).

**3. Virtual hosting**: run one nginx and two names:

```bash
docker run -d --name web -p 8080:80 nginx:1.27-alpine
curl -H 'Host: a.example' -i http://localhost:8080/       # same server, different Host: try changing it
curl -i --resolve app.test:8080:127.0.0.1 http://app.test:8080/    # pretend DNS says app.test = 127.0.0.1
```

Create two `server { server_name ...; }` blocks with different `root`s (mount a config) and watch the `Host` header choose the site. `docker rm -f web` when finished.

**4. Conditional requests**

```bash
curl -sI https://example.com/ | grep -iE 'etag|last-modified|cache-control'
curl -i -H 'If-None-Match: "<paste the etag>"' https://example.com/      # expect 304 Not Modified (if supported)
```

**5. Range request**: `curl -s -r 0-49 https://example.com/ | head -c 100 && echo` and `curl -I https://example.com/ | grep -i accept-ranges`.

**6. Chunked encoding**: `curl --raw -i https://httpbin.org/stream/2` (see the hex sizes); or `printf 'GET /stream/2 HTTP/1.1\r\nHost: httpbin.org\r\nConnection: close\r\n\r\n' | nc httpbin.org 80`.

**7. Pipelining by hand** (against your Python server on 8080): the two-request `printf | nc` command in section 6.2. Observe both responses arrive on one connection in order.

**8. Compare 1.0 and 1.1 connection reuse with tcpdump**: run the Python server twice: with and without `protocol_version = "HTTP/1.1"`, use the client script, and count SYNs.

**9. Time breakdown**

```bash
curl -o /dev/null -s -w 'connect %{time_connect}  ttfb %{time_starttransfer}  total %{time_total}\n' http://example.com/ http://example.com/
```
The **second** transfer in the same `curl` invocation shows `connect 0.000` (a reused connection).

**10. DevTools:** open the browser DevTools → Network → right-click the table header → enable **Protocol** and **Connection ID**. Load a site; you'll see which requests share a connection (HTTP/1.1 sites show ~6 connection IDs; HTTP/2 sites show one).

---

## 13. Common misconceptions

| Misconception | Reality |
|---|---|
| "HTTP/1.1 is 90% faster than HTTP/1.0" | It removes repeated connection setup, which can dominate for small resources, but the gain depends on the page and network; roughly a 2× improvement for sequential fetches in the simplest case, and more with TLS |
| "`Connection: keep-alive` is what enables persistence in 1.1" | Persistence is the **default** in 1.1; keep-alive is only needed for 1.0 clients. `Connection: close` opts *out* |
| "HTTP/1.1 supports parallel requests on one connection" | Only pipelining, which is in-order and basically unused. Parallelism comes from multiple connections |
| "Browsers pipeline requests" | They don't (disabled/removed) |
| "Pipelining and multiplexing are the same" | Pipelining = ordered responses on one connection; HTTP/2 multiplexing = interleaved, independent streams |
| "`Host` header is just informational" | It is what makes virtual hosting and most proxies/Ingress routing work |
| "`Content-Length` is optional" | Without it (or chunked), the connection can't be reused |
| "`no-cache` means 'don't cache'" | It means "cache but revalidate"; `no-store` means don't store |
| "`PUT` and `POST` are interchangeable" | `PUT` replaces at a known URL and is idempotent; `POST` isn't |
| "More connections is always faster" | Beyond a point, it adds slow-start, congestion and server load |
| "HTTP/1.1 is obsolete" | It is still widely used (internal services, simple clients, fallback) and the base of HTTP semantics for 2 and 3 |

---

## 14. Summary

- **HTTP/1.1** keeps connections open by default, so many requests share a TCP (and TLS) connection.
- Every response must be **self-delimiting**: `Content-Length` or **chunked**.
- **`Host`** enables virtual hosting; more **methods** (`PUT`, `PATCH`, `DELETE`, `OPTIONS`...) and **status codes**; robust **caching** (`Cache-Control`, `ETag`, `304`); **range** requests; compression and negotiation; cookies for state.
- Responses on one connection are **strictly ordered**, so a slow one blocks those behind it (**head-of-line blocking**). Pipelining doesn't fix it in practice; **browsers open ~6 connections per host** instead.
- Workarounds (domain sharding, bundling) traded one problem for another; **HTTP/2 multiplexing** and **HTTP/3** solve it properly.
- Keep an eye on security (timeouts, request smuggling, TLS), and tune keep-alive and caching.

---

## 15. Check your understanding

1. What single change makes HTTP/1.1 faster than HTTP/1.0 for a page with many resources, and what header can turn it off?
2. How does a client know where a response ends on a persistent connection? List the methods.
3. Why is the `Host` header mandatory in 1.1?
4. What is head-of-line blocking in HTTP/1.1, and how do browsers work around it?
5. What is the difference between `no-cache` and `no-store`? What does a `304` response contain?
6. Which methods are idempotent, and why does that matter for retries?
7. Why did nginx-to-backend connections not keep alive by default, and what two directives fix it?
8. With RTT = 50 ms, how long do 40 sequential small requests take (ignoring data time) with HTTP/1.0 and with HTTP/1.1 on one persistent connection?

<details>
<summary>Answers</summary>

1. Persistent (reused) connections: no repeated handshakes/slow start. `Connection: close` disables reuse.
2. `HEAD`/`204`/`304` have no body; `Transfer-Encoding: chunked` (zero-length final chunk); `Content-Length: N`; else the connection close ends the body.
3. So many websites can share one IP:port; the server (or proxy) uses `Host` to choose which site the request is for.
4. Responses on one connection must be returned in order, so a slow one delays the rest. Browsers open about 6 parallel connections per host.
5. `no-cache`: may be stored but must be revalidated before reuse. `no-store`: must not be stored at all. A `304` has headers only (no body) and means the cached copy is still valid.
6. `GET`, `HEAD`, `PUT`, `DELETE`, `OPTIONS`, `TRACE` (not `POST`/`PATCH`). Idempotent requests can be retried automatically after a failure without risking a double effect.
7. nginx uses HTTP/1.0 with `Connection: close` to upstreams by default. Use `keepalive N;` in the `upstream` block and `proxy_http_version 1.1;` plus `proxy_set_header Connection "";` in the location.
8. HTTP/1.0: 40 × 2 × 50 ms = 4.0 s. HTTP/1.1: 1 RTT for the handshake + 40 × 1 RTT = 41 × 50 ms = 2.05 s.
</details>

**Practice**

1. Load a website with the DevTools network panel open; count distinct connections and identify the HTTP versions. Find one response with `304` and one with `Cache-Control: max-age`.
2. Write a Python or Go server that supports keep-alive and chunked responses (stream 5 lines, one per second); observe with `curl -N` and `nc`.
3. Configure nginx with two virtual hosts and prove selection by `Host` using `curl -H` and `--resolve`.
4. Measure the effect of connection reuse: fetch 50 URLs with `curl` invoked once with all URLs (reuse) versus 50 separate `curl` invocations, and compare times.
5. Add `Cache-Control` and `ETag` handling to your server; verify `304` responses with `curl -i`.

---

**Next:** [Chapter 27 – HTTP/2 in Detail](27_http_2_in_details.md)
