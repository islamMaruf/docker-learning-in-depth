# Chapter 28: DNS (the Domain Name System) in Detail

> **In one sentence:** DNS is the internet's distributed, hierarchical, cached "phone book": it turns names people can remember (`www.example.com`) into the IP addresses (and other facts) that computers need, through a chain of resolvers and name servers that each know one small piece.

**Level:** 🟢 Beginner → 🟡 Intermediate → 🔴 Expert · **Reading time:** ~70 minutes

**Prerequisites:** [Chapter 22](22_tcp_ip_model.md) and [Chapter 24](24_udp_in_details.md) (UDP and TCP); [Chapter 25](25_http_1_0_in_details.md) (URLs).

---

## What you will learn

- Why DNS exists, and the history that led to it (`HOSTS.TXT`)
- The parts of a **URL** and of a **domain name** (labels, TLD, FQDN, the trailing dot)
- The **hierarchy**: root, TLD, authoritative servers, and the roles of **stub**, **recursive resolver**, and **authoritative** servers
- The complete **resolution process**, with the layers of **caching** and **TTL**
- **Record types** (A, AAAA, CNAME, MX, TXT, NS, SOA, PTR, SRV, CAA...), zone files and **glue**
- The **DNS message** and response codes; UDP vs TCP; **EDNS**, DoT, DoH
- **Security**: cache poisoning, DNSSEC, hijacking, amplification, privacy
- DNS on your machine (`/etc/hosts`, `resolv.conf`, `nsswitch.conf`, systemd-resolved), in **Docker** and **Kubernetes**
- Operations: TTL strategy, migrations, load balancing, troubleshooting
- Hands-on labs with `dig`, `host`, `getent`, `tcpdump`, and a local DNS server in Docker

---

## 1. Why DNS exists

Computers route by **IP addresses** (`93.184.216.34`, `2606:2800:220:1:248:1893:25c8:1946`). People remember **names** (`example.com`). Something has to translate, and it has to do so for billions of names, updated constantly, worldwide, in milliseconds.

### Before DNS: one file
In the ARPANET era (1970s to 1983) a single file, **`HOSTS.TXT`**, maintained by the Stanford Research Institute's Network Information Center, listed every host name and address. Everyone downloaded it periodically. That failed to scale:

| Problem | Why |
|---|---|
| **Size and traffic** | Every machine kept downloading the whole file |
| **Slow updates** | Changes went through one central maintainer; copies were stale for days |
| **Name collisions** | A flat namespace with one authority |
| **Single point of failure** | One file, one place |

In **1983** Paul Mockapetris designed **DNS** (RFCs 882/883, replaced by **RFC 1034/1035 in 1987**, which are still the core of DNS today) with three ideas that solved all of these:

1. **A hierarchy of names** (`www.example.com`), so each organization manages its own part.
2. **Distributed authority** (delegation): thousands of servers each responsible for a slice.
3. **Caching** with expiry times, so most lookups never leave your network.

(The old file lives on as `/etc/hosts` on your machine, which you can still use for overrides.)

---

## 2. Anatomy of a URL and a domain name

```
   https://blog.example.com:443/path/to/page?query=value#section
   └─┬──┘   └──────┬───────┘└┬┘└─────┬─────┘└─────┬─────┘└──┬──┘
   scheme        host       port    path       query    fragment
```

The **host** part is what DNS resolves. A domain name is a sequence of **labels** separated by dots, read **right to left** from most general to most specific:

```
   blog . example . com .          ← the trailing dot is the DNS ROOT
    │        │       │
    │        │       └ top-level domain (TLD)
    │        └──────── second-level domain (what you register)
    └───────────────── subdomain (whatever the owner of example.com creates)
```

| Term | Meaning |
|---|---|
| **Label** | One part between dots (1–63 characters: letters, digits, hyphen; case-insensitive) |
| **FQDN** (fully qualified domain name) | The complete name, formally ending in a dot: `blog.example.com.` The final dot names the **root**; applications usually add it silently |
| **TLD** | Top-level domain. **gTLD**: `.com`, `.org`, `.net`, `.dev`, `.app`... **ccTLD** (country code): `.uk`, `.de`, `.jp`, `.bd`... |
| **Second-level domain (SLD)** | `example` in `example.com`; what you **register** |
| **Subdomain** | Any label added to the left; the owner can create unlimited ones for free (`blog.`, `api.`, `staging.`) |
| **Apex / bare / root domain** | `example.com` itself (the "zone apex") |
| **Registrar / registry / registrant** | The company you buy from (Namecheap...) / the organization that runs the TLD (Verisign for `.com`) / you, the owner |

Limits: a full name is at most **253 characters**. Names may contain **internationalized characters** (e.g. `münchen.de`) which are encoded as **Punycode** (`xn--mnchen-3ya.de`).

