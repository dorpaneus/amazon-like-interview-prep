# Day 9 — Memory Tuning: Hugepages, Swappiness, OOM, NUMA (Wed May 6)

Day 8 was diagnosis. Days 9 and 10 are the tuning lever days. Today: memory. What the kernel actually does with RAM, the knobs that change behavior, and when each knob matters.

This maps to RHCA performance objectives: alternate page sizes, overcommit, swap behavior, NUMA-aware placement, SYSV shared memory.

Interview questions this targets: *"the app's RSS keeps growing — what tools and what tuning?"* and *"we're getting OOM kills — walk me through what's happening at the kernel level."*

---

## Morning Block (3h) — How the kernel thinks about memory

### 9A. The memory mental model (30 min)

What the kernel actually manages:

- **Physical memory** — RAM in DIMMs, divided into **pages** (4 KB default on x86_64).
- **Virtual memory** — each process sees a private virtual address space. The MMU translates virtual → physical via the page table.
- **Anonymous pages** — process heap, stack, malloc'd memory. Not backed by a file.
- **File-backed pages** — page cache (file contents), executable code, mmap'd files.
- **Reclaimable** vs **unreclaimable** — caches can be dropped or written to swap; kernel slab and some other things can't.

Memory pressure resolution order:

1. Reclaim clean page cache (free instantly).
2. Write dirty pages to disk, then reclaim (slower).
3. Swap anonymous pages out (slow).
4. OOM killer.

**Interview-friendly framing**: "Free memory isn't the metric. `MemAvailable` is. Linux uses free RAM for cache as it should. Pressure is when MemAvailable drops and the kernel starts reclaiming, then swapping."

Key `/proc/meminfo` fields beyond Day 8:

- `Active` / `Inactive` — pages on the active/inactive LRU lists
- `AnonPages` — anonymous memory in use
- `Mapped` — pages mmap'd (executables, shared libs, mmap files)
- `Slab` — kernel slab allocator (dentry/inode caches especially)
- `KReclaimable` — kernel memory reclaimable under pressure
- `Committed_AS` — total committed virtual memory (vs `CommitLimit` for overcommit headroom)

### 9B. Page cache and dirty page writeback (30 min)

When you write to a file, by default the data goes to the **page cache** in RAM. The kernel writes it to disk later. Pages waiting to be written are **dirty**.

Why this matters: bursty writes are absorbed by RAM and flushed asynchronously, which is great until the dirty set gets too large and the kernel decides "stop everything and flush" — causing latency spikes that look like the disk is broken.

Tunables in `/proc/sys/vm/`:

| Tunable | Default | Meaning |
|---|---|---|
| `dirty_ratio` | 20 | At this % of usable RAM dirty, **the writing process is throttled and blocked** until writeback catches up. |
| `dirty_background_ratio` | 10 | At this %, kernel **starts** background flushing (no blocking). |
| `dirty_expire_centisecs` | 3000 (30s) | Max age of dirty pages before forced flush. |
| `dirty_writeback_centisecs` | 500 (5s) | Flush thread wake interval. |

There are `*_bytes` variants (`dirty_bytes`, `dirty_background_bytes`) for absolute limits — better on big-RAM systems, since 20% of 256 GB is ~50 GB of dirty pages that no disk can flush quickly.

**Interview frame**: "On high-RAM systems I lower `dirty_ratio`/`dirty_background_ratio` or set the `_bytes` variants to keep the dirty set small enough that the disk can flush it in a few seconds. Otherwise you get sporadic multi-second write stalls when the kernel hits `dirty_ratio` and forces synchronous writeback."

### 9C. Swap and swappiness (45 min)

Swap is disk space used as overflow when RAM is exhausted. Slow (orders of magnitude slower than RAM), but having some swap improves resilience versus none.

**`vm.swappiness`** (0–100, default 60) — the controversial knob. **Not** "how much swap to use." It's the kernel's relative preference for **swapping anonymous pages vs evicting file-backed page cache** when under pressure.

- `swappiness=0` — strongly prefer to evict file cache; only swap anon as last resort
- `swappiness=100` — equally happy to swap anon or evict file cache
- `swappiness=10` — typical for database servers (keep DB pages in RAM; let kernel re-read files)
- `swappiness=60` (default) — reasonable for general workloads

**Common mistake**: setting `swappiness=0` "to disable swap." That's not what it does — at 0, the kernel still swaps if absolutely necessary. To disable swap, use `swapoff -a` and remove swap from `/etc/fstab`.

