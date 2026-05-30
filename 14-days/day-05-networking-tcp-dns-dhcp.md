# 🌐 Day 5 — Networking I: TCP/IP, DNS, DHCP

> [!NOTE]
> **The Goal:** This is the most-tested topic in the entire interview. The "what happens when I load a website" question is a vehicle for the interviewer to probe DNS, TCP, TLS, HTTP, routing, and any layer they want depth on. 
>
> By tonight, you should be able to walk that path for 10 minutes and dive deep on any single step when interrupted.

---

## 🧠 Morning Block — The model and the protocols

### 5a. The OSI/TCP-IP model — keep it useful

Don't recite all 7 OSI layers; nobody uses them in practice. The **practical model** is 5 layers:

| Layer | What lives here | Tools you use |
| :--- | :--- | :--- |
| **5. Application** | HTTP, DNS, SSH, SMTP (TLS sits between L4 and L7) | `curl`, `dig`, `openssl s_client` |
| **4. Transport** | TCP, UDP — port numbers, ordering, reliability | `ss`, `netstat`, `nc` |
| **3. Network** | IP — addressing, routing across networks | `ip`, `ping`, `traceroute`, `mtr` |
| **2. Data link** | Ethernet, Wi-Fi, ARP — frame on a single segment | `ip link`, `ip neigh`, `ethtool` |
| **1. Physical** | Cables, NICs, signal | `ethtool`, `mii-tool`, indicator lights |

> [!TIP]
> **The Golden Rule:** When debugging, **work bottom-up**. Layer 1 broken → nothing above works. Don't waste time checking DNS if the cable is unplugged.

### 5b. The IP layer — addressing and routing (45 min)

**IPv4 addressing**:
- 32 bits, written as 4 octets: `10.0.5.42`
- A subnet is `<address>/<prefix-length>`: `10.0.5.0/24` means first 24 bits are network, last 8 are host → 256 addresses, 254 usable.
- **Network address** (all-zeros host bits) and **broadcast** (all-ones host bits) aren't usable for hosts.

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

> [!IMPORTANT]
> **☁️ The AWS Bridge: VPC Routing & Security**
> In a standard Linux environment, `iptables` or `firewalld` controls port access. In AWS, the network fabric itself intercepts traffic:
> * **Security Groups (SGs):** Stateful, operate at the Instance level. If you allow inbound port 80, the return traffic is automatically allowed.
> * **Network ACLs (NACLs):** Stateless, operate at the Subnet level. If you allow inbound port 80, you *must* explicitly allow the ephemeral ports (32768-60999) outbound, or the return traffic is dropped.
> * **Default Route:** In an AWS public subnet, the default route (`0.0.0.0/0`) points to an Internet Gateway (IGW). In a private subnet, it points to a NAT Gateway.

### 5c. TCP and UDP — the transport layer

**TCP** — reliable, ordered, connection-oriented. **UDP** — best-effort, connectionless, no ordering. Both use **port numbers** (16-bit) for multiplexing.

**The TCP three-way handshake**:
```text
Client                             Server
  | --- SYN (seq=X) -----------> |
  | <-- SYN-ACK (seq=Y, ack=X+1) |
  | --- ACK (ack=Y+1) ---------> |
  |                              |
  | <==== data flowing ========> |
```

Then teardown with FIN/ACK in each direction (4 packets, since each side closes independently):
```text
  | --- FIN ---> |
  | <-- ACK ---- |
  | <-- FIN ---- |
  | --- ACK ---> |
```

> [!WARNING]
> **TIME-WAIT is the high-frequency interview topic.** After an active close, the closing side holds the socket in TIME-WAIT for **2× MSL** (~60s on Linux) so any late-arriving duplicate packets don't confuse a new connection. Lots of TIME-WAITs on a busy server is normal.
> 
> **CLOSE-WAIT is a bug signal.** If you see thousands of CLOSE-WAIT, the application isn't calling `close()` on its sockets. Fix the app, not the kernel.

**Ephemeral port exhaustion**: Outbound connections pick a source port from `net.ipv4.ip_local_port_range` (default `32768-60999` ≈ 28k ports). Lots of short-lived outbound connections + TIME-WAIT can exhaust this. Symptoms: "Cannot assign requested address." Mitigations: `tcp_tw_reuse`, longer-lived connections, more source IPs.

### 5d. DNS — the resolution dance

**Resolution path** when a host looks up `www.example.com`:
1. **Browser cache** — recent lookups (~minutes).
2. **OS resolver cache** — `nscd`, `systemd-resolved`.
3. **`/etc/hosts`** — static overrides; `/etc/nsswitch.conf` sets the order.
4. **Configured DNS resolver** (`/etc/resolv.conf`): The host asks its resolver.
5. **The Recursive Walk:** - Query a **root server**: "who handles `.com`?"
   - Query a **TLD server**: "who handles `example.com`?"
   - Query the **authoritative server**: "what's the A record for `www.example.com`?"
6. Recursive resolver caches the answer and returns it to the host.

> [!TIP]
> **TTLs matter:** When a record is updated, every cache between you and the authoritative still serves the old value until its TTL expires. This is why DNS changes "take time to propagate."

### 5e. DHCP — DORA

The four-message exchange — UDP ports 67 (server) and 68 (client):
1. **Discover** — client broadcasts (`0.0.0.0` → `255.255.255.255`): "anyone got an IP for me?" Includes client MAC.
2. **Offer** — DHCP server(s) on the segment respond with an IP plus lease terms.
3. **Request** — client broadcasts which offer it accepts (broadcast so other servers see the choice and retract their offers).
4. **Acknowledge** — chosen server confirms; client configures the interface.

