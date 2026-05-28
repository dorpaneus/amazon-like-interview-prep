# 🚀 Day 4 — Boot, GRUB, systemd, Recovery

> [!NOTE]
> **The Goal:** The boot chain is one of the highest-value interview topics: it lets the interviewer probe firmware, bootloader, kernel, init, services, dependencies, and recovery in one continuous question. 
> 
> "Walk me through what happens from power-on to login prompt" is the canonical version. By tonight you should be able to do that walk in 5 minutes without notes, and you should have practiced regaining root on a system you "locked yourself out of."

---

## 🧠 Morning Block (3h) — From firmware to login

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

| Breaks | Symptom | Recovery layer |
| :--- | :--- | :--- |
| **MBR/ESP corrupt** | "No bootable device" | Live USB → reinstall GRUB |
| **GRUB config bad** | GRUB rescue prompt | GRUB CLI or live USB |
| **Kernel panic** | Panic on screen, halts | Boot older kernel from menu |
| **initramfs missing module** | "Cannot find root device" | Boot rescue → `dracut --regenerate-all -f` |
| **`/etc/fstab` broken** | Boot drops to emergency | Emergency shell → fix fstab |
| **Service fails to start** | Boots but `multi-user` not reached | `systemctl --failed` + journal |
| **Forgot root password** | Can't log in as root | `rd.break` from GRUB |

### 4B. GRUB2 — config, edits, recovery (45 min)

**Config files** (RHEL/Fedora):
- `/etc/default/grub` — high-level settings (`GRUB_TIMEOUT`, `GRUB_CMDLINE_LINUX`, etc.). **You edit this.**
- `/etc/grub.d/` — scripts that generate the actual config.
- `/boot/grub2/grub.cfg` (BIOS) or `/boot/efi/EFI/<distro>/grub.cfg` (UEFI) — the generated file. **Don't edit by hand**; regenerate with:

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg                 # BIOS
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg        # UEFI on RHEL
```

**Boot-menu editing** (the most useful skill): at the GRUB menu, highlight an entry and press <kbd>e</kbd> to edit kernel parameters for one boot. Find the line beginning with `linux` (or `linux16`/`linuxefi`) and append parameters at the end. <kbd>Ctrl</kbd> + <kbd>x</kbd> or <kbd>F10</kbd> to boot.

**Common boot parameters**:
- `single` or `1` — single-user mode (rescue.target)
- `emergency` — even more minimal than rescue (only `/` mounted, read-only)
- `rd.break` — break in the initramfs, before switch_root
- `systemd.unit=multi-user.target` — override default target
- `init=/bin/bash` — skip systemd entirely (very minimal)
- `selinux=0` — disable SELinux for one boot if you suspect SELinux is blocking

### 4C. Regaining root — the canonical procedure (45 min)

This is **the** boot-recovery interview question. Multiple methods; know `rd.break` cold.

**Method 1: `rd.break` (RHEL/Fedora — works without knowing root password)**:
1. Reboot. At GRUB menu, press <kbd>e</kbd> on the default entry.
2. On the `linux` line, append: `rd.break enforcing=0`
3. <kbd>Ctrl</kbd> + <kbd>x</kbd> to boot. You drop into a `switch_root:/#` prompt — **this is the initramfs**, before the real root has been switched to.
4. Real root is mounted at `/sysroot`, **read-only**. Remount it:
```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
```
5. **Critical**: SELinux. With SELinux enforcing, the new `/etc/shadow` has the wrong context, and you'll be locked out again. 
```bash
touch /.autorelabel              # full relabel on next boot (slow, safe)
```
6. `exit` (out of chroot), `exit` again (out of initramfs shell), system continues boot.

> [!IMPORTANT]
> **☁️ The AWS Bridge: EC2 Root Recovery**
> The `rd.break` trick is a brilliant Linux skill, but **it doesn't work in AWS** because you don't have interactive console access to interrupt GRUB. 
> 
> **If you lose access to an EC2 instance, the AWS recovery method is:**
> 1. Stop the broken instance.
> 2. Detach its root EBS volume.
> 3. Attach the volume to a healthy "Rescue" instance.
> 4. Mount the volume on the Rescue instance.
> 5. Chroot into the mounted volume and fix the issue (e.g., reset `passwd`, fix `fstab`, or fix `sshd_config`).
> 6. Unmount, detach, reattach to the original instance, and start it.