**`vm.vfs_cache_pressure`** (default 100) — how aggressively the kernel reclaims dentry/inode caches relative to page cache. Higher = reclaim them more aggressively (useful when slab grows huge from lots of file metadata).

**Live swap inspection**:

```bash
swapon --show
free -h               # Swap row
cat /proc/swaps
vmstat 1              # si/so columns
```

**Per-process swap usage**:

```bash
for pid in /proc/[0-9]*; do
  swap=$(awk '/VmSwap/ {print $2}' $pid/status 2>/dev/null)
  [[ -n "$swap" && "$swap" -gt 0 ]] && \
    echo "${swap} kB  $(cat $pid/comm 2>/dev/null) (pid=${pid##*/})"
done | sort -rn | head
```

### 9D. Overcommit and the OOM killer (45 min)

Linux allows processes to allocate more virtual memory than physically available — **overcommit**. Sensible because most allocated memory isn't immediately touched. `fork()` in particular relies on this (it COW-clones the parent's address space).

**`vm.overcommit_memory`**:

- `0` (default) — **heuristic** overcommit. Kernel allows reasonable overcommit, rejects obvious nonsense. Most general systems.
- `1` — **always overcommit**. Never refuse `malloc()`. OOM killer becomes more important. HPC/ML with huge sparse allocations.
- `2` — **strict** accounting. Total committed memory cannot exceed `swap + overcommit_ratio% of RAM`. `malloc()` returns NULL when limit would be exceeded.

**`vm.overcommit_ratio`** (default 50) — only meaningful when `overcommit_memory=2`. The % of RAM included in the strict limit.

**Strict overcommit math**: `CommitLimit = swap + (RAM × overcommit_ratio / 100)`. Visible in `/proc/meminfo` as `CommitLimit`; `Committed_AS` is current usage.

**The OOM killer** — when physical memory is exhausted and reclaim can't free anything, the kernel kills processes. Selection scores each process based on:

- RSS (bigger = more likely)
- Time running (newer = more likely)
- `oom_score_adj` (operator override: `-1000` "never kill" to `+1000`)
- Other factors (root processes get a small discount)

```bash
cat /proc/[pid]/oom_score          # 0–1000, higher = more likely killed
cat /proc/[pid]/oom_score_adj      # operator adjustment (-1000 to +1000)
```

**Protecting critical processes**:

```bash
echo -1000 | sudo tee /proc/$PID/oom_score_adj
# Via systemd unit: OOMScoreAdjust=-1000
```

**OOM in logs**:

```bash
dmesg -T | grep -i 'killed process\|oom-kill\|out of memory'
journalctl -k | grep -i oom
```

The kernel prints which process it killed and why, plus the full meminfo snapshot and process list at the time. This is the most valuable forensic data for memory incidents.

**Interview answer for "we got OOM killed"**: "Check `dmesg` for the oom-kill log — kernel tells you which process and why. Investigate the workload: is RSS legitimate (grew naturally), is it a leak (RSS climbing over time), or is the system right-sized? If right-sized but bursty, more RAM or memory limits via cgroups. If a leak, fix the app. If essential and well-behaved, protect with `oom_score_adj`."

### 9E. Hugepages and THP (45 min)

Default page size on x86_64 is 4 KB. For large-memory workloads this is small — a 1 TB process needs 256 million page-table entries. The TLB (translation cache) can't hold that many; TLB misses force page-table walks, which are slow.

**Hugepages** = larger page sizes (2 MB and 1 GB on x86_64). Fewer entries, more TLB hits, faster memory access. Major win for DBs, JVMs, and any workload with large RSS.

**1. Explicit / static hugepages**:

```bash
# Current state
grep -i huge /proc/meminfo
sysctl vm.nr_hugepages

# Allocate 1024 × 2 MB = 2 GB at runtime
sudo sysctl -w vm.nr_hugepages=1024
grep -i hugepages_total /proc/meminfo

# Persistent
echo 'vm.nr_hugepages = 1024' | sudo tee /etc/sysctl.d/hugepages.conf
```

To **use** them, applications must opt in via `mmap(MAP_HUGETLB)` or by mounting `hugetlbfs`. PostgreSQL, Oracle, JVM (`-XX:+UseLargePages`) all have config flags.

**2. Transparent Hugepages (THP)** — kernel automatically promotes 4K pages to 2M when possible, transparently to applications.

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# always [madvise] never  ← current selection in brackets

echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```

Three modes:

- `always` — kernel promotes everywhere aggressively
- `madvise` — only when application calls `madvise(MADV_HUGEPAGE)`
- `never` — disabled

**THP gotcha**: defragmentation cost. To build one 2M hugepage requires 512 contiguous 4K pages. Under fragmentation, the kernel does compaction inline, causing **latency spikes**. **MongoDB, Redis, and many DBs recommend disabling THP** for this reason. The `[madvise]` mode is a reasonable compromise.

Defrag policy:

```bash
cat /sys/kernel/mm/transparent_hugepage/defrag
# always defer defer+madvise madvise [never]
```

**Interview frame**: "Explicit hugepages = guaranteed reserved, ideal for DBs that opt in. THP = automatic but defrag can stall. Common production stance: disable THP for DBs (or set madvise), leave default for most other workloads, use explicit hugepages for the DB."

### 9F. NUMA (30 min)

NUMA = Non-Uniform Memory Access. Modern multi-socket systems have memory attached to each CPU socket; access to your local socket's memory is faster than access to a remote socket's.

```bash
numactl --hardware
# shows "available: N nodes" and which CPUs/memory belong to each
lscpu | grep -i numa
```

A 2-socket EPYC or Xeon typically has 2 NUMA nodes. Remote-node access is ~1.5–2× slower for memory-heavy workloads.

**The problem**: by default the scheduler may run a process on any CPU, and memory may have been allocated on a different node than where the process is currently running. Performance degrades silently.

**Tools**:

```bash
numactl --hardware                         # node topology
numactl --show                              # default policy for current shell
numastat                                    # per-node stats (hits, misses, foreign)
numastat -p <pid>                           # per-process node allocation
numactl --cpunodebind=0 --membind=0 ./myapp # pin to node 0
```

`numastat` counters:

- `numa_hit` — allocation served by intended node
- `numa_miss` — forced to other node (intended was full)
- `numa_foreign` — meant for another node, served by this one
- `other_node` — allocations on this node intended elsewhere

**`numad`** — daemon that automatically migrates processes and memory between nodes for locality. Useful when manual pinning isn't practical.

**On AWS / VMs**: smaller instances are single-NUMA. Big instances (i3.metal, r5.24xlarge, …) have multiple NUMA nodes. Worth checking on memory-intensive workloads.

**Interview frame**: "On multi-socket systems I'd check `numactl --hardware` first. If the workload is memory-heavy and unpinned, `numastat` shows high `numa_foreign` counts — pages being accessed from a different node than allocated. Pin with `numactl --cpunodebind --membind` or use `numad`."

### 9G. SYSV shared memory limits (15 min)

System V shared memory is a legacy IPC mechanism still used by some databases (Oracle, older PostgreSQL). Kernel limits:

- `kernel.shmmax` — max size of a single shared memory segment, bytes
- `kernel.shmall` — total pages of shared memory system-wide
- `kernel.shmmni` — max number of segments

Modern PostgreSQL uses mmap by default and rarely needs these tuned. Mentioned in JD so know they exist.

```bash
sysctl kernel.shmmax kernel.shmall kernel.shmmni
ipcs -m                # current SysV shared memory segments
ipcs -l                # limits
```

---

## Midday Block (2.5h) — Hands-on labs

### Lab 1: Inspect your memory state (20 min)

```bash
cat /proc/meminfo
free -h

grep -E '^(MemTotal|MemAvailable|MemFree|Buffers|Cached|Active|Inactive|AnonPages|Mapped|Slab|SReclaimable|SUnreclaim|SwapTotal|SwapFree|Dirty|Writeback|Committed_AS|CommitLimit|HugePages|AnonHugePages)' /proc/meminfo

# Slab top
sudo slabtop -o | head -20

# Per-process swap usage
for pid in /proc/[0-9]*; do
  swap=$(awk '/VmSwap/ {print $2}' $pid/status 2>/dev/null)
  if [[ -n "$swap" && "$swap" -gt 0 ]]; then
    echo "${swap} kB  $(cat $pid/comm 2>/dev/null) (pid=${pid##*/})"
  fi
done | sort -rn | head

sysctl -a 2>/dev/null | grep -E '^vm\.' | head -40
```

Predict: what's your default swappiness? Is THP enabled?

### Lab 2: Swappiness experiment (30 min)

```bash
sysctl vm.swappiness                    # likely 60

sudo sysctl -w vm.swappiness=10

# Force pressure (~110% of available)
AVAIL=$(awk '/MemAvailable/ {print int($2/1024)}' /proc/meminfo)
PRESS=$((AVAIL * 110 / 100))
stress-ng --vm 1 --vm-bytes ${PRESS}M --timeout 30s &
vmstat 1                                # watch si/so