**Public suffixes:** for `bbc.co.uk` the *registrable* domain is `bbc.co.uk` because `co.uk` is a "public suffix" (like a TLD). Browsers and cookie logic use the **Public Suffix List** to decide what counts as one "site".

Not everything is `www`: `www` is just a customary subdomain, and `example.com` and `www.example.com` are **two different names** that the owner typically points to the same place.

---

## 3. Who is who in DNS

```
   Your app ─► STUB RESOLVER ─► RECURSIVE RESOLVER ─────────► ROOT servers        "ask the .com servers"
   (browser/OS)  (in your OS)    (ISP, company, 8.8.8.8,  ─► TLD servers (.com)  "ask example.com's servers"
                                  1.1.1.1, 9.9.9.9)       ─► AUTHORITATIVE       "here's the answer"
                                                              servers for example.com
```

| Role | Job |
|---|---|
| **Stub resolver** | The tiny resolver in your OS/library (`getaddrinfo()`); it just asks a configured recursive resolver and waits for the final answer |
| **Recursive resolver** (a.k.a. caching resolver, "DNS server" of your ISP, Google Public DNS `8.8.8.8`, Cloudflare `1.1.1.1`, Quad9 `9.9.9.9`, or one you run: Unbound, dnsmasq, BIND) | Does the *legwork*: walks the hierarchy on your behalf, **caches** every answer, and returns the result |
| **Root servers** | Know only where the **TLD** servers are. **13 named server "letters" (a–m.root-servers.net)** but over **1,700 physical instances** worldwide using **anycast** (the same IP is announced from many places; you reach the nearest) |
| **TLD servers** | Know which **authoritative servers** handle each domain in that TLD (e.g. `.com`'s servers know example.com's name servers). They give **referrals**, not final answers |
| **Authoritative servers** | Hold the **actual records** for a zone (`example.com`): the source of truth. Run by the owner, or by a DNS host (Cloudflare, Route 53, Azure DNS, Google Cloud DNS, your registrar) |

Two query styles:

- **Recursive query** (stub → resolver): "give me the *final* answer, and do whatever it takes." Flag `RD` (recursion desired).
- **Iterative queries** (resolver → root/TLD/authoritative): each server answers with either the answer or **a referral**: "I don't know, but ask these servers." The resolver follows the referrals itself.

---

## 4. The resolution process, step by step

You type `www.example.com` in a browser.

### 4.1 On your machine

1. **Browser cache.** Browsers keep their own short-lived DNS cache (Chrome: `chrome://net-internals/#dns`; entries obey TTL, with a floor of about a minute in some browsers).
2. **Operating system.** The browser calls `getaddrinfo("www.example.com")` in the C library, which follows `/etc/nsswitch.conf` (`hosts: files dns`) to decide the sources and order:
   1. **`/etc/hosts`** (local static entries; checked first)
   2. **DNS**: the OS or its local caching service (systemd-resolved, macOS `mDNSResponder`, Windows DNS Client) checks *its* cache and, if empty, asks the configured resolver(s) from `/etc/resolv.conf` (Linux), the network settings (Windows/macOS), or the ones handed out by **DHCP** (Chapters 35–37).
   (Note: modern browsers can bypass the OS and use their **own** resolver, including **DNS over HTTPS**.)
3. The stub sends a **UDP query to port 53** of the recursive resolver.

### 4.2 At the recursive resolver (cache miss)
Assume it has nothing cached (a completely cold cache); real resolvers already know the root and often the `.com` servers:

```
 resolver ──► root server:            "www.example.com A?"
          ◄── referral:               "I don't know, but .com is handled by a.gtld-servers.net (…and others). Here are their IPs (glue)"
 resolver ──► .com TLD server:        "www.example.com A?"
          ◄── referral:               "example.com is handled by a.iana-servers.net, b.iana-servers.net"
 resolver ──► example.com authoritative server:  "www.example.com A?"
          ◄── ANSWER (flag AA):       "www.example.com. 3600 IN A 93.184.216.34"
```

