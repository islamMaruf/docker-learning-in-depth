# Chapter 29: TLS (Transport Layer Security) in Detail

> **In one sentence:** TLS wraps a TCP connection in a secure channel that gives you **confidentiality** (nobody can read the data), **integrity** (nobody can change it unnoticed) and **authentication** (you are really talking to the server you think), by using **certificates** to prove identity, **key exchange** to agree on a secret, and fast **symmetric encryption** for the actual data. **HTTPS = HTTP over TLS.**

**Level:** 🟢 Beginner → 🟡 Intermediate → 🔴 Expert · **Reading time:** ~75 minutes

**Prerequisites:** [Chapter 23](23_tcp_in_details.md) (TCP handshake), [Chapter 26](26_http_1_1_in_details.md) and [Chapter 28](28_dns_domain_name_system_in_details.md) (HTTP and DNS). No math background is required.

---

## What you will learn

- The three problems TLS solves, and what an attacker can do without it
- **SSL vs TLS**, the history of versions, and what is safe today
- The cryptography building blocks in plain language: **symmetric** and **asymmetric** encryption, **hashes**, **digital signatures**, **Diffie-Hellman key exchange**, **AEAD**
- The **TLS 1.3 handshake** step by step (1 round trip), and how it differs from **TLS 1.2** (2 round trips) and the old RSA key exchange
- **Certificates**, **chains of trust**, **CAs**, **SANs**, **SNI**, **ALPN**, **revocation**, **Certificate Transparency**, and **Let's Encrypt/ACME**
- **Forward secrecy**, **session resumption**, **0-RTT** and their trade-offs
- **mTLS** (mutual TLS), HSTS, certificate pinning and real attacks (MITM, downgrade/stripping, Heartbleed)
- Performance, and how TLS sits in the protocol stack (not really "Layer 4")
- Practical work: inspect real connections with `openssl` and `curl`, create your own CA and certificates, run **HTTPS in Docker**, capture and decrypt traffic in Wireshark
- Where post-quantum cryptography fits in

---

## 1. Why TLS? What can go wrong on the network

Data on the internet passes through many machines you don't control: your Wi-Fi router, your ISP, transit networks, the server's data center. Anyone on that path can, with plain **HTTP**:

| Threat | Without TLS |
|---|---|
| **Eavesdropping** | Read passwords, cookies, card numbers, messages |
| **Tampering** | Change what you send or receive: inject ads or malware, alter a bank transfer amount |
| **Impersonation** | Pretend to be the real server (phishing, DNS spoofing, rogue Wi-Fi) |

An analogy: sending cash by taxi is dangerous because the driver, bystanders and anyone following can see the bag. A secure channel is like an **armored, sealed, tamper-evident vehicle** that also has a **verified ID** for the destination. TLS gives every connection that:

| Goal | Meaning | TLS mechanism |
|---|---|---|
| **Confidentiality** | Only the two endpoints can read the data | Symmetric encryption with session keys |
| **Integrity** | Any modification is detected | Authenticated encryption (AEAD) / MACs |
| **Authentication** | The client knows who the server is (and optionally vice versa) | X.509 certificates and digital signatures |

What TLS does **not** do: protect you from a malicious *server*, from malware on your device, from application bugs (SQL injection, XSS), or from phishing sites that have valid certificates for *their own* domain. It secures **the channel**, not the endpoints or the application.

---

## 2. SSL vs TLS and the version history

| Protocol | Year | Status |
|---|---|---|
| SSL 1.0 | (never released) | – |
| **SSL 2.0** | 1995 (Netscape) | **Broken.** Prohibited (RFC 6176) |
| **SSL 3.0** | 1996 | **Broken** (POODLE attack, 2014). Prohibited (RFC 7568) |
| **TLS 1.0** | 1999 (RFC 2246) | **Deprecated** (RFC 8996, 2021); browsers removed it around 2020 |
| **TLS 1.1** | 2006 (RFC 4346) | **Deprecated** (RFC 8996) |
| **TLS 1.2** | 2008 (RFC 5246) | **Still widely used and secure** *if configured well* (AEAD ciphers, ECDHE) |
| **TLS 1.3** | 2018 (RFC 8446) | **Recommended.** Faster, simpler, only strong algorithms |

"SSL certificate", "SSL handshake" and "SSL encryption" are legacy words: everything called SSL today is really **TLS**. Only enable **TLS 1.2 and 1.3**.

### Where TLS sits in the stack
The name "Transport Layer Security" is a bit misleading: TLS is **not a Layer-4 protocol**. It runs **on top of** a reliable transport (TCP) and **underneath** the application protocol (HTTP, SMTP, IMAP, PostgreSQL, ...):

```
 HTTP  (application data)        ← "GET / HTTP/1.1"
 TLS   (record layer: encrypt)   ← encrypts/authenticates each chunk
 TCP   (reliable byte stream)
 IP
 Ethernet / Wi-Fi
```
In OSI terms it is loosely "layers 5–6". (For HTTP/3 the equivalent is **QUIC**, which integrates TLS 1.3 into a UDP-based transport, and **DTLS** is TLS for datagrams.)

---

## 3. The cryptography you need, in plain language

### 3.1 Symmetric encryption: one shared secret key
The same key encrypts and decrypts. Very **fast** (gigabytes per second on modern CPUs with AES hardware instructions). Examples: **AES-128/256-GCM**, **ChaCha20-Poly1305**.
Problem: how do two strangers agree on the key without anyone else learning it?

