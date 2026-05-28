# Day 4 — Boot, GRUB, systemd, Recovery (Fri May 1)

The boot chain is one of the highest-value interview topics: it lets the interviewer probe firmware, bootloader, kernel, init, services, dependencies, and recovery in one continuous question. "Walk me through what happens from power-on to login prompt" is the canonical version. By tonight you should be able to do that walk in 5 minutes without notes, and you should have practiced regaining root on a system you "locked yourself out of."

---

## Morning Block (3h) — From firmware to login

### 4A. The boot sequence (1h)

End-to-end:

1. **Power-on / firmware** — BIOS or UEFI runs. Initializes hardware, runs POST.
  - **BIOS** (legacy) — reads the first 512 bytes of the boot disk (MBR), executes that code.
  - **UEFI** (modern) — reads the **EFI System Partition (ESP)**, a FAT32 partition (mounted `/boot/efi`), and executes a `.efi` file listed in NVRAM boot entries.
2. **Bootloader (GRUB2)** — chosen by firmware. Loads its config (`grub.cfg`), shows menu, loads the chosen kernel + initramfs into memory, jumps to kernel.
3. **Kernel** — decompresses, initializes drivers compiled into it, mounts the **initramfs** as a temporary root.
4. **initramfs** — small in-memory filesystem with just enough drivers/tools to find and mount the real root. On RHEL/Fedora it's built by `dracut`. Loads modules for storage controllers, LVM, LUKS, network (if root is on iSCSI/NFS).
5. **Switch root** — once initramfs has mounted the real `/`, kernel does `switch_root` and exec's `/sbin/init`.
6. **PID 1 = systemd** — reads `/etc/systemd/system/default.target`, walks dependency graph, brings up units in parallel where possible.
7. **Target reached** — typically `multi-user.target` (text login) or `graphical.target` (display manager). Login prompt appears.

**Where things break** (with the recovery layer):

| Breaks                   | Symptom                                   | Recovery layer                             |
| ------------------------ | ----------------------------------------- | ------------------------------------------ |
| MBR/ESP corrupt          | "No bootable device"                      | Live USB → reinstall GRUB                  |
| GRUB config bad          | GRUB rescue prompt                        | GRUB CLI or live USB                       |
| Kernel panic             | Panic on screen, halts                    | Boot older kernel from menu                |
| initramfs missing module | "Cannot find root device"                 | Boot rescue → `dracut --regenerate-all -f` |
| `/etc/fstab` broken      | Boot drops to emergency                   | Emergency shell → fix fstab                |
| Service fails to start   | Boots but `multi-user.target` not reached | `systemctl --failed` + journal             |
| Forgot root password     | Can't log in as root                      | `rd.break` from GRUB                       |

### 4B. GRUB2 — config, edits, recovery (45 min)

**Config files** (RHEL/Fedora):

- `/etc/default/grub` — high-level settings (`GRUB_TIMEOUT`, `GRUB_CMDLINE_LINUX`, etc.). **You edit this.**

- `/etc/grub.d/` — scripts that generate the actual config.

- `/boot/grub2/grub.cfg` (BIOS) or `/boot/efi/EFI/<distro>/grub.cfg` (UEFI) — the generated file. **Don't edit by hand**; regenerate with:

```
grub2-mkconfig -o /boot/grub2/grub.cfg                 # BIOS
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg        # UEFI on RHEL
```

**Boot-menu editing** (the most useful skill): at the GRUB menu, highlight an entry and press **`e`** to edit kernel parameters for one boot. Find the line beginning with `linux` (or `linux16`/`linuxefi`) and append parameters at the end. **`Ctrl-x`** or `F10` to boot.

**Common boot parameters**:

- `single` or `1` — single-user mode (rescue.target)
- `emergency` — even more minimal than rescue (only `/` mounted, read-only)
- `rd.break` — break in the initramfs, before switch_root
- `systemd.unit=multi-user.target` — override default target
- `init=/bin/bash` — skip systemd entirely (very minimal — no `/proc`, no networking, just a shell)
- `quiet`/`splash` — hide kernel output (remove these to see what's happening on boot failure)
- `selinux=0` — disable SELinux for one boot if you suspect SELinux is blocking
- `nomodeset` — skip GPU mode setting if display is broken

**GRUB rescue prompt** (`grub rescue>`) — appears when GRUB can't find its modules. Extreme minimal shell. From here:

```
grub rescue> ls                                    # see partitions: (hd0,gpt2) etc
grub rescue> set prefix=(hd0,gpt2)/boot/grub2
grub rescue> set root=(hd0,gpt2)
grub rescue> insmod normal
grub rescue> normal                                # full menu loads
```

### 4C. Regaining root — the canonical procedure (45 min)

This is **the** boot-recovery interview question. Multiple methods; know `rd.break` cold.

**Method 1: `rd.break` (RHEL/Fedora — works without knowing root password)**:

1. Reboot. At GRUB menu, press `e` on the default entry.

2. On the `linux` line, append: `rd.break enforcing=0`

3. `Ctrl-x` to boot. You drop into a `switch_root:/#` prompt — **this is the initramfs**, before the real root has been switched to.

4. Real root is mounted at `/sysroot`, **read-only**. Remount it:

```
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
```

5. **Critical**: SELinux. With SELinux enforcing, the new `/etc/shadow` has the wrong context, and you'll be locked out again. Two options:

```
touch /.autorelabel              # full relabel on next boot (slow, safe)
```
    Or, faster, relabel just the file you changed:

```
load_policy -i
chcon system_u:object_r:shadow_t:s0 /etc/shadow
```

6. `exit` (out of chroot), `exit` again (out of initramfs shell), system continues boot.

**Why `enforcing=0`**: it disables SELinux for this boot only, which is needed for the relabel-on-next-boot approach to work without errors.

**Method 2: rescue boot from install media** — boot from RHEL/Fedora ISO, choose Troubleshooting → Rescue. It auto-mounts your installed system at `/mnt/sysimage` and offers to chroot. Then `passwd root`, etc. Use this when `rd.break` doesn't work or the disk is encrypted with LUKS and you need extra steps.

**Method 3: `init=/bin/bash`** — older trick. Drops you into a shell with `/` mounted read-only, no systemd, no `/proc`. Remount `/` rw and `passwd`. Less safe (no SELinux awareness, no proper init) but works on systems where `rd.break` isn't available.

### 4D. systemd basics (1h 15min)

systemd is **PID 1** + a unit-based service manager. The mental model: every service, mount, device, target, timer, and socket is a **unit**. Units have **dependencies**, **states**, and a place in **targets** (groups of units).

**Unit types** (file extensions):

- `.service` — a daemon (most common)
- `.target` — group of units (like old runlevels)
- `.socket` — socket activation; service starts when something connects
- `.timer` — replacement for cron (per-user, journaled, dependency-aware)
- `.mount`/`.automount` — filesystem mounts (auto-generated from fstab)
- `.path` — triggered by filesystem events
- `.device` — kernel device units (auto-generated from udev)
- `.slice` — cgroup hierarchy nodes

**Targets you'll see**:

- `default.target` — symlink to whichever target boots by default
- `multi-user.target` — text-mode multi-user (≈ runlevel 3)
- `graphical.target` — adds display manager (≈ runlevel 5)
- `rescue.target` — single-user, root shell, `/` mounted (≈ runlevel 1)
- `emergency.target` — bare minimum: only `/` mounted read-only, no other services
- `reboot.target` / `poweroff.target` — what the names suggest

**The commands you must own**:

```
systemctl status sshd
systemctl start|stop|restart|reload sshd
systemctl enable sshd          # create symlinks for boot startup
systemctl disable sshd
systemctl mask sshd            # symlink to /dev/null — can't be started even manually
systemctl is-active sshd ; systemctl is-enabled sshd

systemctl list-units --failed  # what's broken
systemctl list-unit-files      # all known units, enabled/disabled state
systemctl list-dependencies multi-user.target
systemctl cat sshd             # show effective unit file (with drop-ins)
systemctl edit sshd            # create a drop-in override (preferred over editing unit)

systemctl get-default
systemctl set-default multi-user.target

systemctl daemon-reload        # after editing unit files
```

**Drop-ins** are how you customize without editing vendor unit files. `systemctl edit foo` creates `/etc/systemd/system/foo.service.d/override.conf`. Survives package upgrades.

**Boot analysis**:

```
systemd-analyze                    # total boot time
systemd-analyze blame              # services ranked by startup time
systemd-analyze critical-chain     # the chain on the critical path of boot
systemd-analyze plot > boot.svg    # gorgeous SVG timeline
systemd-analyze verify foo.service # syntax-check a unit
```

**The journal** (systemd's logging — persistent if `/var/log/journal/` exists):

```
journalctl                         # everything
journalctl -b                      # this boot
journalctl -b -1                   # previous boot
journalctl -u sshd                 # just sshd
journalctl -u sshd -f              # follow (like tail -f)
journalctl -u sshd --since "1 hour ago"
journalctl -p err                  # priority err and above (alert/crit/err)
journalctl -k                      # kernel messages (≈ dmesg)
journalctl --disk-usage            # how much journal is taking
journalctl --vacuum-time=2weeks    # purge older
```

**Priority levels** (numeric and name): `0 emerg`, `1 alert`, `2 crit`, `3 err`, `4 warning`, `5 notice`, `6 info`, `7 debug`. `journalctl -p 3` means err and worse.

### 4E. Do I need to reboot? — running vs installed kernel (15 min)

After patching, the box keeps running the *old* kernel until it reboots. The universal, distro-agnostic check is to compare the running kernel against the newest one on disk:

```
# Running kernel
uname -r

# Newest installed kernel (Debian/Ubuntu)
ls -t /boot/vmlinuz-* | head -1
```

If the version in `/boot` is newer than `uname -r`, a kernel reboot is pending.

**Distro signals that are cleaner than eyeballing filenames**:

- **Debian/Ubuntu**: the file `/var/run/reboot-required` exists when a reboot is needed (created by `unattended-upgrades`/`needrestart`); `cat /var/run/reboot-required.pkgs` lists which packages triggered it.
- **RHEL/Fedora**: `needs-restarting -r` (from `dnf-utils`) exits `1` and prints "Reboot is required" when a core lib (kernel, `glibc`, `systemd`, `openssl`…) was updated under running processes.

**The subtler half — reboot vs just restart a service**: a `glibc`/`openssl` upgrade rarely needs a full reboot; often you only need to restart the daemons still mapped to the *old* on-disk library:

```
needrestart                 # interactive: services holding stale libs (Deb/RHEL)
needs-restarting -s         # RHEL: list services needing restart
lsof +c0 | grep -w DEL      # processes mapped to deleted (replaced) libraries
```

(See Day 2 §2C — a process whose `/proc/<pid>/maps` shows a library marked `(deleted)` is still running the old code. That's the same signal, viewed from `/proc`.)

**Avoiding the reboot entirely**: live kernel patching — `kpatch` (RHEL), `canonical-livepatch` (Ubuntu), or Ksplice — applies CVE fixes to the running kernel so you defer the reboot to a maintenance window. Know it exists; the "how do you patch a fleet for a kernel CVE with minimal downtime" question lives here.

---

## Midday Block (2.5h) — Hands-on labs

### Lab 1: Map your boot (30 min)

```
systemd-analyze
systemd-analyze blame | head -20
systemd-analyze critical-chain
systemctl get-default
systemctl list-units --type=target --state=active

# What's failed?
systemctl --failed

# Read the journal for this boot
journalctl -b -p err              # errors only
journalctl -b | head -100         # the early boot lines
```

**Predict before reading**: which service is the slowest? Is it on the critical path? (Slow != critical-path; a slow background service that nothing waits on doesn't extend boot time.)

### Lab 2: Custom service (45 min)

Create a real systemd service to internalize the unit format:

```
# A trivial daemon
sudo tee /usr/local/bin/heartbeat.sh >/dev/null <<'EOF'
#!/bin/bash
while true; do
  echo "$(date) heartbeat" >> /var/log/heartbeat.log
  sleep 5
done
EOF
sudo chmod +x /usr/local/bin/heartbeat.sh

# Unit file
sudo tee /etc/systemd/system/heartbeat.service >/dev/null <<'EOF'
[Unit]
Description=Heartbeat logger
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/heartbeat.sh
Restart=on-failure
RestartSec=3
User=nobody
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now heartbeat
sudo systemctl status heartbeat
sudo journalctl -u heartbeat -f       # Ctrl-C to stop following
```

**Now break it** to see Restart kick in:

```
sudo pkill -f heartbeat.sh
sudo systemctl status heartbeat       # observe restart counter

# Make it fail permanently
sudo sed -i 's|/usr/local/bin/heartbeat.sh|/usr/local/bin/missing|' /etc/systemd/system/heartbeat.service
sudo systemctl daemon-reload
sudo systemctl restart heartbeat
sudo systemctl status heartbeat       # failed; see how many times
sudo journalctl -u heartbeat | tail
```

**Drop-in override** (the right way to customize):

```
sudo systemctl edit heartbeat
# In the editor, add:
# [Service]
# RestartSec=10
# Then save. Inspect:
sudo systemctl cat heartbeat          # see the drop-in below the original
```

Cleanup:

```
sudo systemctl disable --now heartbeat
sudo rm /etc/systemd/system/heartbeat.service
sudo rm -rf /etc/systemd/system/heartbeat.service.d
sudo rm /usr/local/bin/heartbeat.sh /var/log/heartbeat.log
sudo systemctl daemon-reload
```

### Lab 3: The fstab trap (20 min)

The classic boot-breaker. **Take a snapshot of your VM first.**

```
sudo cp /etc/fstab /etc/fstab.bak

# Add a non-existent device with default options
echo "UUID=00000000-0000-0000-0000-000000000000  /mnt/fake  xfs  defaults  0 0" | sudo tee -a /etc/fstab

# Test before rebooting (this is THE habit to develop)
sudo systemctl daemon-reload
sudo mount -a                      # fails — and that's exactly what would happen at boot

# How to make this non-fatal at boot:
# Edit the line: change "defaults" to "defaults,nofail,x-systemd.device-timeout=5"
sudo sed -i 's|defaults  0 0|defaults,nofail,x-systemd.device-timeout=5  0 0|' /etc/fstab
sudo mount -a                      # now no fatal error

# Restore
sudo cp /etc/fstab.bak /etc/fstab
sudo systemctl daemon-reload
```

**The interview answer**: a broken fstab drops the system into emergency mode (or rescue). Always `nofail` on optional mounts. **Always run `mount -a` after editing fstab** — never `reboot && hope`.

### Lab 4: The `rd.break` rescue (40 min)

This is the headline lab. You're going to "lose" the root password and recover it.

**Snapshot the VM first.** Then:

1. Set the root password to something you know currently — `sudo passwd root`. Confirm you know it.

2. Reboot. At GRUB, press `e` on the default entry.

3. Find the `linux` line and append `rd.break enforcing=0` at the end.

4. `Ctrl-x` to boot.

5. You'll see `switch_root:/#`. Run:

```
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel
exit
exit
```

6. System reboots, runs the SELinux relabel (slow, may take minutes), reboots again.

7. Log in with the new root password.

**If you get locked out** because you skipped the `.autorelabel` step: SELinux blocks login because `/etc/shadow` has the wrong context. Repeat the procedure, this time `touch /.autorelabel` correctly.

### Lab 5: Boot to emergency vs rescue, manually (20 min)

```
# Switch to rescue from a running system (will drop you to a single-user prompt)
sudo systemctl rescue                  # similar to runlevel 1
# To return:
# (in the rescue prompt) systemctl default

# Try emergency
sudo systemctl emergency               # only / mounted ro, nothing else
# Press Ctrl-D or systemctl default to come back
```

**Difference**: rescue brings up the root filesystem and runs basic services; emergency is the absolute minimum — just `/` read-only and a shell. Use emergency when even rescue fails.

### Lab 6: Is a reboot pending? (20 min)

**Snapshot the VM first.** The goal is to make "pending reboot" a state you can *see*, three different ways.

```
# Before: record the running kernel
uname -r

# Apply a kernel update (Ubuntu example)
sudo apt update && sudo apt install --only-upgrade linux-image-generic

# Now check pending state WITHOUT rebooting:
uname -r                              # still the old version
ls -t /boot/vmlinuz-* | head -1       # newer version present
cat /var/run/reboot-required          # exists -> reboot needed
cat /var/run/reboot-required.pkgs     # why

# RHEL/Fedora equivalent:
# sudo dnf upgrade kernel
# needs-restarting -r ; echo "exit=$?"   # exit 1 = reboot required

# What still holds old libraries (no reboot needed, just restarts)?
sudo lsof +c0 | grep -w DEL | awk '{print $1}' | sort -u

# Reboot, then confirm the running kernel now matches /boot
sudo reboot
# (after login)
uname -r
```

**The interview answer**: don't reboot reflexively. Distinguish *kernel* pending (needs reboot, or a livepatch) from *library* pending (restart only the affected daemons). `lsof ... DEL` and `/proc/<pid>/maps` tell you exactly which processes are still on the stale library.

---

## Afternoon Block (1.5h) — Drills + Story #4

### Self-check (45 min)

1. Walk me through every step from power button to shell prompt.
2. What's in the initramfs and why is it needed?
3. I edited `/boot/grub2/grub.cfg` directly and my changes disappeared after a kernel update. Why?
4. Difference between `systemctl disable` and `systemctl mask`?
5. Difference between rescue.target and emergency.target?
6. The system boots but `multi-user.target` is never reached. How do you investigate?
7. Walk me through resetting a forgotten root password on a RHEL system.
8. After resetting the root password with `rd.break`, I can't log in as root. Why?
9. I added a line to `/etc/fstab` for an NFS share. Reboot fails into emergency. What did I forget?
10. `systemctl status foo` shows `Active: failed (Result: exit-code)`. How do you find the actual error?
11. What does `systemd-analyze blame` show, and why is it not always the answer to "what's slowing my boot"?
12. A service must start *only after* the network is fully up and DNS is resolvable. What's the right `After=` and `Wants=`/`Requires=` configuration?
13. After a `dnf update`, how do you tell whether the box needs a full reboot versus just a service restart?
14. A security CVE is in the kernel but the service can't take downtime now. What are your options?

<details>
<summary><strong>Answers</strong></summary>

1. Firmware (BIOS/UEFI) → bootloader (GRUB2) loads kernel + initramfs → kernel inits, mounts initramfs → initramfs loads storage drivers, mounts real `/` → `switch_root` to real root → exec `/sbin/init` (systemd, PID 1) → systemd reads `default.target` and brings up dependencies → target reached → login prompt.
2. A small in-memory filesystem with just enough modules and tools to find and mount the real root filesystem. Needed because the kernel doesn't have built-in drivers for every storage controller, LVM, LUKS, multipath, NFS root, etc. Built by `dracut`.
3. `/boot/grub2/grub.cfg` is generated from `/etc/default/grub` and `/etc/grub.d/` scripts. Kernel updates re-run `grub2-mkconfig`, which overwrites it. Edit `/etc/default/grub` instead, then regenerate.
4. `disable` removes the symlinks that auto-start the unit at boot — you can still `systemctl start` it manually. `mask` symlinks the unit to `/dev/null`, blocking it entirely; `start` will refuse.
5. **Rescue**: `/` mounted rw, basic services running, root shell. **Emergency**: only `/` mounted read-only, no services at all, just a shell. Use emergency when rescue itself can't come up.
6. `systemctl --failed` to see broken units. `systemctl list-jobs` to see what's stuck waiting. `journalctl -b -p err`. Often a service in the dependency chain of `multi-user.target` is hung; `systemctl status <unit>` to find it.
7. Reboot → GRUB → `e` → append `rd.break enforcing=0` to the linux line → Ctrl-x → in initramfs shell: `mount -o remount,rw /sysroot`, `chroot /sysroot`, `passwd root`, `touch /.autorelabel`, `exit`, `exit`. System reboots, relabels, reboots again, you log in.
8. SELinux. The new `/etc/shadow` has the wrong context after writing in the chroot. Forgetting `touch /.autorelabel` (or doing the relabel another way) blocks login. Boot back into `rd.break` and create the file.
9. `nofail` and ideally `x-systemd.device-timeout=N`. NFS especially needs `_netdev` so the mount waits for network. Without `nofail`, fstab failures send the system to emergency.mode at boot.
10. `journalctl -u foo` for the unit's own log, plus `journalctl -b -u foo --since "5 min ago"`. Look at ExecStart command, exit code, missing files, permission errors. `systemctl cat foo` to verify the unit file is what you expect (drop-ins included).
11. It shows total time each unit spent in initialization, sorted. Misleading because slow units off the critical path don't extend boot. Use `critical-chain` to see what was actually blocking.
12. `After=network-online.target nss-lookup.target` and `Wants=network-online.target`. `network-online.target` is "real" connectivity, not just "the network service started." `Wants` is a soft dependency that pulls the target in; `Requires` would fail your unit if the target failed.
13. Compare running vs installed kernel (`uname -r` vs newest `/boot/vmlinuz-*`), or use the distro signal: `/var/run/reboot-required` on Debian/Ubuntu, `needs-restarting -r` on RHEL. For libraries (not the kernel), `needrestart` / `needs-restarting -s` / `lsof +c0 | grep DEL` show which *processes* hold deleted libs — restart just those instead of rebooting.
14. Live patch the running kernel (`kpatch`/`canonical-livepatch`/Ksplice) to close the CVE now, then schedule the real reboot for a maintenance window. For a fleet, orchestrate rolling reboots behind a load balancer so capacity stays up.

</details>

### Behavioral (45 min) — Story #4: Bias for Action

Today's LP. Story prompt:
> "Tell me about a time you had to make a decision quickly without having all the information you wanted."

The strongest Bias for Action stories show:

- A clear time pressure (incident, deadline, missed window)
- A reversible decision (you didn't bet the company)
- You weighed risk vs cost of waiting and **chose to act**
- A measurable outcome — and ideally an honest "I'd do X differently" reflection

Avoid: "we had a problem and I solved it fast." That's not Bias for Action; that's just doing your job. The LP is about acting *despite uncertainty*.

STAR. Out loud. Time it. 2-3 minutes.
