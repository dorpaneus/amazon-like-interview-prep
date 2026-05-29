# 📊 Day 8 — Performance Analysis: vmstat, iostat, sar, USE method

> [!NOTE]
> **The Goal:** Week 2 starts. The shift: from "what is broken" to "what is slow, and which resource is the bottleneck." This is hyperscale-operations bread-and-butter — it's the question Amazon's Managed Operations team works on every day.
> 
> By tonight you should reach for the right tool first instead of `top`-ing reflexively, and you should know **Brendan Gregg's USE method** by name (the interviewer will recognize it).

---

## 🧠 Morning Block (3h) — A method, then the tools

### 8A. The USE method (45 min)

The single most useful framework for performance analysis. **Brendan Gregg, "Systems Performance"** — he's the SRE-folklore name to drop.

For every resource in the system, check three things:
- **Utilization** — % of time the resource is busy (CPU 80% busy, disk 95% busy)
- **Saturation** — extra work queued because the resource can't keep up (run queue length, I/O queue depth, page faults)
- **Errors** — error events (NIC errors, ECC errors, retransmits, page faults that swap)

**The resources to walk through**:

| Resource | Utilization | Saturation | Errors |
| :--- | :--- | :--- | :--- |
| **CPU** | `mpstat`, `top` (`%us+sy`) | run queue (`vmstat r`), load avg | rare hardware errors |
| **Memory** | `free`, `/proc/meminfo` | swap activity (`vmstat si/so`), OOM | ECC errors |
| **Disk** | `iostat -xz 1` (`%util`) | `iostat` (`avgqu-sz`, `await`) | `dmesg`, SMART, `iostat` errs |
| **Network** | `sar -n DEV 1` (rx/tx vs limit) | `ss -i` (cwnd, retrans), `nstat` | `ip -s link`, ifconfig errs |
| **System** | open files, PIDs, mounts | (varies) | dmesg, journal |

> [!TIP]
> **Why USE is interview gold:** It forces you to **check every resource before forming a hypothesis**. Most candidates jump to "probably DNS" or "probably the database" without data. The USE walk takes ~3 minutes and rules out 80% of guesses immediately.
> 
> **The Opening Line:** *"I'd start with the USE method — for each resource (CPU, memory, disk, network), check utilization, saturation, and errors. Concretely, I'd run `vmstat 1`, `mpstat -P ALL 1`, `iostat -xz 1`, `sar -n DEV 1`, and `free -m` first."*

### 8B. `vmstat` (30 min)

`vmstat 1` is the single most useful first-run command. One line captures CPU, memory, swap, I/O, and context switches.

```bash
vmstat 1 5
```

**Read in priority order**:

| Column | Meaning |
| :--- | :--- |
| `r` | **Run queue length** — runnable processes. `> nproc` means CPU-bound. |
| `b` | Blocked processes (uninterruptible sleep, usually I/O). |
| `swpd` | KB swapped to disk. |
| `free` | Free memory KB. |
| `buff`/`cache` | Buffer cache (block metadata) / Page cache (file contents). |
| `si` / `so` | Swap **in / out** per second. **Non-zero = memory pressure.** |
| `bi` / `bo` | Blocks **in / out** per second (disk I/O). |
| `cs` | **Context switches/sec.** Sustained high cs (~100k+) = thread contention. |
| `us` / `sy` | User / System (kernel) CPU%. **High `sy` is a red flag** (syscall thrashing/locks). |
| `id` / `wa` | Idle / **I/O wait** (CPUs idle waiting on disk). High `wa` = I/O bottleneck. |
| `st` | **Steal** — hypervisor took CPU from this VM. |

> [!WARNING]
> **Critical Trap:** The **first line** of `vmstat` shows averages since boot. **Ignore it** — only the subsequent lines (the interval samples) are meaningful for "what's happening now."

### 8C. `mpstat` and per-CPU views (15 min)

`mpstat` breaks CPU down per-core — essential when you have an imbalance:

```bash
mpstat -P ALL 1
```

What you're looking for:
- **One CPU 100%, others idle** → single-threaded bottleneck (Python GIL, unsharded workload).
- **Even spread, all busy** → genuine compute-bound.
- **`%iowait` skewed to one CPU** → IRQ affinity bound to that CPU.
- **`%soft` (softirq) high** → network packet processing dominating.

