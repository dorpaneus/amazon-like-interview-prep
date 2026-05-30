# ⚙️ Day 2 — Processes, `/proc`, and the Tools That Read It

> [!NOTE]
> **The Goal:** Yesterday: files are inodes + directory entries. Today: **processes are kernel data structures**, and every tool that shows them (`ps`, `top`, `htop`, `vmstat`, `free`) is just a parser of `/proc`. Once you see this, those tools stop being magic.
>
> The interview question this targets: *"Where do `top` and `ps` get their information?"* — but it also unlocks debugging questions like "process is stuck, what now?" and "memory leak — how do you confirm?"

---

## 🧠 Morning Block — The process model

### 2a. What is a process?

A **process** is a kernel-managed execution context. The kernel keeps a `task_struct` for each one — PID, parent PID, UIDs/GIDs, open files, memory mappings, signal handlers, scheduler state, working directory, namespaces, cgroups.

Key concepts:

- **PID 1** — `init` or `systemd`. Started by the kernel; adopts orphans; reaping it crashes the system.
- **Parent / child** — every process except PID 1 has a parent. `fork()` creates a copy; `exec()` replaces the program. Shell command execution = fork + exec.
- **Process groups & sessions** — for job control and signal delivery to terminal-attached processes.
- **Threads** — share the same memory and FDs but each has its own kernel scheduling entity (PID inside the kernel; what userspace calls TID).

