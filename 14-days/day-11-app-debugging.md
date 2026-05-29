# 🕵️‍♂️ Day 11 — Application Debugging: strace, gdb, valgrind, perf, eBPF

> [!NOTE]
> **The Goal:** The Day 8–10 lever was tuning. Today the lever is **observation** — getting an application to tell you what it's actually doing. 
> 
> Reported interview territory: "Find a memory leak", "the app is hung, what do you check?", "use standard tools to debug an application", "diagnose system and application behavior using eBPF (syscount, gethostlatency)". The JD specifically calls out eBPF — expect it.

---

## 🧠 Morning Block (3h) — The toolkit

### 11A. Choosing the right tool (15 min)

The decision tree:
- **Stuck / hung process** → `strace -p` first. The blocked syscall tells you what it's waiting on (read, futex, poll, etc.).
- **Wrong output / file not found / permission denied** → `strace -e` filtered to the relevant syscall family.
- **Slow but not stuck, CPU-bound** → `perf top` or `perf record`.
- **Slow but not stuck, low CPU** → bottleneck is off-CPU (I/O, locks, network). `bpftrace` / `bcc` off-CPU analysis, or `strace -c` for syscall time.
- **Crashed** → core dump + `gdb`. `coredumpctl` on RHEL/systemd.
- **Memory growth / leak** → first `pmap -x` and RSS over time. If leak suspected, `valgrind --leak-check=full` in dev. In production, eBPF heap profilers.
- **Production system, can't perturb** → eBPF (`bcc` / `bpftrace`). Low overhead, no recompile, no restart.

> [!TIP]
> **The Interview Move:** "I'd start with `strace -p` to see the live syscall pattern. If it's blocked, the syscall tells me what it's waiting on. If it's spinning, I'd switch to `perf top` to see where in userspace it's burning CPU. For production-grade observation without disturbing the process, eBPF tools like `funccount` or `offcputime` cover the same ground at much lower cost."

### 11B. strace — system calls (45 min)

`strace` shows every system call a process makes, with arguments and return values. It's the "talk to me, what are you doing" tool.

```bash
sudo strace -p 12345                      # Attach to a running PID
sudo strace -fp 12345                     # Follow forks (essential for daemons)
strace -e trace=openat,read,write ls      # Filter to interesting syscalls
strace -e trace='!futex' ./myapp          # exclude noise
strace -tt ./myapp                        # microsecond timestamps
strace -c ./myapp                         # summary: count and time per syscall
```

**Reading strace output**:
```text
openat(AT_FDCWD, "/etc/passwd", O_RDONLY|O_CLOEXEC) = 3
read(3, "root:x:0:0:root:/root:/bin/bash\n", 4096) = 32
```
- Each line: `syscall(args) = return_value`
- Negative returns are errors; `strace` decodes the errno: `= -1 ENOENT (No such file or directory)`

**The diagnostic patterns you must recognize**:

| What you see | Diagnosis |
| :--- | :--- |
| `read(N, ...` hanging | Blocked waiting for input — pipe, socket, terminal |
| `futex(..., FUTEX_WAIT, ...` hanging | Waiting on a userspace lock — possibly deadlock |
| `poll(...` / `epoll_wait(...` hanging | Event loop idle (probably fine) |
| `connect(...` hanging | TCP handshake stuck — check the network |
| `openat(...) = -1 ENOENT` | File not found — config file? library? |
| `openat(...) = -1 EACCES` | Permission denied — SELinux? perms? |

> [!CAUTION]
> **Performance Warning:** `strace` is **heavy**. Every syscall traps to ptrace, causing a ~10–100x slowdown. Do not use it on a production hot-path (like a core database). For production, eBPF (`syscount`) is the safer equivalent.

### 11C. gdb, ltrace, and core dumps (40 min)

**`ltrace`** — same idea as strace but for library calls. Useful for "why is this calling libfoo a million times."

**`gdb`** — interactive debugger. 
```bash
sudo gdb -p 12345
```
Inside gdb:
```text
(gdb) bt                           # backtrace of current thread
(gdb) info threads                 # all threads
(gdb) thread apply all bt          # backtrace ALL threads (deadlock investigation)
```

> [!TIP]
> **The Killer Move for Deadlocks:** `thread apply all bt` on the hung process. If three threads are all in `pthread_mutex_lock`, you've found your deadlock and can identify which threads are blocked on which lock.

**Core dumps** — when a process crashes, the kernel writes a core dump file containing the full process memory at the moment of death. 
```bash
coredumpctl list                       # all recent cores
coredumpctl gdb <PID|name>             # open the latest in gdb
```

### 11D. valgrind — memory bugs (25 min)

`valgrind` is **not** for production. It's a dev/test tool with 10–50x slowdown. But for "we have a leak" it's the gold standard.

```bash
# Find leaks
valgrind --leak-check=full --show-leak-kinds=all ./myapp

# Track who allocated leaked memory
valgrind --leak-check=full --track-origins=yes ./myapp

# Race conditions
valgrind --tool=helgrind ./myapp
```

