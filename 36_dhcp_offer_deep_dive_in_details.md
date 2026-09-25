# Chapter 36: DHCP Offer, Layer by Layer

> **In one sentence:** The DHCP **Offer** is the server's reply to a Discover: "here is an address (`yiaddr`) I'm willing to lend you, plus the mask, router, DNS and lease time to go with it", sent from UDP port **67 to 68**, from the server's **real IP** to either the **offered address** (unicast to the client's MAC) or to **`255.255.255.255`** (broadcast), because the client still can't be addressed the normal way.

**Level:** 🟡 Intermediate · **Reading time:** ~50 minutes

**Prerequisite:** [Chapter 35](35_dhcp_discover_deep_dive_in_details.md).

---

## What you will learn

- What the server **decides** between receiving a Discover and sending an Offer (pool, reservations, conflict checks)
- **Every field of the Offer** and how it differs from the Discover
- The **addressing puzzle**: how to reach a client that has no IP yet: **unicast to MAC without ARP** versus **broadcast**, and what governs the choice
- The **options** an Offer carries: subnet mask, router, DNS, lease time, **T1/T2**, server identifier
- What the **client** does when Offers arrive: validate, choose, wait
- **Multiple servers** and **rogue servers**
- How to capture and read an Offer, plus a byte-level example with real checksums

---

## 1. Where we are in DORA

```
Client                                    Server (192.168.1.1)
  │── DISCOVER (broadcast) ───────────────────►│   ← Chapter 35
  │◄─ OFFER  ◄──────────────────────────────── │   ← THIS CHAPTER
  │── REQUEST ────────────────────────────────►│   ← Chapter 37
  │◄─ ACK ─────────────────────────────────────│
```
An Offer is a **proposal, not a commitment**. The client has not got the address yet, and the server has not finalized it either.

---

## 2. What the server does before answering

On receiving the Discover (xid `0x3903F326`, chaddr `aa:bb:cc:11:22:33`):

| Step | Question | Typical logic |
|---|---|---|
| 1 | **Which network is this client on?** | The interface the packet arrived on (or **giaddr** if relayed) picks the **scope/pool** |
| 2 | **Is there a static reservation** for this MAC / client-ID? | If yes, offer that fixed address |
| 3 | **Has this client had a lease before?** | Offer the same address again if still valid/free (stable addresses), or honor option 50 (requested IP) |
| 4 | **Otherwise choose a free address** from the pool | e.g. next unused in `192.168.1.100–.200` |
| 5 | **Is it really free?** | Some servers **ping/ARP-probe** first (ISC dhcpd does an ICMP echo; dnsmasq can too) to detect a static host squatting on the address |
| 6 | **Reserve it temporarily** | Mark it "offered" for a short time (seconds to minutes) so the same address isn't offered to another client |
| 7 | **Build the Offer** | Options from the scope configuration, filtered/prioritized using the client's *Parameter Request List* (option 55) |

Server state after this step (conceptually):

```
Address          Client (MAC / ID)    State     Expires
192.168.1.100    aa:bb:cc:11:22:33    OFFERED   ~ a few seconds/minutes (not a lease yet)
```
If the pool is exhausted, or no scope matches, or the MAC is denied, the server **doesn't answer at all** (there is no "negative offer"). Silent servers are why "obtaining IP address…" hangs.

---

## 3. The Offer message (Layer 7)

Same 240-byte-plus-options layout as Chapter 35. What changes:

| Field | Discover (client→server) | **Offer (server→client)** |
|---|---|---|
| **op** | 1 (BOOTREQUEST) | **2** (BOOTREPLY) |
| **xid** | random | **copied unchanged** (this is how the client knows the Offer is for its Discover) |
| **secs** | client's timer | 0 |
| **flags** | client's choice | **copied** from the Discover (the broadcast bit) |
| **ciaddr** | 0.0.0.0 | 0.0.0.0 |
| **yiaddr** | 0.0.0.0 | **`192.168.1.100`**, the *offered* address ("your IP address") |
| **siaddr** | 0.0.0.0 | Address of the next server for bootstrapping (usually 0, or the TFTP server for PXE) |
| **giaddr** | 0.0.0.0 | Copied from the Discover (0.0.0.0, or the relay's address) |
| **chaddr** | client MAC | **copied**: `aa:bb:cc:11:22:33` |
| **Option 53** | 1 (Discover) | **2 (Offer)** |

The core of an Offer is one field, **`yiaddr`**, plus its options.

### Options in a typical Offer

| Option | Code | Example | Purpose |
|---|---|---|---|
| **DHCP Message Type** | 53 | `2` (Offer) | Says what this is |
| **Server Identifier** | **54** | `192.168.1.1` | *Which server is offering.* The client puts this in its Request to say whom it accepts |
| **IP Address Lease Time** | 51 | `86400` s (24 h) | How long the address is valid |
| **Renewal Time (T1)** | 58 | `43200` s (50%) | When to start renewing with the same server |
| **Rebinding Time (T2)** | 59 | `75600` s (87.5%) | When to broadcast for *any* server |
| **Subnet Mask** | 1 | `255.255.255.0` | The mask the client must use |
| **Broadcast Address** | 28 | `192.168.1.255` | Subnet broadcast |
| **Router** | 3 | `192.168.1.1` | Default gateway (a list is allowed) |
| **DNS Servers** | 6 | `192.168.1.1, 1.1.1.1` | Name servers, in order |
| **Domain Name** | 15 | `home` | Search domain |
| Others | 42 NTP, 66/67 TFTP/boot file, 119 domain search list, 121 classless static routes, 43 vendor-specific, 82 relay info | | As configured |

**T1 and T2 defaults:** if the server omits them, clients use T1 = 0.5 × lease and T2 = 0.875 × lease (RFC 2131).

The Offer in words:
> "Hello `aa:bb:cc:11:22:33` (answering your transaction `0x3903F326`). I am `192.168.1.1`. You may use `192.168.1.100` with mask `255.255.255.0`, gateway `192.168.1.1`, DNS `192.168.1.1` and `1.1.1.1`, for 24 hours."

---

## 4. Layer 4: UDP: ports reversed

| | Discover | Offer |
|---|---|---|
| Source port | 68 | **67** |
| Destination port | 67 | **68** |

The server speaks *from* the well-known server port and replies *to* the well-known client port. Because the destination port is always **68**, the reply reaches the DHCP client process even though the client's IP address isn't configured yet.

Length = 8 + DHCP size. The Offer is usually a bit longer than the Discover because of its options (in our example 302 bytes vs 300).

---

## 5. Layer 3: IP: the addressing puzzle

The server (`192.168.1.1`) is sending to a client that **has no IP address yet**. What goes into the destination fields?

| Choice | Destination IP | Works? |
|---|---|---|
| Unicast to `yiaddr` (`192.168.1.100`) | The offered address | Only if the client's stack accepts IP unicast for an address it hasn't configured, which **most clients (Linux, Windows, macOS) do via a raw/packet socket** and the servers deliver directly to the client's MAC (below) |
| Broadcast | `255.255.255.255` (or the subnet broadcast) | Always works; every host on the link receives it |

RFC 2131 (section 4.1) defines the rule:

1. If **giaddr ≠ 0** (a relay was involved) → send to the **relay** (giaddr), which forwards to the client.
2. Else if **ciaddr ≠ 0** (renewing client) → unicast to ciaddr.
3. Else if the client's **broadcast flag** is **set** → broadcast to `255.255.255.255`.
4. Else → **unicast to `yiaddr`**, using the client's **hardware address (chaddr)** for the Layer-2 destination, with no ARP (the server's stack is told the mapping directly).

So the flag the client sent in Chapter 35 controls the reply's addressing:

| Client flag | Server sends | Frame |
|---|---|---|
| `0x0000` | unicast to `yiaddr` | dst MAC `aa:bb:cc:11:22:33`, dst IP `192.168.1.100` |
| `0x8000` | broadcast | dst MAC `ff:ff:ff:ff:ff:ff`, dst IP `255.255.255.255` |

**In practice** you will meet both. Many implementations (and consumer routers) simply **always broadcast** the Offer because it is the safest choice; Windows clients traditionally set the broadcast flag, Linux `dhclient` traditionally does not. Neither is wrong.

