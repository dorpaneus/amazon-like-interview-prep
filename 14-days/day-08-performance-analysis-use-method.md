# 📊 Day 8 — Performance Analysis: vmstat, iostat, sar, USE method

> [!NOTE]
> **The Goal:** Week 2 starts. The shift: from "what is broken" to "what is slow, and which resource is the bottleneck." This is hyperscale-operations bread-and-butter — it's the question Amazon's Managed Operations team works on every day.
> 
> By tonight you should reach for the right tool first instead of `top`-ing reflexively, and you should know **Brendan Gregg's USE method** by name (the interviewer will recognize it).

---

## 🧠 Morning Block — A method, then the tools

### 8a. The USE method

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

### 8b. `vmstat`

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
| `cs` | **Context switches/sec.** Sustained high cs (~100k+) = thread contention. (The scheduler at work — see 8C.) |
| `us` / `sy` | User / System (kernel) CPU%. **High `sy` is a red flag** (syscall thrashing/locks). |
| `id` / `wa` | Idle / **I/O wait** (CPUs idle waiting on disk). High `wa` = I/O bottleneck. |
| `st` | **Steal** — hypervisor took CPU from this VM. |

> [!WARNING]
> **Critical Trap:** The **first line** of `vmstat` shows averages since boot. **Ignore it** — only the subsequent lines (the interval samples) are meaningful for "what's happening now."

### 8c. The CPU scheduler — CFS, priorities, context switches

`vmstat` just showed you the scheduler's inputs and outputs: `r` (the **run queue** — runnable tasks waiting for a CPU) is what it has to place, and `cs` (**context switches**) is it doing the placing. This section is what sits behind those two columns.

**What the scheduler does:** decide which runnable task runs on which CPU, and for how long. Only *runnable* tasks (state `R`) compete; sleeping (`S`) and I/O-blocked (`D`) tasks aren't on the run queue.