> [!IMPORTANT]
> **☁️ The AWS Bridge: CPU Steal Time (`%st`)**
> If `%st` (steal) is non-zero, the hypervisor is taking cycles away from you. In AWS, this means one of two things:
> 1. **Credit Exhaustion:** You are on a burstable instance (like `t3.medium`) and ran out of CPU credits. Check CloudWatch `CPUCreditBalance`.
> 2. **Noisy Neighbor:** You are on a shared-tenancy instance and another AWS customer on the same physical host is hogging resources.

### 8D. `iostat` and I/O analysis (45 min)

The disk-perf tool. Always use the extended fields:

```bash
iostat -xz 1    # -x extended fields, -z skip idle devices
```

| Column | Meaning |
| :--- | :--- |
| `r/s`, `w/s` | Reads / writes per second (IOPS) |
| `rkB/s`, `wkB/s` | Throughput KB/s |
| `r_await`, `w_await` | **Mean time per request, ms**. Includes queue + service. |
| `aqu-sz` | Average queue depth — saturation signal. |
| `%util` | % of time the device had I/O in flight. |

**Reading it**:
- **High `%util`, low `await`** — disk is busy but fast (e.g., NVMe). Fine.
- **High `%util`, high `await`** — disk is saturated and queueing. Slow.
- **`%util` 100% on a spinning disk with random I/O** — disk is the bottleneck. 

> [!IMPORTANT]
> **☁️ The AWS Bridge: EBS Limits vs OS Metrics**
> `%util` on a multi-queue NVMe/SSD can show 100% while the device is far from saturated. `%util` just measures "any I/O outstanding." 
> 
> On AWS, EBS volumes have strict IOPS and throughput limits. Hitting them shows as elevated `await` at the OS level. **Always cross-reference with CloudWatch:** Look at `VolumeQueueLength` and `VolumeThroughputPercentage`. If AWS is throttling you, `iostat` alone won't explicitly tell you *why*.

### 8E. `sar` and historical data (30 min)

`vmstat`/`iostat` show **now**. `sar` shows **history**. Critical for the "what was the system doing during last night's incident?" question. It stores binary logs in `/var/log/sa/sa<DD>`.

```bash
sudo dnf install -y sysstat
sudo systemctl enable --now sysstat

sar -u                         # CPU all day today
sar -q -s 14:00:00 -e 16:00:00 # Run queue / load between 2pm and 4pm
sar -d -p                      # Disk history (-p resolves names like /dev/sda)
sar -n DEV                     # Network history
sar -r                         # Memory history
```

### 8F. `free`, `/proc/meminfo`, and the memory truth (15 min)

```bash
free -h
grep -E '^(MemAvailable|SwapFree|Dirty|Writeback)' /proc/meminfo
```

> [!WARNING]
> **The Eternal Mistake:** Panicking that `free` is low. **Linux uses free memory for caches.** That is correct behavior. The number that tells you if you have memory pressure is **`available`** (or `MemAvailable` in `/proc/meminfo`). Memory pressure only exists if `available` is dropping sharply AND swap (`si`/`so`) is actively thrashing.

### 8G. PCP — Performance Co-Pilot (20 min)

PCP is the "step up" from `sar` — distributed, longer-retention, plugin architecture. 

```bash
sudo dnf install -y pcp pcp-system-tools
sudo systemctl enable --now pmcd pmlogger

pcp                            # high-level snapshot
pmrep -t 5sec :sar-u           # mimic sar
pcp atop                       # combined process viewer
```
*Note for interview:* Know that `pmcd` is the collector, `pmlogger` archives, and it's built for fleet-wide, extensible telemetry unlike the single-node `sar`.

---

## 💻 Midday Block (2.5h) — Hands-on labs

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

> [!TIP]
> **Predict before reading:** What should `r` (run queue) be on an idle box? What about `wa%`? What about `%util`? Note your idle baseline.

### Lab 2: Generate CPU load and watch it (30 min)

Two terminals. Install `stress-ng` (`sudo dnf install -y stress-ng`).