# Repeat with default
sudo sysctl -w vm.swappiness=60
stress-ng --vm 1 --vm-bytes ${PRESS}M --timeout 30s &
vmstat 1
```

Observation: lower swappiness evicts more page cache before swapping; cache/anon balance shifts.

### Lab 3: OOM killer in action (30 min)

**Warning**: may kill processes. Snapshot the VM first.

```bash
sudo dmesg -wT &                        # watch in background

AVAIL=$(awk '/MemAvailable/ {print int($2/1024)}' /proc/meminfo)
OVER=$((AVAIL * 200 / 100))
stress-ng --vm 1 --vm-bytes ${OVER}M --vm-keep --timeout 60s

sudo dmesg -T | grep -i 'oom\|killed' | tail -30

# OOM scores of current processes
for pid in /proc/[0-9]*; do
  s=$(cat $pid/oom_score 2>/dev/null)
  a=$(cat $pid/oom_score_adj 2>/dev/null)
  c=$(cat $pid/comm 2>/dev/null)
  echo "$s $a ${pid##*/} $c"
done | sort -rn | head -10

# Protect a process
sleep 600 &
PID=$!
echo -1000 | sudo tee /proc/$PID/oom_score_adj
cat /proc/$PID/oom_score                # now very low
kill $PID
```

Read the OOM log carefully — it includes full meminfo, slabinfo, and the process list with RSS at the kill moment. This snapshot is the most valuable forensic artifact you'll get.

### Lab 4: Hugepages allocation (30 min)

```bash
grep -i huge /proc/meminfo
sysctl vm.nr_hugepages

# Reserve 256 × 2 MB = 512 MB
sudo sysctl -w vm.nr_hugepages=256
grep -i huge /proc/meminfo

# Use hugepages from a small C program
cat > /tmp/huge.c <<'EOF'
#include <sys/mman.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
int main(void) {
    size_t sz = 100UL * 1024 * 1024;     // 100 MB
    void *p = mmap(NULL, sz, PROT_READ|PROT_WRITE,
                   MAP_PRIVATE|MAP_ANONYMOUS|MAP_HUGETLB, -1, 0);
    if (p == MAP_FAILED) { perror("mmap"); return 1; }
    memset(p, 1, sz);
    printf("Allocated 100MB hugepages at %p. Sleeping 30s.\n", p);
    sleep(30);
    return 0;
}
EOF
sudo dnf install -y gcc
gcc -o /tmp/huge /tmp/huge.c

# Two terminals:
# Terminal 1:
/tmp/huge
# Terminal 2 while it's running:
grep -i huge /proc/meminfo               # HugePages_Free dropped by 50

# Cleanup
sudo sysctl -w vm.nr_hugepages=0

# THP
cat /sys/kernel/mm/transparent_hugepage/enabled
grep AnonHugePages /proc/meminfo         # how much THP your workload uses
```

### Lab 5: NUMA exploration (30 min)

```bash
sudo dnf install -y numactl
numactl --hardware                       # likely single-node on a VM
numactl --show                           # default policy
numastat                                 # node hits/misses/foreign

# Tooling works even on single-NUMA VMs:
numactl --cpunodebind=0 --membind=0 stress-ng --vm 1 --vm-bytes 256M --timeout 10s &
sleep 1
numastat -p $(pgrep stress-ng | head -1) 2>/dev/null
wait
```

On a real multi-socket box, repeat with `--cpunodebind=1` and compare `numastat -p`. On a single-node VM the lab teaches the tooling.

### Lab 6: Dirty page writeback (20 min)

```bash
sysctl vm.dirty_ratio vm.dirty_background_ratio vm.dirty_expire_centisecs vm.dirty_writeback_centisecs

# Generate write pressure, watch Dirty climb
dd if=/dev/zero of=/tmp/bigfile bs=1M count=2048 &
while sleep 1; do grep -E 'Dirty:|Writeback:' /proc/meminfo; echo ---; done
# Ctrl-C when dd finishes
rm /tmp/bigfile

# Lower the ratio
sudo sysctl -w vm.dirty_ratio=10 vm.dirty_background_ratio=5
dd if=/dev/zero of=/tmp/bigfile bs=1M count=2048 &
while sleep 1; do grep -E 'Dirty:' /proc/meminfo; done
# Ctrl-C, observe Dirty stays smaller

