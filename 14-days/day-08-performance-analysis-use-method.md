# Day 8 — Performance Analysis: vmstat, iostat, sar, USE method (Tue May 5)

Week 2 starts. The shift: from "what is broken" to "what is slow, and which resource is the bottleneck." This is hyperscale-operations bread-and-butter — it's the question Amazon's Managed Operations team works on every day.

By tonight you should reach for the right tool first instead of `top`-ing reflexively, and you should know **Brendan Gregg's USE method** by name (the interviewer will recognize it).

---

## Morning Block (3h) — A method, then the tools

### 8A. The USE method (45 min)

The single most useful framework for performance analysis. **Brendan Gregg, "Systems Performance"** — he's the SRE-folklore name to drop.

For every resource in the system, check three things:

- **Utilization** — % of time the resource is busy (CPU 80% busy, disk 95% busy)
- **Saturation** — extra work queued because the resource can't keep up (run queue length, I/O queue depth, page faults)
- **Errors** — error events (NIC errors, ECC errors, retransmits, page faults that swap)

The **resources** to walk through:

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| **CPU** | `mpstat`, `top` (`%us+sy`) | run queue (`vmstat r`), load average | rare hardware errors |
| **Memory** | `free`, `/proc/meminfo` | swap activity (`vmstat si/so`), OOM kills | ECC errors |
| **Disk** | `iostat -xz 1` (`%util`) | `iostat` (`avgqu-sz`, `await`) | `dmesg`, SMART, `iostat` errors |
| **Network** | `sar -n DEV 1` (rxkB/txkB vs link speed) | `ss -i` (cwnd, retrans), `nstat` | `ip -s link`, ifconfig errors |
| **System** | open files, PIDs, mounts | (varies) | dmesg, journal |

**Why USE is interview gold**: it forces you to **check every resource before forming a hypothesis**. Most candidates jump to "probably DNS" or "probably the database" without data. The USE walk takes ~3 minutes and rules out 80% of guesses immediately.

**The opening line for any "system is slow" interview question**:

> "I'd start with the USE method — for each resource (CPU, memory, disk, network), check utilization, saturation, and errors. That tells me which subsystem to dive into. Concretely, I'd run `vmstat 1`, `mpstat -P ALL 1`, `iostat -xz 1`, `sar -n DEV 1`, and `free -m` first."

That's a memorized opening worth practicing out loud.

### 8B. `vmstat` (30 min)

`vmstat 1` is the single most useful first-run command. One line of output captures CPU, memory, swap, I/O, and context-switch activity.

```bash
vmstat 1 5
```

