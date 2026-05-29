# 🛡️ Day 6 — Networking II: tcpdump, Firewalls, NAT, Connectivity Debugging

> [!NOTE]
> **The Goal:** Yesterday: protocols and the model. Today: the tools you reach for when something is broken. The interview scenario is "I can't reach this service" — and the answer isn't a guess, it's a method. 
> 
> By tonight you should have a layered debugging tree memorized and `tcpdump` BPF filters in muscle memory.

---

## 🧠 Morning Block (3h) — The diagnostic toolkit

### 6A. The connectivity-debugging tree (45 min)

The **single most valuable thing** to internalize today. When asked "service X is unreachable, what do you do," walk this tree out loud:

```text
1. Define the problem
   - From where? To where? Which service/port?
   - When did it start? After what change?
   - Always-failing, or intermittent?

2. Layer 1 (physical / link)
   - ip link show           → is the interface UP, NO-CARRIER?
   - ethtool eth0           → link speed/duplex, link detected?

3. Layer 2 (Ethernet / ARP)
   - ip neigh               → can we ARP the gateway?
   - tcpdump arp            → are ARP requests/replies happening?

4. Layer 3 (IP / routing)
   - ip addr                → do we have an IP on the right subnet?
   - ip route get <dst>     → which route + interface + source IP?
   - ping <gateway>         → reachable?
   - ping <dst>             → end-to-end ICMP

5. DNS (if dst is a name)
   - dig <name>             → resolve correctly?
   - cat /etc/resolv.conf   → right resolvers?
   - getent hosts <name>    → uses nsswitch order

6. Path
   - mtr -rwn <dst>         → where does it die? loss starting where?

7. Layer 4 (TCP/UDP reachability)
   - nc -vz <dst> <port>    → TCP handshake completes?
   - For UDP: nc -vzu, or app-specific test (dig for DNS, etc.)

8. Firewall — local and remote
   - sudo iptables -L -n -v --line-numbers
   - sudo nft list ruleset
   - firewall-cmd --list-all
   - On cloud: security groups / NACLs

9. Layer 7 (application)
   - curl -v / openssl s_client / app-specific
   - Server logs

10. Capture if still mysterious
    - tcpdump on both ends, compare
```

> [!WARNING]
> **The Discipline:** Don't skip layers. "It's probably DNS" is a guess. ICMP first, capture second, change nothing until you have data.

### 6B. tcpdump mastery (1h 15min)

`tcpdump` is the single most powerful network diagnostic tool. Be fast with it.

**Basic invocation**:
```bash
sudo tcpdump -i eth0                  # capture on interface
sudo tcpdump -i any                   # all interfaces (Linux-specific)
sudo tcpdump -nni eth0                # -n no name resolution, -nn no port names either
sudo tcpdump -nni eth0 -c 10          # stop after 10 packets
sudo tcpdump -nni eth0 -w cap.pcap    # write to pcap file (binary, full packets)
sudo tcpdump -r cap.pcap              # read back
```

**The flags you'll use most**:

| Flag | Meaning |
| :--- | :--- |
| `-i <iface>` | interface (`any` for all) |
| `-nn` | no DNS, no port-to-name lookups (faster, clearer) |
| `-c <N>` | stop after N packets |
| `-w <file>` | write pcap (raw packets, replay later or open in Wireshark) |
| `-r <file>` | read pcap |
| `-s 0` | snap full packet (default in modern tcpdump anyway) |
| `-vv` / `-vvv` | more verbose decoding |
| `-X` | hex + ASCII payload (great for HTTP, breaks with TLS) |
| `-A` | ASCII payload only (cleaner for plaintext protocols) |
| `-e` | include link-layer header (MACs) |

**BPF filter expressions** — the language inside the quotes. Memorize these patterns:

```bash
# Host-based
'host 1.2.3.4'
'src host 1.2.3.4'
'dst host 1.2.3.4'
'host 1.2.3.4 and host 5.6.7.8'

# Port-based
'port 443'
'src port 443'
'tcp port 80 or tcp port 443'

# Protocols
'tcp' / 'udp' / 'icmp' / 'arp'

# Combinations — use 'and', 'or', 'not'
'tcp port 443 and host 1.2.3.4'
'not port 22'                                    # essential when SSH'd in

# Network
'net 10.0.0.0/8'

# TCP flags — bitmask matching
'tcp[tcpflags] & tcp-syn != 0'                   # any packet with SYN bit
'tcp[tcpflags] == tcp-syn'                       # SYN only (no ACK = handshake initiation)
'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'         # connection setup or teardown
'tcp[tcpflags] & tcp-rst != 0'                   # resets (often interesting)
```