**Process states** (`ps` STAT column):
* `R` — running or runnable (on a CPU or in run queue)
* `S` — interruptible sleep (most common; waiting on something)
* `D` — uninterruptible sleep (waiting on I/O or kernel; **can't be killed**, even with SIGKILL)
* `T` — stopped (SIGSTOP / Ctrl-Z)
* `Z` — zombie (exited, parent hasn't called `wait()` to reap it)

> [!WARNING]
> **The `D` state matters in interviews:** "I tried `kill -9` and the process won't die." 
> **Answer:** It is in `D` state, blocked in the kernel — typically waiting on stuck I/O (NFS hang, dead disk, hung iSCSI). You can't kill it; you fix the underlying I/O or reboot.

> [!WARNING]
> **Zombies matter too:** Lots of zombies = parent process has a bug (not calling `wait()`). They're harmless individually but exhaust the PID space if unbounded. To kill a zombie, you must kill its *parent*.

### 2b. Signals

Signals are how the kernel and processes communicate asynchronously. Memorize these — they come up:

| Signal              | Number | What it does                                  | Catchable?          |
| ------------------- | ------ | ----------------------------------------- | ------------------- |
| `SIGHUP`            | 1      | Hangup / reload config (by convention)    | Yes                 |
| `SIGINT`            | 2      | Ctrl-C, polite interrupt                  | Yes                 |
| `SIGQUIT`           | 3      | Ctrl-\\, quit + core dump                   | Yes                 |
| `SIGKILL`           | 9      | Terminate immediately                     | **No** |
| `SIGTERM`           | 15     | Polite "please exit" — default for `kill` | Yes                 |
| `SIGSTOP`           | 19     | Pause                                     | **No** |
| `SIGCONT`           | 18     | Resume after STOP                         | Yes                 |
| `SIGUSR1`/`SIGUSR2` | 10/12  | App-defined                               | Yes                 |
| `SIGSEGV`           | 11     | Segfault — invalid memory access          | Yes (rarely caught) |
| `SIGPIPE`           | 13     | Wrote to a closed pipe                    | Yes                 |

**Rules**:
- `kill -9` is the nuclear option. **Try `SIGTERM` first** so the app cleans up (flushes buffers, closes connections, removes lockfiles). Loops asking for `kill -9` immediately are a smell.
- `SIGKILL` and `SIGSTOP` cannot be caught, blocked, or ignored. The kernel handles them.
- A process in `D` state ignores everything, including SIGKILL — it's not running userspace code to handle it.

> [!TIP]
> **Why `SIGHUP` reloads config:** By historical convention, daemons re-read their config on SIGHUP. It's not magic — the daemon's signal handler does it. `nginx -s reload` sends SIGHUP under the hood.

### 2c. `/proc` — the kernel speaks

`/proc` is a **virtual filesystem** populated by the kernel. Every read produces a fresh snapshot. There's no disk involvement — these "files" are kernel functions wearing a filesystem disguise.

**Per-process — `/proc/[pid]/`**:

| Path      | What it contains                                    |
| --------- | --------------------------------------------------- |
| `cmdline` | null-separated argv                                 |
| `comm`    | short process name (15 chars max)                   |
| `cwd`     | symlink to working directory                        |
| `environ` | null-separated environment vars                     |
| `exe`     | symlink to the binary                               |
| `fd/`     | directory of open file descriptors (each a symlink) |
| `io`      | bytes read/written, syscalls                        |
| `limits`  | ulimits (RLIMIT\_\*)                                |
| `maps`    | memory mappings (also smaps for detail)             |
| `status`  | human-readable summary (RSS, threads, state, UIDs)  |
| `stat`    | machine-readable; what ps parses                    |

> [!IMPORTANT]
> **`(deleted)` in `maps` is an ops signal, not a curiosity:** When a package upgrade replaces a shared library, processes that loaded the old one keep running it — `/proc/<pid>/maps` shows the path tagged `(deleted)`. This is the basis of the post-patch "reboot vs just restart the service" decision.

**System-wide — `/proc/`**:

| Path         | What it contains                                                 |
| ------------ | ---------------------------------------------------------------- |
| `cpuinfo`    | per-CPU model, flags, frequency                                  |
| `meminfo`    | memory totals, free, buffers, cached, swap                       |
| `loadavg`    | 1/5/15 min run-queue length + nr\_running/nr\_total + last\_pid  |
| `stat`       | CPU time totals (user/nice/system/idle/iowait/irq/softirq/steal) |
| `sys/`       | tunables (same as sysctl)                                        |

**`/sys` is the sibling:** Device tree, drivers, kernel object hierarchy. Mostly *writable* for tunables (unlike most of `/proc`). Examples:
- `/sys/block/sda/queue/scheduler` — current I/O scheduler
- `/sys/kernel/mm/transparent_hugepage/enabled` — THP setting

> [!IMPORTANT]
> **☁️ The AWS Bridge: Load Average & Steal Time**
> * **D-State and CloudWatch:** If an EC2 instance has processes stuck in `D` state (waiting on a failing EBS volume), the Linux OS load average (`/proc/loadavg`) will skyrocket. However, CloudWatch `CPUUtilization` might stay very low, because the CPU isn't calculating—it's waiting.
> * **Steal Time (`st`):** In `top`, high `st` means the AWS hypervisor (Nitro/Xen) is giving CPU cycles to other tenants. This happens if you are a noisy neighbor on a shared instance, or if you hit CPU credit exhaustion on a burstable instance (like a `t3.micro`).

---

## 💻 Midday Block — Hands-on labs

### Lab 1: Walk the columns of `ps` and `top`

```bash
ps aux | head
```

Each column:
- **USER** / **PID**
- **%CPU** — CPU% over the process's lifetime.
- **%MEM** — RSS / total RAM
- **VSZ** — virtual size in KB (mapped, not necessarily resident — includes shared libs, mmap'd files)
- **RSS** — resident set size in KB (actually in physical RAM right now)
- **STAT** — state code (R/S/D/T/Z) plus modifiers (`s` session leader, `+` foreground group, `<` high prio, `N` low prio, `l` multi-threaded)

> [!TIP]
> **The VSZ vs RSS distinction matters:** VSZ is "address space mapped" — a process can mmap a 100 GB file and have huge VSZ but tiny RSS. RSS is what's actually costing you RAM right now. For "is this process leaking memory?" → **watch RSS over time**.

```bash
top
```

In `top`, press:
- <kbd>1</kbd> — show per-CPU breakdown
- <kbd>M</kbd> — sort by memory
- <kbd>c</kbd> — show full command line
- <kbd>H</kbd> — show threads instead of processes

### Lab 2: Hand-parse `/proc`

```bash
# Pick a long-running process
PID=$(pgrep -f sshd | head -1)

ls -l /proc/$PID/exe       # symlink to binary
ls -l /proc/$PID/cwd       # working directory
cat /proc/$PID/cmdline | tr '\0' ' '; echo
cat /proc/$PID/status      # human-readable
ls /proc/$PID/fd           # open file descriptors
```

**Now reproduce a `ps` field by hand**:

```bash
# RSS for the process, in MB (Field 24 in stat)
awk '{print $24 * 4 / 1024 " MB"}' /proc/$PID/stat
# (multiply by page size, here 4096 bytes; verify with: getconf PAGESIZE)

ps -o pid,rss,cmd -p $PID
```
The numbers should match. You just did what `ps` does.

### Lab 3: Signals in practice

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
kill -SIGTERM $(pgrep sleep)    # polite (sleep dies; Terminal 1 exits)
```

**Now a process that ignores SIGTERM**:

```bash
# Terminal 1
trap 'echo "ignoring TERM"' TERM
sleep 600 &
wait

# Terminal 2
# Send TERM to the shell — gets ignored (parent shell)
# Then send KILL — works
```

### Lab 4: Find the noisy process

This is the realistic interview scenario: "the box is slow, what do you check?"

```bash
# Top by CPU / Memory
top # press P or M

# Top by I/O (often needs installation: sudo dnf install iotop)
sudo iotop -o                   

# Per-process I/O the manual way
cat /proc/$(pgrep -f firefox | head -1)/io 2>/dev/null

# Run-queue length over time
vmstat 1 5                      # column 'r' = runnable processes; > nproc means CPU-bound
```

> [!NOTE]
> If `vmstat`'s `r` column is consistently larger than your CPU count → **CPU-bound**. 
> If `b` (blocked) is high → **I/O-bound**. 
> If `wa%` is high → **I/O-bound**. 
> If `si`/`so` (swap in/out) is non-zero → **Memory pressure**.

---

## 🎯 Afternoon Block — Drills + Story #2

### Self-check

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
<summary><strong>Answers</strong> (click to reveal)</summary>

1. `/proc/stat` (system CPU totals) and `/proc/[pid]/stat` for each process. Computes deltas across intervals. Memory from `/proc/meminfo`.
2. **VSZ** = virtual address space mapped (includes shared libs, unused heap). **RSS** = resident in physical RAM. RSS is what costs you. Watch RSS, not VSZ, for leaks.
3. Process is in `D` (uninterruptible sleep) — blocked in the kernel, typically on stuck I/O. SIGKILL is queued but never delivered until it returns to userspace. Fix the I/O or reboot.
4. I/O wait — CPUs are idle, waiting on disk/network. Check `iostat -xz 1`, look at `await` and `%util` per device. Could be a slow disk, EBS throttle, NFS, etc.
5. Steal time — the hypervisor is giving CPU to other tenants. Noisy-neighbor on shared instance, or you hit CPU credit exhaustion on a burstable instance type (T-class).
6. R running/runnable, S interruptible sleep, D uninterruptible sleep, T stopped, Z zombie.
7. SIGTERM = polite "please exit" — app gets to clean up. SIGKILL = nuclear, app dies immediately, no cleanup. Always try SIGTERM first.
8. `sudo lsof +L1`, find PID, restart it (or signal it to reopen its log).
9. Parent process isn't reaping children — bug in the parent. Restart the parent. Zombies hold a PID slot but no other resources.
10. Depends on what's contributing. Load = average run-queue length + processes in D state. On 4 cores, 12.0 means roughly 3 processes-worth of demand per core — could be CPU-bound work, could be lots of D-state I/O waiters. Look at `top`/`vmstat` to decide.

</details>

### 🤝 Behavioral — Story #2: Dive Deep

Today's LP: **Dive Deep** — heavy weight for SRE/DevOps. Interviewers want to hear that you don't just escalate or guess; you read the data.

Prompt:
> "Tell me about a time you investigated a problem that others had given up on, or where the obvious answer turned out to be wrong."

STAR. Same drill as yesterday: 2-3 minutes, measurable result, "I" not "we", out loud.

> [!TIP]
> The strongest Dive Deep stories share this structure: **expected explanation → went deeper because numbers didn't match → discovered actual root cause → fixed at the source, not the symptom**.