Output columns:

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
```

**Read in priority order**:

| Column | Meaning |
|---|---|
| `r` | **Run queue length** — runnable processes. > nproc means CPU-bound. |
| `b` | Blocked processes (uninterruptible sleep, usually I/O). |
| `swpd` | KB swapped to disk. |
| `free` | Free memory KB. |
| `buff` | Buffer cache KB (block I/O metadata). |
| `cache` | Page cache KB (file contents). |
| `si` / `so` | Swap **in / out** per second. **Non-zero = memory pressure.** |
| `bi` / `bo` | Blocks **in / out** per second (disk I/O). |
| `in` | Interrupts/sec. |
| `cs` | **Context switches/sec.** Sustained high cs (~100k+) suggests too many threads/processes contending. |
| `us` | User CPU%. |
| `sy` | System (kernel) CPU%. **High sy on a non-kernel workload is a red flag** — usually syscalls thrashing or lock contention. |
| `id` | Idle. |
| `wa` | **I/O wait** — CPUs idle waiting on disk. High wa = I/O bottleneck. |
| `st` | **Steal** — hypervisor took CPU from this VM. **Non-zero on EC2 = noisy neighbor** or burstable-instance credit exhaustion. |

**Critical**: the **first line** of `vmstat` shows averages since boot. **Ignore it** — only the subsequent lines (the interval samples) are meaningful for "what's happening now."

### 8C. `mpstat` and per-CPU views (15 min)

`mpstat` breaks CPU down per-core — essential when you have an imbalance:

```bash
mpstat -P ALL 1
```

What you're looking for:

- **One CPU pinned at 100%, others idle** → single-threaded bottleneck (the canonical Python GIL problem, or an unsharded workload).
- **Even spread, all cores busy** → genuine compute-bound.
- **`%iowait` skewed to one CPU** → IRQ affinity bound to that CPU; consider RSS or RPS.
- **`%soft` (softirq) high** → network packet processing dominating; same fix.
- **`%steal` non-zero** → hypervisor.

`top` then `1` (toggle per-CPU) shows the same data live.

### 8D. `iostat` and I/O analysis (45 min)

The disk-perf tool. Get the version with extended fields:

```bash
iostat -xz 1
# -x extended fields, -z skip idle devices
```

The columns that matter:

| Column | Meaning |
|---|---|
| `r/s`, `w/s` | Reads / writes per second (IOPS) |
| `rkB/s`, `wkB/s` | Throughput KB/s |
| `rrqm/s`, `wrqm/s` | Read/write requests **merged** per second (kernel coalescing). High merge ratio = sequential workload. |
| `r_await`, `w_await` | **Mean time per request, ms**. Includes queue + service. |
| `aqu-sz` (or `avgqu-sz`) | Average queue depth — saturation signal. |
| `%util` | % of time the device had I/O in flight. |

**Reading it**:

- **High `%util`, low `await`** — disk is busy but fast (e.g. NVMe at ~80% util with 0.5ms await — fine).
- **High `%util`, high `await`** — disk is saturated and queueing. Slow.
- **`await` >> service time** — requests waiting in queue. Reduce concurrency, faster disk, or batching.
- **`%util` 100% on a spinning disk with random I/O** — disk is the bottleneck. Period.

**Trap**: `%util` on multi-queue NVMe / SSD can show 100% while the device is far from saturated. `%util` measures "any I/O outstanding" not "device busy." For modern devices, look at IOPS and throughput vs the device's spec sheet, and `await` vs expected service time.

**On AWS**: EBS volumes have IOPS and throughput limits. Hitting them shows as elevated `await` and `%util` while the OS-level utilization metrics look fine. **Cross-reference with CloudWatch metrics on the EBS volume itself** (`VolumeQueueLength`, `VolumeThroughputPercentage`) — that's where throttling shows up.

**`iotop` and per-process I/O**:

```bash
sudo iotop -o                  # only processes doing I/O
sudo iotop -oP                 # per-process aggregated (no threads)
```

Or via `/proc/[pid]/io`:

```bash
cat /proc/$(pgrep -f myapp)/io
# read_bytes / write_bytes are the most useful
```

### 8E. `sar` and historical data (30 min)

`vmstat`/`iostat` show **now**. `sar` shows **history**. Critical for the after-the-fact "what was the system doing during last night's incident?"

`sar` is part of the `sysstat` package. By default it samples every 10 minutes via cron and stores a binary log per day in `/var/log/sa/sa<DD>` for the current month.

```bash
# Install / enable
sudo dnf install -y sysstat
sudo systemctl enable --now sysstat

# Read historical data
sar -u                         # CPU all day today (default)
sar -u -f /var/log/sa/sa03     # CPU on day 3 of this month
sar -u -s 14:00:00 -e 16:00:00 # CPU between 2pm and 4pm

# What sar can show
sar -u                         # CPU
sar -r                         # memory
sar -S                         # swap
sar -b                         # I/O totals
sar -d                         # per-device disk
sar -n DEV                     # network
sar -n TCP                     # TCP stats
sar -q                         # run queue and load
sar -W                         # swap pages

