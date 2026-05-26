# Day 11 — Application Debugging: strace, gdb, valgrind, perf, eBPF (Fri May 8)

The Day 8–10 lever was tuning. Today the lever is **observation** — getting an application to tell you what it's actually doing. Five tools, each at a different layer:

| Tool | Layer | When to reach for it |
|---|---|---|
| `strace` | System calls | "What is this process doing right now?" / "Why is it stuck?" |
| `ltrace` | Library calls | Less common; library-level missteps |
| `gdb` | Process state | Stack traces, attaching to a running process, core dumps |
| `valgrind` | Memory access | Leaks, invalid reads/writes, race conditions |
| `perf` | CPU sampling, HW counters | "Where is CPU time going?" / profiling |
| `bpftrace` / `bcc` | In-kernel tracepoints | Production-safe observation at any layer |

Reported interview territory: "find a memory leak", "the app is hung, what do you check?", "use standard tools to debug an application", "diagnose system and application behavior using eBPF (syscount, gethostlatency)". The JD specifically calls out eBPF — expect it.

---

## Morning Block (3h) — The toolkit

### 11A. Choosing the right tool (15 min)

The decision tree:

- **Stuck / hung process** → `strace -p` first. The blocked syscall tells you what it's waiting on (read, futex, poll, etc.).
- **Wrong output / file not found / permission denied** → `strace -e` filtered to the relevant syscall family.
- **Slow but not stuck, CPU-bound** → `perf top` or `perf record`.
- **Slow but not stuck, low CPU** → bottleneck is off-CPU (I/O, locks, network). `bpftrace` / `bcc` off-CPU analysis, or `strace -c` for syscall time.
- **Crashed** → core dump + `gdb`. `coredumpctl` on RHEL/systemd.
- **Memory growth / leak** → first `pmap -x` and RSS over time. If leak suspected, `valgrind --leak-check=full` in dev. In production, eBPF heap profilers.
- **Production system, can't perturb** → eBPF (`bcc` / `bpftrace`). Low overhead, no recompile, no restart.

**The interview move**: "I'd start with `strace -p` to see the live syscall pattern. If it's blocked, the syscall tells me what it's waiting on. If it's spinning, I'd switch to `perf top` to see where in userspace it's burning CPU. For production-grade observation without disturbing the process, eBPF tools like `funccount` or `offcputime` cover the same ground at much lower cost."

### 11B. strace — system calls (45 min)

`strace` shows every system call a process makes, with arguments and return values. It's the "talk to me, what are you doing" tool.

```bash
# Trace a new process
strace ./myapp

# Attach to a running PID
sudo strace -p 12345

# Follow forks (essential for daemons that spawn workers)
sudo strace -fp 12345

# Filter to interesting syscalls
strace -e trace=openat,read,write ls
strace -e trace=network ./myapp           # network family
strace -e trace=file ./myapp              # file family
strace -e trace='!futex' ./myapp          # exclude noise

# Useful flags
strace -tt ./myapp                        # microsecond timestamps
strace -T ./myapp                         # time spent in each syscall
strace -y ./myapp                         # print FD path next to FD number
strace -s 200 ./myapp                     # show longer strings (default 32)
strace -c ./myapp                         # summary: count and time per syscall
strace -o trace.log ./myapp               # write to file
```

**Reading strace output**:

```
openat(AT_FDCWD, "/etc/passwd", O_RDONLY|O_CLOEXEC) = 3
read(3, "root:x:0:0:root:/root:/bin/bash\n", 4096) = 32
close(3) = 0
```

- Each line: `syscall(args) = return_value`
- Negative returns are errors; `strace` decodes the errno: `= -1 ENOENT (No such file or directory)`

**The diagnostic patterns you must recognize**:

| What you see | Diagnosis |
|---|---|
| `read(N, ...` hanging | Blocked waiting for input — pipe, socket, terminal |
| `futex(..., FUTEX_WAIT, ...` hanging | Waiting on a userspace lock — possibly deadlock |
| `poll(...` or `epoll_wait(...` hanging | Event loop idle (probably fine) |
| `connect(...` hanging | TCP handshake stuck — check the network |
| `openat(...) = -1 ENOENT` | File not found — config file? library? |
| `openat(...) = -1 EACCES` | Permission denied — SELinux? perms? |
| Lots of `stat()` returning ENOENT | Library/path search — check `LD_LIBRARY_PATH`, `PATH` |
| `clock_nanosleep(...` | App is literally sleeping — `usleep()` etc. |
| `recvfrom(...) = 0` | Peer closed the socket gracefully |

**Performance warning**: `strace` is **heavy**. Every syscall traps to ptrace, ~10–100x slowdown. Don't use it on a production hot path. For production, eBPF (`syscount`, `funccount`) is the safer equivalent.

**The headline interview answer**: "If a process is hung, I attach with `sudo strace -p <pid>`. The syscall it's blocked in tells me what it's waiting on. `futex_wait` → lock contention, look at threads. `read` on a socket → network issue, look at the peer. `poll`/`epoll_wait` → idle event loop, probably normal."

### 11C. gdb, ltrace, and core dumps (40 min)

**`ltrace`** — same idea as strace but for library calls. Less commonly needed but useful for "why is this calling libfoo a million times":

```bash
ltrace ./myapp
ltrace -p 12345
ltrace -c ./myapp                         # summary
```

**`gdb`** — interactive debugger. The minimum useful set:

```bash
# Attach to running process
sudo gdb -p 12345

# Open binary + core
gdb /usr/bin/myapp /var/lib/systemd/coredump/core.myapp.0.xxxxx
```

Inside gdb:

```
(gdb) bt                           # backtrace of current thread
(gdb) bt full                       # with local variables
(gdb) info threads                  # all threads
(gdb) thread 3                      # switch to thread 3
(gdb) thread apply all bt           # backtrace ALL threads (deadlock investigation)
(gdb) print my_variable
(gdb) list                          # source around current line
(gdb) frame 2                       # move up the stack
(gdb) up / down                     # navigate stack
(gdb) continue                      # resume execution
(gdb) detach                        # detach (process resumes)
(gdb) quit
```

**The killer move for deadlocks**: `thread apply all bt` on the hung process. If three threads are all in `pthread_mutex_lock`, you've found your deadlock and can identify which threads are blocked on which lock.

**Symbols matter** — without debug symbols, backtraces show addresses instead of function names. On RHEL/Fedora:

```bash
sudo dnf install <package>-debuginfo
sudo debuginfo-install <package>      # convenient wrapper
```

**Core dumps** — when a process crashes (segfault, abort, etc.), the kernel can write a core dump file containing the full process memory at the moment of death. **Configure once, debug forever.**

```bash
# Check current setup
ulimit -c                              # 0 = disabled; "unlimited" = enabled
cat /proc/sys/kernel/core_pattern      # where cores go

# Enable for current shell + children
ulimit -c unlimited

# Persistent — RHEL with systemd-coredump (default)
sudo systemctl status systemd-coredump.socket
ls /var/lib/systemd/coredump/

# coredumpctl is the easy way
coredumpctl list                       # all recent cores
coredumpctl info <PID|name>            # details on one
coredumpctl gdb <PID|name>             # open the latest in gdb
coredumpctl dump <name> > /tmp/core    # extract the raw core
```

**Configure crashes to dump in a systemd service**:

```ini
# /etc/systemd/system/myapp.service
[Service]
LimitCORE=infinity
```

Then `systemctl daemon-reload && systemctl restart myapp`.

