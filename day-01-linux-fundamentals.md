# Day 1 — Linux Fundamentals

The interview will pull on these threads constantly: "disk full but `df` says space available" → inode exhaustion. "I deleted the file but disk is still full" → open FD holding the inode. "Hardlink survives the original being deleted, why?" → because the original was never special. Today builds the mental model that makes those answers automatic.

---

## Morning Block (3h) — The Unix file model

### 1A. Everything is a file (45 min)

Mental model: a Linux file is **two things glued together**:

1. An **inode** — metadata + pointers to data blocks (owner, perms, size, timestamps, link count, block list).
2. A **directory entry** — a `(name, inode_number)` pair living inside a directory.

The "file" you see is just a name. The actual file is the inode. This is why:

- A hard link is just another directory entry pointing to the same inode — no data copy.
- `rm` doesn't delete data — it removes one directory entry and decrements link count; data is freed only when count hits 0 *and* no process has the file open.
- You can't hard-link across filesystems — inode numbers are filesystem-local.

**File types** (first char in `ls -l`):

| Char | Type |
|---|---|
| `-` | regular file |
| `d` | directory |
| `l` | symbolic link |
| `b` | block device |
| `c` | character device |
| `p` | named pipe (FIFO) |
| `s` | socket |

### 1B. Permissions deep (1h 15min)

Octal cheat: `r=4, w=2, x=1`. So `755 = rwxr-xr-x`, `644 = rw-r--r--`, `600 = rw-------`.

**Special bits** — high-frequency interview territory:

- **setuid (4xxx)** on executable → process runs as the file's *owner*. Why `passwd` works for non-root users (it needs root to write `/etc/shadow`). Shows as `s` in owner-execute (`S` if execute bit not set).
- **setgid (2xxx)** on file → runs as the file's *group*. On a directory → new files inherit the directory's group (useful for shared workspaces).
- **sticky (1xxx)** on directory → only the file's owner can delete files in it. That's why `/tmp` is `drwxrwxrwt`.

```
-rwsr-xr-x  /usr/bin/passwd       ← setuid
drwxrwsr-x  /shared               ← setgid on dir
drwxrwxrwt  /tmp                  ← sticky (the t)
```

Lowercase `s/t` means the underlying execute bit *is* set. Uppercase `S/T` means it isn't (usually a bug).

**umask** is subtractive at file creation. Default `0022` → files `666-022=644`, dirs `777-022=755`. With `umask 0027`: files `640`, dirs `750`.

### 1C. Hard vs symbolic links (1h)

Today's depth: **the link count column** (column 2 of `ls -l`).

For files: 1 by default. For directories: at least 2 — the directory itself, plus its own `.` entry. A directory with N subdirectories has link count `2 + N` because each subdir's `..` counts as a link to the parent. **This is the answer to "why does my empty-looking directory have link count 5?"**

`ln` quirks worth internalizing:

- `ln target linkname` — hard link
- `ln -s target linkname` — symlink (the target string is stored *verbatim* — relative paths are resolved relative to the symlink's location, which surprises people)
- `readlink linkname` — show the target string
- `stat linkname` vs `stat -L linkname` — without `-L`, you stat the symlink itself; with `-L`, you follow it

**Timestamps trap**:

- **atime** — last access (read). Often disabled with `noatime` mount option.
- **mtime** — last modification of *data*. What `ls -l` shows.
- **ctime** — last change of *inode metadata* (perms, owner, size, link count). **NOT creation time.**

ext4 has `crtime` (creation) in the inode, but POSIX `stat()` doesn't expose it — you need `statx()` or `debugfs -R 'stat <inode>'`.

---

## Midday Block (2.5h) — Hands-on labs

The goal isn't to type commands. It's to **predict the outcome before pressing Enter**, then verify.

### Lab 1: Decode `ls -l` field by field (30 min)

```bash
mkdir ~/day1 && cd ~/day1
touch foo.txt
mkdir bar
ln foo.txt foo-hard.txt
ln -s foo.txt foo-soft.txt
ls -li        # -i adds inode number as first column
```

**Predict, then check**:

- Same inode for `foo.txt` and `foo-hard.txt`? (Yes — same inode, that's what hard link means.)
- Link count on `foo.txt` after the hard link? (2)
- Inode of the symlink — same as `foo.txt`? (No, symlink has its own inode.)
- Compare `stat foo-soft.txt` vs `stat -L foo-soft.txt`. The first describes the link itself; the second describes what it points to.

### Lab 2: Permissions & special bits (45 min)

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

Why doesn't `cp` preserve setuid by default? Security — copying a setuid binary owned by root would let anyone create a privilege-escalation tool. Use `cp -p` to preserve mode.

### Lab 3: Links and deletion — including the FD trick (45 min)

```bash
echo "original content" > original.txt
ln original.txt hard.txt
ln -s original.txt soft.txt

rm original.txt
cat hard.txt    # still works — why?
cat soft.txt    # broken — why?
ls -l soft.txt  # what does it look like?
```

Now the **"disk full but I deleted the file"** classic. Open two terminals:

```bash
# Terminal 1
echo "data" > big.txt
tail -f big.txt    # leave running, holds an FD

# Terminal 2
rm big.txt
ls big.txt         # gone
df -h .            # space NOT freed
lsof | grep big.txt    # still held open!
sudo lsof | grep deleted   # the canonical "find leaked space" command
# Now Ctrl-C the tail in Terminal 1
df -h .            # space freed
```

This is THE answer when an interviewer says "disk is full but I deleted the big log, `df` still shows full." Look for `lsof | grep deleted`, find the process, restart or kill it.

### Lab 4: Timestamps (30 min)

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

## Afternoon Block (1.5h) — Drills + first LP

### Self-check (45 min)

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
<summary>Answers</summary>

1. Type+perms / hard-link count / owner / group / size / mtime / name. Mention special bits if visible.
2. **Symlink required**: across filesystems, or pointing to a directory. **Hard link required**: when the link must survive the original being renamed, moved, or deleted.
3. mtime = file *data* changed. ctime = *inode metadata* changed (perms, owner, size, link count). Editing changes both; `chmod` changes only ctime.
4. `sudo lsof | grep deleted` to find the process still holding the FD. Restart or kill it; space frees when the FD closes.
5. Setuid root. Anyone running it executes as root. If buggy → privilege escalation. Intentional on `passwd`/`sudo`; suspicious on anything user-writable.
6. The directory itself, plus `.`, plus every subdirectory's `..`. Count = `2 + N_subdirs`.
7. Files `640` (rw-r-----), dirs `750` (rwxr-x---).
8. Not in standard `stat`. ext4 stores it as `crtime` but POSIX `stat()` doesn't expose it — use `statx()` or `debugfs -R 'stat <inode>' /dev/sdX`.

</details>

### Behavioral (45 min) — Story #1: Ownership

Today's LP: **Ownership** (most-used LP for SRE/DevOps loops).

Prompt to write a STAR story for:

> "Tell me about a time you took on something outside your job description because it was the right thing for the team or the company."

Write it down (don't memorize — internalize beats):

- **Situation** — 2 sentences max
- **Task** — what *you* specifically chose to take on
- **Action** — bullet what *you* did (not "we")
- **Result** — measurable: numbers, time saved, incidents prevented

Say it out loud. Time it. Target 2–3 minutes. Refine.