# Live (acts like vmstat)
sar 1 5
sar -n DEV 1 5
```

The interview-friendly framing: **"sar gives me a recording I can rewind. When the on-call ticket says 'site was slow at 3am,' I run `sar -q -s 02:55 -e 03:15` to see if the run queue was elevated, then `sar -u` and `sar -d` to localize."**

### 8F. `free`, `/proc/meminfo`, and the memory truth (15 min)

```bash
free -h
free -m
```

The columns:

- `total` — installed memory
- `used` — actually used by processes (excluding caches)
- `free` — completely unused
- `shared` — tmpfs
- `buff/cache` — buffers + page cache (instantly reclaimable)
- `available` — what a new allocation could likely use without swapping (free + most of cache)

**The eternal mistake**: panicking that `free` is low. **Linux uses free memory for caches.** That's correct. The number that tells you if you have memory pressure is **`available`** (or `MemAvailable` in `/proc/meminfo`).

**`/proc/meminfo`** has the full picture:

```bash
grep -E '^(MemTotal|MemFree|MemAvailable|Buffers|Cached|SwapTotal|SwapFree|Dirty|Writeback)' /proc/meminfo
```

Worth noting:

- `Dirty` — pages waiting to be written to disk. Spikes during heavy write workloads.
- `Writeback` — currently being flushed.
- `SwapFree` low + `si/so` non-zero in vmstat → swapping.
- `MemAvailable` very low and dropping → real pressure.

### 8G. PCP — Performance Co-Pilot (20 min)

PCP is the "step up" from sar — distributed, longer-retention, archive-based, plug-in architecture. The JD lists it explicitly.

```bash
sudo dnf install -y pcp pcp-system-tools
sudo systemctl enable --now pmcd pmlogger

# See what's collected
pcp                                # high-level snapshot
pmrep -t 5sec :sar-u               # mimic sar
pmrep -t 5sec mem.util.free network.interface.in.bytes
pcp atop                           # combined process viewer
pmstat                             # vmstat-style live
```

Bullet points sufficient — know the names: `pmcd` (collector daemon), `pmlogger` (archives metrics), `pcp atop` (one of the user-facing viewers). The differentiator from `sar`: PCP archives are queryable across machines, and the metrics namespace is extensible (you can plug in custom collectors).

---

## Midday Block (2.5h) — Hands-on labs

### Lab 1: Snapshot a healthy system (20 min)

Before generating load, capture what "normal" looks like on your VM:

```bash
vmstat 1 5
mpstat -P ALL 1 5
iostat -xz 1 5
free -h
sar -u 1 5
ss -s                              # socket summary
```

**Predict before reading**: what should `r` (run queue) be on an idle box? What about `wa%`? What about `%util`? Note your idle baseline.

### Lab 2: Generate CPU load and watch it (30 min)

Two terminals. **Install `stress-ng`** if you don't have it:

```bash
sudo dnf install -y stress-ng
```

```bash
# Terminal 1 — saturate all cores for 60s
stress-ng --cpu $(nproc) --timeout 60s

# Terminal 2 — watch
vmstat 1
# Note: r climbs to ~nproc, us% goes high, id% drops
mpstat -P ALL 1
# Note: every CPU pinned

# Now run only 1 worker to create imbalance
stress-ng --cpu 1 --timeout 60s
mpstat -P ALL 1
# Note: ONE cpu pinned, others idle → unbalanced workload signal
```

**The pattern to recognize**: if `vmstat` shows 100% CPU but `mpstat -P ALL` shows only one core busy, you have a **single-threaded bottleneck**. The system isn't out of CPU; the *application* is. This distinction is what an interviewer wants to hear.

### Lab 3: Generate I/O load (30 min)

```bash
# Terminal 1 — write 1GB sequentially, then random reads
stress-ng --hdd 1 --hdd-bytes 1G --timeout 60s

# Terminal 2 — watch
iostat -xz 1
# Note: high w/s, wkB/s, await climbing, %util at or near 100
vmstat 1
# Note: bo (blocks out) high, wa% climbing, b (blocked) > 0
```

**Now reproduce the canonical interview scenario** — fsync-bound workload (the database write pattern):

```bash
stress-ng --io 4 --hdd 1 --hdd-opts dsync,fsync --timeout 30s
# Terminal 2
iostat -xz 1
# await goes very high, %util 100, throughput modest
```

The difference in iostat output between async writes (high throughput, lowish await) and sync writes (low throughput, high await) is the fingerprint of a database under fsync pressure.

### Lab 4: Memory pressure and swapping (30 min)

**Caveat**: this can make the box very slow. Reduce the timeout if you only have a small VM.

```bash
# Check current swap setup
free -h
cat /proc/meminfo | grep -i swap
swapon --show

# Generate memory pressure
# (Adjust --vm-bytes to your VM's memory; aim for ~120% of free)
TOTAL=$(awk '/MemAvailable/ {print int($2/1024)}' /proc/meminfo)
stress-ng --vm 1 --vm-bytes ${TOTAL}M --timeout 30s &

