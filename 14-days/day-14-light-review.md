# Day 14 — Light Review & Rest (Mon May 11)

Today's job is **not to learn anything new**. Two solid weeks of work are behind you. Today is consolidation, logistics, and rest. The single biggest mistake at this stage is over-rehearsing into staleness.

The whole day should be **~3 hours of low-intensity activity**, then stop. Trust the prep.

---

## Morning — low-intensity review (90–120 min)

Pick a quiet spot. Coffee. No labs, no new material, no terminals.

### Quick re-reads (45 min)

Skim, don't study. You're refreshing, not learning:

- **The job description** — one read-through. Notice the language they use ("Managed Operations," "European Sovereign Cloud," "24/7 on-call"). Use the same vocabulary in the interview when natural.
- **The 16 Leadership Principles** — one read-through. You're recalibrating, not memorizing.
- **Your story-bank matrix** — your map of which stories tag to which LPs. This is the most useful thing to have fresh.
- **Your two weakest stories from yesterday's mock** — read them once, then recite each one out loud once. Don't perfect them.

### Headline recitals (45 min)

These are the "memorized openers" — the moments where having a smooth, prepared answer reads as senior. Practice each one **once** out loud, slowly. Not for memorization — for warm-up.

**"What happens when I load a website?"**
URL parse → DNS resolution (browser → OS → resolver → recursive walk) → ARP for next hop → TCP three-way handshake → TLS handshake with SNI and cert validation → HTTP request → server processing → response → browser parses, fetches subresources, renders.

**The USE method opener** for any performance question:
"I'd start with the USE method — Utilization, Saturation, Errors for each resource. Concretely, `vmstat 1`, `mpstat -P ALL 1`, `iostat -xz 1`, `sar -n DEV 1`, and `free -m` as my first sweep."

**The hung-process answer**:
"`sudo strace -p <pid>`. The syscall it's blocked in tells me what it's waiting on. `futex_wait` → userspace lock contention. `read` on a socket → I/O / peer issue. `poll`/`epoll_wait` → idle event loop. If no syscall activity, it's spinning in userspace — switch to `perf top -p` or `gdb -p` with `bt`."

**The OOM answer**:
"`dmesg -T | grep -i oom-kill` for the kernel log — it prints which process was killed, current meminfo, and the process list. Investigate: is RSS legitimate or a leak? Is the system right-sized? If essential, raise instance size or use cgroup limits. If a leak, fix the app. Protect critical processes with `oom_score_adj=-1000`."

**The `rd.break` root recovery**:
Reboot → GRUB → press `e` → append `rd.break enforcing=0` → Ctrl-x → in initramfs shell: `mount -o remount,rw /sysroot` → `chroot /sysroot` → `passwd root` → `touch /.autorelabel` → exit, exit. SELinux relabels on next boot.

**The LVM recovery**:
"`vgcfgbackup` archives are in `/etc/lvm/archive/`. Recovery: `pvcreate --uuid <original-UUID> --restorefile <archive> /dev/sdX`, then `vgcfgrestore <vg>`, then `vgchange -ay <vg>`."

**The RPM database recovery**:
"Back up `/var/lib/rpm`, remove `__db.*` indexes, `rpm --rebuilddb`. Then `dnf check` to verify consistency."

**The BDP calculation**:
"BDP = bandwidth × RTT. For 10 Gbps × 100 ms: 10⁹ × 0.1 / 8 = 125 MB. So I'd set `net.core.rmem_max`/`wmem_max` to at least 128 MB, the `tcp_rmem`/`tcp_wmem` max values to match, and verify window scaling is on."

That's it. Don't add more. **Don't drill these to perfection** — they should sound conversational, not memorized.

### Optional: one short STAR re-read (15 min)

If a specific behavioral story still feels uneven, read it once, then recite once. Then stop touching it. Resist the urge to keep tweaking — anxiety likes to disguise itself as preparation.

---

## Midday — logistics & setup (45 min)

This is the part of the day where you actually accomplish something tangible. Boring but important.

### Interview logistics