### 3.2 Asymmetric ("public-key") cryptography: a key pair
Each party has a **public key** (share it with everyone) and a **private key** (keep it secret). They are mathematically linked. Two main uses:

| Use | Operation | Who can do it |
|---|---|---|
| **Encryption** | Encrypt with the **public** key → only the **private** key decrypts | Anyone encrypts; only the owner decrypts |
| **Digital signature** | Sign with the **private** key → anyone verifies with the **public** key | Only the owner signs; anyone verifies |

Analogy: a public key is an open padlock you hand out; the private key is the only key that opens it. A signature is a seal that only you can make but everyone can check. Asymmetric operations are **1,000× slower** than symmetric ones, so TLS uses them only to **authenticate and to agree on keys**, then switches to symmetric encryption for the bulk data (**hybrid encryption**). Examples: **RSA**, **ECDSA**, **Ed25519** (signatures); **ECDH/X25519** (key agreement).

### 3.3 Hash functions
A **hash** (SHA-256, SHA-384) turns any data into a short fixed-size fingerprint. Change one bit of the input and the fingerprint changes completely; you can't feasibly go backwards. Used for signatures, integrity checks and key derivation. MD5 and SHA-1 are broken for security.

### 3.4 MACs and AEAD
A **MAC** (message authentication code) is a keyed hash proving a message wasn't altered. Modern TLS uses **AEAD** ciphers (Authenticated Encryption with Associated Data) such as AES-GCM and ChaCha20-Poly1305 that encrypt **and** authenticate in one step. If a single bit is modified in transit, decryption fails and the connection is torn down.

### 3.5 Diffie-Hellman key exchange
A clever trick: two parties **agree on a shared secret over a public channel** where an eavesdropper sees everything exchanged, and still can't compute the secret. Paint-mixing analogy: both start with the same public yellow paint; each secretly adds a private color; they swap the mixtures; each adds their own secret color again, and both end with the same final color, while an eavesdropper who only saw the mixed paints cannot un-mix them.

A tiny numeric example (real ones use enormous numbers or elliptic curves):

```
Public:  p = 23, g = 5
Alice picks secret a = 6      →  sends A = g^a mod p = 5^6 mod 23 = 8
Bob   picks secret b = 15     →  sends B = g^b mod p = 5^15 mod 23 = 19
Alice computes  B^a mod p = 19^6 mod 23 = 2
Bob   computes  A^b mod p =  8^15 mod 23 = 2        ← the same shared secret, never transmitted!
```

An eavesdropper knows 23, 5, 8 and 19 but not a or b; with huge numbers, recovering them is infeasible. **ECDHE** (elliptic-curve DH, "E" = **ephemeral**: a fresh key each connection) does the same with elliptic curves, using much smaller keys. Diffie-Hellman gives **agreement**, not **authentication**: someone in the middle could run DH with each side separately. That's why the server also **signs** its part with its certificate's private key.

---

## 4. The TLS 1.3 handshake, step by step

After TCP connects, the client and server run the TLS handshake **before** any HTTP is sent. In TLS 1.3 it takes **one round trip**.

```
 Client                                                              Server
   │                                                                   │
   │ ── ClientHello ─────────────────────────────────────────────────► │
   │     · random (32 bytes)                                            │
   │     · supported_versions (TLS 1.3, 1.2)                            │
   │     · cipher suites offered                                        │
   │     · key_share: client's ECDHE PUBLIC value (e.g. X25519)         │
   │     · SNI: server name "example.com"                               │
   │     · ALPN: ["h2", "http/1.1"]                                     │
   │                                                                   │
   │ ◄── ServerHello ───────────────────────────────────────────────── │  (plaintext)
   │     · random · chosen version & cipher suite                       │
   │     · key_share: server's ECDHE PUBLIC value                       │
   │        ═══ both sides now compute the SHARED SECRET and           │
   │            derive handshake keys: everything below is ENCRYPTED ═══│
   │ ◄── EncryptedExtensions (ALPN result, ...)                        │
   │ ◄── Certificate (the server's certificate chain)                   │
   │ ◄── CertificateVerify (a SIGNATURE over the handshake so far,     │
   │                        made with the certificate's PRIVATE key)    │
   │ ◄── Finished (a MAC over the whole handshake)                      │
   │                                                                   │
   │  (client verifies the certificate chain, the name, the signature, │
   │   and the Finished MAC)                                            │
   │ ── Finished ────────────────────────────────────────────────────► │
   │                                                                   │
   │ ══ Application data (HTTP), encrypted with traffic keys ═════════► │
   │ ◄═══════════════════════════════════════════════════════════════ │
```

### What happens, in plain words