Source IP: always the **server's own address** (`192.168.1.1`), because the server already has one.

---

## 6. Layer 2: Ethernet

### Case A: unicast Offer
```
dst MAC: aa:bb:cc:11:22:33   (client, taken from chaddr in the Discover)
src MAC: 11:22:33:44:55:66   (router's LAN interface)
type:    0x0800
```
- A switch that has learned the client's MAC (from the Discover's source) forwards it **only to the client's port**. Other hosts never see it.
- **Why the server knows the MAC without ARP:** ARP would ask "who has 192.168.1.100?", and nobody would answer, since the client hasn't configured it. The server already has the answer in `chaddr` (and in the frame it received), so it constructs the frame directly.

### Case B: broadcast Offer
```
dst MAC: ff:ff:ff:ff:ff:ff, dst IP 255.255.255.255
```
- The switch **floods** it to all ports in the VLAN. Every host receives it and discards it (wrong port/no listener, or `xid` mismatch, or `chaddr` not its own).
- Privacy/noise angle: everyone on the LAN can read the offered address and options; not a secret (DHCP isn't confidential anyway).

### Why unicast is "nicer" but broadcast is safe
Unicast reduces noise and CPU on other hosts; broadcast maximizes compatibility with odd client stacks. Switches and Wi-Fi APs often treat broadcast frames specially (Wi-Fi sends them at the lowest data rate to be heard by everyone, so a broadcast Offer costs more airtime).

---

## 7. A complete Offer, byte by byte

Built with real checksums (reproducible with the script pattern of Chapter 35 §14.3). **Unicast case**, router `11:22:33:44:55:66` → client `aa:bb:cc:11:22:33`:

**Ethernet (14 bytes)**
```
aa bb cc 11 22 33   11 22 33 44 55 66   08 00
```
**IPv4 (20 bytes)**: source `192.168.1.1`, destination `192.168.1.100`
```
45 00 01 4a 00 00 00 00 40 11 f5 ed c0 a8 01 01 c0 a8 01 64
             total length = 0x14a = 330; protocol 17; checksum 0xf5ed; src c0a80101; dst c0a80164
```
**UDP (8 bytes)**
```
00 43   00 44   01 36   c2 13
 67      68      310     checksum 0xc213
```
**DHCP options (first bytes of the options area)**
```
63 82 53 63                     magic cookie
35 01 02                        53: message type = 2 (OFFER)
36 04 c0 a8 01 01               54: server identifier 192.168.1.1
33 04 00 01 51 80               51: lease time 0x00015180 = 86400 s
3a 04 00 00 a8 c0               58: T1 = 0xa8c0 = 43200 s
3b 04 00 01 27 50               59: T2 = 0x12750 = 75600 s
01 04 ff ff ff 00               1 : subnet mask 255.255.255.0
1c 04 c0 a8 01 ff               28: broadcast 192.168.1.255
03 04 c0 a8 01 01               3 : router 192.168.1.1
06 08 c0 a8 01 01 01 01 01 01   6 : DNS 192.168.1.1 and 1.1.1.1
0f 04 68 6f 6d 65               15: domain "home"
ff                              255: end
```
In the fixed part: `op=02`, `yiaddr = c0 a8 01 64` (192.168.1.100), `xid = 39 03 f3 26`, `chaddr = aa bb cc 11 22 33`. Total frame **344 bytes** (302-byte DHCP payload). If it were the broadcast variant: destination MAC `ff:ff:ff:ff:ff:ff`, destination IP `255.255.255.255` (checksums change: IP `0xb7fa`, UDP `0x8420`).

Full-stack summary:

```
Ethernet: 11:22:33:44:55:66 → aa:bb:cc:11:22:33   (or → ff:ff:ff:ff:ff:ff)
  IPv4:   192.168.1.1 → 192.168.1.100           (or → 255.255.255.255)
    UDP:  67 → 68
      DHCP: op=BOOTREPLY xid=0x3903f326 yiaddr=192.168.1.100 chaddr=aa:bb:cc:11:22:33
            opts: 53=Offer 54=192.168.1.1 51=86400 1=255.255.255.0 3=192.168.1.1 6=192.168.1.1,1.1.1.1 ...
```

---

## 8. Receiving and evaluating Offers (client side)

1. The NIC accepts the frame (its own MAC or broadcast); the IP layer accepts it (the client's stack allows this for a DHCP socket even without an address).
2. The DHCP client matches **xid** and **chaddr** to its outstanding Discover; anything else is ignored.
3. It validates: message type 2, server identifier present, offered address plausible, mask consistent.
4. **Waits briefly** (about 1–4 seconds, implementation dependent) to **collect multiple Offers** if more than one server answers.
5. **Chooses** one (commonly the first; some clients prefer the offer that matches their previously leased address, or a server whose configuration they trust).
6. Proceeds to **Request** (Chapter 37) naming that server by option 54.

### Two servers answer?
Redundancy setups (two DHCP servers with split pools, or a failover pair) can both reply. The client gets two Offers (say `.100` from `192.168.1.1` and `.150` from `192.168.1.2`), picks one, and its **broadcast Request** tells the loser to release its reserved address (Chapter 37).

### A rogue server answers?
An attacker or a stray home router also replies, maybe faster, with its own gateway/DNS. The client can't tell (no authentication): whichever Offer it selects wins. Defenses: **DHCP snooping** on switches (only "trusted" ports may send Offers), and monitoring (`nmap --script broadcast-dhcp-discover`, `dhcpdump`, Wireshark filter `bootp.option.dhcp == 2`).

---

## 9. What the client still can't do

After the Offer, the client **must not use the address yet**. It hasn't been *committed* by the server; the address is only reserved provisionally. If the client used it and the server then gave it to someone else, you'd get an IP conflict. The client waits for the **ACK** in Chapter 37, and after that **verifies with ARP** that no other host claims it (Chapter 39).

---

## 10. Lab

Use the private DHCP namespace lab from Chapter 35 §14.2. To compare the two reply styles, configure the client flag and inspect the packets:

```bash
# (namespaces srv/cli, dnsmasq and tcpdump already set up as in Chapter 35)
sudo ip netns exec cli dhclient -1 -v -sf /bin/true -lf /tmp/cli.lease -pf /tmp/cli.pid c0
sudo tcpdump -nn -vv -e -r /tmp/dhcp.pcap 2>/dev/null | grep -A20 -i offer | head -40
```
What to look at in the Offer:
- `Your-IP 192.168.50.10x`, `Server-ID`, `Lease-Time 43200`, `RN` (T1), `RB` (T2), `Subnet-Mask`, `Default-Gateway`, `Domain-Name-Server`.
- Ethernet header: was the destination the client's MAC or the broadcast MAC? (`dnsmasq` honors the client's broadcast flag: `dhclient` sends `0x0000`, so you should see unicast.)

Experiments:
1. Ask dnsmasq for a **fixed lease**: `--dhcp-host=<client-mac>,192.168.50.77`. Confirm `yiaddr` = `.77`.
2. Shrink the pool to a single address, then start two clients (use two more namespaces): the second gets **no Offer**.
3. Start a *second* dnsmasq in another namespace attached to the same wire (or on a bridge) to see two Offers with different Server-IDs and which one `dhclient` picks.
4. Add `--dhcp-option=option:domain-search,lab.local` and see it in the Offer.

**Wireshark filters**: `bootp.option.dhcp == 2` (Offers), `bootp.id == 0x3903f326` (one transaction), `bootp.option.server_id`.

---

## 11. Troubleshooting Offer problems

| Symptom | Cause |
|---|---|
| Discovers arrive, **no Offer** sent | Pool exhausted, no scope for that interface/giaddr, MAC filtered, server firewall/`bind-interfaces`, service down |
| Offer sent but **client doesn't see it** | Switch/AP dropping broadcasts or unicast to an unlearned MAC, VLAN mismatch, DHCP snooping on an untrusted port, wireless client isolation |
| Client shows `169.254.x.x` although the server logs "OFFER" | The return path is broken (see previous row); capture on the client's port |
| Wrong gateway/DNS/network | **Rogue** DHCP server or wrong scope options: check `Server-ID` in the Offer |
| Offer contains an address already in use | Static host inside the pool: server-side conflict detection disabled; use reservations/exclusions |
| Long delays before the Offer | Server performing ping/ARP conflict detection; slow backend (database) |

---

## 12. Common misconceptions

| Misconception | Reality |
|---|---|
| "Offer = the client now has the address" | It's only a proposal; the address is committed at the ACK |
| "The Offer is always unicast" / "always broadcast" | It depends on the client's broadcast flag, relay and server implementation |
| "The server ARPs for the client's IP" | It can't (the client doesn't answer for an unconfigured address); it uses `chaddr` |
| "Source IP of the Offer is 0.0.0.0" | The server has a real address: `192.168.1.1` |
| "Only one Offer can arrive" | Several servers may answer; the client chooses |
| "The client picks the address" | The **server** picks `yiaddr` (the client can only *request* one via option 50) |
| "The Offer contains only an IP" | It carries the whole configuration: mask, router, DNS, lease, T1/T2, and more |
| "Ports are random" | Always 67 → 68 |

