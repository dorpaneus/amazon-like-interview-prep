# 🐧 Day 1 - Linux Fundamentals

> [!NOTE]  
> **The Goal:** The interview will pull on these threads constantly: 
> * "Disk full but `df` says space available" → **inode exhaustion**. 
> * "I deleted the file but disk is still full" → **open FD holding the inode**. 
> * "Hardlink survives the original being deleted, why?" → **because the original was never special**. 
> 
> Today builds the mental model that makes those answers automatic.

---

## 🧠 Morning Block — The Unix file model

### 1a. Everything is a file

# Understanding Linux File Systems: Inodes and Directory Entries

## The Core Mental Model
In Linux, what we typically think of as a "file" is actually two distinct components tied together. The name you see is just a label; the actual file is the underlying data structure. 

1. **The Inode (Index Node):** This is the true representation of the file. It contains the metadata and the physical pointers to the actual data blocks on the disk. An inode stores:
    * Ownership (User and Group)
    * Permissions (Read, Write, Execute)
    * File Size
    * Timestamps (Creation, Modification, Access)
    * Link Count (How many names point to this specific inode)
    * Block List (The disk addresses of the actual data)
    
2. **The Directory Entry:** This is the human-readable part. It is simply a mapping that pairs a `filename` with an `inode_number`. These entries live inside directories.

## Key Implications of this Model

Understanding the separation between names and inodes explains several core Linux behaviors:

* **Hard Links:** Creating a hard link does not copy the file's data. It simply creates a new directory entry (a new name) that points to the *exact same inode*.
* **File Deletion (`rm`):** Running the `rm` command does not instantly erase data from the disk. Instead, it "unlinks" the file by removing the directory entry and decrementing the inode's link count. The actual data blocks are only freed by the system when the link count hits `0` **and** no active processes are currently holding the file open.
* **Filesystem Boundaries:** You cannot create a hard link across different filesystems or partitions. Because inode numbers are generated independently by each filesystem, they are only unique and valid locally.

## Linux File Types Reference

When you list files using the `ls -l` command, the very first character of the output string indicates the type of the file. 

| Character | File Type | Common Use Case |
| :---: | :--- | :--- |
| `-` | **Regular File** | Standard data files (text files, images, scripts, compiled binaries). |
| `d` | **Directory** | A "folder" that contains a list of directory entries. |
| `l` | **Symbolic Link** | A soft link or shortcut that points to another file path (unlike a hard link which points to an inode). |
| `b` | **Block Device** | Hardware devices that read and write data in defined blocks (e.g., hard drives, SSDs). |
| `c` | **Character Device** | Devices that read and write data sequentially, character-by-character (e.g., keyboards, mice, terminals). |
| `p` | **Named Pipe (FIFO)** | Used for advanced inter-process communication, allowing data to flow from one process to another. |
| `s` | **Socket** | Used for network connections or local inter-process communication. |


> [!IMPORTANT]
> **☁️ The AWS Bridge: S3 vs. POSIX & EBS**
> Interviewers love testing if you know where Linux file models end and Cloud storage begins.
> * **EBS (Elastic Block Store):** This is a raw block device. When you format it with `ext4` or `xfs`, all the rules of inodes, hard links, and `df -h` apply perfectly. If an EBS volume runs out of inodes, the instance fails to write data even if AWS shows the volume has gigabytes of free space.
> * **S3 (Simple Storage Service):** This is object storage, *not* a file system. S3 does not have inodes, directories, or traditional file permissions. "Directories" in S3 are just string prefixes in the object key (e.g., `folder/file.txt` is just one long name). You cannot hard link in S3.

### Anatomy of an `ls -l` line — column by column

A single long-listing line packs **seven fields**, space-separated. Take this example:

`-rw-r--r--   1   alice   staff   4096   May 28 10:30   notes.txt`