# Terminal 2
vmstat 1
# Watch: free drops, then si/so go non-zero → swapping
free -h
# available drops sharply
```

When the workload finishes:

```bash
free -h                            # cache repopulates
sar -B 1 5                         # paging activity
```

### Lab 5: A USE-method walk through (30 min)

**Run all five primary commands together** — this is the "first 60 seconds" routine. Make a habit of it.

```bash
# Terminal 1 — keep this running
vmstat 1
```

```bash
# Terminal 2
mpstat -P ALL 1 5
iostat -xz 1 5
sar -n DEV 1 5                     # network throughput per interface
free -h
ss -s
```

This **5-command sweep** is what Brendan Gregg calls the **"60-second performance analysis"**:

1. `uptime` — load averages
2. `dmesg | tail` — recent kernel events
3. `vmstat 1` — system summary
4. `mpstat -P ALL 1` — per-CPU
5. `iostat -xz 1` — disk
6. `free -m` — memory
7. `sar -n DEV 1` — network throughput
8. `sar -n TCP,ETCP 1` — TCP errors
9. `top` — process view

Memorize these. **Saying "I'd run the standard 60-second sweep — `uptime`, `dmesg`, `vmstat`, `mpstat`, `iostat`, `free`, `sar -n DEV`, `sar -n TCP`, `top`" is a notable interview move.**

### Lab 6: Read historical data with sar (20 min)

If sysstat has been running for a while, you have data. If you just installed it, run a stress workload then come back tomorrow. For now:

```bash
# Today's CPU history (10-min samples)
sar -u

# Run queue history
sar -q

# Disk history (per-device)
sar -d -p                          # -p shows /dev/sda etc instead of dev8-0

# Memory history
sar -r

# Network history
sar -n DEV

# A specific time window
sar -u -s 12:00:00 -e 14:00:00

# Switch to another saved day
ls /var/log/sa/
sar -u -f /var/log/sa/sa$(date -d yesterday +%d) 2>/dev/null
```

The frame to use in the interview: **"I'd use `sar` for the historical view of a specific time window, since the alarm fired at a specific time. `vmstat` only shows now."**

### Lab 7: Mock incident — run the diagnosis (20 min)

You're paged: "p99 latency on the API doubled at 03:14." On your idle VM, role-play through the steps out loud, even though you'll find nothing. The **rhythm** matters more than the result.

```bash
# 1. Look at the recent past — was the box different at 03:14?
sar -q -s 03:00:00 -e 03:30:00 2>/dev/null   # run queue
sar -u -s 03:00:00 -e 03:30:00 2>/dev/null   # CPU
sar -d -s 03:00:00 -e 03:30:00 2>/dev/null   # disk
sar -r -s 03:00:00 -e 03:30:00 2>/dev/null   # memory

# 2. Check kernel messages
sudo dmesg -T | grep -i 'oom\|error\|warn' | tail -20
sudo journalctl --since "1 hour ago" -p err

# 3. Now look at "now" baseline to see what's normal
vmstat 1 3
iostat -xz 1 3

