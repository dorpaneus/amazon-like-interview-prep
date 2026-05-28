# 💾 Day 3 — Filesystems, LVM, LUKS, iSCSI

> [!NOTE]
> **The Goal:** Today goes from the inode (Day 1) up the storage stack: physical/virtual disks → LVM → filesystem → mount. Each layer has a failure mode and a recovery procedure. Interviewers love this stack because it's where "things just broke" happens most.
> 
> The objectives this hits: diagnose FS issues, recover corrupted filesystems, recover mis-configured/broken LVM, recover from encrypted FS, identify and fix iSCSI issues.

---

## 🧠 Morning Block (3h) — The storage stack

### 3A. The block-device → filesystem stack (45 min)

Stack from bottom up:

```text
Physical disk / EBS volume / iSCSI LUN
        ↓
Partition (optional — gpt/mbr)
        ↓
LUKS (optional — encryption)
        ↓
LVM Physical Volume (PV)
        ↓
LVM Volume Group (VG)
        ↓
LVM Logical Volume (LV)
        ↓
Filesystem (ext4, xfs, btrfs)
        ↓
Mount point (/var, /home, etc.)
```

You don't need every layer — common stacks:
- **Simple:** disk → partition → ext4 → mount
- **Server:** disk → partition → LVM PV → VG → LV → xfs → mount
- **Encrypted:** disk → partition → LUKS → LVM → xfs → mount (or LVM under LUKS, both work)

