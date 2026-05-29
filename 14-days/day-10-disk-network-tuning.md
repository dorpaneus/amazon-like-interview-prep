# 🖧 Day 10 — Disk and Network Tuning

> [!NOTE]
> **The Goal:** Yesterday: memory. Today: the I/O side — disk schedulers, filesystem layout, and network buffer sizing. The interview targets here are very concrete: *"Walk me through choosing an I/O scheduler for X workload"*, *"Calculate the buffer size for a 10 Gbps × 100 ms RTT path"*, *"Why is my high-BDP transfer maxing out at 30 Mbps despite a 10 Gbps link?"*
> 
> This is also the day that ties most directly to Amazon's hyperscale operations — fleets of hosts, each tuned for a workload, with EBS / network limits to navigate.

---

## 🧠 Morning Block (3h) — Throughput and latency at the kernel boundary

### 10A. I/O schedulers (45 min)

Modern Linux uses **multi-queue (blk-mq)** I/O. Pre-2017 single-queue schedulers (`cfq`, `deadline`, `noop`) are gone on RHEL 8+. The four current options:

| Scheduler | Best for | Why |
| :--- | :--- | :--- |
| **none** | Fast NVMe with deep parallel queues | Device firmware reorders better than the kernel can; scheduling adds overhead |
| **mq-deadline** | SATA SSDs, spinning HDDs, mixed workloads | Separate read/write queues, hard deadlines prevent starvation. Good default. |
| **kyber** | Low-latency reads on fast SSDs | Simple, targets read latency; auto-tunes its own depth |
| **bfq** | Desktops, multi-tenant systems wanting fairness | Budget Fair Queueing — gives each process its share. Higher CPU overhead. Bad for very fast devices. |

**Reading and setting**:

```bash
cat /sys/block/sda/queue/scheduler
# [mq-deadline] kyber bfq none      ← brackets mark current

echo none | sudo tee /sys/block/nvme0n1/queue/scheduler
```

> [!TIP]
> **Interview frame:** "I'd match the scheduler to the device. NVMe → `none` and let the device queue. SATA SSD → `mq-deadline` for predictable latency. HDDs → `mq-deadline` or `bfq` if multi-tenant fairness matters more than throughput. I'd verify the choice with `iostat -xz 1` under realistic load."

### 10B. Filesystem tuning (45 min)

The knobs that matter most for performance.

**Mount options** — set in `/etc/fstab` or `mount -o`:

| Option | Effect |
| :--- | :--- |
| `noatime` | Don't update access times on read. **Big win for read-heavy workloads.** |
| `relatime` | (Default) Update atime only if older than mtime/ctime. Reasonable compromise. |
| `nodiratime` | Skip directory atimes (small additional win on top of relatime) |
| `discard` | Issue TRIM on file deletion. Can cause spikes; better to run `fstrim` periodically. |
| `commit=N` (ext4) | Journal commit interval, seconds (default 5). Raise for throughput, lower for safety. |
| `data=writeback\|ordered\|journal` (ext4) | `ordered` (default) is safe. `writeback` is faster but risks metadata inconsistency on crash. |
| `errors=remount-ro\|panic` | What to do on FS error. `remount-ro` is the safe default. |

> [!WARNING]
> **Trap:** Never set `nobarrier` on a filesystem. Disabling write barriers risks journal corruption on power loss. It is only safe with a hardware battery-backed cache.

**XFS specifics**:
```bash
sudo xfs_info /mnt/data                  # show live config
# At mkfs time:
sudo mkfs.xfs -d agcount=16 /dev/sdb1    # more allocation groups = more parallelism
sudo mkfs.xfs -d su=64k,sw=8 /dev/sdb1   # stripe unit / stripe width (RAID alignment)
```
`agcount` (allocation groups) — XFS divides the volume into AGs, each independently managed. More AGs = more concurrent allocation across threads. Default is auto-sized; bump for write-heavy parallel workloads.

**Alignment matters**:
- Partitions should align to the device's physical block size (usually 4K).
- Modern `parted` defaults handle this; `parted -a optimal` is explicit.
- For striped storage (hardware RAID, EBS gp3): match `mkfs` stripe parameters to the stripe size.