1. **ClientHello**: "Hi. I speak TLS 1.3/1.2. These are the cipher suites I support. Here's my random number and my half of a key exchange (`key_share`). I'm looking for `example.com` (SNI) and I'd like to speak `h2` or `http/1.1` (ALPN)." The client guesses which key-exchange group the server will accept (X25519 almost always), which is why one round trip is enough.
2. **ServerHello**: "Let's use TLS 1.3 with `TLS_AES_128_GCM_SHA256`. Here's my half of the key exchange." Now both sides combine their own private value with the other's public value to compute the same **shared secret** (ECDHE), and run it through a **key derivation function (HKDF)**, together with a hash of the handshake so far, to produce **several keys**: handshake keys, then **traffic keys**. From here on, everything is **encrypted**, including the server's certificate (a privacy improvement over TLS 1.2).
3. **Certificate + CertificateVerify**: the server presents its **certificate chain** (its public key, its name(s), signed by a CA) and **proves it owns the matching private key** by signing a hash of the handshake transcript. This is the authentication step: an impostor with a copy of the certificate but no private key can't produce this signature.
4. **Finished**: each side sends a MAC over the entire handshake, proving that **no one tampered** with any earlier message (this defeats downgrade attacks).
5. **Application data** flows, encrypted with AEAD using keys derived from the shared secret. **Client sends its request right after its Finished**, so the whole thing costs **1 RTT** of TLS, on top of TCP's 1 RTT.

Notes:
- **The pre-master secret is no longer "sent"**. Nothing secret crosses the wire. Both sides *compute* the same secret from public values, so a passive attacker who records everything can't derive it, **even if they later steal the server's private key** (forward secrecy, section 6).
- The random numbers are **32 random bytes** (not "primes like 37 and 47"): they make each session's keys unique and prevent replays.
- **Keys:** the client and server use *different* keys for each direction (client-to-server and server-to-client), plus separate IVs; they're derived independently by both sides, never transmitted.

### The four traffic-key naming
"Client write key" (used by the client to encrypt, and by the server to decrypt) and "server write key" (the opposite direction). It is two secret keys in total per connection (plus IVs), not four different keys.

### TLS 1.3 cipher suites (only five exist, all AEAD)
| Suite | Notes |
|---|---|
| `TLS_AES_128_GCM_SHA256` | Default in most stacks |
| `TLS_AES_256_GCM_SHA384` | |
| `TLS_CHACHA20_POLY1305_SHA256` | Fast on devices without AES hardware (phones) |
| (`TLS_AES_128_CCM_SHA256`, `..._CCM_8_SHA256`) | IoT/constrained |

In 1.3 the suite names only cover the *bulk cipher and hash*. Key exchange (ECDHE/DHE) and the signature algorithm (RSA-PSS, ECDSA, Ed25519) are negotiated separately.

---

## 5. TLS 1.2 (and the old RSA key transport)

TLS 1.2 is still very common. Its handshake needs **two round trips**:

```
 Client                                           Server
   │ ── ClientHello (versions, ciphers, random) ──►│
   │ ◄── ServerHello (chosen cipher, random)       │
   │ ◄── Certificate                               │
   │ ◄── ServerKeyExchange (ECDHE params, signed)  │      ← 1st round trip ends
   │ ◄── ServerHelloDone                           │
   │ ── ClientKeyExchange (client's ECDHE value) ─►│
   │ ── ChangeCipherSpec, Finished ───────────────►│
   │ ◄── ChangeCipherSpec, Finished                │      ← 2nd round trip ends
   │ ══ encrypted application data ═══════════════ │
```

A TLS 1.2 cipher suite name spells out everything, e.g. `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`:
`ECDHE` (key exchange: ephemeral EC Diffie-Hellman) · `RSA` (server signature/authentication) · `AES_128_GCM` (bulk cipher) · `SHA256` (hash/PRF).

### The old RSA key exchange (do not use)
Very old configurations (`TLS_RSA_WITH_AES_128_CBC_SHA` and similar) work like this: the client makes a random **pre-master secret**, **encrypts it with the server's RSA public key**, and sends it; the server decrypts it with its private key; both derive session keys from *pre-master secret + client random + server random*. It's simple to understand, and many tutorials teach it. But:

- **No forward secrecy.** Someone who **records traffic today** and **steals the server's private key years later** can decrypt every recorded session.
- It enabled attacks (Bleichenbacher/ROBOT).
- **TLS 1.3 removed it entirely.** In TLS 1.2 you should only allow `ECDHE` suites.

---

## 6. Forward secrecy

**Perfect forward secrecy (PFS)** means: compromising the server's **long-term private key** later must **not** reveal past sessions.

- With **ephemeral** (EC)DHE, each session's keys derive from temporary values that are **thrown away** after the handshake. The certificate's private key is used only to **sign** (authenticate), never to protect the session secret.
- **TLS 1.3 always provides forward secrecy** (there is no non-ephemeral key exchange). In TLS 1.2 you get it only with `ECDHE`/`DHE` suites.
- Caveat: **session tickets** (below) can weaken forward secrecy if ticket keys live too long, so rotate them frequently.
- This matters because of "**record now, decrypt later**" adversaries.

---

## 7. Certificates and the chain of trust

### 7.1 What a certificate is
An **X.509 certificate** binds a **public key** to an **identity** (mostly domain names), and is **digitally signed by a Certificate Authority (CA)** vouching for it. Important fields:

```
Certificate:
  Version: 3
  Serial Number:          unique within the CA
  Issuer:                 CN=R11, O=Let's Encrypt         ← who signed it
  Validity:               Not Before / Not After          ← lifetime
  Subject:                CN=example.com                  ← (legacy; browsers use the SAN)
  Subject Public Key Info: ECDSA P-256 (or RSA-2048)      ← the server's PUBLIC key
  X509v3 Extensions:
     Subject Alternative Name: DNS:example.com, DNS:www.example.com, DNS:*.api.example.com   ← the names it is valid for
     Key Usage / Extended Key Usage: digitalSignature; TLS Web Server Authentication
     Authority Information Access: CA issuer URL, OCSP URL
     Basic Constraints: CA:FALSE
     Certificate Transparency SCTs
  Signature Algorithm + Signature:  the issuer's signature over all of the above
```