> [!TIP]
> **Why this stack matters in interviews:** When things break, the symptom is at the top (can't access files) but the cause can be at any layer. You learn to **bisect down the stack**: is the disk visible? (`lsblk`) is LUKS open? (`cryptsetup status`) is LVM active? (`vgs`/`lvs`) is the FS clean? (`fsck`/`xfs_repair`) is it mounted? (`mount`/`findmnt`).

**The two filesystems you must know on RHEL/Fedora**:

| Feature | ext4 | XFS |
|---|---|---|
| **Default on** | Older RHEL, Debian/Ubuntu | RHEL 7/8/9, default for `/` |
| **Repair tool**| `fsck.ext4` / `e2fsck` | `xfs_repair` |
| **Shrink** | Yes (`resize2fs` smaller) | **No — XFS cannot shrink** |
| **Grow** | Yes, online | Yes, online (`xfs_growfs`) |
| **Journal** | Optional but always on | Always on |
| **Best for** | Mixed workloads | Large files, parallel I/O |

> [!WARNING]
> **The XFS-can't-shrink trap:** Real interview question. If you grow an XFS volume and need to shrink it, the answer is "you don't — back up, recreate smaller, restore." This is the cost of XFS's design tradeoff for high performance.

### 3B. LVM internals (1h 15min)

LVM is three layers:
1. **Physical Volume (PV)** — a block device (whole disk or partition) initialized with `pvcreate`. Carries metadata at the start describing what it belongs to.
2. **Volume Group (VG)** — pool of PVs. Storage units are **physical extents (PE)**, default 4 MiB.
3. **Logical Volume (LV)** — carved out of a VG. Made of **logical extents (LE)** mapped to PEs. The block device that gets a filesystem.

**Why LVM exists:** Resize without repartitioning, span multiple disks, snapshots, thin provisioning.

> [!CAUTION]
> **Snapshots are copy-on-write:** When you snapshot an LV, the snapshot is initially empty pointers to the origin's blocks. Writes to the origin trigger a copy of the *old* block to the snapshot. 
> **Snapshot full = snapshot becomes invalid**, not paused. This is a high-frequency interview gotcha. Snapshot size needs to be ≥ (expected change rate × time you'll keep it).

**The metadata recovery point** — `vgcfgbackup` automatically saves VG metadata on every change to `/etc/lvm/archive/`. If you wipe a PV header by accident, `vgcfgrestore` brings it back. 

> [!TIP]
> **Interview Scenario:** "You accidentally ran `dd` over the start of `/dev/sdb`, can you recover the LVM?" 
> **Yes.** `vgcfgrestore -f /etc/lvm/archive/vg_data_*.vg vg_data` rewrites the metadata using the archive.

**Common failure modes**:
- **Missing PV** (disk pulled, iSCSI dropped) — `vgs` shows partial. Recover with `vgreduce --removemissing` (LVs touching the missing PV become incomplete) or replace the device and `pvcreate --uuid <old-uuid> --restorefile`.
- **Inactive after boot** — `vgchange -ay` activates.
- **Filter excluding a device** — `/etc/lvm/lvm.conf` has a `filter` line; can hide PVs unintentionally.

> [!IMPORTANT]
> **☁️ The AWS Bridge: LVM vs EBS Elasticity**
> * **The Shift:** In legacy on-prem environments, LVM was mandatory for resizing disks. In AWS, you can dynamically resize an EBS volume on the fly without LVM. Because of this, many modern AMIs do not use LVM by default to keep the stack simple.
> * **The Use Case:** Where LVM *is* still heavily used in AWS is for **Performance Striping**. If a database needs more IOPS or throughput than a single EBS volume can provide, an engineer will attach 4 EBS volumes, create a VG, and create a striped LV across all 4 PVs to multiply the IOPS.

### 3C. LUKS — full-disk encryption (45 min)

LUKS = the standard Linux disk encryption format. Built on `dm-crypt`.

**Architecture**:
- A LUKS-formatted device has a header at the start with: cipher info, master key (encrypted), and **8 key slots** (LUKS1) or up to 32 (LUKS2).
- Each key slot holds a copy of the master key, decryptable by a different passphrase or keyfile.
- Data after the header is encrypted with the master key.

```bash
cryptsetup luksFormat /dev/sdX             # create
cryptsetup luksOpen /dev/sdX cryptdata     # opens, creates /dev/mapper/cryptdata
mkfs.xfs /dev/mapper/cryptdata
mount /dev/mapper/cryptdata /data

cryptsetup luksClose cryptdata             # close (after unmount)

cryptsetup luksDump /dev/sdX               # see header / slots
cryptsetup luksAddKey /dev/sdX             # add another passphrase
cryptsetup luksRemoveKey /dev/sdX          # remove a passphrase
```

> [!WARNING]
> **Header backup is non-negotiable:** Corrupt the header, lose all data. There's no master-key recovery; the master key is *only* in the header, encrypted by your passphrases.

```bash
cryptsetup luksHeaderBackup /dev/sdX --header-backup-file luks-header.bin
# Store this somewhere safe. Without it, header damage = permanent data loss.
cryptsetup luksHeaderRestore /dev/sdX --header-backup-file luks-header.bin
```

**`/etc/crypttab`** — like fstab but for LUKS devices. Format: `name source key options`. Interactive prompt at boot if key is `none`.

### 3D. iSCSI in 15 minutes (15 min)

iSCSI carries SCSI commands over TCP/IP. Server side = **target**. Client side = **initiator**. The exposed disk = **LUN**.

The command-line story:

```bash
# Discover what targets the portal is offering
iscsiadm -m discovery -t sendtargets -p 10.0.0.5

# Log in to a target — disk appears as /dev/sdX
iscsiadm -m node --targetname iqn.2024-01.com.example:storage.lun1 -p 10.0.0.5 --login

# Persistent settings live in /var/lib/iscsi/nodes/...
# Auto-login on boot:
iscsiadm -m node -T <target> -p <portal> --op=update -n node.startup -v automatic

# Logout
iscsiadm -m node --targetname <target> -p <portal> --logout

# See sessions
iscsiadm -m session
```

Common failures:
- **Network reachability** — `nc -vz <portal> 3260` (default iSCSI port).
- **Auth** — CHAP credentials in `/etc/iscsi/iscsid.conf`.
- **IQN mismatch** — `/etc/iscsi/initiatorname.iscsi` must match what the target expects.
- **After reboot disk is gone** — `node.startup` not set to `automatic`.

For HA setups: **multipath** (`multipathd`) groups multiple paths to the same LUN; `multipath -ll` shows current state. If a path fails, I/O continues on another.

---

## 💻 Midday Block (2.5h) — Hands-on labs

You'll need a VM with a few spare virtual disks. If you only have one, attach a couple of small (1–2 GB) virtual disks to it; instructions assume `/dev/sdb` and `/dev/sdc` exist. **Don't run these on a disk that has data you care about.**

### Lab 1: Inspect the existing stack (20 min)

```bash
lsblk -f                       # tree of devices + filesystems + UUIDs + mountpoints
findmnt                        # tree of mounts
df -hT                         # human + filesystem type
df -i                          # inodes — critical for "disk full but du says no"
mount | column -t              # mount options, useful for noatime/errors=remount-ro etc

# Where is each layer?
pvs ; vgs ; lvs                # LVM summary
sudo dmsetup ls --tree         # device-mapper tree (LVM, LUKS, multipath all live here)
```

**Predict before running**: which of your filesystems is XFS vs ext4? What's the mount option on `/`?

### Lab 2: Build an LVM stack (45 min)

```bash
sudo pvcreate /dev/sdb /dev/sdc
sudo pvs

sudo vgcreate vg_lab /dev/sdb /dev/sdc
sudo vgs ; sudo vgdisplay vg_lab

sudo lvcreate -L 500M -n lv_data vg_lab
sudo lvs

sudo mkfs.xfs /dev/vg_lab/lv_data
sudo mkdir /mnt/lab
sudo mount /dev/vg_lab/lv_data /mnt/lab
df -hT /mnt/lab

# Grow the LV and the FS
sudo lvextend -L +200M /dev/vg_lab/lv_data
sudo xfs_growfs /mnt/lab           # XFS: extend FS while mounted
df -hT /mnt/lab

# Or in one step
sudo lvextend -L +200M -r /dev/vg_lab/lv_data
```

**Now break it on purpose** to learn the recovery:

```bash
# Backup metadata (it's auto-archived but practice the command)
sudo vgcfgbackup vg_lab

# "Accidentally" wipe the PV header
sudo dd if=/dev/zero of=/dev/sdb bs=512 count=2048
sudo pvs                        # /dev/sdb missing → VG partial
sudo vgs                        # shows partial

# Recover
sudo ls /etc/lvm/archive/ | grep vg_lab
# Pick the latest backup file
sudo vgcfgrestore -f /etc/lvm/archive/vg_lab_<timestamp>.vg vg_lab
# This rewrites metadata, but the PV UUID is needed:
sudo cat /etc/lvm/backup/vg_lab | grep -A2 "pv0\|/dev/sdb"
# In real recovery you'd: pvcreate --uuid <old-uuid> --restorefile <archive> /dev/sdb
sudo pvcreate --uuid <UUID-from-backup> --restorefile /etc/lvm/archive/vg_lab_<ts>.vg /dev/sdb
sudo vgcfgrestore vg_lab
sudo vgchange -ay vg_lab
sudo pvs ; sudo vgs ; sudo lvs   # back to healthy
```

### Lab 3: Snapshot and rollback (30 min)

```bash
echo "important data v1" | sudo tee /mnt/lab/file.txt
sudo lvcreate -L 100M -s -n lv_data_snap /dev/vg_lab/lv_data

# Make changes
echo "v2 — oops" | sudo tee /mnt/lab/file.txt
cat /mnt/lab/file.txt

# Roll back
sudo umount /mnt/lab
sudo lvconvert --merge /dev/vg_lab/lv_data_snap     # merges snapshot back into origin
sudo mount /dev/vg_lab/lv_data /mnt/lab
cat /mnt/lab/file.txt                                # back to v1
```

Now intentionally fill the snapshot to see invalidation:

```bash
sudo lvcreate -L 50M -s -n lv_data_snap /dev/vg_lab/lv_data
sudo dd if=/dev/zero of=/mnt/lab/big bs=1M count=100   # exceed snapshot size
sudo lvs                                                # snapshot status: 'invalid'
sudo lvremove /dev/vg_lab/lv_data_snap
```

Lesson: **size snapshots for the change rate**, not the size of the origin.

### Lab 4: LUKS — encrypt, key slots, header backup (30 min)

```bash
# Free up a small LV for this
sudo lvcreate -L 200M -n lv_secret vg_lab

sudo cryptsetup luksFormat /dev/vg_lab/lv_secret    # YES, then a passphrase
sudo cryptsetup luksDump /dev/vg_lab/lv_secret      # see slot 0 enabled
sudo cryptsetup luksOpen /dev/vg_lab/lv_secret secret
ls /dev/mapper/secret

sudo mkfs.xfs /dev/mapper/secret
sudo mkdir /mnt/secret
sudo mount /dev/mapper/secret /mnt/secret
echo "classified" | sudo tee /mnt/secret/file

# Backup the header — DO THIS ALWAYS in real life
sudo cryptsetup luksHeaderBackup /dev/vg_lab/lv_secret    --header-backup-file ~/luks-secret-header.bin

# Add a second key
sudo cryptsetup luksAddKey /dev/vg_lab/lv_secret    # asks for existing then new
sudo cryptsetup luksDump /dev/vg_lab/lv_secret      # slot 1 now enabled

# Tear down
sudo umount /mnt/secret
sudo cryptsetup luksClose secret

# Simulate header destruction
sudo dd if=/dev/zero of=/dev/vg_lab/lv_secret bs=1M count=4
sudo cryptsetup luksOpen /dev/vg_lab/lv_secret secret      # fails — no LUKS header

# Restore
sudo cryptsetup luksHeaderRestore /dev/vg_lab/lv_secret    --header-backup-file ~/luks-secret-header.bin
sudo cryptsetup luksOpen /dev/vg_lab/lv_secret secret      # works
sudo mount /dev/mapper/secret /mnt/secret
cat /mnt/secret/file                                       # data intact
```

> [!NOTE]
> The takeaway from the destroy-and-restore cycle: **without the header backup, those 4 MB of zeros would have been permanent total data loss** — every passphrase, every key slot, the master-key wrapper, all gone. The data on disk is still encrypted with the master key, but the key is no longer recoverable.

### Lab 5: XFS repair (20 min)

```bash
# Create a small XFS to corrupt
sudo lvcreate -L 100M -n lv_test vg_lab
sudo mkfs.xfs /dev/vg_lab/lv_test
sudo mkdir /mnt/test
sudo mount /dev/vg_lab/lv_test /mnt/test
echo "data" | sudo tee /mnt/test/file
sudo umount /mnt/test

# Corrupt
sudo dd if=/dev/urandom of=/dev/vg_lab/lv_test bs=512 count=8 seek=64

# Try to mount
sudo mount /dev/vg_lab/lv_test /mnt/test          # fails

# Save metadata image first (always do this in real life)
sudo xfs_metadump /dev/vg_lab/lv_test ~/xfs-meta.img

# Repair
sudo xfs_repair /dev/vg_lab/lv_test               # fix
sudo mount /dev/vg_lab/lv_test /mnt/test
ls /mnt/test                                       # may have lost-and-found entries
```

> [!WARNING]
> **XFS repair rules**:
> - FS must be unmounted (`xfs_repair` refuses on a mounted FS).
> - Run `xfs_repair -n` first — the dry-run, reports without modifying.
> - For ext4: `e2fsck -n /dev/...` is the equivalent dry-run; `e2fsck -y` for auto-yes.
> - **Never run repair on a mounted filesystem** — it corrupts further.

---

## 🎯 Afternoon Block (1.5h) — Drills + Story #3

### Self-check (45 min)

1. Walk me through the full storage stack from physical disk to mount point.
2. I ran `dd if=/dev/zero of=/dev/sdb bs=1M count=10` on a PV. Recover the LVM.
3. Why can't XFS shrink? What do you do if you need to?
4. Difference between LUKS slots and the master key.
5. I lost the LUKS header. Can I recover?
6. My LVM snapshot is showing status `invalid`. What happened?
7. `df` says 80% used, `du -sh /` says 30 GB used on a 100 GB FS. Why the mismatch?
8. iSCSI LUN appeared as `/dev/sdc` yesterday, today it's gone after reboot. Where do you check?
9. I need to grow `/var` which is on an LV. Walk me through the steps with no downtime.
10. `mount` returns "structure needs cleaning" on an ext4 FS. What's the diagnosis and fix?

<details>
<summary><strong>Answers</strong> (click to reveal)</summary>

1. Block device → optional partition → optional LUKS → optional LVM PV → VG → LV → filesystem → mount point. Each layer is independently checkable: `lsblk`, `cryptsetup status`, `pvs`/`vgs`/`lvs`, `xfs_info`/`tune2fs -l`, `findmnt`.
2. `vgcfgbackup` writes archives to `/etc/lvm/archive/`. Find latest, extract the original PV UUID from it (or `/etc/lvm/backup/`). Then `pvcreate --uuid <UUID> --restorefile <archive> /dev/sdb`, then `vgcfgrestore <vg>`, then `vgchange -ay <vg>`.
3. XFS allocates inodes and metadata throughout the volume; there's no provision to relocate them. Solution: back up data, recreate FS at smaller size, restore. Or migrate to a smaller volume.
4. The master key encrypts the data and lives only in the header, encrypted itself. **Each slot** stores a copy of the master key encrypted with a key derived from a passphrase. So adding/removing slots changes who can unlock — but the master key (and thus the data) doesn't change.
5. Only if you have a header backup. Without it, the master key is unrecoverable. Data is technically still encrypted on disk but you have no way to derive the key to decrypt it.
6. Snapshot ran out of space — its CoW area filled before you could merge or delete it. Once invalid, the snapshot is unusable; you can only remove it. Lesson: size snapshots for change rate × retention time.
7. Two common causes: (1) deleted file held open by a process (`lsof +L1`), (2) inode exhaustion — check `df -i`. Also possible: reserved blocks (ext4 reserves 5% for root by default).
8. `iscsiadm -m session` (any active?), `iscsiadm -m node` (any configured?), node startup mode is probably `manual` instead of `automatic`. Also check `/var/log/messages` for connection failures, `nc -vz <portal> 3260` for reachability.
9. `lvextend -L +Xg -r /dev/vgname/var-lv` (the `-r` resizes the FS in the same call: `xfs_growfs` for XFS, `resize2fs` for ext4). All online. If VG is full first: `pvcreate /dev/sdN`, `vgextend vgname /dev/sdN`, then `lvextend`.
10. ext4 journal/metadata corruption. Unmount, run `e2fsck -n /dev/...` first to assess, then `e2fsck -y /dev/...` to repair. Never run on mounted FS. If `/` is affected, boot to rescue / single-user.

</details>

### 🤝 Behavioral (45 min) — Story #3: Insist on the Highest Standards

Today's LP: **Insist on the Highest Standards**. 

Prompt:
> "Tell me about a time you weren't satisfied with how things were running and pushed for a higher bar — even when it created friction."

> [!TIP]
> The strongest stories here have **a defect that others wanted to ship/accept and you blocked or fixed it**: a flaky test treated as "just retry," a runbook everyone tolerated, a postmortem with action items that were never closed. Show: you noticed → you raised it → you owned the fix → measurable improvement after.

STAR. 2-3 minutes. "I" not "we". Out loud, time it, write it down.
