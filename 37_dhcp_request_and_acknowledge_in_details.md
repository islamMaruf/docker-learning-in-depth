# Chapter 37: DHCP Request and Acknowledge

> **In one sentence:** After picking an Offer, the client **broadcasts a DHCP Request** ("I accept *this* address from *that* server"), and the chosen server answers with a **DHCP ACK** that turns the provisional offer into a **lease** with a timer; the client then **checks the address is really free**, configures its interface, and starts the lease clock that drives **renewal (T1)** and **rebinding (T2)**.

**Level:** 🟡 Intermediate · **Reading time:** ~55 minutes

**Prerequisites:** Chapters [35](35_dhcp_discover_deep_dive_in_details.md) and [36](36_dhcp_offer_deep_dive_in_details.md).

---

## What you will learn

- **Why a Request is needed** at all after an Offer
- The exact contents of the **Request** (options 50 and 54) and why it is **broadcast**
- How the server **commits a lease** and builds the **ACK**, and what a **NAK** is
- What the client does **after the ACK**: configure the interface, **ARP-probe** for conflicts, announce itself, **Decline** if needed
- The **DHCP client state machine**: INIT, SELECTING, REQUESTING, BOUND, RENEWING, REBINDING, INIT-REBOOT
- **Lease renewal** (T1, T2 timings and unicast vs broadcast), **Release**, and **Inform**
- How the whole thing looks on the wire, with byte-level examples and a hands-on lab

---

## 1. Why a Request after an Offer?

If the Offer already names an address, why not just start using it? Because:

1. **Several servers may have made Offers.** The client must tell everyone which one it took, so the losers can free their reserved addresses.
2. **The server hasn't committed anything yet.** An Offer is only a temporary hold. The Request asks the server to *bind* the address to this client.
3. **Time may have passed.** Between Offer and Request another client could have taken the address; the server must be able to say **no** (a NAK).
4. **A client may remember an old lease.** Rebooting clients skip Discover and ask directly for the address they had before (INIT-REBOOT, §8).

```
Client                                      Server A (192.168.1.1)      Server B (192.168.1.2)
  │── DISCOVER (broadcast) ───────────────────►│                              │
  │◄── OFFER  .100 ───────────────────────────│                              │
  │◄── OFFER  .150 ─────────────────────────────────────────────────────────  │
  │── REQUEST (broadcast): "I take .100 from server 192.168.1.1" ─────────────►│ (B hears it too: frees .150)
  │◄── ACK  .100 (lease 24 h) ────────────────│
```

---

## 2. Part 1: The DHCP Request

### 2.1 Layer 7: the message

| Field | Value | Notes |
|---|---|---|
| op | `1` | BOOTREQUEST |
| xid | **same as the Discover** (`0x3903F326`) | Same transaction continues |
| flags | as before (`0x8000` in our example) | Tells the server how to reply |
| **ciaddr** | **`0.0.0.0`** | The client does **not** have the address yet, so this stays 0 (only *renewing* clients fill it) |
| yiaddr / siaddr / giaddr | 0 (giaddr set by a relay) | |
| chaddr | `aa:bb:cc:11:22:33` | |

**Options:**

| Option | Value | Purpose |
|---|---|---|
| **53** | `3` (Request) | Message type |
| **50** | `192.168.1.100` | **Requested IP address**: "the one from your Offer" |
| **54** | `192.168.1.1` | **Server Identifier**: "I chose *your* offer" |
| 61 | `01` + MAC | Client identifier |
| 12 | `laptop` | Host name |
| 55 | 1, 3, 6, 15, 28, 51 | Parameters the client wants (repeated) |
| 255 | end | |

**Options 50 + 54 are what make it a *selecting* Request:** the requested address goes in **option 50** (not `ciaddr`) and the chosen server in **option 54**. Any server whose ID does not match understands it lost.

### 2.2 Layer 4: UDP