- [ ] Time confirmed — including time zone, especially if remote with interviewers across regions
- [ ] Link tested (Zoom / Chime / whatever they're using) — open it once, verify it loads
- [ ] Calendar invites accepted; reminders set
- [ ] Names of interviewers if you have them — quick LinkedIn glance for context (don't memorize bios)
- [ ] Recruiter's phone number saved — in case of tech issues

### If remote (most likely)

- [ ] Audio: headphones tested, mic tested, no echo
- [ ] Video: camera angle decent, eye-level if possible, well-lit (face toward window or lamp)
- [ ] Background: tidy or use a neutral virtual background — but if you use one, test it doesn't ghost your edges
- [ ] Connection: wired ethernet if possible, otherwise verify wifi strength
- [ ] Phone: silenced, but reachable for the recruiter
- [ ] Browser: close everything not needed; pin only the interview tab; clear notifications
- [ ] Power: laptop plugged in, full battery
- [ ] Water bottle: filled, lid on, within reach
- [ ] Notepad and pen: physical, not on the screen — for jotting interviewer names or numbers
- [ ] Snack within reach in case of breaks

### If onsite

- [ ] Route checked, transit time + buffer
- [ ] What to wear laid out (business casual is the safe default for AWS — neat but not formal)
- [ ] ID / badge if needed for office access
- [ ] Printed copy of your resume (some interviewers like a hard copy reference)
- [ ] Water bottle, snacks in bag
- [ ] Phone charged

### One last review

- Glance at your **story-bank matrix** for 5 minutes. Just the map, not the stories themselves.
- Confirm you can name your 2-3 "headline" stories without looking — the ones you'd lead with for Ownership, Dive Deep, and Customer Obsession.

That's it. **Close the laptop.**

---

## Afternoon & evening — actively stop

This is the hardest part for most prepared candidates. Doing more does not help now. **Recovery does.**

- **Take a walk.** Outside. 30–60 minutes. Light cardio resets the brain.
- **Eat a normal dinner** at your normal time. Nothing new, nothing heavy.
- **No technical content after dinner.** No skimming, no "just one more re-read."
- **Light entertainment** is fine — a familiar show, a book you're already in, music. Avoid anything emotionally heavy.
- **Sleep at your usual time.** Not dramatically early (causes a wake-up at 2am that ruins everything). Not late (obvious).
- **If you can't sleep**: don't panic. Lie still in the dark. Resting horizontally with eyes closed is ~80% of the benefit of sleep. Don't pick up the phone.

---

## Tuesday morning (interview day)

### The morning ritual

- **Wake up at your normal time.** Not extra early.
- **Eat the breakfast you normally eat.** Not the morning to try anything new.
- **Coffee at normal strength.** Not extra to "boost" — extra caffeine before a high-stakes conversation reliably backfires.
- **Light movement** — a walk, a stretch. Get blood flowing.
- **Shower.** Mental reset.
- **Dress** — get it done early so you're not rushing.

### The 30 minutes before

- Re-read the JD one final time. Quick.
- Glance at your story-bank matrix. Quick.
- **Then close everything.**
- 5–10 minutes of quiet. Slow breathing. Don't run stories in your head.
- Water. Bathroom. Adjust your camera.
- Open the meeting link 2 minutes before the start. Not earlier.

### Inside the interview

A short list to keep in your head — not to recite, just to hold:

1. **Smile in the first 5 seconds.** Sets the tone for both of you.
2. **Their name once at the start.** ("Nice to meet you, Sarah.") Helps you remember it, signals warmth.
3. **Use "I" not "we"** when describing actions.
4. **It's OK to pause.** "Let me think for a moment" beats rambling.
5. **It's OK to say "I don't know"** — when followed by "here's how I'd find out."
6. **Numbers in your results.** Specifics over adjectives.
7. **Your questions at the end** — rotate from your prepared list. Different question for each interviewer.
8. **Reset between interviews.** Don't replay the last one. Water, bathroom, breath.

---

## The panic checklist

If you wake up Tuesday morning and feel suddenly unprepared on some specific thing — *do not* try to learn it. Instead:

- **Topic feels fuzzy?** That's normal under stress. You know more than your panic suggests. If it comes up, walk through it slowly out loud; the structure will return.
- **Story feels weak?** Pick a different story for that LP — that's why the matrix exists.
- **Can't recall a specific command?** "I don't remember the exact flag — here's how I'd find it: `man <tool>`, or the `--help` output, or check `/proc/sys/...`. The capability I'd be looking for is X." That answer is fine. Better than guessing.
- **Going completely blank in an interview?** Say it directly: "Let me think about this for a moment." Take a breath. Restart with whatever you do remember. Interviewers see this all the time — it's not the catastrophe it feels like.

---

## Final note

You've done two full weeks of focused, structured prep. Your story bank is built. You have memorized openers for every high-frequency interview scenario. You understand the storage stack, the boot chain, TCP/IP, the USE method, eBPF tooling, SELinux, and how to recover an RPM database. You've practiced the live-coding scenario.

This is a strong position. The remaining gap between today and Tuesday is **rest**, not study.

The interview is a conversation, not a test. Treat the interviewers like the colleagues they're trying to decide whether you should become.

Good luck Tuesday.