> [!CAUTION]
> **The `not port 22` trick is non-negotiable** when you're SSH'd into the box. Without it, your capture is dominated by your own SSH traffic and may even feed back on itself.

**Reading tcpdump output** — anatomy of a line:
```text
14:32:11.234567 IP 10.0.5.42.51234 > 93.184.216.34.443: Flags [S], seq 12345, win 64240, options [mss 1460,sackOK,...], length 0
```
- **Timestamp**
- **`IP`** — protocol decoded
- **`10.0.5.42.51234`** — source IP and port
- **`>`** — direction
- **`93.184.216.34.443`** — destination IP and port
- **`Flags [S]`** — TCP flags (`S` SYN, `.` ACK only, `S.` SYN-ACK, `F` FIN, `R` RST, `P` PUSH)

**The combinations** to look for in real triage:
- **Lots of `[S]` with no `[S.]` reply** → SYN reaching destination but no response. Firewall dropping inbound, or destination silent.
- **`[S]` then `[R.]` immediately** → port closed (TCP RST is the polite "no service here" reply).
- **`[S]` retransmits, no reply** → packet lost, or firewall silently dropping.
- **Lots of `[R]`** → resets in the middle of a connection. App crashes, intermediaries timing out, RST attacks.

### 6C. Firewalls — iptables, nftables, firewalld (45 min)

Three layers of abstraction on the same kernel netfilter framework:
- **`iptables`** — old userspace tool. Still works on RHEL; rules go through nftables compatibility layer underneath on RHEL 8+.
- **`nftables`** (`nft`) — modern replacement. Single tool for IPv4/IPv6/ARP/bridge. **The default on RHEL 8+.**
- **`firewalld`** — daemon with zone-based abstraction; configures nftables/iptables underneath. **The recommended interface on RHEL.**

**netfilter mental model**: packets traverse **chains** at hooks in the kernel network stack:
- `INPUT` — packets destined for this host
- `OUTPUT` — packets generated by this host
- `FORWARD` — packets routed through this host (only matters if IP forwarding is on)
- `PREROUTING` / `POSTROUTING` — for NAT, before/after routing decision

**firewalld — the right RHEL way**:
```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all                    # active zone, full config

# Open a port
sudo firewall-cmd --add-port=8080/tcp           # runtime only (lost on restart)
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload                      # apply --permanent changes
```

> [!TIP]
> **The interview-friendly debugging move**: When "service is unreachable":
> 1. `sudo nft list ruleset | less` (or `iptables -L -n -v`).
> 2. Look at the *packet counters* (`-v`) on the relevant rule. Counter going up = packets are hitting that rule. Counter zero = packets aren't even arriving at that rule.

> [!WARNING]
> **DROP vs REJECT (Interview Gotcha):** > `DROP` silently discards (sender retransmits, hangs). 
> `REJECT` sends an ICMP unreachable (sender gets immediate "connection refused"). 
> DROP is "stealthier" but worse for legitimate users — they see hangs instead of immediate failure. Security Groups in AWS act as a DROP.

### 6D. NAT — what your router does (15 min)

- **SNAT (source NAT)** — rewrite the source IP outbound. Used when many internal hosts share one public IP (like an AWS NAT Gateway). 
- **DNAT (destination NAT)** — rewrite the destination IP inbound. Used to expose an internal service on a public IP. Port forwarding is DNAT.
- **MASQUERADE** — a special form of SNAT where the source IP is whatever the outbound interface's IP currently is. Used when the public IP is dynamic.

**Conntrack** is what makes NAT work for stateful protocols (TCP) — the kernel remembers connections so reply packets can be reverse-mapped:
```bash
sudo cat /proc/net/nf_conntrack | head    # active connections being tracked
```

> [!IMPORTANT]
> **☁️ The AWS Bridge: Conntrack & NAT Gateways**
> When the **conntrack table fills up** (`nf_conntrack: table full, dropping packet`), new connections fail. 
> * **On-Prem:** You tune `net.netfilter.nf_conntrack_max`. 
> * **In AWS:** If an EC2 instance acts as a NAT (like a bastion or VPN server), it uses conntrack and can hit this limit. However, AWS managed NAT Gateways do *not* use Linux conntrack under the hood, but they have a hard limit of 55,000 concurrent connections to a single destination. 

---

## 💻 Midday Block (2.5h) — Hands-on labs

### Lab 1: tcpdump in anger — capture and read the handshake (30 min)

```bash
# Two terminals.

# Terminal 1 — start capture, exclude SSH
sudo tcpdump -nni any -c 30 'host [www.google.com](https://www.google.com) and not port 22'

# Terminal 2 — generate the traffic
curl -s [https://www.google.com](https://www.google.com) -o /dev/null
```