### 5f. The big walkthrough — "what happens when I load a website"

End-to-end. Practice this until it's smooth. Each numbered step is a place an interviewer might say "wait, dive deeper on that."

1. **You type `https://www.example.com/path` and hit Enter.**
2. **URL parsing** — browser splits scheme (`https`), host (`www.example.com`), port (default 443), path (`/path`).
3. **DNS resolution** — browser cache → OS resolver cache → `/etc/hosts` → configured resolver → if not cached, recursive walk: root → `.com` → `example.com` authoritative → A/AAAA record returned to browser.
4. **ARP / routing** — browser passes the packet to the kernel. Kernel looks up the route to the destination IP, finds the next hop (gateway). ARPs for the gateway's MAC if not cached.
5. **TCP three-way handshake** — SYN → SYN-ACK → ACK. Now there's a TCP connection on port 443.
6. **TLS handshake** — ClientHello (TLS version, cipher suites, SNI = `www.example.com`), ServerHello (chosen cipher, certificate chain), key exchange (ECDHE), Finished. 
7. **Certificate validation** — browser checks: signed by a CA in its trust store, not expired, matches hostname, not revoked.
8. **HTTP request** — `GET /path HTTP/1.1`, headers (`Host: www.example.com`, `User-Agent`, cookies).
9. **Server-side processing** — load balancer (terminates TLS, picks backend), backend app, composes response.
10. **HTTP response** — status line (`200 OK`), headers, body (HTML).
11. **Browser parsing & subresources** — HTML parser starts. `<link>`, `<script>`, `<img>` may trigger another DNS+TCP+TLS+HTTP cycle.
12. **JS execution & rendering** — DOM is built, CSSOM is built, render tree, layout, paint. 

> [!TIP]
> **The Interview Answer:** If the interviewer asks "what could go wrong at each step?", the answer is at every step there's a tool: `dig` for DNS, `ip route get`/`mtr` for routing, `tcpdump`/`ss` for TCP, `openssl s_client`/`curl -v` for TLS+HTTP.

---

## 💻 Midday Block — Hands-on labs

### Lab 1: Inspect the local network stack

```bash
ip addr                              # interfaces and IPs
ip link                              # link-layer state, MAC, MTU
ip route                             # routing table
ip route get 8.8.8.8                 # which route + source IP would be used
ip neigh                             # ARP/ND cache

# Connectivity tests
ping -c 3 google.com                 # tests L3 + DNS
mtr -rwn -c 10 8.8.8.8               # combined ping + traceroute, statistical

# DNS
dig +short google.com
dig +trace google.com                # show the recursive walk yourself
dig -x 8.8.8.8                       # reverse lookup
```

### Lab 2: TCP states and `ss`

```bash
# Listening sockets
ss -tlnp                             # TCP listening, numeric, with process

# Established connections
ss -tan state established
ss -tan state time-wait | head
ss -s                                # summary by state

# Generate connections to see TIME-WAIT
for i in $(seq 1 20); do curl -s https://www.google.com -o /dev/null; done
ss -tan state time-wait | head      # TIME-WAITs from your closures

# Watch handshake live (terminal 2)
sudo tcpdump -nni any 'tcp port 443' -c 20
# In terminal 1
curl -s https://www.google.com -o /dev/null
```

### Lab 3: Building HTTP requests by hand

```bash
# Plain HTTP with nc — see the raw request/response
printf 'GET / HTTP/1.1
Host: example.com
Connection: close

' | nc example.com 80

# HTTPS with openssl — TLS handshake + HTTP
openssl s_client -connect www.google.com:443 -servername www.google.com -quiet 2>/dev/null <<EOF
GET / HTTP/1.1
Host: www.google.com
Connection: close

EOF

# curl with full verbose
curl -v https://www.google.com -o /dev/null

# Just timing
curl -w '
dns:%{time_namelookup}s connect:%{time_connect}s tls:%{time_appconnect}s ttfb:%{time_starttransfer}s total:%{time_total}s
' -s -o /dev/null https://www.google.com
```

### Lab 4: tcpdump for the canonical scenarios

```bash
# DNS queries from this host
sudo tcpdump -nni any port 53 -c 20

# Common BPF filters (be ready to write these in the interview)
# - traffic to a specific host:    'host 1.2.3.4'
# - specific port:                 'port 443'
# - SYN packets only:              'tcp[tcpflags] & tcp-syn != 0'
# - DNS traffic:                   'udp port 53'
# - between two hosts:             'host 1.2.3.4 and host 5.6.7.8'
# - exclude ssh from the capture:  'not port 22'
```

> [!NOTE]
> **The `not port 22` trick** is essential when you're SSH'd into the host — without it, your capture is full of your own SSH traffic.

---

## 🎯 Afternoon Block — Drills + Story #5

### Self-check

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
<summary><strong>Answers</strong> (click to reveal)</summary>

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

### 🤝 Behavioral — Story #5: Customer Obsession

Today's LP: **Customer Obsession**.

Prompt:
> "Tell me about a time you went beyond what was asked to address a customer's underlying need."

> [!TIP]
> For SRE/DevOps roles, "customer" is often **internal**: another engineering team, a partner team, end users of a service you operate. The strongest stories show:
> - You **listened to what they said and noticed what they didn't say**.
> - You **dug into the underlying need**, not the literal request.
> - You delivered something better than what was asked for.
> - A measurable result on *their* terms (their workflow, their incident rate, their on-call burden).

Avoid: "They asked for X, I built X, they were happy." That's not Customer Obsession; that's just delivery. The LP is about **owning the outcome they actually wanted**, even if it meant pushing back on what they initially asked for.

STAR. 2-3 minutes. Out loud. Time it.
