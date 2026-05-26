# Day 2 — Processes, `/proc`, and the Tools That Read It (Wed Apr 29)

Yesterday: files are inodes + directory entries. Today: **processes are kernel data structures**, and every tool that shows them (`ps`, `top`, `htop`, `vmstat`, `free`) is just a parser of `/proc`. Once you see this, those tools stop being magic.

The interview question this targets: *"Where do `top` and `ps` get their information?"* — but also unlocks debugging questions like "process is stuck, what now?" and "memory leak — how do you confirm?"

---

## Morning Block (3h) — The process model

### 2A. What is a process? (45 min)

A **process** is a kernel-managed execution context. The kernel keeps a `task_struct` for each one — PID, parent PID, UIDs/GIDs, open files, memory mappings, signal handlers, scheduler state, working directory, namespaces, cgroups.

Key concepts:

- **PID 1** — `init` or `systemd`. Started by the kernel; adopts orphans; reaping it crashes the system.
- **Parent / child** — every process except PID 1 has a parent. `fork()` creates a copy; `exec()` replaces the program. Shell command execution = fork + exec.
- **Process states** (`ps` STAT column):
  - `R` — running or runnable (on a CPU or in run queue)
  - `S` — interruptible sleep (most common; waiting on something)
  - `D` — uninterruptible sleep (waiting on I/O or kernel; **can't be killed**, even with SIGKILL)
  - `T` — stopped (SIGSTOP / Ctrl-Z)
  - `Z` — zombie (exited, parent hasn't called `wait()` to reap it)
- **Process groups & sessions** — for job control and signal delivery to terminal-attached processes.
- **Threads** — share the same memory and FDs but each has its own kernel scheduling entity (PID inside the kernel; what userspace calls TID).

**The `D` state matters in interviews**: "I tried `kill -9` and the process won't die." Answer: it's in `D` state, blocked in the kernel — typically waiting on stuck I/O (NFS hang, dead disk, hung iSCSI). You can't kill it; you fix the underlying I/O or reboot.

**Zombies matter too**: lots of zombies = parent process has a bug (not calling `wait()`). They're harmless individually but exhaust the PID space if unbounded.

### 2B. Signals (45 min)

Signals are how the kernel and processes communicate asynchronously. Memorize these — they come up:

| Signal | Number | What it does | Catchable? |
|---|---|---|---|
| `SIGHUP` | 1 | Hangup / reload config (by convention) | Yes |
| `SIGINT` | 2 | Ctrl-C, polite interrupt | Yes |
| `SIGQUIT` | 3 | Ctrl-\, quit + core dump | Yes |
| `SIGKILL` | 9 | Terminate immediately | **No** |
| `SIGTERM` | 15 | Polite "please exit" — default for `kill` | Yes |
| `SIGSTOP` | 19 | Pause | **No** |
| `SIGCONT` | 18 | Resume after STOP | Yes |
| `SIGUSR1`/`SIGUSR2` | 10/12 | App-defined | Yes |
| `SIGSEGV` | 11 | Segfault — invalid memory access | Yes (rarely caught) |
| `SIGPIPE` | 13 | Wrote to a closed pipe | Yes |

**Rules**:

- `kill -9` is the nuclear option. **Try `SIGTERM` first** so the app cleans up (flushes buffers, closes connections, removes lockfiles). Loops asking for `kill -9` immediately are a smell.
- `SIGKILL` and `SIGSTOP` cannot be caught, blocked, or ignored. The kernel handles them.
- A process in `D` state ignores everything, including SIGKILL — it's not running userspace code to handle it.

**Why `SIGHUP` reloads config**: by historical convention, daemons re-read their config on SIGHUP. It's not magic — the daemon's signal handler does it. `nginx -s reload` sends SIGHUP under the hood.

### 2C. `/proc` — the kernel speaks (1h 30min)

`/proc` is a **virtual filesystem** populated by the kernel. Every read produces a fresh snapshot. There's no disk involvement — these "files" are kernel functions wearing a filesystem disguise.

**Per-process — `/proc/[pid]/`**:

| Path | What it contains |
|---|---|
| `cmdline` | null-separated argv |
| `comm` | short process name (15 chars max) |
| `cwd` | symlink to working directory |
| `environ` | null-separated environment vars |
| `exe` | symlink to the binary |
| `fd/` | directory of open file descriptors (each a symlink) |
| `io` | bytes read/written, syscalls |
| `limits` | ulimits (RLIMIT_*) |
| `maps` | memory mappings (also smaps for detail) |
| `status` | human-readable summary (RSS, threads, state, UIDs) |
| `stat` | machine-readable; what ps parses |
| `sched` | scheduler stats |
| `syscall` | currently-executing syscall (if any) |
| `task/` | one subdir per thread |

**System-wide — `/proc/`**:

| Path | What it contains |
|---|---|
| `cpuinfo` | per-CPU model, flags, frequency |
| `meminfo` | memory totals, free, buffers, cached, swap |
| `loadavg` | 1/5/15 min run-queue length + nr_running/nr_total + last_pid |
| `stat` | CPU time totals (user/nice/system/idle/iowait/irq/softirq/steal) |
| `sys/` | tunables (same as sysctl) |
| `net/` | networking state (tcp, udp, route, dev stats) |
| `mounts` | currently mounted filesystems |
| `modules` | loaded kernel modules |
| `interrupts` | IRQ counts per CPU |
| `diskstats` | block-device I/O stats (what iostat parses) |
| `slabinfo` | kernel slab allocator stats (root only) |

**`top` reads `/proc/stat` and `/proc/[pid]/stat`** at intervals, computes deltas. CPU% is *always* computed over an interval — that's why the first sample of `top` shows weird values.

**`/sys` is the sibling**: device tree, drivers, kernel object hierarchy. Mostly *writable* for tunables (unlike most of `/proc`). Examples:

- `/sys/block/sda/queue/scheduler` — current I/O scheduler
- `/sys/devices/system/cpu/cpu0/cpufreq/scaling_governor` — CPU governor
- `/sys/kernel/mm/transparent_hugepage/enabled` — THP setting

---

## Midday Block (2.5h) — Hands-on labs

### Lab 1: Walk the columns of `ps` and `top` (45 min)

```bash
ps aux | head
```

Each column:

- **USER** — owner (real UID, resolved to name)
- **PID** — process ID
- **%CPU** — CPU% over the process's lifetime (so long-running mostly-idle daemons show ~0)
- **%MEM** — RSS / total RAM
- **VSZ** — virtual size in KB (mapped, not necessarily resident — includes shared libs, mmap'd files)
- **RSS** — resident set size in KB (actually in physical RAM right now)
- **TTY** — controlling terminal, or `?` for none
- **STAT** — state code (R/S/D/T/Z) plus modifiers (`s` session leader, `+` foreground group, `<` high prio, `N` low prio, `l` multi-threaded)
- **START** — start time
- **TIME** — total CPU time consumed
- **COMMAND** — command line

**The VSZ vs RSS distinction matters**: VSZ is "address space mapped" — a process can mmap a 100 GB file and have huge VSZ but tiny RSS. RSS is what's actually costing you RAM right now. For "is this process leaking memory?" → **watch RSS over time**.

```bash
top
```

In `top`, press:

- `1` — show per-CPU breakdown
- `M` — sort by memory
- `P` — sort by CPU (default)
- `c` — show full command line
- `H` — show threads instead of processes
- `f` — field selector (add/remove columns)
- `t`/`m` — toggle CPU/memory display modes
- `W` — save current config to `~/.toprc`

The header lines decode as:

1. uptime + load averages (1/5/15 min)
2. task counts by state
3. **%Cpu(s)** — `us` user / `sy` system / `ni` nice / `id` idle / `wa` iowait / `hi` hardware irq / `si` softirq / `st` steal (stolen by hypervisor — VMs only)
4. memory: total / free / used / buff/cache
5. swap: total / free / used / available

**Trap**: high `wa` (iowait) means CPUs are idle waiting on I/O. **High `st` on a VM means the hypervisor is starving you** — common AWS noisy-neighbor signal.

### Lab 2: Hand-parse `/proc` (45 min)

```bash
# Pick a long-running process
pgrep -f sshd | head -1
PID=$(pgrep -f sshd | head -1)

ls -l /proc/$PID/exe       # symlink to binary
ls -l /proc/$PID/cwd       # working directory
cat /proc/$PID/cmdline | tr '\0' ' '; echo
cat /proc/$PID/status      # human-readable
cat /proc/$PID/limits
ls /proc/$PID/fd           # open file descriptors

# What ps parses:
cat /proc/$PID/stat
# Field 3 is state, 14/15 are utime/stime, 23 is vsize, 24 is RSS in pages.
# `man 5 proc` is your friend.
```

**Now reproduce a `ps` field by hand**:

```bash
# RSS for the process, in MB
awk '{print $24 * 4 / 1024 " MB"}' /proc/$PID/stat
# (multiply by page size, here 4096 bytes; verify with: getconf PAGESIZE)

ps -o pid,rss,cmd -p $PID
```

The numbers should match. You just did what `ps` does.

### Lab 3: Signals in practice (30 min)

Two terminals.

```bash
# Terminal 1
sleep 600

# Terminal 2
pgrep sleep
kill -SIGSTOP $(pgrep sleep)    # pause it
ps -o pid,stat,cmd -p $(pgrep sleep)    # state? (T)
kill -SIGCONT $(pgrep sleep)    # resume
ps -o pid,stat,cmd -p $(pgrep sleep)    # state? (S — sleeping in nanosleep)
kill -SIGTERM $(pgrep sleep)    # polite
# sleep dies; Terminal 1 exits
```

**Now a process that ignores SIGTERM**:

```bash
# Terminal 1
trap 'echo "ignoring TERM"' TERM
sleep 600 &
wait

# Terminal 2
pgrep -f "sleep 600"            # find the wait or sleep
# Send TERM to the shell — gets ignored (parent shell)
# Then send KILL — works
```

**The takeaway**: well-behaved apps catch `SIGTERM` to clean up. Misbehaving ones may need `SIGKILL`. But `SIGKILL` skips cleanup — you may leave temp files, lockfiles, half-written data.

### Lab 4: Find the noisy process (30 min)

This is the realistic interview scenario: "the box is slow, what do you check?"

```bash
# Top by CPU
top                             # then press P
# Top by memory
top                             # then press M

# Top by I/O — needs iotop (probably need sudo dnf install iotop)
sudo iotop -o                   # -o = only processes doing I/O

# Per-process I/O the manual way
cat /proc/$(pgrep -f firefox | head -1)/io 2>/dev/null
# read_bytes / write_bytes are the most useful

# Run-queue length over time
vmstat 1 5                      # column 'r' = runnable processes; > nproc means CPU-bound
```

If `vmstat`'s `r` column is consistently larger than your CPU count → CPU-bound. If `b` (blocked) is high → I/O-bound. If `wa%` is high → also I/O-bound. If `si`/`so` (swap in/out) is non-zero → memory pressure.

---

## Afternoon Block (1.5h) — Drills + Story #2

### Self-check (45 min)

1. Where does `top` get its data? Be specific.
2. Difference between VSZ and RSS — which one indicates real memory use?
3. I send `kill -9` to a process and it doesn't die. Why?
4. `top` shows `wa: 60%`. What does that mean? Where would you look next?
5. `top` shows `st: 30%` on an EC2 instance. What's happening?
6. Process state codes: what are R, S, D, T, Z?
7. Difference between SIGTERM and SIGKILL — when would you use each?
8. How do you find which process is keeping a deleted file open and eating disk?
9. A process count for `ps` shows hundreds of `<defunct>` entries. What's wrong?
10. Load average is 12.0 on a 4-core box. Is that bad?

<details>
<summary>Answers</summary>

1. `/proc/stat` (system CPU totals) and `/proc/[pid]/stat` for each process. Computes deltas across intervals. Memory from `/proc/meminfo`.
2. **VSZ** = virtual address space mapped (includes shared libs, mmap'd files, unused stack/heap). **RSS** = resident in physical RAM. RSS is what costs you. Watch RSS, not VSZ, for leaks.
3. Process is in `D` (uninterruptible sleep) — blocked in the kernel, typically on stuck I/O (NFS, dead disk, hung block device). SIGKILL is queued but never delivered until it returns to userspace. Fix the I/O or reboot.
4. I/O wait — CPUs are idle, waiting on disk/network. Check `iostat -xz 1`, look at `await` and `%util` per device. Could be a slow disk, EBS throttle, NFS, etc.
5. Steal time — the hypervisor is giving CPU to other tenants. Noisy-neighbor on shared instance, or you're hitting CPU credit exhaustion on a burstable instance type (T-class).
6. R running/runnable, S interruptible sleep, D uninterruptible sleep, T stopped, Z zombie.
7. SIGTERM = polite "please exit" — app gets to clean up. SIGKILL = nuclear, app dies immediately, no cleanup. Always try SIGTERM first.
8. `sudo lsof | grep deleted`, find PID, restart it (or signal it to reopen its log).
9. Parent process isn't reaping children — bug in the parent. Restart the parent. Zombies hold a PID slot but no other resources.
10. Depends on what's contributing. Load = average run-queue length + processes in D state. On 4 cores, 12.0 means roughly 3 processes-worth of demand per core — could be CPU-bound work, could be lots of D-state I/O waiters. Look at `top`/`vmstat` to decide.

</details>

### Behavioral (45 min) — Story #2: Dive Deep

Today's LP: **Dive Deep** — heavy weight for SRE/DevOps. Interviewers want to hear that you don't just escalate or guess; you read the data.

Prompt:
> "Tell me about a time you investigated a problem that others had given up on, or where the obvious answer turned out to be wrong."

STAR. Same drill as yesterday: 2-3 minutes, measurable result, "I" not "we", out loud.

The strongest Dive Deep stories share structure: **expected explanation → went deeper because numbers didn't match → discovered actual root cause → fixed at the source, not the symptom**.