The **private key never appears in the certificate** and never leaves the server (or its HSM/KMS).

### 7.2 Chain of trust

```
   Root CA certificate           (self-signed; pre-installed in OS/browser "trust stores"; kept offline)
        │ signs
   Intermediate CA certificate   (used for daily issuance; can be revoked without replacing the root)
        │ signs
   Leaf (server) certificate     (example.com)   ← the server sends leaf + intermediates; NOT the root
```

The client validates the chain **upwards**: leaf signed by intermediate, intermediate signed by a root it **already trusts**. Failing to send the intermediate is a common misconfiguration ("incomplete chain": works in some browsers, fails in `curl` or Java).

### 7.3 What the client checks
1. The **signature chain** ends at a **trusted root**, and each signature is valid.
2. **Validity dates**: today is inside *Not Before / Not After* (expired certificates are the #1 cause of TLS outages, so automate renewal).
3. **Name matching**: the hostname you requested matches a **Subject Alternative Name** (wildcards cover exactly one label: `*.example.com` matches `www.example.com`, not `example.com` or `a.b.example.com`).
4. **Key usage** allows TLS server authentication; **basic constraints** so leaves can't act as CAs.
5. **Revocation** status where checked: **CRL**, **OCSP** (a status query), or **OCSP stapling** (the server attaches a fresh signed status). Browsers rely largely on pushed lists (e.g. CRLite/CRLSets). The industry is moving away from live OCSP.
6. **Certificate Transparency (CT)**: browsers require public certificates to appear in append-only public logs (SCTs in the certificate), so mis-issuance can be detected. You can search logs at crt.sh.

If any check fails, the browser shows a full-page warning (`NET::ERR_CERT_DATE_INVALID`, `NET::ERR_CERT_AUTHORITY_INVALID`, `ERR_CERT_COMMON_NAME_INVALID`, ...). `curl` says `SSL certificate problem: ...`.

### 7.4 Getting a certificate
- **Domain Validated (DV)**: the CA checks you control the domain, via an HTTP file (`/.well-known/acme-challenge/`) or a **DNS TXT** record (`_acme-challenge`). Automated with the **ACME** protocol. **Let's Encrypt** (launched 2015-2016) issues DV certificates free, for **90 days**, through clients like **Certbot**, **acme.sh**, **Caddy** (automatic), **Traefik**, **cert-manager** (Kubernetes). Free certificates are technically as secure as paid ones.
- **OV/EV** (organization/extended validation): additional identity vetting. Browsers no longer show special green-bar UI for EV, so their value is mostly compliance.
- **Wildcard** (`*.example.com`): requires DNS validation.
- **Lifetimes are shrinking:** publicly trusted certificates were limited to **398 days** (2020), and the CA/Browser Forum has approved a phased reduction toward **~47 days** by 2029. Automate everything.
- **Private CAs**: for internal services, run your own CA (step-ca, Vault PKI, cfssl, cert-manager with a CA issuer, a cloud private CA) and distribute its root to your clients. For local development use **mkcert** (creates a local CA trusted by your browser).
- **Self-signed** certificates work if you explicitly trust them, but browsers warn; use them for tests only.

### 7.5 SNI and ALPN
- **SNI (Server Name Indication):** the client puts the hostname it wants in the ClientHello so a server hosting many HTTPS sites on one IP can pick the right certificate (the TLS equivalent of the HTTP `Host` header). It is sent in clear text in TLS 1.2 and 1.3; **Encrypted Client Hello (ECH)** hides it.
- **ALPN (Application-Layer Protocol Negotiation):** the client and server agree on `h2` or `http/1.1` (or `h3`) during the handshake, with no extra round trip (Chapter 27).

---

## 8. Session resumption and 0-RTT

A full handshake costs time and CPU, so TLS lets a returning client **resume**:

- After the first handshake the server can send a **session ticket** (**PSK**, pre-shared key in TLS 1.3): an encrypted blob containing the session state. On reconnect the client presents it, and both sides skip certificate verification and asymmetric operations. (Servers rotate the key that encrypts tickets.)
- **0-RTT ("early data")** (TLS 1.3): a returning client can send **application data in its very first flight**. Fast, but early data **can be replayed** by an attacker, so it's only safe for **idempotent** requests (`GET`), and servers/CDNs must be configured deliberately (or disable it). It also reduces forward secrecy for that data.

---

## 9. Mutual TLS (mTLS)

In ordinary TLS **only the server** is authenticated. With **mTLS** the server also **requests a client certificate** (`CertificateRequest`), and the client proves possession of its private key (its own `CertificateVerify`). Both sides are authenticated cryptographically. Uses: service-to-service authentication in microservices and **service meshes** (Istio/Linkerd issue short-lived certificates to every pod), Kubernetes API and kubelet, the **Docker daemon's remote API**, VPNs, banking APIs, IoT devices.

---

## 10. Attacks and defenses

| Attack | What happens | Defense |
|---|---|---|
| **Man-in-the-middle** | Attacker relays and reads/modifies traffic | Certificate validation (name + trusted chain), CT logs; never click through warnings; don't disable verification in code (`verify=False`, `curl -k`, `InsecureSkipVerify`) |
| **SSL stripping / downgrade** | Attacker keeps you on `http://` or forces an older protocol | **HSTS** (`Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`), HSTS preload list, redirect HTTP→HTTPS, TLS 1.3 downgrade protection in the handshake, disable old versions |
| **Rogue/compromised CA** | A trusted CA issues a certificate for your domain | **CAA** DNS records, Certificate Transparency monitoring, (mobile apps: certificate/public-key pinning, used carefully; HPKP in browsers was abandoned as too risky) |
| **Stolen private key** | Attacker can impersonate the server | Protect keys (permissions `600`, HSM/KMS), **revoke and reissue**, short certificate lifetimes; forward secrecy protects *past* sessions |
| **Heartbleed** (2014, OpenSSL bug) | Malformed heartbeat requests leaked server memory (including private keys) | Patch, re-key, revoke; lesson: implementation bugs matter even when the protocol is sound |
| **POODLE / BEAST / CRIME / BREACH / Lucky13 / ROBOT** | Attacks on SSL 3.0, CBC ciphers, TLS compression, RSA key exchange | Modern TLS 1.2/1.3 with AEAD/ECDHE, no compression, up-to-date libraries |
| **Replay of 0-RTT data** | Attacker resends captured early data | Only allow safe idempotent early requests |
| **Certificate expiry outages** | Everything breaks at midnight | Monitor and automate renewal |
| **Traffic analysis** | Sizes and timing still leak information | Padding, ECH, application-level measures |

---

## 11. Performance

TLS 1.3 costs one extra round trip per new connection; with resumption/0-RTT, less. Compare the time to first request byte with an RTT of 50 ms (DNS excluded):

| Protocol setup | Round trips before the request is sent | Time |
|---|---|---|
| HTTP | TCP 1 | ≈ 50 ms |
| HTTPS, TLS 1.2 | TCP 1 + TLS 2 | ≈ 150 ms |
| HTTPS, TLS 1.3 | TCP 1 + TLS 1 | ≈ 100 ms |
| HTTPS, TLS 1.3 resumption with 0-RTT | TCP 1 + TLS 0 | ≈ 50 ms |
| HTTP/3 (QUIC, TLS 1.3 integrated) | 1 (0 with 0-RTT) | ≈ 50 ms |

CPU cost is small on modern hardware: symmetric encryption uses **AES-NI** (billions of bytes per second per core), and asymmetric operations (ECDSA/X25519) are cheap and **once per connection**. To keep it fast: reuse connections (HTTP keep-alive, HTTP/2 multiplexing), enable **session resumption**, prefer **ECDSA** certificates and **X25519**, use **OCSP stapling** (or none), avoid oversized certificate chains, and terminate TLS on capable edges (CDN, load balancer, ingress).

---

## 12. Post-quantum cryptography

A large **quantum computer** running Shor's algorithm could break RSA and elliptic-curve cryptography. Nobody has such a machine yet, but attackers can **"harvest now, decrypt later"**, so confidentiality of long-lived secrets needs protection *now*. In August 2024 NIST published its first post-quantum standards: **ML-KEM** (from CRYSTALS-Kyber, key encapsulation, FIPS 203), **ML-DSA** (from Dilithium, signatures, FIPS 204) and **SLH-DSA** (from SPHINCS+, FIPS 205). Browsers and CDNs already deploy **hybrid key exchange** in TLS 1.3 (**X25519MLKEM768**: classical X25519 + ML-KEM, secure if *either* holds), enabled by default in recent Chrome, Firefox, Safari and Cloudflare. Post-quantum **signatures/certificates** are further out. (Symmetric ciphers such as AES-256 and hashes are not broken by quantum computers, only weakened.)

---

## 13. TLS in Docker and Kubernetes

| Topic | Practice |
|---|---|
| **Where to terminate TLS** | Usually at a **reverse proxy** (nginx, Caddy, Traefik, Envoy, HAProxy, cloud load balancer, Kubernetes **Ingress/Gateway**), which forwards plain HTTP (or re-encrypted TLS) to the app containers on the private network |
| **Certificates in containers** | Never bake private keys into images. Mount them as **Docker secrets / volumes / Kubernetes Secrets**, or fetch them from Vault or a cloud secrets manager; certificates must be **rotatable without rebuilding** |
| **Automatic certificates** | Caddy and Traefik obtain and renew Let's Encrypt certificates automatically; **cert-manager** does the same in Kubernetes |
| **Trust store in images** | Slim images may lack `ca-certificates`; **outgoing** HTTPS from a container fails with `certificate verify failed` until you `apt-get install -y ca-certificates` (or use `--no-install-recommends ca-certificates` in Dockerfiles). Custom private CAs must be added (`update-ca-certificates`) |
| **Docker daemon remote API** | Never expose `tcp://0.0.0.0:2375` unauthenticated. Use SSH (`docker -H ssh://...`) or **mutual TLS** on port 2376 (`--tlsverify`, CA + server + client certs) |
| **Registries** | HTTPS by default; "insecure registries" (`insecure-registries`) disable verification: avoid |
| **Service meshes** | Automatic mTLS between pods with short-lived certificates |
| **Health checks, `curl -k`** | Fine for local tests, but never in production paths |

---

## 14. Hands-on labs

You need `openssl` and `curl`. On Windows use WSL2 or Git Bash.

**Lab 1: Look at a real connection**

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | head -40
```
Find: **Protocol** (TLSv1.3), **Cipher** (`TLS_AES_256_GCM_SHA384`), **the certificate chain** (`0 s:` leaf, `1 s:` intermediate), **verify return code: 0 (ok)**, and the **Server public key** type/size.

```bash
curl -v https://example.com/ -o /dev/null 2>&1 | grep -E '^\* (TLS|ALPN|Server certificate|  (subject|start|expire|issuer)|SSL)'
```

**Lab 2: Read the certificate**

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
echo | openssl s_client -connect example.com:443 -servername example.com -showcerts 2>/dev/null | grep -E 's:|i:'      # the chain
```
What is the SAN list? How many days remain? Who signed it (issuer = intermediate)?

**Lab 3: Compare protocol versions**

```bash
openssl s_client -connect example.com:443 -tls1_3 </dev/null 2>&1 | grep -E 'Protocol|Cipher'
openssl s_client -connect example.com:443 -tls1_2 </dev/null 2>&1 | grep -E 'Protocol|Cipher'    # ECDHE-... suites
openssl s_client -connect example.com:443 -tls1_1 </dev/null 2>&1 | head -3                      # should FAIL on modern servers
```

**Lab 4: Verify failures on purpose**

```bash
curl -sS https://expired.badssl.com/ -o /dev/null          # certificate has expired
curl -sS https://wrong.host.badssl.com/ -o /dev/null       # name mismatch
curl -sS https://self-signed.badssl.com/ -o /dev/null      # unknown issuer
curl -sSk https://self-signed.badssl.com/ -o /dev/null && echo "succeeded ONLY because -k disabled verification (never do this in production)"
```

**Lab 5: Build your own CA and a server certificate**

```bash
mkdir tls-lab && cd tls-lab

# 1. A private CA (root)
openssl genrsa -out ca.key 4096
openssl req -x509 -new -key ca.key -sha256 -days 365 -subj "/CN=My Lab CA" -out ca.crt

# 2. A server key and certificate signing request (CSR)
openssl genrsa -out server.key 2048
openssl req -new -key server.key -subj "/CN=localhost" -out server.csr

# 3. The CA signs it, including the SAN that browsers/clients require
printf 'subjectAltName=DNS:localhost,IP:127.0.0.1\nbasicConstraints=CA:FALSE\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth\n' > san.ext
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 30 -sha256 -extfile san.ext

# 4. Inspect and verify
openssl x509 -in server.crt -noout -text | grep -E 'Issuer|Subject:|DNS:|Not After'
openssl verify -CAfile ca.crt server.crt                  # server.crt: OK
```

**Lab 6: HTTPS in Docker with your certificate (nginx)**

```bash
cat > default.conf << 'EOF'
server {
    listen 443 ssl;
    http2 on;                                   # nginx >= 1.25.1
    ssl_certificate     /certs/server.crt;
    ssl_certificate_key /certs/server.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    add_header Strict-Transport-Security "max-age=31536000" always;
    location / { root /usr/share/nginx/html; }
}
server { listen 80; return 301 https://$host$request_uri; }
EOF
docker run -d --name tls -p 8443:443 -p 8080:80 \
  -v "$PWD/default.conf":/etc/nginx/conf.d/default.conf:ro \
  -v "$PWD":/certs:ro nginx:1.27-alpine

curl -sS https://localhost:8443/ -o /dev/null                       # FAILS: unknown CA
curl -sS --cacert ca.crt https://localhost:8443/ -o /dev/null && echo "OK: trusted because we gave curl our CA"
curl -sSI --cacert ca.crt https://localhost:8443/ | head -5         # HTTP/2 200, strict-transport-security header
openssl s_client -connect localhost:8443 -CAfile ca.crt -servername localhost </dev/null 2>/dev/null | grep -E 'Verif|Protocol|Cipher'
curl -sSI http://localhost:8080/ | head -3                          # 301 redirect to https
docker rm -f tls
```

Now break things deliberately and observe the error messages: request `https://127.0.0.2:8443` (name/IP mismatch; not in the SAN), change the system date backwards or use `faketime` (not yet valid), remove the SAN, or replace the CA.

**Lab 7: Mutual TLS**

```bash
# a client certificate signed by the same CA
openssl genrsa -out client.key 2048
openssl req -new -key client.key -subj "/CN=alice" -out client.csr
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 30 -sha256
# add to the nginx server block:   ssl_client_certificate /certs/ca.crt;  ssl_verify_client on;
# then:
curl --cacert ca.crt --cert client.crt --key client.key https://localhost:8443/    # works
curl --cacert ca.crt https://localhost:8443/                                       # 400 "No required SSL certificate was sent"
```

**Lab 8: Watch a handshake in Wireshark and decrypt it**

```bash
export SSLKEYLOGFILE=$HOME/tls-keys.log
curl https://example.com/ -o /dev/null
# Wireshark: capture, filter "tls"; expand ClientHello (see supported_versions, cipher suites, key_share, SNI, ALPN),
# ServerHello, and the encrypted handshake records. Preferences → Protocols → TLS → (Pre)-Master-Secret log filename → the file above → HTTP appears decrypted.
```
Observe: in TLS 1.3 the certificate is *not* visible in the capture (it's encrypted), while in a TLS 1.2 capture (`curl --tlsv1.2 --tls-max 1.2 ...`) it is.

**Lab 9: Time the handshake**

```bash
curl -o /dev/null -s -w 'dns %{time_namelookup}  tcp %{time_connect}  tls %{time_appconnect}  ttfb %{time_starttransfer}\n' https://example.com/
curl --tlsv1.2 --tls-max 1.2 -o /dev/null -s -w 'TLS1.2: tcp %{time_connect} tls %{time_appconnect}\n' https://example.com/
```
`tls - tcp` is the handshake cost: about **2 RTT** for TLS 1.2 and **1 RTT** for TLS 1.3.

**Lab 10: Audit a server**: use the online SSL Labs test (ssllabs.com/ssltest) or the open-source **`testssl.sh`** (`docker run --rm -ti drwetter/testssl.sh example.com`) to check protocols, ciphers, forward secrecy, certificate chain and known vulnerabilities. Use Mozilla's SSL Configuration Generator (ssl-config.mozilla.org) for sane server configs.

**Lab 11: Certificate Transparency**: search `crt.sh?q=%25.yourdomain.com` to see every publicly logged certificate for a domain (great for spotting unexpected issuance).

---

## 15. Troubleshooting

| Error | Meaning | Fix |
|---|---|---|
| `certificate has expired` / `ERR_CERT_DATE_INVALID` | Past *Not After* (or your machine's clock is wrong) | Renew (automate!); check system time |
| `unable to get local issuer certificate` / `unknown CA` / `ERR_CERT_AUTHORITY_INVALID` | Chain doesn't lead to a trusted root: **missing intermediate**, private/self-signed CA, or the client lacks `ca-certificates` | Serve the full chain (`fullchain.pem`); install the CA in the client trust store; `apt-get install ca-certificates` in slim images |
| `hostname mismatch` / `ERR_CERT_COMMON_NAME_INVALID` | Requested name isn't in the SAN | Reissue with correct SANs; use the right hostname (not the raw IP) |
| `tlsv1 alert protocol version` / `no protocols available` | Client and server share no TLS version | Enable TLS 1.2/1.3; update the old client |
| `no shared cipher` / `handshake failure` | No common cipher suites | Loosen/modernize the cipher list |
| `ERR_SSL_PROTOCOL_ERROR`, `wrong version number` | Speaking TLS to a plain-HTTP port (or vice versa) | Check the port (443 vs 80/8080) and the scheme |
| `unknown ca` alert from the server in mTLS | Client cert not signed by a CA the server trusts | Fix the client CA configuration |
| Works in browser, fails in `curl`/Java/Python | Incomplete chain (browsers fetch missing intermediates), or different trust stores | Send the full chain; test with `openssl s_client -showcerts` |
| `SSL_ERROR_SYSCALL`, resets after ClientHello | Firewall/IDS blocking, SNI-based filtering, MTU issues | Test from another network; `tcpdump` |
| HSTS: cannot click through a warning | The site sent HSTS earlier | Fix the certificate; (dev) clear HSTS for the domain in the browser |
| Docker: `x509: certificate signed by unknown authority` | Container lacks the CA (custom/corporate MITM proxy) | Add the CA to the image, or mount the bundle |
| Docker: `x509: certificate is valid for X, not Y` | Registry/service accessed by a name not in the certificate | Use the certificate's name (or `--add-host`), or reissue |

---

## 16. Common misconceptions

| Misconception | Reality |
|---|---|
| "SSL and TLS are different things in use today" | Only TLS is used; "SSL certificate" is a legacy label |
| "TLS is Layer 4" | It sits *above* TCP and *below* HTTP |
| "The browser encrypts the pre-master secret with the server's public key" | Only in the obsolete RSA key exchange. Modern TLS uses (EC)DHE: nothing secret is sent; both sides compute the secret |
| "The random numbers in Hello messages must be prime/small" | They're 32 random bytes to make each session unique |
| "The padlock means the site is safe/trustworthy" | It means the *connection* is encrypted to the domain shown; phishing sites can have valid certificates |
| "Free certificates are less secure" | Same cryptography and trust; just shorter-lived and automated |
| "HTTPS is only for logins/payments" | Everything should be HTTPS: it prevents tampering/injection and is required by browser features (service workers, geolocation, HTTP/2 in browsers) |
| "TLS makes sites slow" | 1 extra RTT (0 with resumption/0-RTT), negligible CPU |
| "Certificate = private key" | The certificate is public; the **private key** must stay secret |
| "Encrypting with the private key is how signatures work" | Signing is a distinct operation (RSA-PSS/ECDSA/EdDSA) done with the private key, verified with the public key |
| "TLS protects the data on the server" | Only in transit. Data at rest needs its own protection |
| "If I use HTTPS my app is secure" | TLS doesn't stop application bugs or a compromised endpoint |

---

## 17. Summary

- **TLS** provides **confidentiality, integrity and authentication** for TCP connections; **HTTPS = HTTP over TLS**. Use **TLS 1.3** (and 1.2 with ECDHE + AEAD); **SSL and TLS ≤ 1.1 are dead**.
- **Hybrid crypto:** asymmetric crypto (certificates, signatures, ECDHE) sets up the connection; fast **symmetric AEAD** encryption (AES-GCM/ChaCha20) protects the data.
- **TLS 1.3 handshake (1 RTT):** ClientHello (key share, SNI, ALPN) → ServerHello (key share) → *encrypted* EncryptedExtensions, Certificate, CertificateVerify (signature), Finished → client Finished → application data. Secrets are **computed, never sent**.
- **Forward secrecy** comes from ephemeral (EC)DHE and is always on in TLS 1.3; the old **RSA key exchange** is obsolete.
- **Certificates** (X.509, SANs) are signed by CAs in a **chain** ending at a root in your trust store; clients check chain, dates, name, usage and revocation; **Certificate Transparency** and **CAA** guard against mis-issuance; **ACME/Let's Encrypt** automates issuance; lifetimes are shrinking, so automate renewal.
- **Resumption/0-RTT** speed up reconnects (0-RTT is replayable); **mTLS** authenticates clients too; **HSTS** prevents downgrades.
- In Docker/Kubernetes: terminate TLS at a proxy/ingress, keep keys out of images, automate certificates, include CA roots, and never disable verification.
- **Post-quantum** hybrid key exchange (X25519MLKEM768) is already deployed to counter "harvest now, decrypt later".

---

## 18. Check your understanding

1. Name the three security properties TLS provides and the mechanism behind each.
2. Why does TLS use both asymmetric and symmetric cryptography?
3. In the TLS 1.3 handshake, what is sent in the ClientHello's `key_share`, and how do the client and server end up with the same shared secret without sending it?
4. What does the server's `CertificateVerify` prove that the certificate alone doesn't?
5. What is forward secrecy, and why did the old RSA key exchange lack it?
6. List four things a client checks on a server certificate. What is the most common operational cause of certificate errors?
7. Why is the certificate chain "leaf + intermediate" sent, but not the root?
8. What are SNI and ALPN for?
9. What is the risk of 0-RTT data?
10. A container's `curl https://api.internal` says "unable to get local issuer certificate", while your browser works. Give two likely causes and fixes.
11. In Diffie-Hellman with p = 23, g = 5, Alice's secret a = 4 and Bob's secret b = 3, compute both public values and the shared secret.

<details>
<summary>Answers</summary>

1. Confidentiality: symmetric encryption; integrity: AEAD/MAC (tamper detection); authentication: certificates and digital signatures.
2. Asymmetric operations solve identity and key agreement but are slow; symmetric ciphers are very fast for bulk data. TLS combines them.
3. The client's ephemeral ECDHE **public** value. Each side combines its own private value with the other's public value; the math (DH) makes both results equal, but an eavesdropper who sees only the public values cannot compute it.
4. That the server actually holds the private key matching the certificate's public key (it signs a hash of the handshake transcript), so a copy of the certificate isn't enough to impersonate it. It also binds the signature to *this* handshake.
5. Forward secrecy means recorded past traffic stays secret even if the server's long-term private key is later stolen. In RSA key exchange the session secret was encrypted with that long-term key, so anyone with the key and recorded traffic could decrypt everything.
6. Any four of: signature chain to a trusted root, validity dates, name matches SAN, key usage/constraints, revocation status, CT. Most common problem: expiry (and missing intermediates).
7. The client already has the root in its trust store, and a root sent by the server wouldn't add trust; the intermediate is needed so the client can build the path from leaf to root.
8. SNI: tells the server which hostname you want so it can pick the right certificate. ALPN: negotiates the application protocol (`h2`, `http/1.1`, `h3`) during the handshake.
9. Early data can be replayed by an attacker, so only idempotent requests are safe, and forward secrecy for that data is weaker.
10. The container lacks the CA certificates (`ca-certificates` not installed in a slim image) or the service uses a private/corporate CA the container doesn't trust (or an incomplete chain). Fix: install/mount the CA bundle, add the private CA (`update-ca-certificates`), serve the full chain.
11. A = 5⁴ mod 23 = 625 mod 23 = 4 (625 − 621); B = 5³ mod 23 = 125 mod 23 = 10 (125 − 115). Shared = B^a = 10⁴ mod 23 = 10000 mod 23 = 18 (23 × 434 = 9982); check A^b = 4³ mod 23 = 64 mod 23 = 18 ✓.
</details>

**Practice**

1. Use `openssl s_client` to inspect three sites; record protocol, cipher, key type, chain length, issuer and expiry date. Which uses ECDSA vs RSA?
2. Complete labs 5 and 6, then intentionally break each check (wrong name, expired/not-yet-valid date, unknown CA, missing intermediate) and note the exact error message each produces in `curl` and in a browser.
3. Capture a TLS 1.2 and a TLS 1.3 handshake in Wireshark; draw the message sequences and note which are encrypted in each.
4. Add HSTS and a redirect to your nginx, and test `curl -I http://localhost:8080` and `https://localhost:8443`.
5. Set up automatic certificates with Caddy in Docker for a domain you control (or a local ACME server like `smallstep/step-ca`), and observe renewal.
6. Read RFC 8446 section 2 (protocol overview) and compare its handshake diagram to the one in this chapter.

---

**Next:** [Chapter 30 – The Internet Protocol (IP) in Detail](30_internet_protocol_ip_in_details.md)