Source port **68** → destination port **67** (same as Discover).

### 2.3 Layer 3: still a broadcast!

| | Value |
|---|---|
| Source IP | `0.0.0.0` (the client still doesn't own an address) |
| Destination IP | **`255.255.255.255`** |

**Why broadcast when the client knows the server's address (from option 54)?**
1. The client **must not use the offered address** as its source yet, so it has no valid source for a unicast conversation.
2. **The other servers must hear it** to release their offers.
3. It keeps things simple: the same rule as Discover.

(When a client **renews** later, it *does* have an address and unicasts. See §7.)

### 2.4 Layer 2: Ethernet

```
dst MAC: ff:ff:ff:ff:ff:ff   src MAC: aa:bb:cc:11:22:33   type 0x0800
```
Flooded by the switch, just like the Discover.

### 2.5 Byte-level view (with real checksums)

```
IPv4: 45 00 01 48 00 00 00 00 40 11 79 a6 | 00 00 00 00 | ff ff ff ff     (checksum 0x79a6, total len 328)
UDP : 00 44  00 43  01 34  84 56                                           (68→67, length 308, checksum 0x8456)
DHCP options:
   63 82 53 63                          magic cookie
   35 01 03                             53: Request
   3d 07 01 aa bb cc 11 22 33           61: client-ID
   32 04 c0 a8 01 64                    50: requested IP 192.168.1.100
   36 04 c0 a8 01 01                    54: server ID   192.168.1.1
   0c 06 6c 61 70 74 6f 70              12: "laptop"
   37 06 01 03 06 0f 1c 33              55: parameter request list
   ff                                   end
```
Frame size 342 bytes, same as Discover.

---

## 3. What the server does with the Request

```
Server 192.168.1.1 receives the Request:
  1. Is option 54 (server ID) == me?     No → I lost: release any address I had offered to this client, ignore.
                                          Yes → continue
  2. Is option 50 (requested IP) still available for this client (still reserved to it, valid for this subnet)?
        Yes → COMMIT: write lease (MAC/ID, IP, start, expiry) to the lease database; send ACK
        No  → send NAK ("that address isn't available; start over")
```
**Lease record (conceptually):**

```
IP address       Client ID / MAC       Start           Expiry (start + 24 h)   State
192.168.1.100    aa:bb:cc:11:22:33     2026-09-26 10:00  2026-09-27 10:00       ACTIVE   hostname=laptop
```
ISC dhcpd writes it to `dhcpd.leases`; dnsmasq to `dnsmasq.leases` (expiry-epoch, MAC, IP, hostname, client-ID); Windows Server to its DHCP database; consumer routers to their in-memory tables (the "DHCP client list" in the admin page).

---

## 4. Part 2: The DHCP ACK

### 4.1 Layer 7

Nearly identical to the Offer (Chapter 36), with different message type and confirmed contents:

| Field / Option | Offer | **ACK** |
|---|---|---|
| op | 2 | 2 |
| xid | copied | copied |
| **yiaddr** | offered address | **the address now leased to you** (`192.168.1.100`) |
| Option 53 | 2 | **5** (ACK) |
| Options 54, 51, 58, 59, 1, 3, 6, 15, 28 | as configured | as configured, now final (a server may adjust the lease time) |

The ACK is the authoritative configuration: the client should apply *these* values, even if they differ slightly from the Offer (e.g. the lease time).

### 4.2 Layers 4-2

Same choices as the Offer (Chapter 36 §5):

| Case | Frame |
|---|---|
| Client flag `0x0000` | Unicast: MAC `aa:bb:cc:11:22:33` ← `11:22:33:44:55:66`, IP `192.168.1.1` → `192.168.1.100`, UDP 67→68 |
| Client flag `0x8000` | Broadcast: `ff:ff:ff:ff:ff:ff`, IP `192.168.1.1` → `255.255.255.255`, UDP 67→68 |

Example (unicast): IPv4 `45 00 01 4a … 40 11 f5 ed c0 a8 01 01 c0 a8 01 64`, UDP `00 43 00 44 01 36 bf 13`, DHCP payload 302 bytes, frame 344 bytes. Only the message type byte (`35 01 05`) and checksum differ from the Offer.

### 4.3 The NAK

If the server can't honor the Request it sends **DHCPNAK** (type 6), typically **broadcast** (the client's address is not valid on this link) with a message option (56) such as "wrong network". Causes:

- The requested address was given to someone else or is outside the client's subnet (a laptop moved to a different network while trying to reuse its old lease: **INIT-REBOOT**),
- the offer expired,
- the server lost its lease database.

The client's reaction: **discard the configuration and restart at Discover.** NAKs are why moving a laptop between offices "just works": the old address is rejected, and a fresh one is negotiated.

---

## 5. After the ACK: the client's checklist

```
1. Receive ACK, validate xid / chaddr / options.
2. Record the lease: address, mask, router, DNS, server ID, T1, T2, expiry.
3. (Many clients) ARP-PROBE the address:  "Who has 192.168.1.100?"  sent with sender IP 0.0.0.0 (RFC 5227)
      • If someone answers → conflict → send DHCPDECLINE (type 4), wait ~10 s, restart at Discover.
      • Silence → the address is free.
4. Configure the interface:  ip addr add 192.168.1.100/24 dev eth0
                             ip route add default via 192.168.1.1
                             write DNS servers (resolv.conf / systemd-resolved / registry)
5. Send a gratuitous ARP / ARP announcement so neighbors update their caches.
6. Start the timers T1, T2 and lease expiry. State = BOUND.
```
Then the host can ARP for the router (Chapter 39), resolve names (DNS), and open connections.

`DHCPDECLINE` tells the server "this address is in use elsewhere; don't lease it (for now)", so a rogue static host inside the pool doesn't keep causing conflicts.

---

## 6. The lease timeline

Lease time L = 24 h (86,400 s):

```
0h ─────── T1 = 0.5 L (12 h) ─────── T2 = 0.875 L (21 h) ─────── L (24 h)
BOUND      RENEWING (unicast)        REBINDING (broadcast)        EXPIRED
```

| Time | Client state | What it sends | Where |
|---|---|---|---|
| 0 | **BOUND** | Nothing, uses the address | |
| **T1** (50%) | **RENEWING** | `DHCPREQUEST` with **ciaddr = its IP**, **unicast** to the server that granted the lease | If ACK arrives → lease timer restarts (a new full lease from now) |
| **T2** (87.5%) | **REBINDING** | `DHCPREQUEST`, **broadcast** | "Any server: extend my lease" (the original may be dead) |
| **L** (100%) | Expired | Must **stop using the address**, deconfigure, go back to INIT | Discover from scratch |

A **renewal Request** differs from the first:

| | Initial Request | Renewal (RENEWING) | Rebinding |
|---|---|---|---|
| Source IP | 0.0.0.0 | the client's leased IP | leased IP |
| Destination IP | 255.255.255.255 | **the server's IP (unicast)** | 255.255.255.255 |
| **ciaddr** | 0 | **its IP** | its IP |
| Option 50 / 54 | present | **absent** | absent |
| Frame dst MAC | broadcast | server's MAC (via ARP) | broadcast |

The renewal is a normal unicast conversation, since the client now owns a valid address (byte-level: same layout as the Request, with `ciaddr = c0 a8 01 64`, no options 50/54, IP `192.168.1.100 → 192.168.1.1`, frame still 342 bytes).

**Lease length is a policy trade-off:** short leases (minutes, guest Wi-Fi) reclaim addresses quickly but add DHCP traffic; long leases (days) reduce traffic but hold addresses for machines that left. Typical home routers: 12–24 h; corporate 8 h–8 days; hotspots 15 min–2 h.

---