---

## 13. Summary

- The **Offer** is the server's proposal: `op=2`, same `xid` and `chaddr`, **`yiaddr`** = the offered address, option 53 = 2, option 54 = the server's identity, plus mask, router, DNS, lease time, **T1/T2**.
- Sent **from `192.168.1.1:67` to port 68**, either **unicast to `yiaddr` with the client's MAC as Layer-2 destination (no ARP)** or **broadcast**, depending on the broadcast flag, relay and implementation.
- The server first picks a scope, honors reservations, avoids conflicts and **temporarily reserves** the address; if it can't offer, it stays silent.
- The client matches `xid`, may collect several Offers, picks one, and **doesn't use the address until the ACK**.
- No authentication, so use **DHCP snooping** against rogue servers.

---

## 14. Check your understanding

1. Which fields are copied from Discover to Offer, and which are new?
2. What is `yiaddr`, and who chooses it?
3. Why can't the server ARP for the client's offered address?
4. According to RFC 2131, when is the Offer broadcast and when unicast?
5. What do options 54, 51, 58 and 59 mean?
6. Why is the Offer "not yet" the client's address?
7. What happens when two servers answer? When none answers?
8. How can you detect a rogue DHCP server?

<details>
<summary>Answers</summary>

1. Copied: `xid`, `flags`, `giaddr`, `chaddr`. New/changed: `op=2`, `yiaddr`, option 53 = 2, option 54 (server ID) and the configuration options (mask, router, DNS, lease, T1, T2).
2. "Your IP address": the address offered to the client; the **server** chooses it (from a reservation, previous lease/requested IP or the pool).
3. The client hasn't configured that address, so it would not answer an ARP request. The server uses the client's MAC from `chaddr` instead.
4. Via relay (`giaddr` ≠ 0): to the relay. Renewing (`ciaddr` ≠ 0): unicast to ciaddr. Otherwise broadcast if the flag is set, else unicast to `yiaddr` at the client's MAC.
5. 54 = server identifier; 51 = lease time; 58 = T1 renewal time (50%); 59 = T2 rebinding time (87.5%).
6. The server only reserved it provisionally; it becomes a lease when the server ACKs the client's Request.
7. Two: the client picks one and its Request names the chosen server. None: the client retries Discover with backoff and eventually falls back to link-local.
8. Watch for Offers from unexpected Server-IDs (`nmap --script broadcast-dhcp-discover`, Wireshark `bootp.option.dhcp == 2`), enable DHCP snooping on switches.
</details>

**Practice**

1. Capture an Offer from your own network. Identify `yiaddr`, server ID, lease, T1/T2, router, DNS, and whether it was unicast or broadcast.
2. In the lab, reserve an address for the client MAC and prove the Offer changes.
3. Explain (draw) the Offer's path through a switch in the unicast and broadcast cases.
4. Modify the Chapter 35 Python script to construct the Offer bytes from §7 and verify the IP and UDP checksums.
5. Find out what happens on your home router when you plug a second router into a LAN port (capture the two Offers).

---

**Next:** [Chapter 37 – DHCP Request and Acknowledge](37_dhcp_request_and_acknowledge_in_details.md)