The resolver **caches** each piece (the `.com` servers for two days, `example.com`'s name servers, the final answer for its TTL), then returns the answer to the stub, which caches it in turn and hands it to the browser.

### 4.3 Then the real work begins
Only now can the browser open a TCP (and TLS) connection to `93.184.216.34` and send the HTTP request. **DNS is on the critical path of every new connection to a new name**. (That's why `<link rel="dns-prefetch">` and `preconnect` exist.)

### 4.4 Typical timing
| Situation | Time |
|---|---|
| Answer in the browser or OS cache | ~0–1 ms |
| Answer in the resolver's cache (very common) | ~1–30 ms (one round trip to the resolver) |
| Cold resolution walking root → TLD → authoritative | ~50–300 ms (several round trips to distant servers) |

### 4.5 Why the design scales
- The **root servers** are only asked for **TLD delegations**, and those are cached for **2 days**, so they see a small fraction of the world's queries.
- A resolver serving thousands of users answers most queries from cache: the first user to look up `example.com` pays the cost, and everyone else within the **TTL** benefits.
- Authority is **delegated**: `.com` delegates `example.com` to its owner, who can delegate `dev.example.com` further. No one runs "all of DNS".

### 4.5b Negative caching
"That name does not exist" is cached too (**NXDOMAIN**), for the time given by the zone's **SOA minimum/negative TTL**. That's why a newly created record may "not exist" for a few minutes for someone who asked just before you created it.

---

## 5. TTL and caching

Every record carries a **TTL** (time to live, in seconds): how long a cache may keep it.

```
www.example.com.   300   IN   A   93.184.216.34
                   ↑ TTL
```

Caches count down: `dig` shows the *remaining* TTL when the answer comes from a cache (run it twice and watch the number fall). When it hits 0, the record must be re-fetched.

| TTL | Pros | Cons | Use for |
|---|---|---|---|
| **Low** (30–300 s) | Changes take effect quickly; easy failover | More queries, slightly slower on average, higher DNS cost | Records you may need to change fast: load balancers, failover, migrations |
| **High** (3600–86400 s) | Fewer queries, better resilience if authoritative servers are unreachable | Changes are slow to take effect | Stable records (MX, NS, most static sites) |

**Important truths**

- **"DNS propagation" is a myth of waiting caches.** There is no push; each cache independently expires. A change takes *up to the old TTL* to be seen everywhere (some resolvers or applications hold longer than the TTL. Java's default and some libraries are examples).
- **Migration recipe:** 1–2 days before, lower the TTL (e.g. 60 s); wait for the old TTL to pass; change the record; verify; then raise the TTL again.
- **Layers:** application (browser, Java, Node) → OS cache → local forwarder → recursive resolver. Flushing "DNS cache" on your machine clears only some of them.

---

## 6. DNS records

Authoritative servers store **resource records (RRs)**: `name  TTL  class  type  data`. Class is almost always `IN` (Internet).

| Type | Purpose | Example |
|---|---|---|
| **A** | Name → IPv4 address | `www  300 IN A  93.184.216.34` |
| **AAAA** | Name → IPv6 address | `www  300 IN AAAA 2606:2800:220:1:248:1893:25c8:1946` |
| **CNAME** | Alias: "this name is really *that* name" | `blog  300 IN CNAME  sites.hosting.example.` |
| **MX** | Mail servers for a domain, with **priority** (lower = preferred) | `@ IN MX 10 mail1.example.com.` / `@ IN MX 20 mail2.example.com.` |
| **TXT** | Free text, widely used for verification and email policy | `@ IN TXT "v=spf1 include:_spf.google.com ~all"` |
| **NS** | The authoritative name servers for a zone (delegation) | `@ IN NS ns1.example.com.` |
| **SOA** | Zone metadata: primary server, admin e-mail, **serial**, refresh/retry/expire, negative-cache TTL | one per zone |
| **PTR** | Reverse lookup: IP → name (`34.216.184.93.in-addr.arpa.`) | used by mail servers and logs |
| **SRV** | Service location (host + port + priority + weight) | `_sip._tcp IN SRV 10 60 5060 sip.example.com.` (also used by Kubernetes, XMPP, Active Directory) |
| **CAA** | Which certificate authorities may issue TLS certificates | `@ IN CAA 0 issue "letsencrypt.org"` |
| **HTTPS / SVCB** | Service binding: advertises HTTP/2, HTTP/3, ports, ECH | modern browsers use it |
| **DS / DNSKEY / RRSIG / NSEC(3)** | DNSSEC (section 9.2) | |

Email-related TXT records: **SPF** (`v=spf1 ...`, which servers may send mail for the domain), **DKIM** (`selector._domainkey`, public key for message signatures), **DMARC** (`_dmarc`, policy for failures).

### Rules and gotchas
- **CNAME cannot coexist with other records at the same name**, and therefore **cannot be used at the zone apex** (`example.com`), which needs `SOA` and `NS`. Providers offer **ALIAS/ANAME/"CNAME flattening"** as a workaround.
- **MX and NS targets must be names with A/AAAA records**, not CNAMEs.
- A name can have **several A records**: resolvers return them (often rotated), a crude form of **round-robin load balancing**.
- **Wildcards**: `*.example.com IN A ...` matches any name that has no more specific record.
- **Glue records**: if `example.com`'s name server is `ns1.example.com`, the parent (`.com`) must also supply that server's IP ("glue"), otherwise you'd need `example.com` to find the server for `example.com`, an infinite loop.

### A zone file

```
$ORIGIN example.com.
$TTL 3600
@    IN SOA ns1.example.com. hostmaster.example.com. (
          2025010101 ; serial   (increase on every change)
          7200       ; refresh  (how often secondaries check)
          3600       ; retry
          1209600    ; expire
          300 )      ; negative-caching TTL

     IN NS    ns1.example.com.
     IN NS    ns2.example.com.

@    IN A     192.0.2.10
@    IN AAAA  2001:db8::10
www  IN CNAME @                 ; (or point at the same A record)
mail IN A     192.0.2.20
@    IN MX 10 mail.example.com.
@    IN TXT   "v=spf1 mx -all"
api  300 IN A 192.0.2.30
*.dev IN A    192.0.2.40
```

Zones are replicated from a **primary** to **secondary** servers by **zone transfers** (AXFR/IXFR over **TCP**), keyed by the SOA serial number.

---

## 7. The DNS message, ports and transport

A DNS message, the same shape for queries and responses, is tiny:

```
┌───────────────────────────────────────────────┐
│ Header (12 bytes): ID | flags | QDCOUNT | ANCOUNT | NSCOUNT | ARCOUNT
├───────────────────────────────────────────────┤
│ Question section:   name, type, class           e.g. www.example.com. A IN
├───────────────────────────────────────────────┤
│ Answer section:     resource records
├───────────────────────────────────────────────┤
│ Authority section:  NS records (referrals), SOA
├───────────────────────────────────────────────┤
│ Additional section: extra useful records (glue, EDNS OPT)
└───────────────────────────────────────────────┘
```

Header flags worth knowing: `QR` (query/response), `AA` (authoritative answer), `TC` (truncated: retry over TCP), `RD` (recursion desired), `RA` (recursion available), `AD` (authenticated data: DNSSEC verified), and the **RCODE**:

| RCODE | Name | Meaning |
|---|---|---|
| 0 | **NOERROR** | Success (may still have zero answers: "NODATA": the name exists but not that type) |
| 2 | **SERVFAIL** | The server couldn't complete the lookup (broken delegation, unreachable servers, DNSSEC failure) |
| 3 | **NXDOMAIN** | The name doesn't exist |
| 5 | **REFUSED** | The server won't answer you (policy) |

### UDP first, TCP when needed
DNS uses **port 53**. Queries normally use **UDP** because a typical exchange is one small question and one small answer: no handshake, no connection state on busy servers, and one round trip. A TCP lookup would need at least an extra round trip for the handshake (plus close). (Rough cost: about 2 round trips on TCP versus 1 on UDP; not "6×".) A UDP client that gets no answer simply **retries** (typically after ~1–2 s, or another server).

**DNS falls back to TCP** when:
1. The UDP answer is **truncated** (`TC=1`): originally at 512 bytes; with **EDNS(0)** the client advertises a larger buffer (commonly 1232 bytes since the 2020 "DNS Flag Day" recommendation, to avoid IP fragmentation). Big responses (DNSSEC, many records) trigger it.
2. **Zone transfers** (AXFR/IXFR).
3. The server or policy requires it, and for **DNS over TLS** (TCP 853), **DNS over HTTPS** (TCP 443 / HTTP/2 or HTTP/3), **DNS over QUIC** (UDP 853).

Server operators must allow **both UDP and TCP** on port 53.

---

## 8. DNS on your machine, in Docker and in Kubernetes

### 8.1 Linux/macOS/Windows
| File / tool | Purpose |
|---|---|
| `/etc/hosts` (Windows: `C:\Windows\System32\drivers\etc\hosts`) | Static name → IP entries checked **before** DNS (`127.0.0.1 localhost`); handy for overrides |
| `/etc/nsswitch.conf` | Lookup order: `hosts: files dns` (files = `/etc/hosts`) |
| `/etc/resolv.conf` | Resolver settings: `nameserver 1.1.1.1`, `search corp.example.com` (suffixes tried for short names), `options ndots:1 timeout:2 attempts:3` |
| systemd-resolved (Ubuntu and others) | A local caching stub at **127.0.0.53**; `resolvectl status`, `resolvectl query example.com`, `resolvectl flush-caches` |
| macOS | `scutil --dns`; flush: `sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder` |
| Windows | `ipconfig /displaydns`, `ipconfig /flushdns` |

The **`search` list** and **`ndots`** matter: with `ndots:5` (Kubernetes default) a name like `api.example.com` (2 dots < 5) is first tried with each search suffix (`api.example.com.default.svc.cluster.local`, ...), producing several failing queries before the real one, which is a classic source of slowness. A trailing dot (`api.example.com.`) makes a name absolute and skips the search list.

### 8.2 Docker
- On the **default bridge network**, containers copy the host's resolvers into `/etc/resolv.conf`.
- On **user-defined networks** Docker runs an **embedded DNS server** at **127.0.0.11** in each container; it resolves **container names, service names and network aliases** to container IPs and forwards everything else to the host's resolvers. That's why `ping db` works from another container on the same user-defined network (but not on the default bridge).
- Override with `--dns 1.1.1.1`, `--dns-search example.com`, `--add-host myhost:10.0.0.5` (writes `/etc/hosts`), or the daemon's `/etc/docker/daemon.json` (`{"dns": ["1.1.1.1"]}`).
- Common problem: containers can't resolve names because the host uses a VPN/corporate resolver or a `127.0.0.53` stub, so set `--dns` or fix the daemon config. Also check that UDP/53 isn't blocked.

### 8.3 Kubernetes
Each pod's resolver points to the cluster DNS service (CoreDNS). **Services** get names like `my-svc.my-namespace.svc.cluster.local` → the Service's cluster IP; short names work via the `search` list. Headless services return pod IPs; `SRV` records expose ports. Watch `ndots:5` (above) and CoreDNS latency in busy clusters.

---

## 9. Security and privacy

### 9.1 Attacks
| Attack | How | Defenses |
|---|---|---|
| **Cache poisoning** (Kaminsky, 2008) | Race the real answer with forged responses to get a false record cached (e.g. bank → attacker IP) | Source-port and query-ID randomization, 0x20 case randomization, **DNSSEC**, DNS cookies; keep resolvers patched |
| **DNS hijacking** | Malware/router compromise changes your resolver settings, or an attacker takes over registrar/DNS-provider accounts | Strong 2FA and registry locks, monitor NS changes, DoH/DoT, secure routers |
| **Amplification / reflection DDoS** | Spoofed-source small queries (`ANY`, large TXT/DNSSEC records) to **open resolvers** flood the victim with big answers | Don't run open resolvers, response-rate limiting, BCP38 filtering, minimal `ANY` responses |
| **Subdomain takeover** | A dangling CNAME points to a deleted cloud resource that an attacker re-claims | Remove stale records; inventory DNS |
| **Typosquatting / homograph** | Look-alike domains, Unicode confusables | Browser IDN policy, monitoring |
| **Tunneling/exfiltration** | Data hidden in DNS queries | DNS monitoring, egress control |
| **Outages of the DNS provider** | (The 2016 Dyn attack took down many major sites) | Multiple DNS providers, sensible TTLs |

### 9.2 DNSSEC (integrity)
**DNSSEC** adds **digital signatures** to DNS data so a validating resolver can prove an answer really came from the zone owner and wasn't altered: `RRSIG` signatures over each record set, `DNSKEY` public keys, and `DS` records in the parent zone forming a **chain of trust** from the **root** down. It provides **authenticity and integrity, not confidentiality**. Adoption is partial; a mistake (expired signatures, wrong `DS`) makes a domain **unreachable** for validating resolvers (`SERVFAIL`). Check with `dig +dnssec example.com` and look for the `ad` flag when asking a validating resolver.

### 9.3 Privacy: DoT and DoH
Classic DNS is **unencrypted**: the local network and your ISP see every name you look up. **DNS over TLS** (DoT, port 853, RFC 7858), **DNS over HTTPS** (DoH, RFC 8484, port 443), and **DNS over QUIC** encrypt the stub-to-resolver hop. This protects against eavesdropping and tampering on that leg, but the resolver still sees your queries (choose one you trust), and DoH inside browsers can bypass corporate DNS policy. (Encrypted ClientHello, ECH, helps hide the name in TLS as well.)

---

## 10. Operations and design

- **Use two or more NS records** on different networks/providers for resilience; anycast DNS hosts help.
- **Keep the SOA serial increasing** on each change (a date-based `YYYYMMDDnn` format is common).
- **DNS-based load balancing:** multiple A records (round-robin), **weighted/latency/geo-based** answers (Route 53, Cloudflare), **health-checked failover**. But clients and resolvers cache, so DNS gives coarse and slow control; combine with a real load balancer.
- **Split-horizon (split-brain) DNS:** answer differently for internal and external clients (internal IPs inside the company).
- **Private zones:** cloud VPC private DNS; Kubernetes' `cluster.local`.
- **Email deliverability** depends on correct MX, SPF, DKIM, DMARC, and PTR.
- **Automating certificates:** ACME **DNS-01** challenges create a TXT record `_acme-challenge.example.com` to prove control (works for wildcards).
- **Performance tips:** reduce the number of distinct hostnames a page uses; `dns-prefetch`/`preconnect` for third parties; run a local caching resolver on servers with heavy lookups; avoid `ndots` traps; use connection reuse (HTTP/2) so you resolve less often.

---

## 11. Hands-on labs

Install tools if needed: `apt-get install -y dnsutils` (Debian/Ubuntu), `dnf install bind-utils`, `brew install bind`.

**Lab 1: Basic lookups**

```bash
dig example.com                 # full response: QUESTION, ANSWER, AUTHORITY, ADDITIONAL; note "Query time" and the TTL
dig +short example.com          # just the answer
dig example.com AAAA +short
dig example.com MX +short
dig example.com TXT +short
dig example.com NS +short
dig example.com SOA +short
host example.com                # friendly summary
getent hosts example.com        # what YOUR OS resolver (nsswitch: hosts, files, dns) returns; what apps see
```

**Lab 2: Watch caching and TTL**

```bash
dig example.com | grep -A1 'ANSWER SECTION'; sleep 5; dig example.com | grep -A1 'ANSWER SECTION'
# the TTL number decreases if answered from a cache; compare with an authoritative answer:
dig @$(dig +short NS example.com | head -1) example.com | grep -A1 'ANSWER SECTION'     # full TTL, flag "aa"
```

**Lab 3: Trace the hierarchy (what a resolver does)**

```bash
dig +trace example.com
# root servers (.) → NS for com. → NS for example.com. → final A record
```

Read each stage: who was asked, what it returned (referral or answer), and the TTLs (2 days on the delegations).

**Lab 4: Choose the resolver and compare**

```bash
for s in 8.8.8.8 1.1.1.1 9.9.9.9; do echo -n "$s: "; dig @$s example.org | grep 'Query time'; done
dig @1.1.1.1 www.github.com CNAME +short
```

**Lab 5: Watch the packets**

```bash
sudo tcpdump -i any -nn port 53 &
dig example.com +norecurse @198.41.0.4         # ask a root server (a.root-servers.net) directly: you get a referral to .com, not an answer
dig example.com +bufsize=512 +tcp              # force TCP; see the handshake and the length-prefixed messages
dig example.com                                 # 1 UDP query + 1 UDP response
```

**Lab 6: NXDOMAIN, SERVFAIL, and flags**

```bash
dig doesnotexist.example.com | grep -E 'status|flags'       # status: NXDOMAIN
dig dnssec-failed.org | grep status                         # SERVFAIL on a validating resolver (deliberately broken DNSSEC test domain)
dig +dnssec example.com | grep -E 'flags|RRSIG'            # ad flag and signatures when validated
```

**Lab 7: Reverse DNS**

```bash
dig -x 8.8.8.8 +short          # dns.google.
host 1.1.1.1
```

**Lab 8: `/etc/hosts` override and the search order**

```bash
echo '127.0.0.1 myapp.test' | sudo tee -a /etc/hosts
getent hosts myapp.test         # 127.0.0.1 (from files)
dig myapp.test +short           # empty! dig queries DNS directly and ignores /etc/hosts
ping -c1 myapp.test
sudo sed -i '/myapp.test/d' /etc/hosts
```

**Lab 9: DNS in Docker**

```bash
docker network create labnet
docker run -d --name web --network labnet nginx:1.27-alpine
docker run --rm --network labnet alpine:3.20 sh -c 'cat /etc/resolv.conf; nslookup web; wget -qO- http://web | head -3'   # nameserver 127.0.0.11
docker run --rm alpine:3.20 sh -c 'cat /etc/resolv.conf; nslookup web'      # default bridge: cannot resolve "web"
docker run --rm --dns 9.9.9.9 --add-host demo.local:10.1.2.3 alpine:3.20 sh -c 'cat /etc/resolv.conf /etc/hosts | tail -4'
docker rm -f web; docker network rm labnet
```

**Lab 10: Run your own authoritative server + resolver locally with CoreDNS or dnsmasq**

```bash
mkdir dns-lab && cd dns-lab
cat > Corefile << 'EOF'
lab.test:1053 {
    file /zone.db
    log
}
.:1053 {
    forward . 1.1.1.1
    cache 30
    log
}
EOF
cat > zone.db << 'EOF'
$ORIGIN lab.test.
@   3600 IN SOA ns.lab.test. admin.lab.test. 1 7200 3600 1209600 300
    3600 IN NS  ns.lab.test.
ns  3600 IN A   127.0.0.1
www 60   IN A   10.10.10.10
api 60   IN CNAME www
EOF
docker run -d --name coredns -p 1053:1053/udp -p 1053:1053/tcp \
  -v "$PWD/Corefile":/Corefile:ro -v "$PWD/zone.db":/zone.db:ro coredns/coredns:1.11.3 -conf /Corefile
dig @127.0.0.1 -p 1053 www.lab.test +noall +answer          # 10.10.10.10, flag aa (authoritative)
dig @127.0.0.1 -p 1053 api.lab.test +noall +answer          # CNAME → www → A
dig @127.0.0.1 -p 1053 example.com +noall +answer           # forwarded to 1.1.1.1 and cached 30 s
docker logs coredns | tail; docker rm -f coredns
```

You've just run an authoritative server and a forwarding cache. Change the record TTL to 5 s and edit the zone to watch clients re-query.

**Lab 11: Measure DNS cost in a page load.** In Chrome DevTools → Network → click a request → Timing → "DNS Lookup"; or `curl -w 'dns %{time_namelookup}s connect %{time_connect}s total %{time_total}s\n' -o /dev/null -s https://example.com`. Repeat: DNS time drops to ~0 once cached by your OS/resolver.

---

## 12. Troubleshooting

| Symptom | Likely cause | What to do |
|---|---|---|
| `Could not resolve host` / `Temporary failure in name resolution` | No resolver reachable, wrong `resolv.conf`, network down, firewall blocking UDP/53 | `cat /etc/resolv.conf`, `ping 1.1.1.1` (IP works?), `dig @1.1.1.1 example.com` |
| Works with `dig` but not in the app (or vice versa) | `dig` bypasses `/etc/hosts`, nsswitch and the app's own caches | Test with `getent hosts name`; check `/etc/hosts`, the app's DNS cache (JVM `networkaddress.cache.ttl`), Docker's DNS |
| Old IP after a change | Cached record (TTL not expired), or the app cached it | `dig +norecurse`, check TTL; flush the OS/browser cache; wait out the TTL; lower TTL before the next change |
| Different answers from different resolvers | Geo/latency DNS, stale caches, split-horizon, poisoned or filtering resolver | Compare `dig @8.8.8.8` vs `@1.1.1.1` vs authoritative |
| `NXDOMAIN` for a name that should exist | Typo, record missing at the *authoritative* server, delegation (NS) wrong, negative caching | `dig +trace`, query the authoritative servers directly |
| `SERVFAIL` | Broken delegation, unreachable authoritative servers, DNSSEC validation failure | `dig +trace`, `dig +cd` (checking disabled), check DS records |
| Slow first request; fast afterwards | Cold resolution, or a dead first resolver in `resolv.conf` (timeout 5 s per try) | Reorder or fix resolvers; use a local cache |
| Slow lookups in Kubernetes | `ndots:5` search expansion, CoreDNS overloaded | Use FQDNs with a trailing dot, tune `ndots`, node-local DNS cache |
| Email rejected/spam | Missing/incorrect MX, SPF, DKIM, DMARC, PTR | Check each record with `dig TXT`, `dig -x` |
| Certificate issuance fails (DNS-01) | TXT record not published/propagated, wrong provider | `dig TXT _acme-challenge.example.com @authoritative` |
| Container can't resolve names | Host resolver is `127.0.0.53`/VPN, blocked UDP/53, default bridge (no name resolution between containers) | `--dns`, user-defined network, check `/etc/resolv.conf` inside the container |

---

## 13. Common misconceptions

| Misconception | Reality |
|---|---|
| "DNS only maps names to IPv4 addresses" | It stores many record types (AAAA, MX, TXT, SRV, CAA, ...) |
| "There are only 13 root servers" | 13 *addresses/names* (a–m), but 1,700+ anycast instances |
| "Root servers know where facebook.com is" | They know only who runs each TLD; TLD servers know who runs the domain; the domain's servers know the answer |
| "DNS changes take 24–48 hours to propagate" | Only as long as the **old TTL** (and buggy caches); with a low TTL, minutes |
| "DNS uses UDP because it's faster" | Because one small query/answer needs no connection; it uses **TCP** when answers are large, for zone transfers, and for DoT/DoH |
| "The browser talks directly to root/TLD servers" | Your **recursive resolver** does the walking; your stub asks it |
| "`www` is required" | It's just a subdomain by convention |
| "A CNAME can be at the apex" | It can't (it would conflict with SOA/NS records) |
| "Each domain has one IP" | It can have many (round-robin, geo, anycast) |
| "DNSSEC encrypts DNS" | It signs records for authenticity; it doesn't hide anything. Use DoT/DoH for privacy |
| "`dig` shows what my app sees" | `dig` talks straight to DNS; apps use the OS resolver (`getaddrinfo`), `/etc/hosts`, caches, and search lists |
| "Flushing my DNS cache fixes everything" | Only your local layers; the recursive resolver and others may still hold the old data until TTL expires |

---

## 14. Summary

- **DNS** = a distributed, hierarchical, cached database mapping names to records; born in 1983 to replace `HOSTS.TXT`, defined by RFC 1034/1035.
- Names are read **right to left**: root (`.`) → TLD → domain → subdomain. **Delegation** gives each organization control of its slice.
- Roles: **stub resolver** (your OS) → **recursive resolver** (does the work and caches) → **root → TLD → authoritative** servers (referrals, then the answer).
- **Caching with TTLs** at every layer makes it fast; changes take as long as the old TTL; failures are cached too (negative caching).
- **Records:** A, AAAA, CNAME, MX, TXT, NS, SOA, PTR, SRV, CAA, HTTPS; CNAME can't live at the apex; glue records solve the chicken-and-egg problem of name servers inside their own zone.
- Transport: **UDP 53** by default, **TCP 53** for large answers, transfers; **DoT/DoH/DoQ** for privacy; **DNSSEC** for authenticity.
- Docker: embedded DNS at **127.0.0.11** for user-defined networks; Kubernetes: CoreDNS and `ndots`.
- Tools: `dig`, `dig +trace`, `getent hosts`, `resolvectl`, `tcpdump port 53`.

---

## 15. Check your understanding

1. Read `mail.eu.example.co.uk.` from right to left and name each part. What does the trailing dot mean?
2. What is the difference between a stub resolver, a recursive resolver, and an authoritative server?
3. What do root and TLD servers return to a resolver looking up `www.example.com`?
4. You change an A record with TTL 3600 at 10:00. When can everyone be sure to see the new value, and how would you have made it faster?
5. Why can't you put a CNAME at `example.com`, and what can you use instead?
6. When does DNS use TCP?
7. `dig example.com` works, but `curl example.com` fails to resolve on the same machine. Give two possible causes.
8. What does a container on a user-defined Docker network use to resolve other containers, and what address does its `resolv.conf` show?
9. What does DNSSEC protect against, and what does DoH protect against? Do they overlap?
10. Why do resolvers cache NXDOMAIN answers?

<details>
<summary>Answers</summary>

1. Root `.` → `uk` (ccTLD) → `co.uk` (public suffix/second-level) → `example` (the registered domain) → `eu` (subdomain) → `mail` (host). The trailing dot denotes the DNS root; the name is fully qualified.
2. Stub: OS library that asks a configured resolver and waits. Recursive resolver: walks the hierarchy for you and caches the results. Authoritative: holds the zone's real records and gives definitive answers.
3. Root: a referral to the `.com` TLD servers (with glue). TLD: a referral to example.com's authoritative name servers. Only the authoritative server returns the actual A record.
4. Up to 11:00 (old TTL 1 hour): every cache holding the old record expires by then. Lower the TTL (e.g. to 60 s) a day before the change, wait out the old TTL, then change it.
5. A CNAME can't coexist with the SOA/NS records the apex requires. Use A/AAAA records, or the provider's ALIAS/ANAME/CNAME-flattening feature.
6. Truncated (large) responses, zone transfers (AXFR/IXFR), and DNS over TLS (also DoH/DoQ variants use TCP/QUIC); also when a server or policy requires it.
7. `dig` bypasses `/etc/hosts` and nsswitch/OS caches, so the system resolver (or `resolv.conf`, `/etc/hosts` entry, search domain, broken local stub) is the problem, or curl uses a different resolver/proxy/IPv6 path.
8. Docker's embedded DNS server; `resolv.conf` shows `nameserver 127.0.0.11`.
9. DNSSEC: forged or altered answers (authenticity/integrity). DoH: eavesdropping/tampering on the client-to-resolver hop (confidentiality). They address different threats and complement each other.
10. To avoid hammering authoritative servers with repeated queries for names that don't exist (governed by the SOA negative-caching TTL).
</details>

**Practice**

1. Use `dig +trace` for three domains (a `.com`, a ccTLD and a subdomain) and draw the referral chain for each, with TTLs.
2. Set up your own domain or a free subdomain at a DNS host (or use the CoreDNS lab), create A, CNAME, MX and TXT records, and verify each with `dig` at the authoritative server and at 8.8.8.8.
3. Lower a record's TTL to 30 s, change it, and watch `dig @8.8.8.8` until the new value appears; measure how long it takes.
4. Capture a DNS query and response with Wireshark; identify the transaction ID, flags, question, and answer; then force a truncated response (`+bufsize=512` on a large TXT record) and see the TCP retry.
5. In Docker, compare name resolution between two containers on the default bridge and on a user-defined network. Explain the difference.

---

**Next:** [Chapter 29 – TLS (Transport Layer Security) in Detail](29_tls_transport_layer_security_in_details.md)