**Read it together**:
- Find the DNS query (UDP 53 to your resolver) and reply.
- Find the SYN, SYN-ACK, ACK to port 443.
- Find the TLS records (TLS handshake — first ones are cleartext).
- Find the encrypted application data.
- Find the FIN/ACK teardown.

Walk through it line by line. If you can identify each packet, you can answer "what does this tcpdump output show me?" cold.

### Lab 2: Diagnose a "broken" port (30 min)

```bash
# Start a server you can reach
sudo dnf install -y nmap-ncat 2>/dev/null
nc -l 9999 &                                     # listener on TCP 9999
PID=$!

# From the same host
nc -vz 127.0.0.1 9999                            # should succeed

# Now block it
sudo firewall-cmd --add-rich-rule='rule family=ipv4 source address=127.0.0.1 port port=9999 protocol=tcp drop'

# Re-test
nc -vz 127.0.0.1 9999                            # hangs (DROP)
# In another terminal, start a capture before the test
sudo tcpdump -nni lo -c 5 'port 9999' &
nc -vz -w 3 127.0.0.1 9999                       # SYN with no reply visible

# Now switch DROP to REJECT
sudo firewall-cmd --remove-rich-rule='rule family=ipv4 source address=127.0.0.1 port port=9999 protocol=tcp drop'
sudo firewall-cmd --add-rich-rule='rule family=ipv4 source address=127.0.0.1 port port=9999 protocol=tcp reject'

# Re-test
nc -vz -w 3 127.0.0.1 9999                       # immediate "Connection refused" (ICMP back)

# Cleanup
sudo firewall-cmd --remove-rich-rule='rule family=ipv4 source address=127.0.0.1 port port=9999 protocol=tcp reject'
kill $PID 2>/dev/null
```

> [!NOTE]
> **Lesson**: DROP shows up in capture as SYN with no reply (and the client retransmits, eventually times out). REJECT shows up as immediate ICMP unreachable, client fails fast. **You can tell DROP vs REJECT from the packet capture alone.**

### Lab 3: Walking the firewall (30 min)

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all

# What's active now in nftables?
sudo nft list ruleset | head -80

# Practice: open HTTP/HTTPS permanently
sudo firewall-cmd --add-service=http --add-service=https --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-services

# Inspect packet counters on rules — find what's actively dropping
sudo iptables -L -n -v --line-numbers | head -40
# Look at the pkts and bytes columns. Non-zero on a DROP rule = active drops happening.
```

### Lab 4: Capture-and-replay workflow (30 min)

This is the production workflow — capture, transfer, analyze offline.

```bash
# Capture for 10 seconds, rotate every 5 seconds, keep 2 files
sudo tcpdump -nni any -G 5 -W 2 -w /tmp/cap-%Y%m%d-%H%M%S.pcap 'tcp port 443' &
PID=$!
sleep 11
sudo kill $PID

ls -la /tmp/cap-*.pcap

# Read, count, summarize
for f in /tmp/cap-*.pcap; do
  echo "=== $f ==="
  tcpdump -nnr $f | wc -l
done

# Show only resets
for f in /tmp/cap-*.pcap; do
  tcpdump -nnr $f 'tcp[tcpflags] & tcp-rst != 0'
done
```

### Lab 5: Mock incident — full diagnostic walk (30 min)

Open a fresh terminal. The scenario: a teammate Slacks you "the API at `internal-api.example.com:8443` is down for me, but the dashboards say it's up." Walk the tree out loud, command by command.

```bash
# 1. Define
# "down" — connection refused, timeout, slow, wrong response?

# 2. L1
ip link show

# 3. L3
ip route get $(dig +short internal-api.example.com | head -1) 2>/dev/null || ip route get 8.8.8.8

# 4. DNS
dig internal-api.example.com
getent hosts internal-api.example.com

# 5. Path
mtr -rwn -c 5 8.8.8.8                  # stand-in destination

# 6. L4 reachability
nc -vz -w 3 google.com 443             # works as a control test
nc -vz -w 3 internal-api.example.com 8443    # would fail (doesn't exist)

# 7. Local firewall — is it OUR box dropping outbound?
sudo nft list ruleset | grep -i drop | head

