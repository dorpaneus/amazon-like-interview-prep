# ⚙️ Day 2 — Processes, `/proc`, and the Tools That Read It

> [!NOTE]
> **The Goal:** Yesterday: files are inodes + directory entries. Today: **processes are kernel data structures**, and every tool that shows them (`ps`, `top`, `htop`, `vmstat`, `free`) is just a parser of `/proc`. Once you see this, those tools stop being magic.
>
> The interview question this targets: *"Where do `top` and `ps` get their information?"* — but it also unlocks debugging questions like "process is stuck, what now?" and "memory leak — how do you confirm?"

---

## 🧠 Morning Block — The process model

### 2a. What is the kernel?

The **kernel** is the core of the OS — the only software that talks directly to the hardware (CPU, RAM, disks, NICs). Everything else (`bash`, `nginx`, `python`) is a **userspace** program that must *ask* the kernel for anything real.

Two mental models:

- **Kernel space vs user space.** The CPU runs at two privilege levels. Kernel space (ring 0) can touch hardware and all memory; user space (ring 3) cannot. A crash in user space kills one process; a crash in kernel space — a **kernel panic** — takes down the whole box.
- **The syscall is the door between them.** A program can't read a file or send a packet itself; it makes a **system call** (`read`, `write`, `open`, `fork`, `execve`…) and the kernel does it on its behalf. `strace` (Day 11) shows exactly these doors being knocked on; eBPF (also Day 11) is how modern tools watch the kernel safely from the inside.

> [!TIP]
> **The two-word version:** the kernel is a **resource manager** (shares CPU, memory, and devices between processes) and a **protection boundary** (stops processes from stomping on each other or the hardware). `ps`, `top`, and `/proc` are all just userspace asking the kernel: *"what are you managing right now?"*

> [!IMPORTANT]
> **☁️ AWS Bridge:** `uname -r` shows your running kernel. On Amazon Linux 2023 it's patched via `dnf`, and many fixes only take effect after a reboot (this is the same reason behind the `(deleted)` libraries note in 2e). Kernel tunables live under `/proc/sys` and are set with `sysctl` / `/etc/sysctl.d/`.

### 2b. What is a process?

A **process** is a kernel-managed execution context. The kernel keeps a `task_struct` for each one — PID, parent PID, UIDs/GIDs, open files, memory mappings, signal handlers, scheduler state, working directory, **namespaces, cgroups** (see 2c).

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

### 2c. Namespaces & cgroups — how the kernel isolates a process

The `task_struct` in 2b had **namespaces** and **cgroups** fields. These are the two kernel features **containers are built from** — there is no "container" object in the kernel, just a normal process with these two things configured.

- **Namespaces = what a process can *see*.** Each type virtualizes one global resource so the process thinks it owns it:
  - `pid` — its own PID numbering (its own PID 1)
  - `net` — its own interfaces, routes, ports
  - `mnt` — its own mount table / filesystem view
  - `uts` — its own hostname
  - `ipc` — its own shared memory / semaphores
  - `user` — its own UID/GID map (root inside ≠ root outside)
- **cgroups (control groups) = what a process can *use*.** They **limit and account** for resources: CPU shares, memory ceiling, block I/O, PID count. Blow past the memory cgroup limit and the kernel OOM-kills *inside that group* (ties directly into Day 9's OOM material — the OOM you usually hit in the cloud is a cgroup limit, not the host running dry).

> [!TIP]
> **One line to remember:** **namespaces isolate (visibility), cgroups limit (resources).**

```bash
ls -l /proc/$$/ns/      # namespaces of your current shell
cat /proc/$$/cgroup     # cgroup membership of your shell
lsns                    # list namespaces system-wide
systemd-cgls            # cgroup tree (systemd puts each service in its own cgroup)
```

> [!IMPORTANT]
> **☁️ AWS Bridge — this *is* ECS/Fargate:** A container task is just a process fenced in with namespaces (its own network, PID, mount view) and capped with cgroups (the CPU/memory from your task definition). When an ECS task shows `OOMKilled`, that's the **memory cgroup limit** being hit — *not* the host running out of RAM. Fargate then wraps each task in its own micro-VM (Firecracker) for a hardware isolation layer on top of these.

### 2d. Signals

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

### 2e. `/proc` — the kernel speaks

`/proc` is a **virtual filesystem** populated by the kernel. Every read produces a fresh snapshot. There's no disk involvement — these "files" are kernel functions wearing a filesystem disguise. (This is the kernel-as-resource-manager from 2a answering questions about itself.)

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
| `ns/`     | the process's namespaces (the symlinks from 2c)     |
| `cgroup`  | which cgroups the process belongs to                |
| `status`  | human-readable summary (RSS, threads, state, UIDs)  |
| `stat`    | machine-readable; what ps parses                    |

> [!IMPORTANT]
> **`(deleted)` in `maps` is an ops signal, not a curiosity:** When a package upgrade replaces a shared library, processes that loaded the old one keep running it — `/proc/<pid>/maps` shows the path tagged `(deleted)`. This is the basis of the post-patch "reboot vs just restart the service" decision (and the kernel-patch reboot caveat from 2a).

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

> [!TIP]
> **Bonus — see the isolation from 2c:** `ls -l /proc/$PID/ns/` shows the namespace symlinks, and `cat /proc/$PID/cgroup` shows the cgroups. On a host running containers, compare these between a container process and a host process — the inode numbers in `ns/` differ when they're isolated.

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
11. What's the difference between kernel space and user space, and what is a kernel panic?
12. Namespaces vs cgroups — one line each. Which one does an ECS `OOMKilled` relate to?
13. A "container" — what kernel features is it actually made of? Is there a "container" object in the kernel?

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
11. Kernel space (ring 0) is the privileged level that touches hardware and all memory; user space (ring 3) cannot and must use syscalls to ask the kernel for anything. A **kernel panic** is an unrecoverable fault in kernel space — unlike a userspace crash that kills one process, a panic halts the whole machine.
12. **Namespaces** isolate *what a process can see* (PID, net, mount, hostname, IPC, users). **cgroups** limit *what a process can use* (CPU, memory, I/O, PID count). `OOMKilled` is a **cgroup** memory-limit hit — the task's group exceeded its cap, not the host running out of RAM.
13. Namespaces (isolation) + cgroups (limits), applied to an otherwise normal process. There is **no** "container" object in the kernel — "container" is a userspace convention (runtime + image) over those two primitives. Fargate adds a Firecracker micro-VM around it for hardware isolation.

</details>

### 🤝 Behavioral — Story #2: Dive Deep

Today's LP: **Dive Deep** — heavy weight for SRE/DevOps. Interviewers want to hear that you don't just escalate or guess; you read the data.

Prompt:
> "Tell me about a time you investigated a problem that others had given up on, or where the obvious answer turned out to be wrong."

STAR. Same drill as yesterday: 2-3 minutes, measurable result, "I" not "we", out loud.

> [!TIP]
> The strongest Dive Deep stories share this structure: **expected explanation → went deeper because numbers didn't match → discovered actual root cause → fixed at the source, not the symptom**.
