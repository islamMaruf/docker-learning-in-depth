# Chapter 25: HTTP/1.0 in Detail

> **In one sentence:** HTTP is the plain-text request/response language that browsers and servers speak; version 1.0 (1996) is its first widely used form, in which **every request opened a new TCP connection, got one response, and closed it**, simple and clear, but costly.

**Level:** 🟢 Beginner → 🟡 Intermediate · **Reading time:** ~45 minutes

**Prerequisites:** [Chapter 22](22_tcp_ip_model.md) and [Chapter 23](23_tcp_in_details.md) (TCP handshake, sockets, ports).

---

## What you will learn

- What **HTTP** is and where it sits in the stack (it is an application-layer protocol that runs over TCP)
- The **URL**: scheme, host, port, path, query
- The exact format of an **HTTP request** and **response** (start line, headers, blank line, body)
- The **methods** (`GET`, `HEAD`, `POST`), **status codes** and important **headers** of HTTP/1.0
- The step-by-step **life cycle** of one HTTP/1.0 transaction over TCP
- Why "one connection per request" hurts, with correct latency arithmetic
- How to speak HTTP **by hand** (`nc`, `curl`), build a **tiny HTTP/1.0 server and client** in Python, and watch it in `tcpdump`
- HTTP/0.9 → 1.0 → 1.1 → 2 → 3 in one picture

---

## 1. What is HTTP?

**HTTP** (HyperText Transfer Protocol) is an **application-layer** protocol that defines:

- how a **client** (browser, `curl`, a mobile app, another server) **asks** for a *resource* (a page, image, JSON document, ...);
- how a **server** **answers**, with either the resource or an explanation of why not;
- the **format** of these messages, the **methods**, **status codes** and **headers**.

HTTP is **not** about routing, reliability or ordering; it *assumes* a reliable, ordered byte stream underneath and therefore runs over **TCP** (HTTP/1.x and HTTP/2) or over QUIC (HTTP/3):

```
 HTTP        ← what to say (requests, responses)               Layer 7
 TCP         ← reliable, ordered delivery, ports               Layer 4
 IP          ← addressing and routing                          Layer 3
 Ethernet    ← delivery on the local link                      Layer 2
```

HTTP is **stateless** at the protocol level: each request/response pair is independent, and the server isn't required to remember earlier requests. (Cookies, tokens and sessions are added by applications to get "state" on top.)

### A little history

| Year | Version | Highlights |
|---|---|---|
| 1991 | **HTTP/0.9** | One-line request `GET /page`; response is just HTML; no headers, no status codes |
| **1996** | **HTTP/1.0** (RFC 1945) | Adds version number, **headers**, **status codes**, content types (so images and other files can be served), `POST` and `HEAD` |
| 1997/1999 | HTTP/1.1 (RFC 2068, RFC 2616; revised 2014 in RFC 7230–7235 and 2022 in RFC 9110–9112) | Persistent connections by default, mandatory `Host` header, chunked transfer, caching improvements, more methods |
| 2015 | HTTP/2 (RFC 7540, now RFC 9113) | Binary framing, multiplexing over one TCP connection, header compression |
| 2022 | HTTP/3 (RFC 9114) | HTTP over QUIC (UDP) |

The web of 1996 was mostly **static documents**: pages with a handful of small images, browsed over dial-up modems (14.4–56 kbit/s). HTTP/1.0's simplicity fit that world well.

---

## 2. URLs: how a resource is named

```
   http://example.com:8080/docs/intro.html?lang=en#section2
   └┬─┘   └────┬────┘└─┬─┘└──────┬───────┘ └──┬───┘ └───┬───┘
  scheme     host   port       path         query    fragment
```

| Part | Meaning |
|---|---|
| **scheme** | Protocol: `http` (default port 80), `https` (443) |
| **host** | Server name (resolved to an IP by DNS) or an IP address |
| **port** | Optional; omitted if it's the scheme's default |
| **path** | Which resource on that server (`/docs/intro.html`). `/` = the site root |
| **query** | Parameters after `?` (`lang=en`) |
| **fragment** | After `#`; used only by the browser (**never sent** to the server) |