## 7. The client state machine (RFC 2131)

```
                      ┌──────────────────────── NAK / lease expired ────────────────────────┐
                      ▼                                                                     │
   ┌─────────┐   send DISCOVER   ┌───────────┐  receive OFFER(s)   ┌────────────┐  send REQUEST   │
   │  INIT   │ ─────────────────►│ SELECTING │ ───────────────────►│ REQUESTING │ ────────────────┘
   └─────────┘                   └───────────┘   choose one        └────────────┘
       ▲  ▲                                                             │ receive ACK (address check OK)
       │  └── DECLINE / conflict ───────────────────────────────────────┤
       │                                                                 ▼
   ┌─────────────┐  boot with an old lease    ┌────────┐   T1    ┌──────────┐   T2    ┌───────────┐
   │ INIT-REBOOT │ ──── REQUEST (broadcast) ─►│ BOUND  │ ───────►│ RENEWING │ ───────►│ REBINDING │
   └─────────────┘                            └────────┘         └──────────┘         └───────────┘
                                                  ▲   ACK              │   ACK               │
                                                  └────────────────────┴────────────────────┘
```
- **INIT-REBOOT:** after a reboot or reconnect, a client with a still-valid remembered lease skips Discover/Offer and **broadcasts a Request** with option **50** (its old address) and *no* option 54 → the server ACKs it (fast, 2 packets) or NAKs it (moved to another network).
- Other client-initiated messages: **DHCPRELEASE** (type 7: "I'm leaving; free my address", unicast to the server, sent when you disconnect or run `dhclient -r`) and **DHCPINFORM** (type 8: "I already have an address configured statically; just give me the options", the server replies with an ACK containing options only).

---

## 8. Renewal for real: what you can see

```bash
ip -d addr show eth0                     # valid_lft 85231sec preferred_lft 85231sec  ← the remaining lease
nmcli -f DHCP4 device show eth0          # dhcp_lease_time, expiry, options
cat /var/lib/dhcp/dhclient*.leases       # (Debian/Ubuntu dhclient) lease {...} with renew/rebind/expire dates
resolvectl status | head -30             # DNS obtained from the lease
sudo dhclient -r eth0                    # DHCPRELEASE
sudo dhclient -v eth0                    # a fresh DORA
```
`valid_lft` counts down to 0 at expiry; when it passes T1 you will see a renewal in the logs (`journalctl -u NetworkManager | grep -i dhcp`, or `DHCPREQUEST for … to <server>`).

Windows: `ipconfig /all` (shows **Lease Obtained** and **Lease Expires**), `ipconfig /release`, `ipconfig /renew`. macOS: `ipconfig getpacket en0`.

---

## 9. Lab: complete DORA and renewal in namespaces

Reuse the setup of Chapter 35 §14.2 (namespaces `srv` and `cli`, dnsmasq, tcpdump), but with a **very short lease** so you can watch T1 happen:

```bash
sudo ip netns add srv; sudo ip netns add cli
sudo ip link add s0 type veth peer name c0
sudo ip link set s0 netns srv; sudo ip link set c0 netns cli
sudo ip netns exec srv ip addr add 192.168.50.1/24 dev s0; sudo ip netns exec srv ip link set s0 up
sudo ip netns exec cli ip link set c0 up
sudo ip netns exec srv tcpdump -nn -vv -e -i s0 -w /tmp/dora.pcap 'udp port 67 or udp port 68' &
sleep 1
# lease of 2 minutes → T1 ≈ 60 s. (dnsmasq's minimum lease is 2 minutes.)
sudo ip netns exec srv dnsmasq --no-daemon --port=0 --interface=s0 --bind-interfaces \
     --dhcp-range=192.168.50.100,192.168.50.150,2m --dhcp-leasefile=/tmp/dora.leases --log-dhcp &
sleep 1
# this time let dhclient CONFIGURE the interface (it only changes the namespace's c0);
# the -sf script would touch /etc/resolv.conf, so use a tiny script that configures only address+route:
cat > /tmp/dh-script.sh <<'EOF'
#!/bin/sh
case "$reason" in
  BOUND|RENEW|REBIND|REBOOT) ip addr flush dev "$interface"; ip addr add "$new_ip_address/$new_subnet_mask" dev "$interface" 2>/dev/null || ip addr add "$new_ip_address/24" dev "$interface";;
esac
exit 0
EOF
chmod +x /tmp/dh-script.sh
sudo ip netns exec cli dhclient -d -v -sf /tmp/dh-script.sh -lf /tmp/dora.lease -pf /tmp/dora.pid c0 &
sleep 5
sudo ip netns exec cli ip -br addr          # 192.168.50.10x/24 configured
sleep 65                                    # wait past T1 (≈60 s)
sudo pkill tcpdump
sudo tcpdump -nn -vv -e -r /tmp/dora.pcap 2>/dev/null | grep -E 'DHCP-Message|Request|Discover|Offer|ACK|length' | head -40
```
Look for **six** packets: Discover (broadcast), Offer, Request (broadcast, options 50 & 54, ciaddr 0), ACK, then at ~60 s a **Request unicast** to `192.168.50.1` with `ciaddr` set (renewal) and its ACK. Compare the two Requests' `Client-IP` (`ciaddr`) fields, IP addresses and destination MACs.

Also try:

```bash
sudo ip netns exec cli dhclient -r -v -sf /tmp/dh-script.sh -lf /tmp/dora.lease -pf /tmp/dora.pid c0   # RELEASE packet (unicast, type 7)
cat /tmp/dora.leases                                                                            # the server's table: entry gone
```
Clean up:
```bash
sudo pkill dhclient; sudo pkill dnsmasq; sudo pkill tcpdump
sudo ip netns del srv; sudo ip netns del cli; rm -f /tmp/dora.* /tmp/dh-script.sh
```

---

## 10. Failure and edge cases

| Situation | What happens |
|---|---|
| **Request unanswered** | Client retries with backoff, eventually restarts from Discover |
| **NAK** | Client drops the config and starts over |
| **Address conflict** found by ARP probe | DECLINE, wait, restart |
| **Server dies during the lease** | Client renews at T1 (no answer), then broadcasts at T2 (another server may take over, if it has the lease data: failover pair), else expires at L and re-Discovers |
| **Laptop changes network** | Link-change triggers INIT-REBOOT → NAK from the new network's server → fresh DORA |
| **Two servers hand out overlapping pools** | Duplicate-address conflicts; the Decline/ARP check catches some; fix the config |
| **Clock changes / hibernation** | Leases are relative; clients re-validate after resume |
| **DHCP failover** | Two servers sharing lease state (ISC/Kea failover, Windows failover, or split-scope 80/20) keep service going |

---

## 11. Security recap
- Unauthenticated: rogue servers, starvation and spoofed Release/Decline are possible → **DHCP snooping** (also builds the IP-MAC binding table used by Dynamic ARP Inspection and IP Source Guard).
- **Option 82** (relay agent info) can bind leases to a switch port.
- **DHCP authentication (RFC 3118)** exists but is essentially unused; in practice you rely on network controls (802.1X, port security).
- Leases leak host names and MACs to the whole segment; use randomized MACs and privacy options (client-ID choices) on untrusted Wi-Fi.

---

## 12. Docker and DHCP once more

| Environment | How the container/VM gets its address |
|---|---|
| Default `docker0` bridge or user-defined bridge | Docker's internal IPAM assigns from the subnet at container start (no DORA) |
| `macvlan` network | Static from Docker IPAM, or an external DHCP plugin can lease from your real DHCP server |
| VirtualBox/VMware NAT, libvirt `virbr0` | Built-in DHCP server (dnsmasq) leases to the VM's virtual NIC |
| Cloud VM | The provider's DHCP service on boot; lease renewed as in this chapter |
| Kubernetes pods | CNI plugins assign (host-local IPAM, etc.), not DHCP |

