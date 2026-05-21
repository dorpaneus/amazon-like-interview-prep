# 14-Day Curriculum — AWS Systems Engineer Interview Prep

**Interview date:** Tuesday, May 12, 2026
**Start date:** Tuesday, April 28, 2026
**Daily commitment:** ~7 hours every day, including weekends

---

## Daily structure (~7 hours)

| Block | Duration | Focus |
|---|---|---|
| Morning | ~3h | Concepts + reading. New material lands here while you're fresh. |
| Midday | ~2.5h | Hands-on labs. Run commands, break things, observe. |
| Afternoon | ~1.5h | Drills + behavioral. Past-question repetition, one LP/STAR story per day. |

Each topic block follows: **Context → Mental model → Tool walkthrough → Hands-on exercises → Self-check questions** (with folded answers).

---

## Week 1 — Foundations & Core Troubleshooting

| Day | Date | Focus | LP Story |
|---|---|---|---|
| 1 | Linux fundamentals — files, permissions, `ls -l` columns, hard/symlinks, basic commands | Ownership |
| 2 | Processes & `/proc` — `ps`, `top`, signals, where tools get their data | Dive Deep |
| 3 | Filesystems — mount, `fsck`, LVM recovery, LUKS, iSCSI | Insist on the Highest Standards |
| 4 | Boot & systemd — GRUB, services, recovery, regaining root | Bias for Action |
| 5 | Networking I — TCP/IP, DNS, DHCP, "what happens when I load a website" | Customer Obsession |
| 6 | Sun May 3 | Networking II — `tcpdump`, `ss`, `mtr`, firewall, traffic inspection | Earn Trust |
| 7 | Mon May 4 | Scripting & log parsing — Python, regex, Bash idioms (the live-coding question) | Deliver Results |

## Week 2 — Performance, Advanced, Behavioral

| Day | Date | Focus | LP Story |
|---|---|---|---|
| 8 | Tue May 5 | Performance analysis — `vmstat`, `iostat`, `sar`, USE method, PCP | Are Right, A Lot |
| 9 | Wed May 6 | Memory tuning — hugepages, swappiness, NUMA, OOM, overcommit | Learn and Be Curious |
| 10 | Thu May 7 | Disk/network tuning — I/O schedulers, BDP, buffer sizing | Invent and Simplify |
| 11 | Fri May 8 | App debugging — `strace`, `gdb`, `valgrind`, `perf`, eBPF/bcc | Think Big |
| 12 | Sat May 9 | Security & packages — SELinux, PAM, LDAP/Kerberos, dnf/RPM recovery | Have Backbone; Disagree and Commit |
| 13 | Sun May 10 | Behavioral intensive — STAR for remaining LPs, mock loop | (covers Frugality, Hire/Develop, Earth's Best Employer, Success and Scale) |
| 14 | Mon May 11 | Light review + rest — drills, no new material, sleep early | Final mock + rest |

---

## Topic coverage map

How the 14 days cover each section of the prep doc:

| Reference doc section | Days covered |
|---|---|
| Past interview questions (`ls -l`, slow app, DNS, DHCP, log parsing, perf debugging, `top` data source, symlinks) | 1, 2, 5, 6, 7, 8, 11 |
| RHCA Troubleshooting — methodology, system info | 2, 8 |
| RHCA Troubleshooting — boot & startup | 4 |
| RHCA Troubleshooting — hardware, kernel modules | 9, 11 |
| RHCA Troubleshooting — filesystems, LVM, LUKS, iSCSI | 3 |
| RHCA Troubleshooting — package management, RPM | 12 |
| RHCA Troubleshooting — network connectivity | 5, 6 |
| RHCA Troubleshooting — application debugging | 7, 11 |
| RHCA Troubleshooting — SELinux, PAM, LDAP, Kerberos | 12 |
| RHCA Troubleshooting — kdump, SystemTap, sosreport | 11, 12 |
| RHCA Performance — analysis tools (vmstat, iostat, sar, PCP) | 8 |
| RHCA Performance — kernel tuning (sysctl, /proc/sys) | 9, 10 |
| RHCA Performance — eBPF, SystemTap, perf | 11 |
| RHCA Performance — process priorities, tuned, cgroups | 11 |
| RHCA Performance — memory (hugepages, swap, NUMA) | 9 |
| RHCA Performance — disk/FS tuning (schedulers, dirty pages) | 10 |
| RHCA Performance — network tuning (BDP, buffers) | 10 |
| Amazon Leadership Principles (16) — STAR stories | 1–13 (one/day, intensive on 13) |

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
- [ ] LVM recovery procedure (vgcfgrestore) memorized
- [ ] BPF filter syntax for the common cases (`tcp[tcpflags]`, `not port 22`, etc.)
- [ ] Live-coding log-parsing solution rehearsed end-to-end