# 8. Capture during the failing attempt
sudo tcpdump -nni any -c 10 'host internal-api.example.com' &
nc -vz -w 3 internal-api.example.com 8443
```

---

## 🎯 Afternoon Block (1.5h) — Drills + Story #6

### Self-check (45 min)

1. Walk the connectivity-debugging tree, layer by layer.
2. I see `[S]` packets in tcpdump going out, but no `[S.]` coming back. What are the possibilities?
3. DROP vs REJECT — what's the difference in packet behavior, and which would the user prefer?
4. Why is `not port 22` in your tcpdump filter when you're SSH'd in?
5. Write the BPF filter for: SYN packets to port 443 from any source.
6. Explain what a TCP RST in the middle of a session typically means.
7. nftables vs iptables vs firewalld — what's the relationship?
8. Service is up locally (`ss -tlnp` shows it), but unreachable from another host. Where do you look first?
9. What does `MASQUERADE` do and where would you use it?
10. `nf_conntrack: table full, dropping packet` in dmesg. What's happening, what's the fix?
11. I capture on the source and the destination simultaneously. SYN leaves source, never arrives at destination. What's the next move?
12. tcpdump shows no DNS reply for a query. Where do you check next?

<details>
<summary><strong>Answers</strong> (click to reveal)</summary>

1. Define problem → L1 (link state) → L2 (ARP) → L3 (IP, route, gateway, ICMP) → DNS → path (mtr) → L4 reachability (nc -vz) → firewall (local + remote, plus cloud security groups) → L7 (curl/openssl) → capture if still stuck.
2. (a) Network drops the packet en route, (b) destination's firewall DROPs (silent), (c) destination has no listener and the kernel is silently dropping (extremely unusual — should send RST), (d) routing problem on the return path so the SYN-ACK doesn't make it back, (e) destination is overloaded and `tcp_max_syn_backlog` is full.
3. DROP = silently discard, sender hangs and retransmits. REJECT = send ICMP unreachable, sender fails immediately. REJECT is friendlier to legitimate users; DROP is preferred against scanners (gives them less info).
4. Without it, every packet you receive triggers a tcpdump output line that you ship over your SSH session, which is itself a packet you receive — capture explodes and the box may struggle. Standard hygiene.
5. `'tcp dst port 443 and tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack == 0'` — SYN bit set, ACK bit not set, dst port 443.
6. The peer (or an intermediary) closed the connection abnormally. Could be: app crashed, stateful firewall timed out and is now sending RST to both sides, load balancer removed the backend mid-flight, app called `close()` with unread data in the socket buffer. Usually correlates with errors in the application log on one side.
7. `firewalld` is a daemon providing zone-based abstraction; it programs `nftables` underneath on modern RHEL (or `iptables` on older). `nftables` is the modern in-kernel framework. `iptables` is the legacy command and on RHEL 8+ is a compatibility shim over nftables. Use firewalld when you can; drop to nft for fine control.
8. (1) Destination's local firewall (`firewall-cmd --list-all` on the dest), (2) any cloud security group / NACL between, (3) the listener is bound to `127.0.0.1` instead of `0.0.0.0` (`ss -tlnp` will show `127.0.0.1:port` vs `*:port` or `0.0.0.0:port`).
9. SNAT where the source IP is whatever the outbound interface currently has — used when public IP is dynamic (typical home router NAT). Programmed as `-j MASQUERADE` in iptables NAT table.
10. The kernel's connection-tracking table is full and new connections can't be tracked, so they're dropped. Bump `net.netfilter.nf_conntrack_max` (raise it), and `nf_conntrack_buckets` (the hash table size, ~max/4 is reasonable). Investigate why so many connections — could be legitimate growth or a connection leak / SYN flood.
11. Packet is being dropped between source and destination. Capture at intermediate hops if accessible. If not, `mtr` to find where loss starts. Possible causes: route blackhole, MTU issue (set `df` bit, see if "fragmentation needed" comes back), upstream firewall, ECMP path issue.
12. Was the query sent to the right resolver? Check `/etc/resolv.conf` and what tcpdump shows as destination. Is the resolver reachable on UDP 53 (`nc -vzu <resolver> 53` doesn't work cleanly for UDP — use `dig @<resolver> example.com`)? Is something between (firewall) blocking UDP 53? Try TCP fallback (`dig +tcp`).

</details>

### 🤝 Behavioral (45 min) — Story #6: Earn Trust

Today's LP: **Earn Trust**.

Prompt:
> "Tell me about a time you had to deliver bad news, push back on a stakeholder, or admit you were wrong."

> [!TIP]
> Earn Trust stories work in three flavors — pick whichever fits a real story you have:
> - **Vocally self-critical**: you noticed something *you* did wrong, raised it before anyone else did, owned the fix. Bonus if it would have stayed hidden.
> - **Honest delivery**: you told a leader/customer something they didn't want to hear (slipped deadline, design flaw, security risk), with the data and a path forward.
> - **Built credibility through consistent follow-through**: a relationship was strained or distrustful, and you rebuilt it through specific actions over time.

Avoid: stories where you just listened well or "communicated clearly." That's table stakes. Earn Trust requires friction — bad news, hard truth, or self-criticism.

STAR. 2-3 minutes. Out loud. Time it.