**CFS — the Completely Fair Scheduler** (the default from 2007 to 2023, kernel 2.6.23 onward; still what you'll meet on most production boxes and in interviews):
- The idea in one line: give every runnable task a **fair share** of CPU. No fixed time-slices.
- It tracks a per-task **virtual runtime (`vruntime`)** — roughly, how much CPU the task has already had — and always runs the task with the *lowest* `vruntime`, kept sorted in a red-black tree. A task that's had less runs next; fairness emerges from "always pick whoever's behind."

**Priority — `nice` and real-time:**
- **Niceness** runs from **-20 (highest priority) to +19 (lowest)**. It *weights* how fast `vruntime` accrues: a low-nice (high-priority) task accrues virtual time more slowly, so the scheduler picks it more often. `nice -n N cmd` to launch, `renice N -p PID` to change a running one. Lowering below 0 needs root.
- **Real-time classes** (`SCHED_FIFO`/`SCHED_RR`, set with `chrt`) sit *above* all normal tasks — a runnable RT task preempts every `nice` task regardless of niceness. Used for latency-critical work; dangerous because a runaway RT task can starve the box.

> [!NOTE]
> **EEVDF (kernel 6.6+, late 2023):** newer kernels replaced CFS with EEVDF (Earliest Eligible Virtual Deadline First). Same goal — fair sharing via virtual runtime — but it adds explicit **virtual deadlines** so latency-sensitive tasks run *sooner* without being allowed to consume more total CPU. For interviews the CFS mental model (fair share via `vruntime`) is still the right answer; mention EEVDF as "what replaced it, same fairness idea plus better latency."

**Context switches — voluntary vs involuntary:** a switch = the kernel saves one task's CPU state and loads another's.
- **Voluntary** — the task gives up the CPU itself because it *blocks* (waiting on I/O, a lock, a sleep). Lots of these = I/O-bound or lock-heavy.
- **Involuntary** — the kernel *preempts* it (its slice ended, or a higher-priority task woke up). Lots of these = CPU contention: more runnable work than cores.
- `vmstat`'s `cs` is the sum; `pidstat -w 1` splits them per process (`cswch/s` voluntary, `nvcswch/s` involuntary). Each switch costs ~1–3µs plus cold cache/TLB afterward — negligible once, painful at 100k+/s.

**Inspect it:**
```bash
pidstat -w 1                 # per-process voluntary/involuntary context switches
chrt -p $(pgrep -n sshd)     # scheduling policy + priority of a process
nice -n 10 some_batch_job    # launch de-prioritized
sudo renice -5 -p 1234       # bump a running process up
taskset -cp 0,1 1234         # pin a process to specific CPUs (affinity)
```

> [!TIP]
> **Interview hook — "load average is high but CPU% is low":** load average = run-queue length **plus** `D`-state tasks, averaged over 1/5/15 min (the EWMA from Day 2's `/proc/loadavg`). A high load with low `%us`/`%sy` means the queue is full of **I/O-blocked** (`D`) tasks, not CPU-bound ones — the scheduler isn't the problem, the disk/network is. This is exactly the USE "saturation" signal from 8A.

> [!IMPORTANT]
> **☁️ The AWS Bridge: the scheduler can only divide what the hypervisor grants.** On a burstable **T-class** instance, a single-threaded peg can be *either* an app limit (one core) *or* **credit exhaustion** — and they look different: app limit shows high `%us` on one core, credit exhaustion shows high **`%st`** (steal) with the run queue not especially long, because the hypervisor simply isn't scheduling your vCPU. Check CloudWatch `CPUCreditBalance` to tell them apart. On large multi-socket instances, `taskset`/IRQ affinity matters for keeping a task on its NUMA-local CPU (ties into the `%soft`/IRQ-affinity note in 8D).

### 8d. `mpstat` and per-CPU views

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

### 8e. `iostat` and I/O analysis

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

### 8f. `sar` and historical data

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

### 8g. `free`, `/proc/meminfo`, and the memory truth

```bash
free -h
grep -E '^(MemAvailable|SwapFree|Dirty|Writeback)' /proc/meminfo
```

> [!WARNING]
> **The Eternal Mistake:** Panicking that `free` is low. **Linux uses free memory for caches.** That is correct behavior. The number that tells you if you have memory pressure is **`available`** (or `MemAvailable` in `/proc/meminfo`). Memory pressure only exists if `available` is dropping sharply AND swap (`si`/`so`) is actively thrashing.

### 8h. PCP — Performance Co-Pilot

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

## 💻 Midday Block — Hands-on labs

### Lab 1: Snapshot a healthy system

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

### Lab 2: Generate CPU load and watch it

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

### Lab 3: Priorities & context switches

Build on the CPU load from Lab 2, but now watch the *scheduler*.

```bash
# Saturate all cores, then watch the run queue and context switches
stress-ng --cpu $(nproc) --timeout 60s &
vmstat 1                 # r climbs to ~nproc; note cs (context switches/sec)
pidstat -w 1             # per-process: cswch/s (voluntary) vs nvcswch/s (involuntary)
```

**Now make priority visible.** Pin two CPU hogs to the *same* core so they must compete, with opposite nice values:

```bash
taskset -c 0 nice -n  19 stress-ng --cpu 1 --timeout 30s &   # de-prioritized
sudo taskset -c 0 nice -n -5 stress-ng --cpu 1 --timeout 30s &   # prioritized (needs root)

# Watch the split — the high-priority worker should get the larger share of core 0
top -p "$(pgrep -d, stress-ng)"      # or: pidstat -u 1
chrt -p "$(pgrep -n stress-ng)"      # show one worker's scheduling policy/priority
```

> [!TIP]
> **What to notice:** on a *busy* core the nice values clearly skew the CPU split; on an *idle* core they barely matter, because there's nothing to compete with. That's the whole point of CFS fairness — niceness only decides who wins *under contention*. Watch `pidstat -w`: pure CPU hogs show mostly **involuntary** switches (preemption); add `--io` workers and you'll see **voluntary** ones (blocking on I/O) climb instead.

### Lab 4: Generate I/O load

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

### Lab 5: Memory pressure and swapping

```bash
TOTAL=$(awk '/MemAvailable/ {print int($2/1024)}' /proc/meminfo)
stress-ng --vm 1 --vm-bytes ${TOTAL}M --timeout 30s &

# Terminal 2
vmstat 1           # Watch: free drops, then si/so go non-zero → swapping
free -h            # available drops sharply
```

### Lab 6: A USE-method walk through

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

### Lab 7: Mock incident — run the diagnosis

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

## 🎯 Afternoon Block — Drills + Story #8

### Self-check

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
13. What does the CPU scheduler do, and what's CFS's core idea? What replaced it, and why?
14. Voluntary vs involuntary context switch — which `pidstat` field shows each, and what does a high count of each imply?
15. `nice`/`renice` vs `chrt` — what does each control? Does `nice -n -20` guarantee a process more CPU on an otherwise idle box?

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
13. The scheduler decides which runnable task runs on which CPU and for how long. **CFS** (Completely Fair Scheduler, default 2007–2023) gives each task a fair share by tracking per-task **virtual runtime** and always running whoever has had the least, kept in a red-black tree — no fixed time-slices. Kernel 6.6 (2023) replaced it with **EEVDF**, which keeps the fairness goal but adds explicit virtual deadlines so latency-sensitive tasks run sooner without taking more total CPU.
14. **Voluntary** (`cswch/s`): the task gave up the CPU itself by blocking — on I/O, a lock, or a sleep. A high count means I/O-bound or lock-contended work. **Involuntary** (`nvcswch/s`): the kernel preempted it (slice expired, or a higher-priority task woke). A high count means CPU contention — more runnable work than cores. `vmstat`'s `cs` is the two summed.
15. `nice`/`renice` set the **niceness** (-20..+19) that weights a normal `SCHED_OTHER` task's share of CPU. `chrt` sets the **scheduling policy** and real-time priority (`SCHED_FIFO`/`RR`), which outranks every nice value. On an idle box niceness barely matters — with no contention even a `+19` task gets the CPU on demand; nice only changes who wins when tasks actually compete.

</details>

### 🤝 Behavioral — Story #8: Are Right, A Lot

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