The client turns this into: (1) DNS lookup of the host, (2) TCP connection to `host:port`, (3) an HTTP request whose start line contains only the **path + query**.

---

## 3. HTTP/1.0 message format

HTTP/1.0 is **text**: lines separated by `CRLF` (`\r\n`, carriage return + line feed). Both requests and responses have the same shape:

```
start line              ← request line OR status line
Header-Name: value      ← zero or more headers
Header-Name: value
                        ← an empty line (just CRLF) ends the headers
optional body           ← bytes; length known by Content-Length (or by connection close in 1.0)
```

### 3.1 Request

```
GET /docs/intro.html?lang=en HTTP/1.0\r\n
User-Agent: curl/8.5.0\r\n
Accept: text/html\r\n
\r\n
```

**Request line:** `METHOD SP request-target SP HTTP-version`

| Part | Example | Meaning |
|---|---|---|
| Method | `GET` | What to do |
| Request target | `/docs/intro.html?lang=en` | Which resource |
| Version | `HTTP/1.0` | Protocol version the client speaks |

Then headers, then the blank line, then the body (only for methods like `POST`).

**A `POST` with a body:**

```
POST /login HTTP/1.0\r\n
Content-Type: application/x-www-form-urlencoded\r\n
Content-Length: 27\r\n
\r\n
username=john&password=1234
```

`Content-Length` tells the server exactly how many body bytes follow (here 27).

### 3.2 Response

```
HTTP/1.0 200 OK\r\n
Date: Mon, 01 Jan 2024 12:00:00 GMT\r\n
Server: Apache/1.3.6\r\n
Content-Type: text/html\r\n
Content-Length: 137\r\n
\r\n
<html><body><h1>Hello</h1> ... </body></html>
```

**Status line:** `HTTP-version SP status-code SP reason-phrase`. The **3-digit code** is for programs; the **reason phrase** (`OK`, `Not Found`) is just human-readable text.

### 3.3 Methods in HTTP/1.0