```bash
# Terminal 1 — saturate all cores for 60s
stress-ng --cpu $(nproc) --timeout 60s

# Terminal 2 — watch
vmstat 1           # Note: r climbs to ~nproc, us% goes high, id% drops
mpstat -P ALL 1    # Note: every CPU pinned
```

Now run only 1 worker to create imbalance:

```bash
stress-ng --cpu 1 --timeout 60s
mpstat -P ALL 1    # Note: ONE cpu pinned, others idle 
```
**Lesson:** If `vmstat` shows high CPU but `mpstat` shows only one core busy, you have a **single-threaded bottleneck**. The system isn't out of CPU; the *application* is.

### Lab 3: Generate I/O load (30 min)

```bash
# Terminal 1 — write 1GB sequentially
stress-ng --hdd 1 --hdd-bytes 1G --timeout 60s

# Terminal 2 — watch
iostat -xz 1       # Note: high w/s, await climbing, %util near 100
vmstat 1           # Note: bo high, wa% climbing, b > 0
```

**The fsync-bound database scenario:**
```bash
stress-ng --io 4 --hdd 1 --hdd-opts dsync,fsync --timeout 30s
# iostat shows await goes very high, %util 100, but throughput is modest
```

### Lab 4: Memory pressure and swapping (30 min)

```bash
TOTAL=$(awk '/MemAvailable/ {print int($2/1024)}' /proc/meminfo)
stress-ng --vm 1 --vm-bytes ${TOTAL}M --timeout 30s &

# Terminal 2
vmstat 1           # Watch: free drops, then si/so go non-zero → swapping
free -h            # available drops sharply
```

### Lab 5: A USE-method walk through (30 min)

Run this **5-command sweep** — the "60-second performance analysis":

1. `uptime` — load averages
2. `dmesg | tail` — recent kernel events
3. `vmstat 1` — system summary
4. `mpstat -P ALL 1` — per-CPU
5. `iostat -xz 1` — disk
6. `free -m` — memory
7. `sar -n DEV 1` — network throughput
8. `sar -n TCP,ETCP 1` — TCP errors
9. `top` — process view

### Lab 6: Mock incident — run the diagnosis (20 min)

You're paged: "p99 latency on the API doubled at 03:14." Role-play the steps out loud on your VM.

```bash
# 1. Look at the recent past — was the box different at 03:14?
sar -q -s 03:00:00 -e 03:30:00 2>/dev/null   # run queue
sar -u -s 03:00:00 -e 03:30:00 2>/dev/null   # CPU
sar -d -s 03:00:00 -e 03:30:00 2>/dev/null   # disk

# 2. Check kernel messages
sudo dmesg -T | grep -i 'oom\|error\|warn' | tail -20

# 3. Look at "now" baseline
vmstat 1 3
```

> [!TIP]
> **The Interview Answer:** "I'd cross-reference the time the alarm fired with `sar` for system-level metrics, and the application's logs for the layer-7 view. Most incidents are diagnosed at the intersection of 'what changed in infra' and 'what changed in the workload.'"

---

## 🎯 Afternoon Block (1.5h) — Drills + Story #8

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
<summary><strong>Answers</strong> (click to reveal)</summary>

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

### 🤝 Behavioral (45 min) — Story #8: Are Right, A Lot

Today's LP: **Are Right, A Lot**. 

Prompt:
> "Tell me about a decision you made with limited information that turned out to be right. How did you get to that judgment?"

> [!TIP]
> Are Right, A Lot stories are subtle — interviewers are tired of "I made the right call." The strongest ones show:
> - A genuine **fork** in the road, with reasonable people disagreeing.
> - You sought **disconfirming** evidence (the LP asks you to "work to disconfirm your beliefs").
> - A specific decision-making process — not "I just had a feeling."
> - An outcome that validated the call, ideally with a counter-factual ("if we'd gone the other way, X would have happened").

Avoid: This LP is *not* about being the smartest person in the room. It's about **judgment under uncertainty**. The "I just knew" answer fails. The "here's how I weighed the signals and what I deliberately tried to disprove" answer wins.

STAR. 2-3 minutes. Out loud. Time it.
