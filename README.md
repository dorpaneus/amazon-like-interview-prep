# 🚀 14-Day Curriculum — AWS Systems Engineer Interview Prep

> [!NOTE]
> **Target Roles:** Systems Engineer / DevOps Engineer (Managed Operations) / Systems Dev Engineer II
> **Daily Commitment:** ~3 hours every day, including weekends
> **Progress:** Days 1–11 written · Days 12–14 pending

This repository contains everything required to pass a heavy infrastructure and behavioral interview loop. It is designed to be completely self-contained—no endless Googling required.

---

## ⏱️ Daily Structure (~3 hours)

| Block | Focus |
| :--- | :--- |
| **🌅 Morning** | **Concepts + Reading.** New material lands here while you're fresh. |
| **💻 Midday** | **Hands-on Labs.** Run commands, break things, observe. |
| **🎯 Afternoon** | **Drills + Behavioral.** Past-question repetition, one LP/STAR story per day. |

> [!TIP]
> **Standard Topic Flow:** 
> Context → Mental model → Tool walkthrough → Hands-on exercises → Self-check questions (with folded answers).

---

## 🗓️ Week 1 — Foundations & Core Troubleshooting

| Day | Topic | AWS Context | LP Story |
| :--- | :--- | :--- | :--- |
| 1 | [Linux fundamentals — files, permissions, hard/symlinks](14-days/day-01-linux-fundamentals.md) | IAM, S3 objects vs POSIX | **Ownership** |
| 2 | [Processes & `/proc` — `ps`, `top`, signals, zombie processes](14-days/day-02-processes-proc-signals.md) | EC2 instance state, ECS | **Dive Deep** |
| 3 | [Filesystems — mount, `fsck`, LVM recovery, LUKS, iSCSI](14-days/day-03-filesystems-lvm-luks-iscsi.md) | EBS volumes, EFS, Instance Store | **Insist on the Highest Standards** |
| 4 | [Boot & systemd — GRUB, services, recovery, regaining root](14-days/day-04-boot-grub-systemd.md) | EC2 Rescue, AMI lifecycle | **Bias for Action** |
| 5 | [Networking I — TCP/IP, DNS, DHCP, "loading a website"](14-days/day-05-networking-tcp-dns-dhcp.md) | Route53, VPC subnets, IGW | **Customer Obsession** |
| 6 | [Networking II — `tcpdump`, `ss`, `mtr`, firewalls, NAT](14-days/day-06-networking-tcpdump-firewalls-nat.md) | Security Groups, NACLs, NAT GW | **Earn Trust** |
| 7 | [Scripting & log parsing — Python, regex, Bash idioms](14-days/day-07-scripting-log-parsing.md) | CloudWatch Logs, Lambda | **Deliver Results** |

## 🗓️ Week 2 — Performance, Advanced, Behavioral

| Day | Topic | AWS Context | LP Story |
| :--- | :--- | :--- | :--- |
| 8 | [Performance analysis — `vmstat`, `iostat`, `sar`, USE method](14-days/day-08-performance-analysis-use-method.md) | CloudWatch metrics, EC2 sizing | **Are Right, A Lot** |
| 9 | [Memory tuning — hugepages, swappiness, NUMA, OOM](14-days/day-09-memory-tuning.md) | Memory-optimized instances | **Learn and Be Curious** |
| 10 | [Disk/network tuning — I/O schedulers, BDP, buffer sizing](14-days/day-10-disk-network-tuning.md) | EBS IOPS, Nitro Enclaves | **Invent and Simplify** |
| 11 | [App debugging — `strace`, `gdb`, `valgrind`, `perf`, eBPF](14-days/day-11-app-debugging.md) | X-Ray, application-level traces | **Think Big** |
| 12 | [Security & packages — SELinux, PAM, LDAP/Kerberos](14-days/day-12-security-package-management.md) | IAM Identity Center, Systems Manager | **Have Backbone; Disagree & Commit** |
| 13 | [Behavioral intensive — STAR for remaining LPs, mock loop](14-days/day-13-behavioral-intensive.md) | Amazon Culture fit | **Frugality, Hire/Develop, Success & Scale** |
| 14 | [Light review + rest — drills, no new material, sleep early](14-days/day-14-light-review.md) | Final prep | *(Final mock + rest)* |

---

## 📂 Repository Layout

```text
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
    ├── day-11-app-debugging.md
    ├── day-12-security-package-management.md
    ├── day-13-behavioral-intensive.md
    └── day-14-light-review.md
```

---

## 📚 Story Bank Target — Leadership Principles

By Day 13 you should have **~12–15 stories** tagged to LPs, with at least one strong story per LP. 

> [!WARNING]
> **The most heavily-weighted LPs for SysEng/DevOps Managed Operations:**
> Focus your strongest, most detailed technical stories here.
> 
> *   **Ownership** (on-call, "your service is your problem")
> *   **Dive Deep** (root cause analysis, complex debugging)
> *   **Insist on the Highest Standards** (operational excellence, automation)
> *   **Bias for Action** (incident response, moving fast safely)
> *   **Earn Trust** (blameless postmortems, cross-team work)
> *   **Customer Obsession** (uptime and reliability as a customer-facing metric)

---

## ✅ Pre-loop Checklist (Day 14 Review)

- [ ] Story bank covering all 16 LPs, each rehearsed out loud in STAR format.
- [ ] At least one **incident story** walked through end-to-end (timeline, mitigation, root cause, action items).
- [ ] One **mentoring / cross-team leadership** story.
- [ ] One **"disagreed with a decision"** story.
- [ ] One **"data contradicted intuition"** story.
- [ ] Refreshed AWS basics (EC2, S3, IAM, VPC) — interviewers will frame Linux questions inside AWS scenarios.
- [ ] 2–3 thoughtful questions prepared for *each* interviewer (specific to their scope, not generic).
- [ ] "What happens when I load a website?" walked end-to-end without notes.
- [ ] Brendan Gregg's **USE method** opener memorized.
- [ ] `rd.break` root recovery procedure memorized.
- [ ] LVM recovery procedure (`vgcfgrestore`) memorized.
- [ ] BPF/tcpdump filter syntax for common cases (`tcp[tcpflags]`, `not port 22`, etc.).
- [ ] Live-coding log-parsing solution rehearsed end-to-end (Python or Bash).