| Method | Purpose | Body? | Properties |
|---|---|---|---|
| **GET** | Retrieve a resource | Request: no. Response: yes | **Safe** (doesn't change server state), **idempotent** (repeating has the same effect), cacheable |
| **HEAD** | Like `GET` but the server sends **headers only** (no body) | Response: never | Use to check existence, size (`Content-Length`) or freshness (`Last-Modified`) without downloading |
| **POST** | Send data to the server to be processed (submit a form, create something) | Request: yes | **Not** safe, **not** idempotent |

(RFC 1945 also lists `PUT`, `DELETE`, `LINK` and `UNLINK` as "additional" methods that servers weren't required to support. `OPTIONS`, `TRACE`, `CONNECT` came with HTTP/1.1; `PATCH` later, in RFC 5789.)

### 3.4 Status codes

| Class | Meaning | Common codes in HTTP/1.0 |
|---|---|---|
| **1xx** | Informational | (none defined in 1.0; introduced in 1.1) |
| **2xx** | Success | **200 OK**, 201 Created, 202 Accepted, **204 No Content** |
| **3xx** | Redirection | **301 Moved Permanently**, **302 Moved Temporarily** ("Found"), **304 Not Modified** |
| **4xx** | Client error | **400 Bad Request**, **401 Unauthorized** (needs authentication), **403 Forbidden**, **404 Not Found** |
| **5xx** | Server error | **500 Internal Server Error**, 501 Not Implemented, 502 Bad Gateway, **503 Service Unavailable** |

Quick rules of thumb: **2xx** = worked; **3xx** = go elsewhere (see the `Location` header) or use your cache; **4xx** = *you* asked wrongly; **5xx** = *the server* failed. `401` = "who are you?", `403` = "I know who you are, and no."

### 3.5 Important headers in HTTP/1.0

| Header | Direction | Purpose |
|---|---|---|
| `Content-Type` | both | Type of the body (`text/html`, `image/jpeg`, `application/json`); a *MIME type* |
| `Content-Length` | both | Body size in bytes |
| `Content-Encoding` | response | e.g. `gzip` |
| `Date` | both | When the message was generated |
| `Server` | response | Server software name |
| `User-Agent` | request | Client software |
| `Accept` | request | Content types the client can handle |
| `Referer` [sic] | request | The page that linked here |
| `Location` | response | Where to go (with 3xx) |
| `Last-Modified`, `Expires`, `If-Modified-Since`, `Pragma: no-cache` | | Caching (conditional requests: `If-Modified-Since` → `304`) |
| `Authorization`, `WWW-Authenticate` | | Basic authentication (**credentials are only Base64-encoded, not encrypted**, so useless without TLS) |
| `Connection: keep-alive` | both | A **non-standard extension** popular in late HTTP/1.0 clients/servers, asking to keep the connection open (becomes the default in 1.1) |
| `Host` | request | **Not required in 1.0**, but many clients sent it. It's **mandatory in HTTP/1.1** and enables *virtual hosting* (several sites on one IP) |

Unknown headers must be ignored: that's how HTTP stays **extensible**.

---

## 4. The life of one HTTP/1.0 transaction

Suppose you open `http://192.168.0.6/world-war`:

```
 Browser (client)                                           Web server :80
      │                                                          │
 1.   │ ─── SYN ───────────────────────────────────────────────► │   TCP three-way
      │ ◄── SYN+ACK ───────────────────────────────────────────── │   handshake
      │ ─── ACK + "GET /world-war HTTP/1.0 ..." ───────────────► │ 2. request (may ride on the 3rd segment)
      │                                                          │ 3. server parses the request, finds the
      │                                                          │    file or runs a program (CGI), builds the response
      │ ◄── "HTTP/1.0 200 OK ... <html>..." ─────────────────── │ 4. response (possibly many TCP segments)
      │ ◄── FIN ───────────────────────────────────────────────── │ 5. server closes: "that's the end of the body"
      │ ─── ACK, FIN ──────────────────────────────────────────► │
      │ ◄── ACK ───────────────────────────────────────────────── │   6. connection closed
 5'.  │ parse HTML → find <img>, <link>, <script> → each needs ANOTHER request (start again at 1)
```

Detailed steps:

1. **Open a TCP connection** to the server's port (80). One round trip (SYN → SYN+ACK). The client's final ACK can already carry the request.
2. **Send the request** text. TCP splits it into segments if needed and handles retransmission.
3. **Server processes it**: parse the request line and headers; map the path to a file (`/world-war` → `/var/www/html/world-war.html`) or a program/database; check permissions.
4. **Server sends the response**: status line, headers, blank line, body.
5. **Server closes the connection.** In HTTP/1.0 this is *how the end of the response is marked* if there's no `Content-Length`: the client keeps reading until the connection closes. (With `Content-Length`, the client knows exactly when the body ends.)
6. **Client parses the response**; the browser renders HTML and discovers more resources (stylesheets, scripts, images), each requiring **a brand-new connection**.

Who does what: the browser's HTTP code calls `socket()`, `connect()`, `send()`, `recv()` (system calls, Chapter 2); the **kernel's TCP/IP stack** does the handshake, segmentation, retransmission and closing; **DNS** happens before step 1 to turn the name into an IP.

---

## 5. Why "one request per connection" is costly (with correct numbers)

Let **RTT** be the round-trip time to the server (say 100 ms).

**Cost of fetching one small resource on a fresh connection:**

| Phase | Time |
|---|---|
| TCP handshake | 1 RTT (the client can send its request with the final ACK) |
| Request out, response back | 1 RTT (plus the transmission time of the data) |
| Connection close | Runs *after* the response arrives, so it doesn't delay the page, but the server keeps state around in `TIME_WAIT` |
| **Total to receive the response** | **≈ 2 RTT + transfer time** (+ 1–2 RTT for TLS if HTTPS, + DNS the first time) |

For a page with 1 HTML file and 4 images:

- **Sequentially** (one after another): 5 × 2 RTT = **10 RTT = 1.0 s** at 100 ms RTT, before counting data transfer.
- **In parallel** (say 4 connections at once): HTML first (2 RTT), then the 4 images together (2 RTT) → about **4 RTT = 0.4 s**.

Browsers of the HTTP/1.0 era therefore **opened several connections in parallel** (typically 2–4, later 6 per host). So the problem was **not** strictly "fully sequential", but "each resource pays connection costs, and the number of parallel connections is limited".

Beyond latency, per-request connections have other costs:

1. **TCP slow start** (Chapter 23): every new connection starts with a tiny congestion window, so short transfers never reach full speed. Reusing a warm connection is far faster.
2. **Server load:** every connection consumes a socket, a file descriptor, kernel memory and CPU for handshake and teardown; the closing side accumulates `TIME_WAIT` entries. Thousands of connections per second, each carrying a few KB, is wasteful.
3. **Network load:** extra packets (SYN, SYN+ACK, ACK, FIN×2, ACK×2 = 7 packets) around every request.
4. **No prioritization or multiplexing:** requests on separate connections compete without coordination.
5. **Repeated headers:** every request re-sends `User-Agent`, `Accept`, cookies, etc. (HTTP/2's header compression addresses this).

The relative overhead is dramatic for tiny resources: transferring a 2 KB icon may take 2 RTT of setup for 0.1 RTT of actual data.

### What changed
- **Keep-alive / persistent connections** (an extension to 1.0, the default in **HTTP/1.1**): reuse one TCP connection for many requests → no repeated handshake/slow start.
- **Pipelining** (1.1): send several requests without waiting; responses must come back in order. Poorly supported and rarely enabled (head-of-line blocking, buggy proxies).
- **HTTP/2:** many concurrent request/response streams multiplexed on *one* TCP connection (still susceptible to TCP-level head-of-line blocking).
- **HTTP/3:** streams over QUIC/UDP; one lost packet only delays its own stream.

Chapters 26 and 27 cover HTTP/1.1 and HTTP/2.

---

## 6. HTTP by hand

The best way to understand HTTP is to *type it*.

### 6.1 With `nc` (netcat)

```bash
printf 'GET / HTTP/1.0\r\nHost: example.com\r\n\r\n' | nc example.com 80
```

You will see something like `HTTP/1.0 200 OK` (or `HTTP/1.1 200 OK`; a modern server replies in the highest version it supports, even if the request said 1.0) followed by headers, a blank line, and the HTML. Note `\r\n` twice at the end, which is the blank line. Without it the server keeps waiting. (Many sites now answer plain HTTP with a `301` redirect to HTTPS.)

### 6.2 With `curl`

```bash
curl -v --http1.0 http://example.com/           # -v shows the request and response headers
curl -i http://example.com/                     # include response headers
curl -I http://example.com/                     # HEAD request: headers only
curl -X POST -d 'username=john&password=1234' -v http://httpbin.org/post
curl -v -H 'Connection: keep-alive' --http1.0 http://example.com/
```

### 6.3 Watch the TCP under the HTTP

Terminal 1: `sudo tcpdump -i any -nn -A 'tcp port 8080'` (`-A` prints the payload as ASCII).
Terminal 2: run Python's built-in server, which **speaks HTTP/1.0 by default**, then request a page:

```bash
python3 -m http.server 8080 &
curl -s http://127.0.0.1:8080/ > /dev/null
curl -s http://127.0.0.1:8080/ > /dev/null      # a second request: a brand-new TCP connection (new source port)
```

You will see: `[S]`, `[S.]`, `[.]`; then the ASCII `GET / HTTP/1.1` (curl speaks 1.1; use `--http1.0` for exactly 1.0); the response `HTTP/1.0 200 OK` with `Server: SimpleHTTP/0.6 Python/3.x`; and then a `[F.]`, the **server closing the connection**. Note the source ports differ between the two requests, because that's one connection per request.

---

## 7. Code: a tiny HTTP/1.0 client and server (Python)

### Client

```python
# http10_client.py
import socket

def http_get(host, port, path):
    sock = socket.create_connection((host, port))            # TCP handshake
    request = (f"GET {path} HTTP/1.0\r\n"
               f"Host: {host}\r\n"
               f"User-Agent: my-tiny-client\r\n"
               f"\r\n")                                       # blank line ends the headers
    sock.sendall(request.encode("ascii"))

    chunks = []
    while True:
        data = sock.recv(4096)
        if not data:                                          # server closed → end of response
            break
        chunks.append(data)
    sock.close()

    raw = b"".join(chunks)
    head, _, body = raw.partition(b"\r\n\r\n")                # split headers from body
    return head.decode("iso-8859-1"), body

headers, body = http_get("127.0.0.1", 8080, "/")
print(headers)
print("---- body ----")
print(body[:300].decode("utf-8", "replace"))
```

### Server

```python
# http10_server.py
import socket

def handle(client):
    request = b""
    while b"\r\n\r\n" not in request:                          # read until the end of the headers
        chunk = client.recv(4096)
        if not chunk:
            return
        request += chunk

    request_line = request.split(b"\r\n", 1)[0].decode("iso-8859-1")
    try:
        method, path, version = request_line.split(" ")
    except ValueError:
        body, status = b"<h1>400 Bad Request</h1>", "400 Bad Request"
    else:
        if method not in ("GET", "HEAD"):
            body, status = b"<h1>501 Not Implemented</h1>", "501 Not Implemented"
        elif path == "/":
            body, status = b"<html><body><h1>Welcome!</h1></body></html>", "200 OK"
        else:
            body, status = b"<h1>404 Not Found</h1>", "404 Not Found"

    head = (f"HTTP/1.0 {status}\r\n"
            f"Content-Type: text/html\r\n"
            f"Content-Length: {len(body)}\r\n"
            f"\r\n").encode("ascii")
    client.sendall(head + (b"" if request.startswith(b"HEAD") else body))
    # HTTP/1.0: the server closes the connection after every response (caller does client.close())

server = socket.socket()
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(("0.0.0.0", 8080))
server.listen(50)
print("listening on :8080")
while True:
    client, addr = server.accept()
    with client:
        handle(client)
```

(One request at a time: fine for learning. Real servers use threads, processes or event loops.)

Try `curl -v localhost:8080/`, `curl -I localhost:8080/`, `curl -v localhost:8080/nope`, `curl -X DELETE -v localhost:8080/`, and `printf 'garbage\r\n\r\n' | nc localhost 8080`.

**Socket life cycle:** client `socket → connect → send → recv (until close) → close`; server `socket → bind → listen → accept → recv → send → close`.

---

## 8. HTTP/1.0 with Docker

```bash
docker run -d --name web -p 8080:80 nginx:1.27-alpine
curl -v --http1.0 http://localhost:8080/
docker logs web                    # access log: shows "GET / HTTP/1.0" and the 200 status
docker rm -f web
```

`-p 8080:80` publishes the container's TCP port 80 (Chapter 18). The nginx access log lines record the request line, status and body size: a good way to see how servers view HTTP requests. Inside another container on the same network you can `curl http://web/` by name (Docker DNS).

---

## 9. Common misconceptions

| Misconception | Reality |
|---|---|
| "HTTP is secure/private" | Plain HTTP is readable and modifiable by anyone on the path. HTTPS = HTTP inside TLS (Chapter 29) |
| "HTTP is the same as TCP or the web" | HTTP is one application protocol among many that runs over TCP |
| "HTTP/1.0 had no way to reuse a connection" | The standard didn't, but `Connection: keep-alive` was a widely used extension |
| "HTTP/1.0 required the `Host` header" | It didn't; `Host` became mandatory in 1.1 (which enabled virtual hosting) |
| "HTTP/1.0 could only do GET" | It had `GET`, `HEAD` and `POST` (and optional extras) |
| "The reason phrase matters" | Only the 3-digit code is significant to programs |
| "HTTP/1.0 is fully sequential" | Browsers opened several connections in parallel, but each resource still paid a handshake |
| "Each HTTP/1.0 request costs 4.5 RTT" | Roughly **2 RTT** to get a response (1 for TCP, 1 for request/response); teardown is off the critical path |
| "Statelessness means no cookies or sessions" | Servers may keep state above HTTP (cookies, tokens); the *protocol* doesn't require it |
| "A 404 means the server is down" | A 404 means the server is up and says the resource doesn't exist |

---

## 10. Summary

- **HTTP** = a text request/response protocol over TCP. A message = start line + headers + blank line + optional body.
- **HTTP/1.0 (1996, RFC 1945):** methods `GET`, `HEAD`, `POST`; status codes; headers; content types.
- Life cycle: DNS → TCP handshake → request → server processes → response → server closes the connection → repeat for every resource.
- **Cost:** ≈ 2 RTT per resource on a new connection, plus TCP slow start and server overhead; browsers mitigated it with a few parallel connections.
- **Fixes:** persistent connections (1.1), multiplexing (HTTP/2), QUIC (HTTP/3).
- HTTP is **stateless**, **extensible** (headers), and best learned by typing requests with `nc`/`curl` and watching them with `tcpdump`.

---

## 11. Check your understanding

1. What are the three parts of an HTTP request message, and what marks the end of the headers?
2. Which status code class means "client error"? What is the difference between 401 and 403?
3. Why does HTTP run on TCP rather than UDP?
4. In HTTP/1.0, how does the client know a response body has ended if there is no `Content-Length`?
5. Compute the minimum time to fetch 6 resources one after another on a fresh connection each, with RTT = 80 ms (ignore data transfer time).
6. Give two costs of one-connection-per-request besides latency.
7. What does `HEAD` do and when would you use it?

<details>
<summary>Answers</summary>

1. Request line, headers, and optional body, separated by a blank line (`\r\n\r\n` after the last header).
2. 4xx. 401 = authentication required/failed ("who are you?"); 403 = understood and refused ("you may not").
3. HTTP needs reliable, ordered delivery of complete messages, which TCP provides (HTTP/3 uses QUIC, which reimplements it over UDP).
4. The server closes the TCP connection; the client reads until end-of-stream.
5. About 2 RTT per resource: 6 × 2 × 80 ms = **960 ms** (1 RTT for the handshake + 1 RTT for the request/response).
6. TCP slow start on each fresh connection; server resources (sockets, `TIME_WAIT`); extra packets; repeated headers. (Any two.)
7. It returns only the headers of a `GET` response, useful to check existence, size or modification time without downloading the body.
</details>

**Practice**

1. Use `nc` to request a page of your choice with HTTP/1.0 and identify each line of the response.
2. Run the Python server, then use `curl -v` for a valid path, a missing path, `HEAD`, and `DELETE`; explain each status code you get.
3. Capture two consecutive `curl` requests with `tcpdump`; prove that each used a different TCP connection (different source ports) and that the server sent the first FIN.
4. Extend the server to serve real files from a directory (careful with paths like `../`: what is "directory traversal" and how do you prevent it?), sending the correct `Content-Type`.
5. Measure with `curl -w '%{time_connect} %{time_starttransfer} %{time_total}\n'` against a remote site and relate the numbers to RTTs.

---

**Next:** [Chapter 26 – HTTP/1.1 in Detail](26_http_1_1_in_details.md)