**Interview answer for "the app crashed at 3 AM"**: "First check `coredumpctl list` for a core. If present, `coredumpctl gdb <name>` opens it. `thread apply all bt` for the full state at crash. Read the signal — SIGSEGV usually means null/dangling pointer; SIGABRT means an explicit `abort()` (often from `assert` or `glibc` detecting heap corruption). If no core, check `journalctl -u <unit>` and `dmesg` — sometimes the kernel logs the segfault with the faulting address."

### 11D. valgrind — memory bugs (25 min)

`valgrind` is **not** for production. It's a dev/test tool with 10–50x slowdown. But for "we have a leak" it's the gold standard.

The tools in valgrind (use `--tool=NAME`):

- **`memcheck`** (default) — detect leaks, invalid reads/writes, use-after-free, uninitialized memory
- **`helgrind`** — race conditions in threaded code
- **`callgrind`** — call-graph profiling (combine with `kcachegrind` for visualization)
- **`massif`** — heap profiler; tracks heap usage over time

```bash
sudo dnf install -y valgrind

# Find leaks
valgrind --leak-check=full --show-leak-kinds=all ./myapp

# Track who allocated leaked memory
valgrind --leak-check=full --track-origins=yes ./myapp

# Race conditions
valgrind --tool=helgrind ./myapp

# Heap profiling
valgrind --tool=massif ./myapp
ms_print massif.out.<pid>             # text view
massif-visualizer massif.out.<pid>     # GUI if available
```

**Reading memcheck output**:

```
==12345== 40 bytes in 1 blocks are definitely lost in loss record 1 of 1
==12345==    at 0x4C2BB6F: malloc (vg_replace_malloc.c:299)
==12345==    by 0x4005A1: leak_function (main.c:8)
==12345==    by 0x4005C2: main (main.c:15)
```

Each leak shows the allocation stack. "Definitely lost" = leaked, no reachable pointer. "Indirectly lost" = leaked because a parent was lost. "Possibly lost" = reachable pointer to interior (suspicious but not always a bug). "Still reachable" = not really leaks, just not freed at exit (usually fine).