# 4. The application
# logs, request rate, error rate
# (this is where you'd hit your APM / dashboards)
```

The interview-friendly sentence: **"I'd cross-reference the time the alarm fired with `sar` for system-level metrics, and the application's own logs/dashboards for the layer-7 view. Most production incidents are diagnosed at the intersection of 'what changed in infra' and 'what changed in the workload.'"**

---

## Afternoon Block (1.5h) — Drills + Story #8

### Self-check (45 min)

1. What is the USE method? Walk through it for CPU.
2. `vmstat 1` — what do `r`, `b`, `wa`, `st`, `si/so`, `cs` mean, and what does each tell you?
3. Why is the first line of `vmstat 1` output unreliable?
4. `top` shows 100% CPU. `mpstat -P ALL` shows only one core busy. What's the diagnosis?
5. `%util` 100% on a modern NVMe disk — does that mean it's saturated?
6. `iostat` shows `await` of 200ms and `%util` 100%. Walk me through what's happening and how you'd dig in.
7. `wa%` is 60% in vmstat. What does that mean and what's next?
8. `st%` is 30% on an EC2 instance. What's happening and what would you do?
9. `free` shows only 200MB free but you're not under memory pressure. Reconcile.
10. What's the difference between `vmstat`, `sar`, and PCP?
11. The alert fired at 02:14 last night and the box looks fine now. How do you investigate?
12. Your application is single-threaded and pegging one core. What's your move?

<details>
<summary>Answers</summary>

1. **Utilization**, **Saturation**, **Errors** for every resource. For CPU: utilization = `top`'s `%us+%sy` (or `mpstat`); saturation = run queue length (`vmstat r`) and load average; errors = rare, mostly hardware (correctable cache errors etc).
2. `r` runnable processes (>nproc = CPU-bound), `b` blocked (usually I/O wait), `wa` CPU idle waiting on I/O, `st` time stolen by hypervisor, `si/so` swap in/out (non-zero = memory pressure), `cs` context switches/sec (sustained 100k+ = thrashing).
3. It's the average since boot, not a sample. Useless for "what's happening now."
4. Single-threaded bottleneck. The application can't use more than one core. Example: Python with the GIL, single-threaded event loop, unsharded workload. The fix is in the application (parallelize, shard work) or move to multiple processes.
5. Not necessarily. `%util` measures "any I/O outstanding," not "device fully busy." Modern NVMe handles many parallel queues; %util can hit 100% well below saturation. Look at IOPS and throughput vs device spec, and `await` vs expected service time.
6. The disk is queueing. Service time is much higher than usual, requests are stacking up. Steps: (a) `iotop -oP` to find the noisy process, (b) check `aqu-sz` for queue depth, (c) on cloud, check whether you're hitting IOPS limits (CloudWatch `VolumeQueueLength`, throttling metrics), (d) investigate the workload — is it sync writes (fsync), random I/O, or just volume? (e) consider faster storage class, more parallelism, batching, or fewer fsyncs.
7. CPUs are idle, waiting on I/O. The bottleneck is disk or network I/O, not CPU. Move to `iostat` to localize. Could also be NFS, iSCSI, or a hung block device (then `b` column is also high and processes are in D state).
8. Hypervisor is taking CPU from your VM. Either noisy neighbor on a shared instance, or you're on a burstable T-class instance and have exhausted CPU credits. Check CloudWatch `CPUCreditBalance`. Fix: instance type change, reserved/dedicated, or right-size the workload.
9. Linux uses free memory for caches (page cache + buffers). That memory is reclaimable instantly; "free" is misleading. The truth is `MemAvailable` in `/proc/meminfo` (or `available` column in `free -h`). Memory pressure shows up as `MemAvailable` low **and** swap activity (`si/so` in vmstat).
10. **vmstat** = live, single-host, system summary, no history. **sar** = single-host with historical samples (10 min default), persistent in `/var/log/sa/`. **PCP** = distributed, plugin-extensible, longer retention, designed for fleets.
11. Pull `sar` data from the time window: `sar -q -s 02:00:00 -e 02:30:00`, `sar -u`, `sar -d`, `sar -r`, `sar -n DEV`. Cross-reference with `dmesg`, `journalctl --since`, application logs, and dashboards. Most incidents are at the intersection of an infra change and a workload change.
12. Either parallelize the workload (multiprocessing, sharding) or vertically scale the CPU clock. **Throwing more cores at a single-threaded process does not help.** If it's an off-the-shelf service, check if it has a thread/worker config you can turn up.

</details>

### Behavioral (45 min) — Story #8: Are Right, A Lot

Today's LP. Story prompt:

> "Tell me about a decision you made with limited information that turned out to be right. How did you get to that judgment?"

Are Right, A Lot stories are subtle — interviewers are tired of "I made the right call." The strongest ones show:

- A genuine **fork** in the road, with reasonable people disagreeing.
- You sought **disconfirming** evidence (the LP says "seek diverse perspectives and work to disconfirm your beliefs").
- A specific decision-making process — not "I just had a feeling."
- An outcome that validated the call, ideally with a counter-factual ("if we'd gone the other way, X").

A pitfall: this LP is *not* about being smart. It's about **judgment under uncertainty**. The "I just knew" answer fails. The "here's how I weighed the signals and what I deliberately tried to disprove" answer wins.

STAR. 2-3 minutes. Out loud. Time it.