### 10C. Network buffers and the BDP (45 min)

**Bandwidth-Delay Product** — the foundational interview concept.

> BDP = bandwidth × round-trip time

This is the amount of data "in flight" on the wire at any moment. **To fill a pipe, your TCP receive window (and the sender's buffer) must be at least BDP.** If smaller, you bottleneck on flow control regardless of how fast the link is.

| Path | Bandwidth | RTT | BDP |
| :--- | :--- | :--- | :--- |
| LAN | 10 Gbps | 1 ms | 1.25 MB |
| Same-region AWS | 10 Gbps | 5 ms | 6.25 MB |
| Cross-region AWS | 1 Gbps | 80 ms | 10 MB |
| Trans-Atlantic | 1 Gbps | 100 ms | 12.5 MB |
| Trans-Pacific high-bandwidth | 10 Gbps | 200 ms | 250 MB |

> [!NOTE]
> **Math reminder:** BDP in bytes = bandwidth (bps) × RTT (s) / 8.
> For 10 Gbps × 100 ms: `(10 * 10^9) * 0.1 / 8 = 125 MB`

**The TCP buffer sysctls**:

```bash
# Kernel-level hard caps (limits what apps can request via SO_RCVBUF/SO_SNDBUF)
sysctl net.core.rmem_max net.core.wmem_max

# TCP auto-tuning — three values: min default max
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem
```

TCP auto-tunes its window between the min and max from `tcp_rmem`/`tcp_wmem`, based on the observed BDP. **But it can't exceed `rmem_max`/`wmem_max`** — those are the hard ceilings.

**For high-BDP environments** (long-distance, fat-pipe):
```text
# /etc/sysctl.d/99-network-tuning.conf
net.core.rmem_max = 268435456           # 256 MB
net.core.wmem_max = 268435456
net.ipv4.tcp_rmem = 4096 87380 134217728     # min default max
net.ipv4.tcp_wmem = 4096 65536 134217728
net.ipv4.tcp_window_scaling = 1         # must be on for >64KB windows
```

> [!WARNING]
> **Window scaling (RFC 1323)** — TCP's window field is 16 bits, max 64 KB. Window scaling shifts it, allowing windows up to ~1 GB. **Default on**. Without it, no amount of buffer tuning helps on high-BDP paths.

**Other useful network sysctls**:
- `net.core.netdev_max_backlog` (1000) — Packet queue between NIC and IP stack. Raise if seeing drops here.
- `net.core.somaxconn` (4096) — Max `listen()` backlog. Raise for high-accept-rate servers.
- `net.ipv4.tcp_max_syn_backlog` (1024) — Half-open SYN backlog. Raise under SYN load.
- `net.ipv4.tcp_tw_reuse` (2) — Reuse TIME-WAIT for outbound (safe).

**UDP buffers** — UDP has **no auto-tuning**. Apps must size `SO_RCVBUF` explicitly. System-wide caps come from `net.core.rmem_max`. If a UDP service is losing packets and `netstat -su` shows `RcvbufErrors` > 0, bump `SO_RCVBUF` in the app and `rmem_max` system-wide.

> [!TIP]
> **Interview-ready BDP answer:** "For 10 Gbps × 100 ms, BDP is 125 MB. I'd set `net.core.rmem_max` and `wmem_max` to at least 128 MB, the `tcp_rmem`/`tcp_wmem` max values to match, and verify TCP window scaling is on. Then check with `ss -ti` during a transfer — `cwnd` and `wscale` should reflect the larger window."

### 10D. TCP congestion control (15 min)

`net.ipv4.tcp_congestion_control` — algorithm deciding how the sender ramps up and reacts to loss/delay.

| Algorithm | Behavior |
| :--- | :--- |
| `cubic` | Linux default. Loss-based. Aggressive ramp. Good on LAN, mediocre on noisy or shared paths. |
| `bbr` | Google's. Model-based — estimates bandwidth and RTT, doesn't wait for loss. Much better on high-RTT or shared paths. |

To switch:
```bash
sudo modprobe tcp_bbr
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

# BBR pairs with fq qdisc
sudo sysctl -w net.core.default_qdisc=fq
```

**When to switch**: cross-region, high-BDP, internet-facing services where packet loss isn't congestion (just noise) — BBR is usually a clear win. Within a clean datacenter LAN, the difference is small.

### 10E. NIC offloads, queues, and IRQs (30 min)

Network performance at high PPS depends on offloading work to the NIC and spreading load across CPUs.

**Offloads** — push segmentation/checksum work to hardware:
```bash
ethtool -k eth0                # show all offloads
ethtool -K eth0 tso on          # TCP segmentation offload
ethtool -K eth0 gso on          # generic segmentation offload
```

> [!WARNING]
> **LRO gotcha**: Large Receive Offload aggregates incoming TCP segments before delivering to the stack. Saves CPU on hosts. **Breaks routers and forwarding hosts** because they need to re-segment for the next hop — turn it off in those roles.

**Ring buffers** — between NIC and driver:
```bash
ethtool -g eth0                # current and max
ethtool -G eth0 rx 4096 tx 4096
```

**Multi-queue (RSS)** — NIC hashes flows across N receive queues, each tied to a CPU's IRQ. Distributes interrupt load.
```bash
ethtool -L eth0 combined 8       # set to 8 (typically = number of cores up to NIC max)
```

**NIC drop counters** — the place to look when packets are missing:
```bash
ip -s link show eth0            # rx_dropped, tx_dropped, errors
```
Drops in `rx_dropped` typically mean the receive ring filled before the kernel drained it — bump rings, more RX queues, or pin IRQs.

> [!IMPORTANT]
> **☁️ The AWS Bridge: ENA Networking limits**
> In a bare-metal environment, tuning `ethtool` ring buffers and RSS queues is mandatory for 10G+ speeds. 
> 
> In AWS, you use the Elastic Network Adapter (ENA). The ENA driver handles RSS and IRQ affinity automatically. Instead of messing with `ethtool`, if you see dropped packets in AWS, your first check should be **CloudWatch `NetworkPacketsIn` vs your Instance Type's limits.** Each instance type has a hard hardware cap on throughput (e.g., 5 Gbps, 25 Gbps) and packets per second (PPS). If you hit that cap, the hypervisor drops the packets before they even reach your OS ring buffers. 

---

## 💻 Midday Block (2.5h) — Hands-on labs

### Lab 1: I/O scheduler inspection and switching (30 min)

```bash
# Discover your devices
lsblk

# Current scheduler
for dev in $(lsblk -d -n -o NAME); do
  sched=$(cat /sys/block/$dev/queue/scheduler 2>/dev/null)
  rot=$(cat /sys/block/$dev/queue/rotational 2>/dev/null)
  echo "$dev (rotational=$rot): $sched"
done

# Switch on the fly (won't persist across reboot)
DEV=$(lsblk -d -n -o NAME | head -1)
echo "Testing on /dev/$DEV"

# Test with mq-deadline
echo mq-deadline | sudo tee /sys/block/$DEV/queue/scheduler
sudo dnf install -y fio
sudo fio --name=randread --filename=/tmp/fio.test --size=200M --rw=randread \
  --bs=4k --iodepth=32 --runtime=10 --time_based --group_reporting

# Compare IOPS and latency between the two runs
rm /tmp/fio.test
```

### Lab 2: Filesystem mount options (30 min)

```bash
# Inspect what's currently mounted with what options
findmnt -t ext4,xfs -o TARGET,SOURCE,FSTYPE,OPTIONS

# Create a test FS
sudo lvcreate -L 500M -n lv_mountlab vg_lab 2>/dev/null || true
sudo mkfs.xfs -f /dev/vg_lab/lv_mountlab
sudo mkdir -p /mnt/mountlab

# Default mount
sudo mount /dev/vg_lab/lv_mountlab /mnt/mountlab
findmnt /mnt/mountlab           # see relatime in options
sudo umount /mnt/mountlab

# noatime + nodiratime mount
sudo mount -o noatime,nodiratime /dev/vg_lab/lv_mountlab /mnt/mountlab
findmnt /mnt/mountlab

# Test atime behavior
sudo touch /mnt/mountlab/testfile
sudo stat /mnt/mountlab/testfile        # note atime
sudo cat /mnt/mountlab/testfile         # read it
sudo stat /mnt/mountlab/testfile        # atime unchanged with noatime

# Cleanup
sudo umount /mnt/mountlab
sudo lvremove -y /dev/vg_lab/lv_mountlab 2>/dev/null
```

### Lab 3: Compute and set TCP buffers (30 min)

```bash
# Current settings
sysctl net.core.rmem_max net.core.wmem_max
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem
sysctl net.ipv4.tcp_window_scaling

# Compute BDP for a scenario you choose
# Example: 10 Gbps × 100 ms RTT
python3 -c "print('BDP MB:', 10e9 * 0.1 / 8 / 1024 / 1024)"

# Apply higher limits (try non-persistent first)
sudo sysctl -w net.core.rmem_max=134217728
sudo sysctl -w net.core.wmem_max=134217728
sudo sysctl -w 'net.ipv4.tcp_rmem=4096 87380 67108864'
sudo sysctl -w 'net.ipv4.tcp_wmem=4096 65536 67108864'

# Inspect live TCP windows during a transfer (Terminal 1)
ss -ti                                  # Look for cwnd, wscale, send/recv buffers

# Generate traffic in another terminal:
# curl -o /dev/null [https://speed.cloudflare.com/__down?bytes=100000000](https://speed.cloudflare.com/__down?bytes=100000000)
```

### Lab 4: Congestion control switching (20 min)

```bash
sysctl net.ipv4.tcp_available_congestion_control
sysctl net.ipv4.tcp_congestion_control

# Load BBR
sudo modprobe tcp_bbr
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
sudo sysctl -w net.core.default_qdisc=fq

# Verify with ss
ss -tin state established | grep -i bbr | head

# Run a test transfer (cubic vs bbr)
sudo sysctl -w net.ipv4.tcp_congestion_control=cubic
curl -w 'speed: %{speed_download} B/s, total: %{time_total}s\n' \
  -o /dev/null -s [https://speed.cloudflare.com/__down?bytes=100000000](https://speed.cloudflare.com/__down?bytes=100000000)

sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
curl -w 'speed: %{speed_download} B/s, total: %{time_total}s\n' \
  -o /dev/null -s [https://speed.cloudflare.com/__down?bytes=100000000](https://speed.cloudflare.com/__down?bytes=100000000)
```

### Lab 5: NIC inspection (20 min)

```bash
# Pick an interface
IFACE=$(ip -o link show | awk -F': ' '$2 !~ /lo/ {print $2; exit}')
echo "Using: $IFACE"

# Offloads
sudo ethtool -k $IFACE | head -30

# Rings
sudo ethtool -g $IFACE

# Coalescing
sudo ethtool -c $IFACE | head -20

# Channels
sudo ethtool -l $IFACE

# Live stats including drops
sudo ethtool -S $IFACE 2>/dev/null | grep -E 'drop|miss|error|rx_packets|tx_packets' | head -20
```

---

## 🎯 Afternoon Block (1.5h) — Drills + Story #10

### Self-check (45 min)

1. NVMe device, mixed read/write workload — which I/O scheduler and why?
2. Compute the BDP for 10 Gbps × 50 ms RTT. Express in MB.
3. After setting `tcp_rmem` max to 64 MB, your throughput is unchanged on a high-RTT path. What did you miss?
4. UDP-based service is losing packets. `netstat -su` shows `RcvbufErrors`. What's the fix?
5. Difference between `cubic` and `bbr`. When would you choose each?
6. Mount option `noatime` — what does it do, and why is it commonly recommended?
7. Why might you set `nobarrier` on a filesystem, and why is it usually a bad idea?
8. Explain RSS, RPS, and RFS — when do you need each?
9. `ethtool -S` shows `rx_dropped` climbing. Walk me through the diagnosis.
10. What's TCP window scaling and why does it matter on high-BDP paths?
11. You're seeing many `TIME_WAIT` sockets and seeing `EADDRNOTAVAIL` errors on outbound connect. What sysctls help?
12. A VM's disk shows %util 100% in iostat but await is 1 ms. Real bottleneck or not?

<details>
<summary><strong>Answers</strong> (click to reveal)</summary>

1. `none` (no-op). Fast NVMe handles its own queueing in firmware; kernel scheduling adds latency. The device's hardware queues + parallelism mean kernel reordering doesn't help.
2. 10e9 × 0.05 / 8 = 62.5 MB. (Bandwidth in bits, RTT in seconds, divide by 8 for bytes.)
3. Probably window scaling. If `tcp_window_scaling=0`, the max window is 64 KB regardless of buffer size — you bottleneck on flow control. Verify with `sysctl net.ipv4.tcp_window_scaling` and `ss -ti` (look for `wscale`).
4. Bump `SO_RCVBUF` in the application (and `net.core.rmem_max` system-wide if the app can't get the size it asked for). UDP has no auto-tuning, so the receive buffer must be sized for burst tolerance.
5. **cubic**: loss-based, aggressive ramp, Linux default. Great on LAN. **bbr**: model-based, estimates bandwidth and RTT, doesn't react to packet loss the same way. Much better on high-RTT or lossy paths. Switch to BBR for cross-region or internet-facing services where loss ≠ congestion.
6. Skip access-time updates on read. Every read normally triggers a metadata write to update atime — expensive on read-heavy workloads. `noatime` eliminates this. (`relatime`, the default since 2009, is a compromise but `noatime` is a bit faster.)
7. Disables write barriers — the FS journal no longer enforces ordering between data and metadata on commits. Faster, but a crash can corrupt the journal and leave the FS in a bad state. Only safe with battery-backed cache or where corruption is acceptable.
8. **RSS** (hardware): NIC hashes flows to multiple receive queues, each bound to a CPU IRQ. Distributes interrupt load. **RPS** (software): kernel hashes packets to CPUs when hardware doesn't have enough queues. **RFS** (software): RPS that follows the consumer process, improving cache locality. Use RSS if your NIC supports enough queues; fall back to RPS/RFS otherwise.
9. Drops at the NIC ring level — packets arrived faster than the kernel could drain the ring. (a) Bump ring size (`ethtool -G`). (b) More RX queues (`ethtool -L`). (c) Check IRQ distribution — one CPU pegged? Enable RSS or pin IRQs. (d) On AWS, check Instance Network Limits in CloudWatch. 
10. TCP's window field is 16 bits, max 64 KB. Window scaling (RFC 1323) negotiates a shift, allowing windows up to ~1 GB. Without it, you can't use more than 64 KB in flight regardless of buffer sizes — so high-BDP paths bottleneck on the window, not bandwidth.
11. `net.ipv4.tcp_tw_reuse=2` lets the kernel reuse TIME-WAIT sockets for new outbound connections (safe). Widen `net.ipv4.ip_local_port_range`. Possibly lower `net.ipv4.tcp_fin_timeout`. Best long-term fix: connection pooling in the app so you're not making zillions of short-lived connections.
12. Not necessarily. `%util` measures "any I/O outstanding" — on modern multi-queue devices it can be 100% while the device is far from saturated. Look at IOPS and throughput vs device spec, and `await` vs expected service time. 1 ms await is healthy; the device isn't queueing.

</details>

### 🤝 Behavioral (45 min) — Story #10: Invent and Simplify

Today's LP: **Invent and Simplify**. 

Prompt:
> "Tell me about a time you found a way to simplify or eliminate complexity — code, process, system — that others had accepted."

> [!TIP]
> The strongest Invent and Simplify stories share a structure:
> - A status quo everyone tolerated as "just how it is".
> - You questioned it.
> - You came up with something cleaner — often by **removing** something rather than adding.
> - A measurable benefit: less code, less infrastructure, lower cost, lower on-call burden.
> - Bonus: a thing you decided **not** to build.

**Pitfalls:**
- *"I built X feature"* → that's invention without simplification. The LP wants both, especially the simplification half.
- *"I refactored a function"* → too small. Look for system-level wins.

The Amazon flavor of this LP is **"look for ideas everywhere — not invented here is not a virtue."** A story where you adapted an external idea (open-source pattern, vendor approach, industry practice) and applied it to your context lands well.

STAR. 2-3 minutes. Out loud. Time it.