# Reset
sudo sysctl -w vm.dirty_ratio=20 vm.dirty_background_ratio=10
rm /tmp/bigfile
```

Use case for lowering: any service where a sudden flush would cause user-visible latency. DBs, write-heavy services, latency-SLO services.

---

## Afternoon Block (1.5h) — Drills + Story #9

### Self-check (45 min)

1. What does `vm.swappiness` actually control?
2. System has 4 GB swap, 32 GB RAM, `swappiness=10`. Will it ever swap?
3. Difference between `MemFree` and `MemAvailable`?
4. We're getting OOM kills. Walk me through your investigation.
5. How do you protect a process from being OOM-killed?
6. Explicit hugepages vs Transparent Hugepages?
7. Why do MongoDB/Redis docs recommend disabling THP?
8. The host writes for 30s, stalls for 10s, resumes. What's happening at the kernel level and what do you tune?
9. `overcommit_memory=0` vs `=1` vs `=2` — when would you use each?
10. NUMA — how do you tell if your workload is suffering from cross-node access?
11. How do you find per-process swap usage?
12. `Slab` is growing without bound in /proc/meminfo. What's happening and how do you investigate?

<details>
<summary>Answers</summary>

1. Relative preference for swapping anonymous pages vs evicting file-backed page cache under memory pressure. **NOT** "how much to use swap."
2. Yes, potentially. `swappiness=10` means "strongly prefer not to," not "never." Under severe pressure with nothing cache-y left to evict, kernel will swap.
3. `MemFree` = completely untouched RAM. `MemAvailable` = `MemFree` + most of buffers/cache (reclaimable). MemAvailable is the realistic "how much could a new allocation get."
4. `dmesg -T | grep -i oom-kill` — kernel logs the killed process and full meminfo snapshot. Investigate: was RSS legitimate or a leak (track over time)? System right-sized? If essential, raise instance size or use cgroup memory limits. If a leak, fix the app. Protect critical processes with `oom_score_adj=-1000`.
5. `echo -1000 | sudo tee /proc/$PID/oom_score_adj`. Or via systemd: `OOMScoreAdjust=-1000`. Note this shifts the kill to another process.
6. Explicit: reserved by sysadmin via `vm.nr_hugepages`, apps must opt in (mmap MAP_HUGETLB, hugetlbfs, DB config). Predictable. THP: kernel automatically promotes 4K→2M opportunistically. Easier but defrag can cause latency spikes.
7. Defragmentation cost. Building a 2M hugepage requires 512 contiguous 4K pages; under fragmentation kernel does compaction inline, causing pauses. Strict-latency DBs prefer predictable 4K, explicit hugepages reserved at startup, or `madvise` mode.
8. Dirty pages accumulated until `vm.dirty_ratio` was hit; kernel forced synchronous writeback, blocking the writer. Lower `dirty_ratio` and `dirty_background_ratio`, or set `dirty_bytes` and `dirty_background_bytes` to absolute values appropriate for your disk's flush rate.
9. `0` (heuristic, default) — general purpose. `1` (always allow) — HPC/ML with huge sparse allocations, never want malloc to fail. `2` (strict) — paranoid environments wanting hard guarantees against overcommit, accept allocation failures over OOM kills.
10. `numastat` — look at `numa_miss`, `numa_foreign`, `other_node` counters. Per-process: `numastat -p <pid>`. High values for an unpinned process = workload running cross-node. Pin with `numactl --cpunodebind --membind`.
11. Walk `/proc/[pid]/status` for `VmSwap`. Lab 1 has the one-liner. Or `smem -t -k -s swap`.
12. Likely dentry/inode cache growth. `slabtop` to see which slab is biggest (often `dentry`). Mitigate: raise `vm.vfs_cache_pressure` (counterintuitively higher = reclaim these caches *more aggressively*), or `echo 2 > /proc/sys/vm/drop_caches` for emergency relief (don't do this blindly in production).

</details>

### Behavioral (45 min) — Story #9: Learn and Be Curious

Today's LP. Story prompt:

> "Tell me about a time you intentionally went outside your area of expertise to learn something new. How did it pay off?"

This LP is undervalued by candidates — they think it's about being smart. It's not. It's about **deliberate** learning and **applied** curiosity.

Strong stories show:

- Specific technical/domain area you didn't know
- Why you chose to learn it (problem to solve, gap you noticed)
- How you went about it (resources, mentors, side projects)
- What you did with it after (applied to work, taught others)
- Measurable impact

Weak: "I love learning new things and read a lot of blogs." That's a personality trait, not a story.

STAR. 2-3 minutes. Out loud. Time it.