| # | Example        | Field                         | What it means |
| - | -------------- | ----------------------------- | ------------- |
| 1 | `-rw-r--r--`   | **File type + permissions** | The *first* character is the file type (`-` file, `d` dir, `l` symlink, …). The next 9 characters are three permission triads — **owner**, **group**, **other** — each `rwx`. A trailing `.` means an SELinux context is present; a trailing `+` means a POSIX ACL is set. |
| 2 | `1`            | **Hard-link count** | How many directory entries (names) point at this inode. Regular files start at 1; directories start at 2 and gain 1 for every subdirectory, because each subdir's `..` links back to the parent. |
| 3 | `alice`        | **Owner (user)** | The user that owns the inode. This is who the *first* permission triad applies to, and what a setuid binary runs as. |
| 4 | `staff`        | **Group** | The owning group — who the *middle* permission triad applies to. (`ls -ln` shows the numeric UID/GID instead of names.) |
| 5 | `4096`         | **Size** | Size in **bytes** by default (`ls -lh` makes it human-readable). For a directory this is the size of its entry table, not the contents within. For a device file you'll see `major, minor` numbers here instead of a byte count. |
| 6 | `May 28 10:30` | **Modification time (mtime)** | When the file's *data* last changed. `ls -lc` shows ctime (inode change), `ls -lu` shows atime (access), and `ls -l --full-time` shows full precision. |
| 7 | `notes.txt`    | **Name** | The filename — i.e. the directory entry itself. For a symlink, `ls -l` appends `-> target` showing where it points. |

> [!TIP]
> **Quick way to remember the order:** > perms → links → owner → group → size → time → name.

### 1b. Permissions deep

Octal cheat: `r=4, w=2, x=1`. So `755 = rwxr-xr-x`, `644 = rw-r--r--`, `600 = rw-------`.

**Special bits** — high-frequency interview territory:

- **setuid (4xxx)** on executable → process runs as the file's *owner*. Why `passwd` works for non-root users (it needs root to write `/etc/shadow`). Shows as `s` in owner-execute (`S` if execute bit not set).
- **setgid (2xxx)** on file → runs as the file's *group*. On a directory → new files inherit the directory's group (useful for shared workspaces).
- **sticky (1xxx)** on directory → only the file's owner can delete files in it. That's why `/tmp` is `drwxrwxrwt`.

```text
-rwsr-xr-x  /usr/bin/passwd       ← setuid
drwxrwsr-x  /shared               ← setgid on dir
drwxrwxrwt  /tmp                  ← sticky (the t)
```

Lowercase `s/t` means the underlying execute bit *is* set. Uppercase `S/T` means it isn't (usually a bug).

**umask** acts as a filter that *turns off* permissions at file creation (it is a bitwise mask, not simple subtraction). Default `0022` strips write access from group and others → new files get `644` (`rw-r--r--`), dirs `755` (`rwxr-xr-x`). With `umask 0027`, it strips write from group, and all access from others → files get `640` (`rw-r-----`), dirs `750` (`rwxr-x---`).

### 1c. Hard vs symbolic links

Today's depth: **the link count column** (column 2 of `ls -l`).

For files: 1 by default. For directories: at least 2 — the directory itself, plus its own `.` entry. A directory with N subdirectories has link count `2 + N` because each subdir's `..` counts as a link to the parent. **This is the answer to "why does my empty-looking directory have link count 5?"**

`ln` quirks worth internalizing:

- `ln target linkname` — hard link
- `ln -s target linkname` — symlink (the target string is stored *verbatim* — relative paths are resolved relative to the symlink's location, which surprises people)
- `readlink linkname` — show the target string
- `stat linkname` vs `stat -L linkname` — without `-L`, you stat the symlink itself; with `-L`, you follow it

> [!CAUTION]
> **Timestamps trap:**
> - **atime** — last access (read). Often disabled with `noatime` mount option.
> - **mtime** — last modification of *data*. What `ls -l` shows.
> - **ctime** — last change of *inode metadata* (perms, owner, size, link count). **NOT creation time.**

ext4 has `crtime` (creation) in the inode, but POSIX `stat()` doesn't expose it — you need `statx()` or `debugfs -R 'stat <inode>'`.

---

## 💻 Midday Block — Hands-on labs

The goal isn't to type commands. It's to **predict the outcome before pressing <kbd>Enter</kbd>**, then verify.

### Lab 1: Decode `ls -l` field by field

```bash
mkdir ~/day1 && cd ~/day1
touch foo.txt
mkdir bar
ln foo.txt foo-hard.txt
ln -s foo.txt foo-soft.txt
ls -li        # -i adds inode number as first column
```

> [!NOTE]
> **Predict, then check:**
> - Same inode for `foo.txt` and `foo-hard.txt`? (Yes — same inode, that's what hard link means.)
> - Link count on `foo.txt` after the hard link? (2)
> - Inode of the symlink — same as `foo.txt`? (No, symlink has its own inode.)
> - Compare `stat foo-soft.txt` vs `stat -L foo-soft.txt`. The first describes the link itself; the second describes what it points to.

### Lab 2: Permissions & special bits

```bash
ls -l /usr/bin/passwd          # observe the s

# umask experiment
umask 0077
touch ~/day1/private.txt
ls -l ~/day1/private.txt       # what perms? predict first
umask 0022
touch ~/day1/normal.txt
ls -l ~/day1/normal.txt

# Setuid demo (clean up after!)
sudo install -m 4755 /bin/whoami /tmp/myid
/tmp/myid                      # what user is reported?
ls -l /tmp/myid
sudo rm /tmp/myid

# Sticky bit
ls -ld /tmp                    # see the t
mkdir /tmp/scratch && chmod 1777 /tmp/scratch
ls -ld /tmp/scratch
```

> [!WARNING]
> Why doesn't `cp` preserve setuid by default? **Security** — copying a setuid binary owned by root would let anyone create a privilege-escalation tool. Use `cp -p` to preserve mode.

### Lab 3: Links and deletion — including the FD trick

```bash
echo "original content" > original.txt
ln original.txt hard.txt
ln -s original.txt soft.txt

rm original.txt
cat hard.txt    # still works — why?
cat soft.txt    # broken — why?
ls -l soft.txt  # what does it look like?
```

Now for the classic interview scenario. Open two terminals:

```bash
# Terminal 1
echo "data" > big.txt
tail -f big.txt    # leave running, holds an FD

# Terminal 2
rm big.txt
ls big.txt         # gone
df -h .            # space NOT freed
```

> [!TIP]
> **The Interview Answer:**
> When an interviewer says "disk is full but I deleted the big log," the answer is almost always an open File Descriptor (FD) holding the inode. 

Find the leaked space using this command:

```bash
# The canonical flag to find open files with 0 link count (deleted)
sudo lsof +L1          
```

Finally, press <kbd>Ctrl</kbd> + <kbd>C</kbd> in Terminal 1 to kill the tail process. Run `df -h .` again and verify the space is freed!

### Lab 4: Timestamps

```bash
touch t.txt
stat t.txt

cat t.txt > /dev/null    # changes atime (unless mounted noatime)
stat t.txt

echo "x" >> t.txt        # changes mtime AND ctime (size changed = inode changed)
stat t.txt

chmod 600 t.txt          # changes ctime only — metadata, not data
stat t.txt
```

Predict which of atime/mtime/ctime changes before each command.

---

## 🎯 Afternoon Block — Drills + first LP

### Self-check

Answer out loud as if interviewing. Force structure — not "uh, the thing in column…". Answers folded below.

1. Walk through every column of `ls -l /etc/shadow`.
2. Hard link vs symlink — give one case where you *must* use each.
3. Difference between `mtime` and `ctime`?
4. Disk is 100% full but I just deleted a 50 GB log. `df` still shows full. What now?
5. `-rwsr-xr-x root root` on a binary. What's the `s` and what's the security implication?
6. Why might a directory have link count 5 even though I never ran `ln`?
7. What perms does `umask 0027` give to new files and new dirs?
8. Where is a file's creation time on ext4?

<details>
<summary><strong>Answers</strong> (click to reveal)</summary>

1. Type+perms / hard-link count / owner / group / size / mtime / name. Mention special bits if visible.
2. **Symlink required**: across filesystems, or pointing to a directory. **Hard link required**: when the link must survive the original being renamed, moved, or deleted.
3. mtime = file *data* changed. ctime = *inode metadata* changed (perms, owner, size, link count). Editing changes both; `chmod` changes only ctime.
4. Run `sudo lsof +L1` to find the process still holding the unlinked file descriptor. Restart or kill it; space frees when the FD closes.
5. Setuid root. Anyone running it executes as root. If buggy → privilege escalation. Intentional on `passwd`/`sudo`; suspicious on anything user-writable.
6. The directory itself, plus `.`, plus every subdirectory's `..`. Count = `2 + N_subdirs`.
7. Files `640` (rw-r-----), dirs `750` (rwxr-x---).
8. Not in standard `stat`. ext4 stores it as `crtime` but POSIX `stat()` doesn't expose it — use `statx()` or `debugfs -R 'stat <inode>' /dev/sdX`.

</details>

### 🤝 Behavioral — Story #1: Ownership

Today's LP: **Ownership** (most-used LP for SRE/DevOps loops).

Prompt to write a STAR story for:
> "Tell me about a time you took on something outside your job description because it was the right thing for the team or the company."

Write it down (don't memorize — internalize beats):

- **Situation** — 2 sentences max
- **Task** — what *you* specifically chose to take on
- **Action** — bullet what *you* did (not "we")
- **Result** — measurable: numbers, time saved, incidents prevented

Say it out loud. Time it. Target 2–3 minutes. Refine.
