# Day 5 — Networking I: TCP/IP, DNS, DHCP (Sat May 2)

This is the most-tested topic in the entire interview. The "what happens when I load a website" question is a vehicle for the interviewer to probe DNS, TCP, TLS, HTTP, routing, and any layer they want depth on. By tonight, you should be able to walk that path for 10 minutes and dive deep on any single step when interrupted.

Reported questions this hits: DNS walkthrough, "what happens when I load a website," DHCP, debugging connectivity issues, network buffer concepts (preview for Day 10).

---

## Morning Block (3h) — The model and the protocols

### 5A. The OSI/TCP-IP model — keep it useful (30 min)

Don't recite all 7 OSI layers; nobody uses them in practice. The **practical model** is 5 layers:

| Layer | What lives here | Tools you use |
|---|---|---|
| **5. Application** | HTTP, DNS, SSH, SMTP, TLS sits between L4 and L7 | `curl`, `dig`, `openssl s_client` |
| **4. Transport** | TCP, UDP — port numbers, ordering, reliability | `ss`, `netstat`, `nc` |
| **3. Network** | IP — addressing, routing across networks | `ip`, `ping`, `traceroute`, `mtr` |
| **2. Data link** | Ethernet, Wi-Fi, ARP — frame on a single segment | `ip link`, `ip neigh`, `ethtool` |
| **1. Physical** | Cables, NICs, signal | `ethtool`, `mii-tool`, indicator lights |

**The rule**: when debugging, **work bottom-up**. Layer 1 broken → nothing above works. Don't waste time checking DNS if the cable is unplugged.

### 5B. The IP layer — addressing and routing (45 min)

**IPv4 addressing**:

- 32 bits, written as 4 octets: `10.0.5.42`
- A subnet is `<address>/<prefix-length>`: `10.0.5.0/24` means first 24 bits are network, last 8 are host → 256 addresses, 254 usable.
- **Network address** (all-zeros host bits) and **broadcast** (all-ones host bits) aren't usable for hosts.
- Common subnet sizes:

| Prefix | Mask | Hosts |
|---|---|---|
| /24 | 255.255.255.0 | 254 |
| /25 | 255.255.255.128 | 126 |
| /26 | 255.255.255.192 | 62 |
| /27 | 255.255.255.224 | 30 |
| /30 | 255.255.255.252 | 2 (point-to-point) |

**Private (RFC 1918) ranges** — memorize:

- `10.0.0.0/8`
- `172.16.0.0/12` (i.e. `172.16.0.0` through `172.31.255.255`)
- `192.168.0.0/16`
- `169.254.0.0/16` is **link-local** — auto-assigned when DHCP fails ("APIPA"). Seeing this on an interface is a red flag.

**Routing logic** — when a host has a packet for `203.0.113.5`:

1. Walk the routing table top-down (most specific match wins — longest prefix).
2. If destination is on a directly-connected subnet → ARP for the destination's MAC, send directly.
3. Else → send to the matching route's gateway (ARP for the *gateway's* MAC).
4. If no route matches → **default route** (`0.0.0.0/0`).
5. No default route either → "Network unreachable."

```bash
ip route                    # routing table
ip route get 8.8.8.8        # which route would be chosen?
```

**ARP** (the L2/L3 glue): IP packets need a destination MAC on the local segment. ARP broadcasts "who has 10.0.5.1?" and the answer is cached.

```bash
ip neigh                    # ARP cache
```

**IPv6 in 60 seconds**: 128 bits, hex with colons: `2001:db8::1`. `::` collapses one run of zeros. **Link-local** addresses start `fe80::/10` and are auto-configured. **Stateless Address Autoconfig (SLAAC)** lets hosts derive addresses from router advertisements without DHCP. Don't get deep, but don't be surprised by IPv6 in traceroutes.

### 5C. TCP and UDP — the transport layer (1h)

**TCP** — reliable, ordered, connection-oriented. **UDP** — best-effort, connectionless, no ordering. Both use **port numbers** (16-bit) for multiplexing.

**The TCP three-way handshake**:

```
Client                          Server
  | --- SYN (seq=X) -----------> |
  | <-- SYN-ACK (seq=Y, ack=X+1) |
  | --- ACK (ack=Y+1) ---------> |
  |                              |
  | <==== data flowing ========> |
```

Then teardown with FIN/ACK in each direction (4 packets, since each side closes independently):

```
  | --- FIN ---> |
  | <-- ACK ---- |
  | <-- FIN ---- |
  | --- ACK ---> |
```

**TCP states** (visible in `ss -tan`):