**Production memory leak workflow** (when valgrind isn't an option):
1. Watch RSS over time with `ps` or `cat /proc/$PID/status | grep VmRSS`.
2. Confirm linear growth (not just startup allocation).
3. `pmap -x <pid>` to see what's growing (heap, anon, file mappings).
4. In dev with the same code: valgrind to identify the specific leak.
5. Or use eBPF heap profilers (`memleak.py` from bcc tools) — production-safe.

### 11E. perf — profiling and hardware counters (35 min)

`perf` is the Linux profiler. Three modes you'll use:

**Live sampling** (like top, but for functions):
```bash
sudo perf top -p 12345                  # specific process
```

**Record + report** (the profiling workflow):
```bash
sudo perf record -g -p 12345 -- sleep 30      # record 30 seconds of PID with stacks
sudo perf report --no-children                # interactive viewer
```

**Hardware counters** (CPU performance characteristics):
```bash
sudo perf stat ./myapp
```
Look for `IPC` (instructions per cycle). `> 1` is good, `< 0.5` suggests stalls. Low IPC + high branch-misses → unpredictable branches. Low IPC + high cache-misses → memory-bound workload.

> [!IMPORTANT]
> **☁️ The AWS Bridge: Hardware PMUs**
> `perf stat` relies on Hardware Performance Monitoring Units (PMUs) inside the CPU to count cache misses and instructions per cycle. 
> 
> **Historically, AWS disabled PMUs on virtualized EC2 instances** for security. If you run `perf stat` on a standard `t3` or `m5` instance, many hardware counters will show `<not supported>`. To get true hardware profiling in AWS, you must use **Bare Metal instances** (`.metal`) or dedicated hosts where PMU access is exposed.

**Flame graphs** — Brendan Gregg's visualization for `perf record` output. Wider bars = more CPU time. The visual lets you spot "this one function is 60% of execution" at a glance.

### 11F. eBPF and bcc tools (40 min)

eBPF is the modern replacement for SystemTap. You can run sandboxed programs inside the kernel that hook tracepoints **with low overhead and no recompile**. 

**Install on RHEL/Fedora**:
```bash
sudo dnf install -y bcc-tools bpftrace
ls /usr/share/bcc/tools/                 # ~100 ready tools
```

**The bcc tools you should know by name**:

| Tool | What it does |
| :--- | :--- |
| `execsnoop` | Every `exec()` system-wide. Great for "what's spawning processes?" |
| `opensnoop` | Every `open()` system-wide. Lighter than strace. |
| `syscount` | Count syscalls system-wide or per-process. |
| `gethostlatency` | DNS resolution latency |
| `biolatency` | Block I/O latency histogram |
| `offcputime` | Where is on-CPU time vs off-CPU time going? |
| `memleak` | Heap-allocation tracking — production-safe leak detection |

**`bpftrace`** — the ad-hoc tracing language:
```bash
# Count syscalls system-wide
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'
```

---

## 💻 Midday Block (2.5h) — Hands-on labs

### Lab 1: strace in anger (30 min)

```bash
# A simple program to attach to
sleep 600 &
PID=$!

# Watch it - it's blocked in nanosleep
sudo strace -p $PID

# Filter
sudo strace -e trace=clock_nanosleep,nanosleep -p $PID

# A "find the file the app is looking for" exercise
strace -e openat -e fault=openat:error=ENOENT cat /etc/hostname 2>&1 | head
# Real value: when an app fails, strace -e openat tells you which file is missing

kill $PID 2>/dev/null
```

### Lab 2: gdb attach and inspect (30 min)

```bash
# Create a tiny program with debug symbols
sudo dnf install -y gcc gdb
cat > /tmp/loop.c <<'EOF'
#include <stdio.h>
#include <unistd.h>
void inner(int *x) {
    while (1) {
        (*x)++;
        sleep(1);
    }
}
int main(void) {
    int counter = 0;
    inner(&counter);
    return 0;
}
EOF
gcc -g -O0 -o /tmp/loop /tmp/loop.c

# Run it
/tmp/loop &
PID=$!

# Attach
sudo gdb -p $PID
```
Inside gdb: `bt`, `info threads`, `print counter`, `continue`, `detach`.

### Lab 3: Core dumps - generate and inspect (20 min)

```bash
# A program that crashes
cat > /tmp/crash.c <<'EOF'
#include <stdio.h>
int main(void) {
    int *p = NULL;
    *p = 42;            // segfault
    return 0;
}
EOF
gcc -g -O0 -o /tmp/crash /tmp/crash.c

ulimit -c unlimited
/tmp/crash
# Segmentation fault (core dumped)

coredumpctl info crash
sudo coredumpctl gdb crash
```
Inside gdb: `bt` to see where it crashed. 

### Lab 4: valgrind on a leaky program (30 min)

```bash
sudo dnf install -y valgrind

cat > /tmp/leak.c <<'EOF'
#include <stdlib.h>
#include <string.h>
char *leak_one(void) {
    char *p = malloc(40);
    strcpy(p, "this memory will never be freed");
    return p;
}
int main(void) {
    for (int i = 0; i < 3; i++) {
        leak_one();        // returned pointer dropped — leaked
    }
    return 0;
}
EOF
gcc -g -O0 -o /tmp/leak /tmp/leak.c

valgrind --leak-check=full --show-leak-kinds=all --track-origins=yes /tmp/leak
```
Read the output: Look for "definitely lost" and the allocation stack.

### Lab 5: eBPF / bcc tools tour (30 min)

```bash
sudo dnf install -y bcc-tools bpftrace

# execsnoop - watch every process exec
sudo /usr/share/bcc/tools/execsnoop &
EXEC_PID=$!
sleep 10
date >/dev/null
sudo kill $EXEC_PID

# syscount (5 seconds; per-process)
sudo /usr/share/bcc/tools/syscount -P 5

# gethostlatency - DNS timing
sudo /usr/share/bcc/tools/gethostlatency &
GH_PID=$!
host google.com
sleep 3
sudo kill $GH_PID

# Ad-hoc bpftrace: syscalls per process
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }' &
BT_PID=$!
sleep 5
sudo kill -SIGINT $BT_PID
# Will print the @ map of comm -> count when interrupted
```

---

## 🎯 Afternoon Block (1.5h) — Drills + Story #11

### Self-check (45 min)

1. App is hung at 0% CPU. What's your first move and what are you looking for?
2. What does `futex_wait` in strace mean and how is it different from a regular blocked `read`?
3. Why is `strace` unsafe in production hot paths, and what's the eBPF equivalent for syscall counting?
4. A user reports "the program can't find its config file." What strace flag reproduces the issue fast?
5. App crashes nightly. How do you make sure you have a core dump on the next crash?
6. You attach gdb to a hung multi-threaded process. What single command gives you the most useful information?
7. Difference between `valgrind` and eBPF `memleak.py` for finding leaks?
8. `perf stat` shows IPC of 0.3. What does that suggest, and what would you check next?
9. What does `offcputime` tell you that `perf top` doesn't?
10. The JD specifically mentions `syscount` and `gethostlatency`. Walk me through what each does and when you'd use it.
11. You want to know how often a specific kernel function is called over 30 seconds. What's the shortest command that gives you that?

<details>
<summary><strong>Answers</strong> (click to reveal)</summary>

1. `sudo strace -p <pid>`. If it's blocked in a syscall, the syscall tells you what it's waiting on (read on a socket = waiting on peer, futex_wait = waiting on a userspace lock, etc.). If strace shows nothing, it's spinning in userspace — switch to `perf top -p <pid>` or `gdb -p <pid>` with `bt`.
2. `futex_wait` is the kernel side of a userspace lock (`pthread_mutex`, condition vars, etc.). The thread is waiting for another thread to wake it up. A blocked `read` is waiting on an I/O source. 
3. strace traps every syscall to ptrace, ~10-100x slowdown. Production equivalent: `bcc/syscount` or `bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'`. Both run in-kernel with minimal overhead.
4. `strace -e openat <cmd>` (or attach to running PID). Watch for openat returning -1 ENOENT for the missing file.
5. (a) For systemd services: `LimitCORE=infinity` in the unit. (b) Verify systemd-coredump is active: `systemctl status systemd-coredump.socket`. 
6. `thread apply all bt`. Backtrace of every thread. If three threads are all in `pthread_mutex_lock` you've located the deadlock.
7. valgrind: dev/test only (10-50x slowdown), tracks every allocation, very precise. memleak.py (eBPF): production-safe, samples allocations. Use valgrind in CI; use memleak.py on a leaky production process.
8. Low IPC (instructions per cycle) means the CPU is stalled. Check `perf stat` for cache-misses (memory-bound) or branch-misses (unpredictable branches).
9. `perf top` samples *on-CPU* time — only counts time the process is actively running. `offcputime` (bcc) samples *off-CPU* time — time blocked, sleeping, waiting on locks/I/O/scheduler. 
10. `syscount`: counts syscalls system-wide or per-process. Use to see "which syscall is dominating". `gethostlatency`: measures DNS resolution time per request. Use when a service is slow and you suspect DNS.
11. `sudo bpftrace -e 'kprobe:<func_name> { @ = count(); }'` and Ctrl-C after 30 seconds.

</details>

### 🤝 Behavioral (45 min) — Story #11: Think Big

Today's LP: **Think Big**. 

Prompt:
> "Tell me about a time you proposed something more ambitious than what was being asked for. How did you sell it, and how did it play out?"

> [!TIP]
> Think Big is one of the trickier LPs because the failure mode is **looking grandiose**. The best stories are *grounded ambition*:
> - A real, specific bigger goal that you saw and others didn't.
> - A concrete proposal — not just "we should do better" but "here's what 10× looks like, and here's how we get there."
> - A demonstration of how you got others on board: communication, prototype, data, building coalition.
> - Outcome — even if you didn't fully achieve the big version, what got done that wouldn't have without your push?

**Pitfalls:**
- Vagueness — "I had a vision" without specifics.
- Solo flag-planting — "I proposed the moon and they didn't listen" reads as poor judgment, not bold thinking.
- Borrowed scope — describing the company's strategy, not your contribution.

STAR. 2-3 minutes. Out loud. Time it.