**Production memory leak workflow** (when valgrind isn't an option):

1. Watch RSS over time with `ps` or `cat /proc/$PID/status | grep VmRSS`.
2. Confirm linear growth (not just startup allocation).
3. `pmap -x <pid>` to see what's growing (heap, anon, file mappings).
4. In dev with the same code: valgrind to identify the specific leak.
5. Or use eBPF heap profilers (`memleak.py` from bcc tools) — production-safe.

### 11E. perf — profiling and hardware counters (35 min)

`perf` is the Linux profiler. Three modes you'll use:

**Live sampling** (like top, but for what functions are running):

```bash
sudo perf top                           # live, system-wide
sudo perf top -p 12345                  # specific process
sudo perf top -g                        # with call graph
```

Press `?` for help inside `perf top`. You'll see function-level CPU% across the system.

**Record + report** (the profiling workflow):

```bash
sudo perf record -g -p 12345 -- sleep 30      # record 30 seconds of PID with stacks
sudo perf record -g ./myapp                    # record running myapp end-to-end

sudo perf report                               # interactive viewer
sudo perf report --no-children                 # don't aggregate child calls into parents
sudo perf report --stdio                       # printable
```

**Hardware counters** (CPU performance characteristics):

```bash
sudo perf stat ./myapp
# Output includes:
#   task-clock, context-switches, cpu-migrations, page-faults
#   cycles, instructions, branches, branch-misses, cache-misses
#   IPC (instructions per cycle) — > 1 is good, < 0.5 suggests stalls
```

A surprisingly useful diagnostic: low IPC + high branch-misses → branch prediction failing → code has unpredictable branches. Low IPC + high cache-misses → memory-bound workload, not compute-bound. The fix is different in each case.

**Flame graphs** — Brendan Gregg's visualization for `perf record` output. The classic SRE-folklore profiling artifact:

```bash
sudo perf record -g -F 99 -p <pid> -- sleep 30
sudo perf script > out.perf

# One-time clone
git clone --depth=1 https://github.com/brendangregg/FlameGraph
./FlameGraph/stackcollapse-perf.pl out.perf > out.folded
./FlameGraph/flamegraph.pl out.folded > flame.svg
# Open flame.svg in a browser
```

Wider bars = more CPU time. The visual lets you spot "this one function is 60% of execution" at a glance.

**Interview frame**: "For CPU profiling I'd run `perf record -g -F 99 -p <pid> -- sleep 30` for 30 seconds at 99Hz, then either `perf report` for an interactive view or a flame graph for sharing. `perf stat` for hardware-level context — IPC, cache misses, branch misses can tell you if it's compute-bound, memory-bound, or branch-prediction-bound."

### 11F. eBPF and bcc tools (40 min)

eBPF is the modern replacement for SystemTap. The selling point: you can run sandboxed programs inside the kernel that hook tracepoints, kprobes, uprobes, or perf events, **with low overhead and no recompile**. The JD specifically lists eBPF tools.

Three layers of tooling:

1. **bcc tools** — pre-built Python/C tools using libbcc. `dnf install bcc-tools` (binaries in `/usr/share/bcc/tools/` on RHEL).
2. **bpftrace** — high-level scripting language for ad-hoc tracing. Like awk for the kernel.
3. **libbpf** + custom programs — for production tooling.

**Install on RHEL/Fedora**:

```bash
sudo dnf install -y bcc-tools bpftrace
ls /usr/share/bcc/tools/                 # ~100 ready tools
```

**The bcc tools you should know by name** (the JD lists `syscount` and `gethostlatency` specifically):

| Tool | What it does |
|---|---|
| `execsnoop` | Every `exec()` system-wide. Great for "what's spawning processes?" |
| `opensnoop` | Every `open()` system-wide. Lighter than strace. |
| `syscount` | Count syscalls system-wide or per-process. |
| `tcpconnect` / `tcpaccept` / `tcplife` | TCP connections start/end with duration |
| `gethostlatency` | DNS resolution latency |
| `biolatency` | Block I/O latency histogram |
| `biosnoop` | Per-I/O block-device trace |
| `funccount` | Count function calls in any binary |
| `funclatency` | Histogram of function durations |
| `offcputime` | Where is on-CPU time vs off-CPU time going? |
| `memleak` | Heap-allocation tracking — production-safe leak detection |
| `runqlat` | Run queue latency (scheduling delays) |
| `cachestat` | Page cache hit/miss rates |

Example invocations:

```bash
# Live exec tracing — see every new process
sudo /usr/share/bcc/tools/execsnoop

# What files is the system opening?
sudo /usr/share/bcc/tools/opensnoop

# Syscall histogram per second, per-process
sudo /usr/share/bcc/tools/syscount -P -i 1

# DNS resolution latency
sudo /usr/share/bcc/tools/gethostlatency

# Block I/O latency as a histogram
sudo /usr/share/bcc/tools/biolatency 10 1
# Will show a histogram of I/O latencies over 10 seconds

# Find off-CPU time (sleeping, waiting on locks, I/O)
sudo /usr/share/bcc/tools/offcputime -p <pid> 30
```

**`bpftrace`** — the ad-hoc tracing language:

```bash
# Count syscalls system-wide
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'
# Ctrl-C to print

# Histogram of read() durations
sudo bpftrace -e '
  tracepoint:syscalls:sys_enter_read { @s[tid] = nsecs; }
  tracepoint:syscalls:sys_exit_read /@s[tid]/ {
    @ns = hist((nsecs - @s[tid])/1000);
    delete(@s[tid]);
  }'

# Trace what's calling kfree
sudo bpftrace -e 'kprobe:kfree { @[kstack] = count(); }'
```

The one-liners are gold for "show me what the kernel is doing right now without restarting anything."

**Interview-friendly framing**: "eBPF is what I'd reach for in production where strace is too heavy. `execsnoop` tells me what processes are being launched; `opensnoop` what files are being opened; `tcplife` what TCP connections are happening and how long they last. For latency, `biolatency` and `funclatency` give me histograms. For off-CPU analysis, `offcputime`. All low-overhead, no restarts, no recompile."

---

## Midday Block (2.5h) — Hands-on labs

### Lab 1: strace in anger (30 min)

```bash
# A simple program to attach to
sleep 600 &
PID=$!

# Watch it - it's blocked in nanosleep
sudo strace -p $PID
# Ctrl-C to detach

# Filter
sudo strace -e trace=clock_nanosleep,nanosleep -p $PID
# Ctrl-C

# A more interesting target - watch sshd
PID=$(pgrep -f sshd | head -1)
sudo strace -p $PID -e trace='!futex,!nanosleep' -tt 2>&1 | head -20
# Ctrl-C

# Summary mode on a command
strace -c -e trace='!futex' ls /usr/bin
# See which syscalls are most frequent

# A "find the file the app is looking for" exercise
strace -e openat -e fault=openat:error=ENOENT cat /etc/hostname 2>&1 | head
# Real value: when an app fails, strace -e openat tells you which file is missing

# Cleanup
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

Inside gdb:

```
(gdb) bt
(gdb) info threads
(gdb) print counter
(gdb) print *x
(gdb) frame 1                  # move up to caller
(gdb) list
(gdb) info locals
(gdb) continue                  # let it run
(gdb) Ctrl-C                    # interrupt again
(gdb) detach
(gdb) quit
```

```bash
kill $PID
```

### Lab 3: Core dumps - generate and inspect (20 min)

```bash
# Check coredump setup
ulimit -c
coredumpctl --version             # systemd-coredump

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

# Run it
/tmp/crash
# Segmentation fault (core dumped)

# Find and open the core
coredumpctl list | tail -5
coredumpctl info crash
sudo coredumpctl gdb crash
```

Inside gdb:

```
(gdb) bt                       # see where it crashed
(gdb) frame 0                  # at the crash site
(gdb) print p
(gdb) list
(gdb) quit
```

```bash
# Cleanup
rm /tmp/crash /tmp/crash.c
```

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
# Read the output:
# - "definitely lost: 120 bytes in 3 blocks"
# - Each leak stack: malloc → leak_one → main, with line numbers

# A use-after-free scenario
cat > /tmp/uaf.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
int main(void) {
    int *p = malloc(sizeof(int));
    *p = 1;
    free(p);
    printf("%d\n", *p);    // use after free
    return 0;
}
EOF
gcc -g -O0 -o /tmp/uaf /tmp/uaf.c
valgrind /tmp/uaf
# memcheck reports "Invalid read of size 4" with stack

# Cleanup
rm /tmp/leak* /tmp/uaf*
```

### Lab 5: perf for CPU profiling (30 min)

```bash
sudo dnf install -y perf

# A CPU-bound program
cat > /tmp/cpu.c <<'EOF'
#include <stdio.h>
double work(int iters) {
    double sum = 0;
    for (int i = 0; i < iters; i++)
        sum += i * 0.5;
    return sum;
}
int main(void) {
    for (int i = 0; i < 100; i++)
        printf("%f\n", work(1000000));
    return 0;
}
EOF
gcc -g -O2 -o /tmp/cpu /tmp/cpu.c

# Hardware counters
sudo perf stat /tmp/cpu > /dev/null
# Note: IPC, branches, branch-misses, cache-misses

# Sampling
sudo perf record -g -F 99 /tmp/cpu > /dev/null
sudo perf report --stdio | head -40
# work() should dominate

# Live system-wide
sudo perf top -F 99
# Press 'q' to quit

# Build a flame graph (optional - takes more setup)
sudo perf record -g -F 99 -p $(pgrep -f cpu | head -1) -- sleep 5 2>/dev/null
sudo perf script > /tmp/out.perf 2>/dev/null
git clone --depth=1 https://github.com/brendangregg/FlameGraph /tmp/FlameGraph 2>/dev/null
/tmp/FlameGraph/stackcollapse-perf.pl /tmp/out.perf > /tmp/out.folded 2>/dev/null
/tmp/FlameGraph/flamegraph.pl /tmp/out.folded > /tmp/flame.svg 2>/dev/null
ls -la /tmp/flame.svg
# Open /tmp/flame.svg in a browser

# Cleanup
rm /tmp/cpu /tmp/cpu.c
```

### Lab 6: eBPF / bcc tools tour (30 min)

```bash
sudo dnf install -y bcc-tools bpftrace
ls /usr/share/bcc/tools/ | head -30

# execsnoop - watch every process exec
sudo /usr/share/bcc/tools/execsnoop &
EXEC_PID=$!
# Run things in another window for a few seconds, then:
sleep 10
ls /tmp >/dev/null
date >/dev/null
sudo kill $EXEC_PID

# opensnoop - file open tracing
sudo /usr/share/bcc/tools/opensnoop &
OPEN_PID=$!
cat /etc/hostname >/dev/null
sleep 5
sudo kill $OPEN_PID

# syscount
sudo /usr/share/bcc/tools/syscount -P 5
# (5 seconds; per-process)

# gethostlatency - DNS timing
sudo /usr/share/bcc/tools/gethostlatency &
GH_PID=$!
host google.com
host example.com
sleep 3
sudo kill $GH_PID

# biolatency - block I/O latency histogram over 5 seconds
sudo /usr/share/bcc/tools/biolatency 5 1
# (will print a histogram after 5 seconds)

# Ad-hoc bpftrace: syscalls per process
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }' &
BT_PID=$!
sleep 5
sudo kill -SIGINT $BT_PID
# Will print the @ map of comm -> count when interrupted

# Ad-hoc bpftrace: histogram of TCP send sizes
sudo bpftrace -e 'kprobe:tcp_sendmsg { @bytes = hist(arg2); }' &
BT_PID=$!
curl -s https://www.google.com -o /dev/null
sleep 3
sudo kill -SIGINT $BT_PID
```

If `bcc-tools` isn't installed or some tools fail, `bpftrace` one-liners are an excellent fallback — same kernel mechanism.

---

## Afternoon Block (1.5h) — Drills + Story #11

### Self-check (45 min)

1. App is hung at 0% CPU. What's your first move and what are you looking for?
2. What does `futex_wait` in strace mean and how is it different from a regular blocked `read`?
3. Why is `strace` unsafe in production hot paths, and what's the eBPF equivalent for syscall counting?
4. A user reports "the program can't find its config file." What strace flag reproduces the issue fast?
5. App crashes nightly. How do you make sure you have a core dump on the next crash?
6. The crashed process used 8 GB of RAM. The core file is also 8 GB. Why, and how would you reduce that?
7. You attach gdb to a hung multi-threaded process. What single command gives you the most useful information?
8. Difference between `valgrind` and eBPF `memleak.py` for finding leaks?
9. `perf stat` shows IPC of 0.3. What does that suggest, and what would you check next?
10. What does `offcputime` tell you that `perf top` doesn't?
11. The JD specifically mentions `syscount` and `gethostlatency`. Walk me through what each does and when you'd use it.
12. You want to know how often a specific kernel function is called over 30 seconds. What's the shortest command that gives you that?

<details>
<summary>Answers</summary>

1. `sudo strace -p <pid>`. If it's blocked in a syscall, the syscall tells you what it's waiting on (read on a socket = waiting on peer, futex_wait = waiting on a userspace lock, etc.). If strace shows nothing (process not making syscalls), it's spinning in userspace — switch to `perf top -p <pid>` or `gdb -p <pid>` with `bt` to see where.
2. `futex_wait` is the kernel side of a userspace lock (`pthread_mutex`, condition vars, etc.). The thread is waiting for another thread to wake it up. A blocked `read` is waiting on an I/O source. Different problems: futex_wait → likely lock contention or deadlock; read → I/O issue.
3. strace traps every syscall to ptrace, ~10-100x slowdown. Production equivalent: `bcc/syscount` or `bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'`. Both run in-kernel with minimal overhead.
4. `strace -e openat <cmd>` (or attach to running PID). Watch for openat returning -1 ENOENT for the missing file. You'll see exactly which path the app tried.
5. (a) `ulimit -c unlimited` for interactive shells. (b) For systemd services: `LimitCORE=infinity` in the unit. (c) Verify systemd-coredump is active: `systemctl status systemd-coredump.socket`. (d) Test by sending SIGSEGV to a test instance: `kill -SEGV <test_pid>`, confirm `coredumpctl list` shows it.
6. Cores contain process memory by default. Compress with the systemd default (already gzipped). Set `coredump.conf` `ExternalSizeMax` and `JournalSizeMax`. Filter unmapped memory via `/proc/<pid>/coredump_filter`. Or use minidumps with breakpad/crashpad for production crash collection.
7. `thread apply all bt`. Backtrace of every thread. If three threads are all in `pthread_mutex_lock` you've located the deadlock; the lock addresses identify which locks. Bonus: `info threads` first to see thread count and states.
8. valgrind: dev/test only (10-50x slowdown), tracks every allocation, very precise, identifies the exact line that leaked. memleak.py (eBPF): production-safe (low overhead), samples allocations, gives you the stack of the biggest unfreed allocations after a run. Use valgrind in CI; use memleak.py on a leaky production process you can't restart.
9. Low IPC (instructions per cycle) means the CPU is stalled. Check `perf stat` for cache-misses (memory-bound — pointer chasing, large working set), branch-misses (unpredictable branches), or low-frequency CPU (cpufreq governor stuck, thermal throttling). For deep dive, `perf record -e cache-misses` and check the report.
10. `perf top` samples *on-CPU* time — only counts time the process is actively running. `offcputime` (bcc) samples *off-CPU* time — time blocked, sleeping, waiting on locks/I/O/scheduler. For "low CPU but slow," `offcputime` is what you want.
11. `syscount`: counts syscalls system-wide or per-process. Use to see "which syscall is dominating" — usually finds inefficient I/O patterns. `gethostlatency`: measures DNS resolution time per request. Use when a service is slow and you suspect DNS — saves you having to add tracing in the app.
12. `sudo bpftrace -e 'kprobe:<func_name> { @ = count(); }'` and Ctrl-C after 30 seconds. Or `sudo funccount -d 30 <func_name>` from bcc-tools.

</details>

### Behavioral (45 min) — Story #11: Think Big

Today's LP. Story prompt:

> "Tell me about a time you proposed something more ambitious than what was being asked for. How did you sell it, and how did it play out?"

Think Big is one of the trickier LPs because the failure mode is **looking grandiose**. The best stories are *grounded ambition*:

- A real, specific bigger goal that you saw and others didn't.
- A concrete proposal — not just "we should do better" but "here's what 10× looks like, and here's how we get there."
- A demonstration of how you got others on board: communication, prototype, data, building coalition.
- Outcome — even if you didn't fully achieve the big version, what got done that wouldn't have without your push?

What kills these stories:

- Vagueness — "I had a vision" without specifics.
- Solo flag-planting — "I proposed the moon and they didn't listen" reads as poor judgment, not bold thinking.
- Borrowed scope — describing the company's strategy, not your contribution.

Strong shape: *"The team's plan was X. I argued for Y, which was [10x the scope / a different paradigm / a fundamentally simpler approach]. I [made the case using data / built a prototype / found allies in adjacent teams]. We ended up shipping [Y, or a version of Y, or X with some of Y's principles] and the impact was [measurable]."*

STAR. 2-3 minutes. Out loud. Time it.