| State | Meaning |
|---|---|
| `LISTEN` | Server waiting for SYN |
| `SYN-SENT` | Client sent SYN, waiting |
| `SYN-RECV` | Server got SYN, sent SYN-ACK, waiting for ACK |
| `ESTABLISHED` | Connection open |
| `FIN-WAIT-1` / `FIN-WAIT-2` | Closing side waiting for peer's FIN |
| `CLOSE-WAIT` | Peer closed; we still need to call close() |
| `TIME-WAIT` | Local close complete; holding port to catch stragglers |
| `CLOSED` | Done |

**TIME-WAIT is the high-frequency interview topic**. After active close, the closing side holds the socket in TIME-WAIT for **2× MSL** (~60s on Linux) so any late-arriving duplicate packets don't confuse a new connection on the same 4-tuple. **Lots of TIME-WAITs on a busy server is normal**; you tune with `net.ipv4.tcp_tw_reuse=1` (safe) — **not** `tcp_tw_recycle` (removed since kernel 4.12; was unsafe with NAT).

**CLOSE-WAIT is a bug signal**. If you see thousands of CLOSE-WAIT, the application isn't calling `close()` on its sockets. Fix the app, not the kernel.

**Ephemeral port exhaustion**: outbound connections pick a source port from `net.ipv4.ip_local_port_range` (default `32768 60999` ≈ 28k ports). Lots of short-lived outbound connections + TIME-WAIT can exhaust this. Symptoms: "Cannot assign requested address." Mitigations: `tcp_tw_reuse`, longer-lived connections, more source IPs.

**Key TCP concepts**:

- **MSS (Maximum Segment Size)** — payload per TCP segment, derived from MTU minus headers. Standard Ethernet MTU 1500 → IP+TCP headers 40 → MSS 1460.
- **Receive window** — flow control; how much the receiver is willing to accept. Scales up with `tcp_window_scaling`.
- **Congestion control** — `cubic` (Linux default), `bbr` (Google's, latency-aware). Set with `net.ipv4.tcp_congestion_control`.
- **Nagle's algorithm** — coalesces small writes; disable with `TCP_NODELAY` for latency-sensitive apps.
- **Keepalives** — `tcp_keepalive_time` etc; periodic empty packets to detect dead peers. Many apps set their own.

**UDP** is much simpler: send a datagram, hope it arrives. No connection state, no ordering, no retransmit. Used by DNS, DHCP, NTP, QUIC (which adds reliability on top in userspace), and most "real-time" things where retransmit-after-delay is worse than loss.

### 5D. DNS — the resolution dance (45 min)

**Record types you must know**:

| Type | Meaning |
|---|---|
| `A` | IPv4 address |
| `AAAA` | IPv6 address |
| `CNAME` | Alias to another name |
| `NS` | Authoritative name servers for a zone |
| `MX` | Mail servers (with priority) |
| `TXT` | Arbitrary text — SPF, DKIM, verification |
| `PTR` | Reverse — IP back to name (used in `in-addr.arpa`) |
| `SOA` | Start of Authority — zone metadata, refresh times |
| `SRV` | Service location (used by Kerberos, AD) |

**Resolution path** when a host looks up `www.example.com`:

1. **Browser cache** — recent lookups (~minutes).
2. **OS resolver cache** — `nscd`, `systemd-resolved`, varies by distro.
3. **`/etc/hosts`** — static overrides; `/etc/nsswitch.conf` sets the order (`hosts: files dns`).
4. **Configured DNS resolver** (in `/etc/resolv.conf` or via `systemd-resolved`): the host asks its resolver — typically a recursive resolver at the ISP, in the VPC, or `8.8.8.8`/`1.1.1.1`.
5. The recursive resolver does the actual walk if it doesn't have a cached answer:
   - Query a **root server** (one of 13 anycasted root letter clusters): "who handles `.com`?"
   - Root replies with NS records for `.com` TLD servers.
   - Query a `.com` TLD server: "who handles `example.com`?"
   - TLD replies with NS records for `example.com`'s authoritative servers.
   - Query the authoritative: "what's the A record for `www.example.com`?"
   - Authoritative replies with the answer.
6. Recursive resolver caches the answer (TTL-bounded) and returns it to the host.

**Iterative vs recursive**: the *client* asks recursively ("just give me the answer"). The *resolver* does iterative queries (it walks the tree itself). Authoritative servers usually only answer for their zones, not recurse for clients.

**Transport**: UDP port 53 by default. Falls back to TCP for responses too large (>512 bytes traditionally; **EDNS0** raised this but TCP is still used for zone transfers and DNSSEC).

**TTLs matter**: when a record is updated, every cache between you and the authoritative still serves the old value until its TTL expires. This is why DNS changes "take time to propagate."

### 5E. DHCP — DORA (15 min)

The four-message exchange — UDP ports 67 (server) and 68 (client):

1. **Discover** — client broadcasts (`0.0.0.0` → `255.255.255.255`): "anyone got an IP for me?" Includes client MAC.
2. **Offer** — DHCP server(s) on the segment respond with an IP plus lease terms (mask, gateway, DNS, NTP, lease time, etc.).
3. **Request** — client broadcasts which offer it accepts. Broadcast (not unicast) so other servers see the choice and retract their offers.
4. **Acknowledge** — chosen server confirms; client configures the interface.

**Lease lifecycle**:

- **T1 (50% of lease)** — client tries to renew, unicast to the original server. Usually succeeds.
- **T2 (~87.5%)** — if T1 failed, client broadcasts a `REBINDING` request to *any* DHCP server.
- **Lease expiration** — if both failed, client releases and starts over with Discover.

**DHCP relay**: clients broadcast, broadcasts don't cross routers. So routers run a **DHCP relay** (`ip helper-address` on Cisco) that forwards the Discover to a DHCP server on another segment. This is how enterprise networks have one central DHCP server for many subnets.

### 5F. The big walkthrough — "what happens when I load a website" (45 min)

End-to-end. Practice this until it's smooth. Each numbered step is a place an interviewer might say "wait, dive deeper on that."

1. **You type `https://www.example.com/path` and hit Enter.**
2. **URL parsing** — browser splits scheme (`https`), host (`www.example.com`), port (default 443), path (`/path`).
3. **DNS resolution** — browser cache → OS resolver cache → `/etc/hosts` → configured resolver → if not cached, recursive walk: root → `.com` → `example.com` authoritative → A/AAAA record returned to browser.
4. **ARP / routing** — browser passes the packet to the kernel. Kernel looks up the route to the destination IP, finds the next hop (gateway). ARPs for the gateway's MAC if not cached.
5. **TCP three-way handshake** — SYN → SYN-ACK → ACK. Now there's a TCP connection on port 443.
6. **TLS handshake** — ClientHello (TLS version, cipher suites, SNI = `www.example.com`), ServerHello (chosen cipher, certificate chain), key exchange (ECDHE), Finished. With **TLS 1.3** this is 1 round-trip; with **TLS 1.2** it's 2 round-trips. SNI lets one IP serve many hostnames with different certs.
7. **Certificate validation** — browser checks: signed by a CA in its trust store, not expired, matches hostname, not revoked (OCSP/CRL — most browsers use stapled OCSP).
8. **HTTP request** — `GET /path HTTP/1.1`, headers (`Host: www.example.com`, `User-Agent`, cookies, `Accept-Encoding: gzip`).
9. **Server-side processing** — load balancer (terminates TLS, picks backend), backend app (may hit cache like Redis, may query DB, runs business logic), composes response.
10. **HTTP response** — status line (`200 OK`), headers (content-type, length, cache directives, `Set-Cookie`), body (HTML).
11. **Browser parsing & subresources** — HTML parser starts. Each `<link>`, `<script>`, `<img>`, `@font-face` may trigger another DNS+TCP+TLS+HTTP cycle (or reuse existing connections via HTTP/2 multiplexing or HTTP/3 over QUIC).
12. **JS execution & rendering** — DOM is built, CSSOM is built, render tree, layout, paint. `DOMContentLoaded`, then `load`, then user-visible interactivity.
13. **Connection lifecycle** — connection may be reused (`Connection: keep-alive` is default in HTTP/1.1). Eventually FIN-ACK teardown.

If the interviewer asks "what could go wrong at each step?", the answer is at every step there's a tool: `dig`/`nslookup` for DNS, `ip route get`/`mtr` for routing, `tcpdump`/`ss` for TCP, `openssl s_client`/`curl -v` for TLS+HTTP.

---

## Midday Block (2.5h) — Hands-on labs

### Lab 1: Inspect the local network stack (30 min)

```bash
ip addr                              # interfaces and IPs
ip -br addr                          # brief
ip link                              # link-layer state, MAC, MTU
ip route                             # routing table
ip route get 8.8.8.8                 # which route + source IP would be used
ip neigh                             # ARP/ND cache

# Connectivity tests
ping -c 3 8.8.8.8                    # raw IP — tests L3
ping -c 3 google.com                 # tests L3 + DNS
mtr -rwn -c 10 8.8.8.8               # combined ping + traceroute, statistical

# DNS
cat /etc/resolv.conf                 # current resolvers
cat /etc/nsswitch.conf | grep hosts  # lookup order

dig google.com                       # full answer with sections
dig +short google.com
dig +trace google.com                # show the recursive walk yourself
dig MX gmail.com
dig -x 8.8.8.8                       # reverse lookup
```

**Predict before running**: what's your default gateway? Which DNS servers are configured? What's the TTL on `google.com`'s A record?

### Lab 2: TCP states and `ss` (30 min)

```bash
# Listening sockets
ss -tlnp                             # TCP listening, numeric, with process

# Established connections
ss -tan state established
ss -tan state time-wait | head
ss -s                                # summary by state

# What's listening on port 22?
ss -tlnp | grep :22

# Generate connections to see TIME-WAIT
for i in $(seq 1 20); do curl -s https://www.google.com -o /dev/null; done
ss -tan state time-wait | head      # TIME-WAITs from your closures

# Watch handshake live (terminal 2)
sudo tcpdump -nni any 'tcp port 443' -c 20
# In terminal 1
curl -s https://www.google.com -o /dev/null
```

**Read the tcpdump output**: identify the SYN, SYN-ACK, ACK, then the TLS records, then the FIN/ACK closing.

### Lab 3: Walking DNS with `dig +trace` (20 min)

```bash
dig +trace +nodnssec www.amazon.com
```

Read the output top-down — you'll see:

1. Roots returning the `.com` NS records.
2. `.com` returning the `amazon.com` NS records.
3. `amazon.com`'s authoritative servers returning the A record (or a CNAME first).

**Compare with a cached lookup**:

```bash
dig www.amazon.com                  # one-shot, asks your resolver
# Run it again
dig www.amazon.com                  # TTL is lower — proof of caching
```

Watch the `;; Query time:` line drop on the second run.

### Lab 4: Building HTTP requests by hand (30 min)

```bash
# Plain HTTP with nc — see the raw request/response
printf 'GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n' | nc example.com 80

# HTTPS with openssl — TLS handshake + HTTP
openssl s_client -connect www.google.com:443 -servername www.google.com -quiet 2>/dev/null <<EOF
GET / HTTP/1.1
Host: www.google.com
Connection: close

EOF

# curl with full verbose
curl -v https://www.google.com -o /dev/null

# Just timing
curl -w '\ndns:%{time_namelookup}s connect:%{time_connect}s tls:%{time_appconnect}s ttfb:%{time_starttransfer}s total:%{time_total}s\n' -s -o /dev/null https://www.google.com
```

**The timing breakdown is the practical version of the big walkthrough** — DNS time, TCP connect time, TLS handshake time, time-to-first-byte, total. This is exactly how you'd diagnose "the website is slow" in production.

### Lab 5: DHCP in action (20 min)

```bash
# What's the current lease?
sudo cat /var/lib/dhclient/*.lease 2>/dev/null
# or for systemd-networkd / NetworkManager:
nmcli -p connection show           # find your connection
nmcli connection show <name> | grep -i dhcp

# Watch DHCP traffic during a renew (run in two terminals)
# Terminal 1
sudo tcpdump -nni any -s0 'port 67 or port 68' -vv
# Terminal 2 — force a renew
sudo dhclient -r ; sudo dhclient   # for traditional dhclient
# or
sudo nmcli connection down <name> && sudo nmcli connection up <name>
```

You'll see Discover/Offer/Request/Ack in the tcpdump.

### Lab 6: tcpdump for the canonical scenarios (30 min)

```bash
# DNS queries from this host
sudo tcpdump -nni any port 53 -c 20

# Slow connection — capture and analyze
sudo tcpdump -nni any -w /tmp/cap.pcap port 80 &
PID=$!
curl http://example.com >/dev/null
sleep 2 ; sudo kill $PID

# Read it back filtered
tcpdump -nnr /tmp/cap.pcap | head -30

# Common BPF filters (be ready to write these in the interview)
# - traffic to a specific host:    'host 1.2.3.4'
# - specific port:                 'port 443'
# - SYN packets only:              'tcp[tcpflags] & tcp-syn != 0'
# - DNS traffic:                   'udp port 53'
# - between two hosts:             'host 1.2.3.4 and host 5.6.7.8'
# - exclude ssh from the capture:  'not port 22'
```

**The `not port 22` trick** is essential when you're SSH'd into the host — without it, your capture is full of your own SSH traffic.

---

## Afternoon Block (1.5h) — Drills + Story #5

### Self-check (45 min)

1. Walk me through what happens when I type `https://www.amazon.com` and press Enter.
2. Three-way handshake — packet by packet, what's in each one?
3. What is TIME-WAIT and why does it exist? When is it a problem?
4. CLOSE-WAIT vs TIME-WAIT — what does each tell you?
5. Walk through DNS resolution from cache to authoritative.
6. UDP vs TCP — when would you choose UDP for an application?
7. DORA — what is it, and what port numbers does DHCP use?
8. I see `169.254.x.x` on my interface. What's wrong?
9. Difference between `traceroute` and `mtr` — which would you reach for and why?
10. `dig` shows the answer in milliseconds the second time and 80ms the first time. Why?
11. Ephemeral port exhaustion — what is it, what's the symptom, how do you mitigate?
12. I run `curl https://example.com` and it hangs. Walk me through how you'd diagnose, layer by layer.

<details>
<summary>Answers</summary>

1. URL parse → DNS (browser → OS → /etc/hosts → resolver → recursive walk) → ARP for next hop → TCP 3WHS → TLS handshake (with SNI, cert validation) → HTTP request → server processing (LB → app → maybe cache/DB) → HTTP response → browser parses HTML, fetches subresources, renders.
2. SYN (client→server, seq=X), SYN-ACK (server→client, seq=Y, ack=X+1), ACK (client→server, ack=Y+1). Establishes both directions of sequence numbers.
3. After active close, the closing side holds the 4-tuple in TIME-WAIT for 2×MSL (~60s) so late duplicate packets from the old connection don't get accepted as part of a new connection on the same tuple. Problem on busy outbound clients — can lead to ephemeral port exhaustion. Mitigate with `tcp_tw_reuse=1`.
4. **TIME-WAIT** = normal for the side that closed first; just hold the tuple briefly. **CLOSE-WAIT** = peer closed but our app didn't call `close()`. Many CLOSE-WAITs = application bug, file descriptor leak.
5. Browser cache → OS cache → `/etc/hosts` (per nsswitch order) → configured resolver. Resolver may have it cached. If not: query root for `.com` NS, query `.com` for `example.com` NS, query `example.com` for the record. Cache result for TTL.
6. UDP for low-latency where retransmit-after-delay is worse than loss: DNS, DHCP, NTP, VoIP/streaming, gaming, telemetry. Also when you want to do reliability in userspace (QUIC).
7. DHCP four-message exchange: Discover (broadcast), Offer, Request (broadcast — so other servers retract), Acknowledge. Server is UDP/67, client is UDP/68.
8. Link-local (RFC 3927). Means DHCP failed and the OS auto-assigned. Real cause is usually no DHCP server reachable: check cable, check switch port, check VLAN, check that DHCP relay is configured if the server isn't on the local segment.
9. `traceroute` is one-shot; `mtr` is continuous and statistical (loss%, latency stddev). For diagnosing intermittent issues, `mtr -rwn` for a steady sample. For a quick "where does it die," `traceroute` is fine.
10. First lookup walked from your resolver out to the authoritative; second was served from your resolver's cache. TTL bounds how long the cache holds it.
11. Outbound connections need a unique local port. Default range ~28k. Lots of short-lived connections + TIME-WAIT can exhaust it. Symptom: `connect()` fails with EADDRNOTAVAIL ("Cannot assign requested address"). Mitigate with `tcp_tw_reuse`, longer-lived connections, more source IPs, larger `ip_local_port_range`.
12. Layer-by-layer. **L3**: `ping example.com` — if no DNS, `dig example.com` first. **L3 reachability**: `mtr` to find where it dies. **L4**: `nc -vz example.com 443` — does the TCP handshake succeed? **L7 TLS**: `openssl s_client -connect example.com:443` — does TLS handshake complete, cert valid? **L7 HTTP**: `curl -v` with timing flags. Capture with `tcpdump -nni any host example.com` to see exactly where it stalls.

</details>

### Behavioral (45 min) — Story #5: Customer Obsession

Today's LP. Story prompt:

> "Tell me about a time you went beyond what was asked to address a customer's underlying need."

For SRE/DevOps roles, "customer" is often **internal**: another engineering team, a partner team, end users of a service you operate. The strongest stories show:

- You **listened to what they said and noticed what they didn't say**.
- You **dug into the underlying need**, not the literal request.
- You delivered something better than what was asked for.
- A measurable result on *their* terms (their workflow, their incident rate, their on-call burden).

Avoid: "they asked for X, I built X, they were happy." That's not Customer Obsession; that's just delivery. The LP is about **owning the outcome they actually wanted**, even if it meant pushing back on what they initially asked for.

STAR. 2-3 minutes. Out loud. Time it.