### 4D. systemd basics (1h 15min)

systemd is **PID 1** + a unit-based service manager. The mental model: every service, mount, device, target, timer, and socket is a **unit**. Units have **dependencies**, **states**, and a place in **targets** (groups of units).

**Targets you'll see**:
- `default.target` — symlink to whichever target boots by default
- `multi-user.target` — text-mode multi-user (≈ runlevel 3)
- `graphical.target` — adds display manager (≈ runlevel 5)
- `rescue.target` — single-user, root shell, `/` mounted (≈ runlevel 1)
- `emergency.target` — bare minimum: only `/` mounted read-only, no other services

**The commands you must own**:

```bash
systemctl list-units --failed  # what's broken
systemctl cat sshd             # show effective unit file (with drop-ins)
systemctl edit sshd            # create a drop-in override (preferred over editing unit)
systemctl daemon-reload        # RUN THIS after editing unit files
```

**Boot analysis**:
```bash
systemd-analyze                    # total boot time
systemd-analyze blame              # services ranked by startup time
systemd-analyze critical-chain     # the chain on the critical path of boot
```

**The journal** (systemd's logging):
```bash
journalctl -b                      # this boot
journalctl -u sshd -f              # follow (like tail -f)
journalctl -p err                  # priority err and above (alert/crit/err)
journalctl -k                      # kernel messages (≈ dmesg)
```

### 4E. Do I need to reboot? — running vs installed kernel (15 min)

After patching, the box keeps running the *old* kernel until it reboots. The universal check is to compare the running kernel against the newest one on disk:

```bash
uname -r                           # Running kernel
ls -t /boot/vmlinuz-* | head -1    # Newest installed kernel 
```

**The subtler half — reboot vs just restart a service**: A `glibc`/`openssl` upgrade rarely needs a full reboot; often you only need to restart the daemons still mapped to the *old* on-disk library:

```bash
needs-restarting -s         # RHEL: list services needing restart
lsof +c0 | grep -w DEL      # processes mapped to deleted (replaced) libraries
```

---

## 💻 Midday Block (2.5h) — Hands-on labs

### Lab 1: Map your boot (30 min)

```bash
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

> [!TIP]
> **Predict before reading:** Which service is the slowest? Is it on the critical path? (Slow != critical-path; a slow background service that nothing waits on doesn't extend boot time.)

### Lab 2: Custom service (45 min)

Create a real systemd service to internalize the unit format:

```bash
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

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now heartbeat
sudo systemctl status heartbeat
sudo journalctl -u heartbeat -f       # Ctrl-C to stop following
```

**Now break it** to see Restart kick in:

```bash
sudo pkill -f heartbeat.sh
sudo systemctl status heartbeat       # observe restart counter
```

**Drop-in override** (the right way to customize):

```bash
sudo systemctl edit heartbeat
# In the editor, add:
# [Service]
# RestartSec=10
# Then save. Inspect:
sudo systemctl cat heartbeat          # see the drop-in below the original
```

### Lab 3: The fstab trap (20 min)

The classic boot-breaker. **Take a snapshot of your VM first.**

```bash
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

> [!WARNING]
> **The Interview Answer:** A broken fstab drops the system into emergency mode. Always use `nofail` on optional mounts (like external EBS data volumes). **Always run `mount -a` after editing fstab** — never `reboot && hope`.

### Lab 4: The `rd.break` rescue (40 min)

This is the headline lab. You're going to "lose" the root password and recover it.

**Snapshot the VM first.** Then:
1. Set the root password to something you know currently — `sudo passwd root`. Confirm you know it.
2. Reboot. At GRUB, press <kbd>e</kbd> on the default entry.
3. Find the `linux` line and append `rd.break enforcing=0` at the end.
4. <kbd>Ctrl</kbd> + <kbd>x</kbd> to boot.
5. You'll see `switch_root:/#`. Run:

```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel
exit
exit
```

> [!CAUTION]
> **If you get locked out** because you skipped the `.autorelabel` step: SELinux blocks login because `/etc/shadow` has the wrong context. Boot back into `rd.break` and `touch /.autorelabel` correctly.

### Lab 5: Boot to emergency vs rescue, manually (20 min)

```bash
# Switch to rescue from a running system (will drop you to a single-user prompt)
sudo systemctl rescue                  # similar to runlevel 1
# To return: systemctl default

# Try emergency
sudo systemctl emergency               # only / mounted ro, nothing else
# Press Ctrl-D or systemctl default to come back
```

### Lab 6: Is a reboot pending? (20 min)

```bash
# Before: record the running kernel
uname -r

# Apply a kernel update (Ubuntu example)
sudo apt update && sudo apt install --only-upgrade linux-image-generic

# Now check pending state WITHOUT rebooting:
uname -r                              # still the old version
ls -t /boot/vmlinuz-* | head -1       # newer version present
cat /var/run/reboot-required          # exists -> reboot needed

# What still holds old libraries (no reboot needed, just restarts)?
sudo lsof +c0 | grep -w DEL | awk '{print $1}' | sort -u
```

---

## 🎯 Afternoon Block (1.5h) — Drills + Story #4

### Self-check (45 min)

1. Walk me through every step from power button to shell prompt.
2. What's in the initramfs and why is it needed?
3. I edited `/boot/grub2/grub.cfg` directly and my changes disappeared after a kernel update. Why?
4. Difference between `systemctl disable` and `systemctl mask`?
5. Difference between rescue.target and emergency.target?
6. The system boots but `multi-user.target` is never reached. How do you investigate?
7. Walk me through resetting a forgotten root password on a local RHEL VM vs an AWS EC2 instance.
8. After resetting the root password with `rd.break`, I can't log in as root. Why?
9. I added a line to `/etc/fstab` for an NFS share. Reboot fails into emergency. What did I forget?
10. What does `systemd-analyze blame` show, and why is it not always the answer to "what's slowing my boot"?

<details>
<summary><strong>Answers</strong> (click to reveal)</summary>

1. Firmware (BIOS/UEFI) → bootloader (GRUB2) loads kernel + initramfs → kernel inits, mounts initramfs → initramfs loads storage drivers, mounts real `/` → `switch_root` to real root → exec `/sbin/init` (systemd, PID 1) → systemd reads `default.target` and brings up dependencies → target reached → login prompt.
2. A small in-memory filesystem with just enough modules and tools to find and mount the real root filesystem. Needed because the kernel doesn't have built-in drivers for every storage controller, LVM, LUKS, multipath, NFS root, etc. Built by `dracut`.
3. `/boot/grub2/grub.cfg` is generated from `/etc/default/grub` and `/etc/grub.d/` scripts. Kernel updates re-run `grub2-mkconfig`, which overwrites it. Edit `/etc/default/grub` instead, then regenerate.
4. `disable` removes the symlinks that auto-start the unit at boot — you can still `systemctl start` it manually. `mask` symlinks the unit to `/dev/null`, blocking it entirely; `start` will refuse.
5. **Rescue**: `/` mounted rw, basic services running, root shell. **Emergency**: only `/` mounted read-only, no services at all, just a shell. Use emergency when rescue itself can't come up.
6. `systemctl --failed` to see broken units. `systemctl list-jobs` to see what's stuck waiting. `journalctl -b -p err`. Often a service in the dependency chain of `multi-user.target` is hung; `systemctl status <unit>` to find it.
7. Local: `rd.break` via GRUB editing, chroot, passwd, relabel. AWS: Detach EBS volume, attach to rescue instance, mount, chroot, passwd, detach, reattach to original.
8. SELinux. The new `/etc/shadow` has the wrong context after writing in the chroot. Forgetting `touch /.autorelabel` blocks login.
9. `nofail` and ideally `x-systemd.device-timeout=N`. NFS especially needs `_netdev` so the mount waits for network. Without `nofail`, fstab failures send the system to emergency.mode at boot.
10. It shows total time each unit spent in initialization, sorted. Misleading because slow units off the critical path don't extend boot. Use `critical-chain` to see what was actually blocking.

</details>

### 🤝 Behavioral (45 min) — Story #4: Bias for Action

Today's LP: **Bias for Action**

Prompt:
> "Tell me about a time you had to make a decision quickly without having all the information you wanted."

> [!TIP]
> The strongest Bias for Action stories show:
> - A clear time pressure (incident, deadline, missed window).
> - A **reversible** decision (you didn't bet the company).
> - You weighed risk vs cost of waiting and **chose to act**.
> - A measurable outcome — and ideally an honest "I'd do X differently" reflection.

Avoid: "We had a problem and I solved it fast." That's not Bias for Action; that's just doing your job. The LP is about acting *despite uncertainty*.

STAR. Out loud. Time it. 2-3 minutes.