---

## 13. Common misconceptions

| Misconception | Reality |
|---|---|
| "The client uses the address after the Offer" | Only after the **ACK** (and a conflict check) |
| "Request is unicast to the server" | The initial Request is a **broadcast** (renewals are unicast) |
| "The requested IP goes in `ciaddr`" | Only in renewal/rebinding; the initial Request uses **option 50** |
| "ACK and Offer are identical" | Similar structure, but the ACK **commits** the lease and is authoritative |
| "The lease is renewed when it expires" | Renewal starts at **T1 = 50%**; expiry is the last resort |
| "A NAK is an error to fix" | It's normal, e.g. after moving between networks |
| "DHCP finishes after the ACK" | Leases are live agreements: renew, release, expire |
| "Static IPs conflict only if two people type the same one" | A static address inside a DHCP pool will be handed out too |

---

## 14. Summary

- **Request** = "I accept *this* address (option 50) from *that* server (option 54)", **broadcast** so losing servers release their offers; `ciaddr` stays 0.
- **ACK** = the server **commits the lease** and confirms the final configuration; **NAK** = "no, start over".
- After ACK the client **probes with ARP**, configures address/route/DNS, announces itself and starts the timers.
- **T1 (50%)** unicast renewal; **T2 (87.5%)** broadcast rebinding; at **100%** the address must be dropped. **Release** and **Decline** are the client's other tools; **INIT-REBOOT** shortens the process after a reboot.
- The whole DORA takes four packets and typically well under a second on a healthy network.

---

## 15. Check your understanding

1. Why does the client still broadcast the Request? Give two reasons.
2. Which options identify the chosen server and the chosen address in the Request?
3. How does a server decide between ACK and NAK?
4. What does the client do between receiving the ACK and using the address?
5. When are renewals sent, and how do they differ from the initial Request?
6. What is INIT-REBOOT, and what happens if the laptop moved networks?
7. What is DHCPDECLINE for? DHCPRELEASE? DHCPINFORM?
8. Lease is 8 hours. When are T1 and T2?

<details>
<summary>Answers</summary>

1. The client has no valid address to use as a source; and other servers that made Offers must hear it to free their reservations.
2. Option 54 (server identifier) and option 50 (requested IP address).
3. If the requested address is still reserved for this client and valid for its subnet it commits a lease and sends ACK; otherwise NAK.
4. Validates the ACK, usually ARP-probes the address for conflicts (Decline if taken), then configures the interface/route/DNS and announces with gratuitous ARP.
5. At T1 (50%) unicast to the leasing server with `ciaddr` set and no options 50/54; at T2 (87.5%) broadcast. The initial Request is broadcast with `ciaddr` 0 and options 50/54.
6. A client with a remembered lease broadcasts a Request for its old address at boot; the server ACKs it or, on another network, NAKs it and the client falls back to Discover.
7. Decline: "that address is in use, don't lease it". Release: "I'm done, free my lease". Inform: "I have an address; send me only configuration options."
8. T1 = 4 h, T2 = 7 h.
</details>

**Practice**

1. Run §9 and produce a timeline table: time, packet type, src/dst IP, src/dst MAC, ciaddr, yiaddr.
2. Make the client request a specific address (`dhclient` with `request` config or `udhcpc -r`), and see the option 50 in a Discover.
3. Add a `--dhcp-host` reservation and prove the same address after release/renew.
4. On your real network, read your current lease (`nmcli`, lease file, `ipconfig /all`) and compute when T1 and T2 fall.
5. Draw the state machine from memory, and annotate each arrow with the message sent.

---

**Next:** [Chapter 38 – Hub, Switch and Router: Network Devices](38_hub_switch_router_network_devices_in_details.md)
