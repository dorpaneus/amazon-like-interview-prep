# 14-Day Curriculum — AWS Systems Engineer Interview Prep

**Daily commitment:** ~3 hours every day, including weekends
**Progress:** Days 1–11 written · Days 12–14 pending

---

## Daily structure (~3 hours)

| Block | Duration | Focus |
|---|---|---|
| Morning | ~1h | Concepts + reading. New material lands here while you're fresh. |
| Midday | ~1.5h | Hands-on labs. Run commands, break things, observe. |
| Afternoon | ~0.5h | Drills + behavioral. Past-question repetition, one LP/STAR story per day. |

Each topic block follows: **Context → Mental model → Tool walkthrough → Hands-on exercises → Self-check questions** (with folded answers).

---

## Week 1 — Foundations & Core Troubleshooting

| Day | Topic | LP Story |
|---|---|---|
| 1 | [Linux fundamentals — files, permissions, `ls -l` columns, hard/symlinks, basic commands](14-days/day-01-linux-fundamentals.md) | Ownership |
| 2 | [Processes & `/proc` — `ps`, `top`, signals, where tools get their data](14-days/day-02-processes-proc-signals.md) | Dive Deep |
| 3 | [Filesystems — mount, `fsck`, LVM recovery, LUKS, iSCSI](14-days/day-03-filesystems-lvm-luks-iscsi.md) | Insist on the Highest Standards |
| 4 | [Boot & systemd — GRUB, services, recovery, regaining root](14-days/day-04-boot-grub-systemd.md) | Bias for Action |
| 5 | [Networking I — TCP/IP, DNS, DHCP, "what happens when I load a website"](14-days/day-05-networking-tcp-dns-dhcp.md) | Customer Obsession |
| 6 | [Networking II — `tcpdump`, `ss`, `mtr`, firewall, traffic inspection](14-days/day-06-networking-tcpdump-firewalls-nat.md) | Earn Trust |
| 7 | [Scripting & log parsing — Python, regex, Bash idioms (the live-coding question)](14-days/day-07-scripting-log-parsing.md) | Deliver Results |

## Week 2 — Performance, Advanced, Behavioral

| Day | Topic | LP Story |
|---|---|---|
| 8 | [Performance analysis — `vmstat`, `iostat`, `sar`, USE method, PCP](14-days/day-08-performance-analysis-use-method.md) | Are Right, A Lot |
| 9 | [Memory tuning — hugepages, swappiness, NUMA, OOM, overcommit](14-days/day-09-memory-tuning.md) | Learn and Be Curious |
| 10 | [Disk/network tuning — I/O schedulers, BDP, buffer sizing](14-days/day-10-disk-network-tuning.md) | Invent and Simplify |
| 11 | [App debugging — `strace`, `gdb`, `valgrind`, `perf`, eBPF/bcc](14-days/day-11-app-debugging.md) | Think Big |
| 12 | [Security & packages — SELinux, PAM, LDAP/Kerberos, dnf/RPM recovery](14-days/day-12-app-debugging.md) | Have Backbone; Disagree and Commit |
| 13 | [Behavioral intensive — STAR for remaining LPs, mock loop](14-days/day-13-app-debugging.md) | Frugality, Hire/Develop, Earth's Best Employer, Success and Scale |
| 14 | [Light review + rest — drills, no new material, sleep early]((14-days/day-14-app-debugging.md))  | Final mock + rest |

---

## Repository layout

```
.
├── README.md                # this file
├── LICENSE
└── 14-days/
    ├── day-01-linux-fundamentals.md
    ├── day-02-processes-proc-signals.md
    ├── day-03-filesystems-lvm-luks-iscsi.md
    ├── day-04-boot-grub-systemd.md
    ├── day-05-networking-tcp-dns-dhcp.md
    ├── day-06-networking-tcpdump-firewalls-nat.md
    ├── day-07-scripting-log-parsing.md
    ├── day-08-performance-analysis-use-method.md
    ├── day-09-memory-tuning.md
    ├── day-10-disk-network-tuning.md
    └── day-11-app-debugging.md
```

---

## Story bank target — Leadership Principles

By Day 13 you should have ~12–15 stories tagged to LPs, with at least one strong story per LP. Distribution across the curriculum:

| Day | LP introduced | Cumulative coverage |
|---|---|---|
| 1 | Ownership | 1/16 |
| 2 | Dive Deep | 2/16 |
| 3 | Insist on the Highest Standards | 3/16 |
| 4 | Bias for Action | 4/16 |
| 5 | Customer Obsession | 5/16 |
| 6 | Earn Trust | 6/16 |
| 7 | Deliver Results | 7/16 |
| 8 | Are Right, A Lot | 8/16 |
| 9 | Learn and Be Curious | 9/16 |
| 10 | Invent and Simplify | 10/16 |
| 11 | Think Big | 11/16 |
| 12 | Have Backbone; Disagree and Commit | 12/16 |
| 13 | Frugality + Hire/Develop + Earth's Best Employer + Success and Scale (intensive day) | 16/16 |
| 14 | (Review only — no new stories) | 16/16 |

**The most heavily-weighted LPs for SysEng/DevOps Managed Operations** (focus the strongest stories here):

- Ownership (on-call, "your service is your problem")
- Dive Deep (root cause analysis, debugging)
- Insist on the Highest Standards (operational excellence)
- Bias for Action (incident response)
- Earn Trust (blameless postmortems, cross-team work)
- Customer Obsession (uptime as a customer-facing metric)

---

## Pre-loop checklist (Day 14 review)

- [ ] Story bank covering all 16 LPs, each rehearsed in STAR format
- [ ] At least one incident story walked through end-to-end (timeline, mitigation, root cause, action items)
- [ ] One mentoring / cross-team leadership story
- [ ] One "disagreed with a decision" story
- [ ] One "data contradicted intuition" story
- [ ] Refreshed AWS basics (EC2, S3, IAM, VPC) — interviewer scenarios assume them
- [ ] 2–3 thoughtful questions prepared for each interviewer (specific to their work, not generic)
- [ ] "What happens when I load a website" walked end-to-end without notes
- [ ] USE method opener memorized
- [ ] `rd.break` root recovery procedure memorized
- [ ] LVM recovery procedure (`vgcfgrestore`) memorized
- [ ] BPF filter syntax for the common cases (`tcp[tcpflags]`, `not port 22`, etc.)
- [ ] Live-coding log-parsing solution rehearsed end-to-end
