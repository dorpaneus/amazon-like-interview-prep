# Day 7 — Scripting and Log Parsing (Mon May 4)

End of week one. Today is the **live-coding day**. The reported question — "extract IPs and error codes from log files" — is the most likely live-coding scenario for this role. We'll write that code, but the real skill is **what you say while you type**: narrating tradeoffs, asking clarifying questions, recognizing when the toy answer is wrong for production scale.

The interviewer is watching for: Python or Bash fluency? Do you stream or load the whole file? Do you know what regex you wrote? Do you handle malformed input? Do you know when to reach for `awk`, when to reach for Python, and when neither is the right answer?

---

## Morning Block (3h) — Tools and idioms

### 7A. Choosing the right tool (30 min)

The first thing to **say out loud** in a live-coding session: "What scale are we at?" The answer drives the tool.

| Scale | Tool |
|---|---|
| One file, a few MB, one-off | `grep`/`awk`/`sort` one-liner |
| One file, GBs, recurring | Python streaming, or `awk` |
| Many files across hosts, recurring | Centralized logging (CloudWatch Insights, Athena, Elasticsearch) |
| Real-time analysis | Stream processor (Kinesis, Kafka + consumers), or eBPF for live system data |

**The interview-winning answer** to the log-parsing question is *not* the cleverest regex. It's:

1. Ask the scale and frequency.
2. Solve the toy version cleanly.
3. **Note that at production scale you wouldn't grep files** — logs would already be in CloudWatch Logs Insights / Athena / Elasticsearch, queryable by structured fields. The grep-the-file solution is for ad-hoc debugging on one box.

That sentence alone separates you from candidates who only show off regex.

### 7B. Bash hygiene (45 min)

Bash is everywhere in ops. Bad Bash silently corrupts things. **Five rules** to internalize:

**1. Always set strict mode**:

```bash
#!/bin/bash
set -euo pipefail
IFS=$'\n\t'
```

- `set -e` — exit on any command failure
- `set -u` — error on unset variables (`$undefined` becomes a hard error)
- `set -o pipefail` — a pipeline fails if *any* command in it fails (default is "only the last")
- `IFS=$'\n\t'` — field splitting only on newlines and tabs, not spaces (avoids the classic spaces-in-filenames bug)

**2. Quote all variable expansions**:

```bash
# Wrong — breaks on spaces, glob chars
rm $file

# Right — survives any filename
rm "$file"
```

**3. Use `[[` not `[`**:

```bash
[[ -f "$file" && -r "$file" ]]    # safer, supports && and ||
[ -f "$file" -a -r "$file" ]       # POSIX, brittle, deprecated
```

**4. `mktemp` for temp files**:

```bash
tmp=$(mktemp)
trap 'rm -f "$tmp"' EXIT             # clean up even on failure
```

Don't use `/tmp/foo.$$` — predictable, race-prone, and not cleaned up on failure.

**5. `command -v` to test for a binary**:

```bash
if ! command -v jq >/dev/null 2>&1; then
    echo "jq required" >&2
    exit 1
fi
```

**Useful built-ins worth knowing**:

- `${var:-default}` — use default if unset/empty
- `${var:?error}` — error if unset/empty (great for required env vars)
- `${var%suffix}` / `${var#prefix}` — strip suffix/prefix
- `${var//old/new}` — replace all
- `(( arithmetic ))` — integer math, no `$` needed
- `<()` and `>()` — process substitution; e.g. `diff <(cmd1) <(cmd2)`

### 7C. The text toolkit — `grep`, `awk`, `sed`, `cut`, `sort`, `uniq` (45 min)

You should be able to write these one-liners cold.

```bash
# Lines matching a pattern
grep ERROR /var/log/app.log
grep -i error file.log              # case-insensitive
grep -v INFO file.log               # invert (exclude)
grep -c ERROR file.log              # just count
grep -E 'ERROR|WARN' file.log       # extended regex (ERE) for alternation
grep -P '\d{3}' file.log            # PCRE — for \d, lookarounds, etc

# 5 lines of context around each match
grep -B 2 -A 2 ERROR file.log       # before / after
grep -C 3 ERROR file.log            # both, 3 lines

# Files matching, recursively, only filenames
grep -rln ERROR /var/log/

# Whole-word, line numbers
grep -wn '^bash$' /etc/shells
```

**`awk`** — column-oriented; the right tool when fields matter:

```bash
# Print column 7 (URL in default Apache log format)
awk '{print $7}' access.log

# Filter then project
awk '$9 == 500 {print $1, $7}' access.log

# Sum a column
awk '{sum += $10} END {print sum}' access.log

# Count occurrences (the awk version of sort | uniq -c)
awk '{count[$1]++} END {for (ip in count) print count[ip], ip}' access.log | sort -rn | head

# Custom field separator
awk -F: '{print $1}' /etc/passwd                  # all usernames
```

**`sed`** — stream editor, mostly for find-and-replace:

```bash
sed 's/old/new/'        file        # replace first occurrence per line
sed 's/old/new/g'       file        # replace all
sed -i 's/old/new/g'    file        # in-place edit
sed -n '10,20p'         file        # print lines 10-20 only
sed '/^#/d'             file        # delete lines starting with #
```

**The classic count-and-rank pipeline**:

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
# top 10 IPs by request count

# Equivalent in pure awk:
awk '{c[$1]++} END {for (k in c) print c[k], k}' access.log | sort -rn | head
```

### 7D. Python — patterns for live-coding (1h)

**The skeleton** for any "process a log file" question:

```python
#!/usr/bin/env python3
import re
import sys
from collections import Counter

def main(path):
    ip_re = re.compile(r'\b(?:\d{1,3}\.){3}\d{1,3}\b')
    err_re = re.compile(r'\b(?:[45]\d{2})\b')

    ips, codes = Counter(), Counter()

    with open(path) as f:
        for line in f:                    # streaming, line by line
            for ip in ip_re.findall(line):
                ips[ip] += 1
            for code in err_re.findall(line):
                codes[code] += 1

    print("Top 10 IPs:")
    for ip, count in ips.most_common(10):
        print(f"  {count:6d}  {ip}")

    print("\nError codes:")
    for code, count in codes.most_common():
        print(f"  {count:6d}  {code}")

if __name__ == '__main__':
    if len(sys.argv) != 2:
        sys.exit(f"usage: {sys.argv[0]} <logfile>")
    main(sys.argv[1])
```

**Things to say out loud while writing it**:

- *"I'm streaming line-by-line so this works on multi-GB files without loading everything."*
- *"I'm using `Counter` because counting is the obvious data structure."*
- *"This regex catches anything that *looks* like an IPv4. `0.0.0.0` and `999.999.999.999` would both match — I'd validate with `ipaddress.ip_address(...)` if accuracy mattered."*
- *"The 4xx/5xx regex is naive — it'd match `400` inside a port number or content length. In real log parsing I'd anchor to the position where the status appears, e.g. by parsing the log format properly."*

**The thing that separates intermediate from senior** — when the interviewer says "now make it 100x faster" or "this file is 50 GB":

- **Compile the regex once** (already done above).
- **Pre-filter with `'in'` before regex** — if `'500' not in line` you can skip the regex.
- **`mmap` for very large files** — but lose line orientation; usually not worth the complexity.
- **Parallelize** — split file into chunks, process with `multiprocessing.Pool`, merge counters.
- **Or: stop. The right answer is "this shouldn't be a Python script. This is a CloudWatch Logs Insights query / Athena query / Elasticsearch search."**

**Apache/Nginx log parsing — the right way**:

```python
import re
# Combined Log Format:
# host ident user [time] "method path proto" status size "referer" "ua"
LOG_RE = re.compile(
    r'(?P<ip>\S+) \S+ \S+ \[(?P<time>[^\]]+)\] '
    r'"(?P<method>\S+) (?P<path>\S+) \S+" '
    r'(?P<status>\d{3}) (?P<size>\S+)'
)

with open('access.log') as f:
    for line in f:
        m = LOG_RE.match(line)
        if not m:
            continue                       # malformed; skip (and maybe count)
        if m['status'].startswith(('4', '5')):
            print(m['ip'], m['status'], m['path'])
```

**Use named groups.** Use `match.groupdict()` if you need a dict. Anchor to *positions* in the format, not "anything that looks like a number."

**Edge cases to mention** (even if you don't code them all):

- **Malformed lines** — count them and report; don't crash.
- **Multi-line entries** (Java stack traces, JSON pretty-printed) — line-by-line breaks. State machine or join-by-timestamp.
- **Log rotation** mid-script — if tailing, watch for inode change.
- **Compressed logs** — `gzip.open()` or `subprocess.Popen(['zcat', f])`.
- **Encoding** — pass `encoding='utf-8', errors='replace'` if the file might have garbage bytes.
- **IPv6** — `\b(?:\d{1,3}\.){3}\d{1,3}\b` won't match. Use `ipaddress` module to validate after a coarser regex.

### 7E. Regex essentials (30 min)

The patterns you'll write under pressure:

| Pattern | Matches |
|---|---|
| `\d` | digit (PCRE only — in basic regex use `[0-9]`) |
| `\s` / `\S` | whitespace / non-whitespace |
| `\w` / `\W` | word char `[a-zA-Z0-9_]` / non-word |
| `\b` | word boundary |
| `^` / `$` | start / end of line |
| `.` | any char except newline |
| `*` `+` `?` | 0+ / 1+ / 0 or 1 |
| `{n,m}` | between n and m times |
| `[abc]` | character class |
| `[^abc]` | negated class |
| `(...)` | group |
| `(?:...)` | non-capturing group |
| `(?P<name>...)` | named group (Python) |
| `\1` / `\g<name>` | backreference |

**Greedy vs lazy**: `.*` is greedy (matches as much as possible). `.*?` is lazy (matches as little as possible). For "everything between two brackets," prefer `\[([^\]]*)\]` over `\[(.*?)\]` — character-class-negation is faster and correct.

**Anchoring discipline** — the most common bug. `\d{3}` matches `500` *anywhere*, including inside `1500`. Use `\b\d{3}\b`.

**Compile once, reuse** in Python:

```python
PATTERN = re.compile(r'...')        # at module level
PATTERN.findall(line)               # in the loop
```

---

## Midday Block (2.5h) — Hands-on labs

### Lab 1: Generate a log file to work on (15 min)

```bash
mkdir -p ~/day7 && cd ~/day7
python3 - <<'EOF' > access.log
import random, datetime
random.seed(42)
ips = ['10.0.5.42','10.0.5.43','192.168.1.5','172.16.0.10','203.0.113.7']
methods = ['GET']*8 + ['POST']*2
paths = ['/','/api/users','/api/orders','/login','/health','/static/app.css']
statuses = [200]*70 + [301]*5 + [404]*15 + [500]*8 + [503]*2
sizes = [random.randint(100,9000) for _ in range(20)]
start = datetime.datetime(2026, 5, 1, 10, 0, 0)
for i in range(2000):
    ts = (start + datetime.timedelta(seconds=i*3)).strftime('%d/%b/%Y:%H:%M:%S +0000')
    ip = random.choice(ips)
    m = random.choice(methods)
    p = random.choice(paths)
    s = random.choice(statuses)
    sz = random.choice(sizes)
    print(f'{ip} - - [{ts}] "{m} {p} HTTP/1.1" {s} {sz} "-" "Mozilla/5.0"')
EOF
wc -l access.log
head -3 access.log
```

You now have a 2000-line Apache combined-format log to attack.

### Lab 2: One-liner ladder (30 min)

Write each of these without looking at the answer first. Then check.

1. Top 5 IPs by request count.
2. All 5xx responses, only the IP and the path.
3. Count of each status code.
4. Total bytes served (sum of column 10).
5. Requests per minute (the timestamp is column 4).

<details>
<summary>Answers</summary>

```bash
# 1
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -5

# 2
awk '$9 ~ /^5/ {print $1, $7}' access.log

# 3
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# 4
awk '$10 ~ /^[0-9]+$/ {sum += $10} END {print sum}' access.log

# 5
awk -F'[][]' '{print substr($2, 1, 17)}' access.log | sort | uniq -c | sort -rn | head
# (the [][] field separator splits on the brackets around the timestamp)
```

</details>

### Lab 3: The Python live-coding solution, end to end (45 min)

Write this from scratch on paper or in a fresh file before looking. **Time yourself — 15 minutes target.**

Requirements:

- Read a log file path from `argv[1]`.
- Print top 10 IPs by request count.
- Print count of each 4xx and 5xx status code.
- Print top 10 paths that returned errors.
- Stream the file (don't load it all).
- Handle malformed lines gracefully (count and report).

Then compare with this reference:

```python
#!/usr/bin/env python3
"""Parse Apache/Nginx combined-format access log."""
import re
import sys
from collections import Counter

LOG_RE = re.compile(
    r'(?P<ip>\S+) \S+ \S+ \[(?P<time>[^\]]+)\] '
    r'"(?P<method>\S+) (?P<path>\S+) \S+" '
    r'(?P<status>\d{3}) (?P<size>\S+)'
)

def parse(path):
    ips = Counter()
    error_codes = Counter()
    error_paths = Counter()
    malformed = 0

    with open(path, encoding='utf-8', errors='replace') as f:
        for line in f:
            m = LOG_RE.match(line)
            if not m:
                malformed += 1
                continue

            ips[m['ip']] += 1
            status = m['status']
            if status[0] in ('4', '5'):
                error_codes[status] += 1
                error_paths[m['path']] += 1

    return ips, error_codes, error_paths, malformed

def main():
    if len(sys.argv) != 2:
        sys.exit(f"usage: {sys.argv[0]} <logfile>")

    ips, codes, paths, bad = parse(sys.argv[1])

    print("Top 10 IPs:")
    for ip, n in ips.most_common(10):
        print(f"  {n:6d}  {ip}")

    print("\nError codes:")
    for code, n in sorted(codes.items()):
        print(f"  {n:6d}  {code}")

    print("\nTop 10 error paths:")
    for path, n in paths.most_common(10):
        print(f"  {n:6d}  {path}")

    if bad:
        print(f"\nMalformed lines skipped: {bad}", file=sys.stderr)

if __name__ == '__main__':
    main()
```

```bash
chmod +x parselog.py
./parselog.py access.log
```

**Things to verbalize as you write (rehearse this)**:

- *"I'm using a regex match for the whole line so I get structured fields rather than fishing for status codes and IPs separately. The naive approach would mismatch e.g. a 500-byte size field with the 500 status."*
- *"I'm using `Counter.most_common()` because it's already sorted internally."*
- *"I'm counting malformed lines instead of crashing on them. In production I'd log them somewhere for inspection."*
- *"This is fine for a single file. At fleet scale, I'd push these logs to CloudWatch Logs and run an Insights query — `stats count() by status` and `filter status >= 400` — that runs against indexed data instead of grep'ing files."*

### Lab 4: Make it 10x faster (30 min)

Take your script. Generate a bigger log (1M lines):

```bash
python3 - <<'EOF' > big.log
import random, datetime
random.seed(1)
ips=['10.0.5.42','10.0.5.43','192.168.1.5','172.16.0.10','203.0.113.7']
methods=['GET']*8+['POST']*2
paths=['/','/api/users','/api/orders','/login','/health']
statuses=[200]*70+[301]*5+[404]*15+[500]*8+[503]*2
start=datetime.datetime(2026,5,1)
with open('big.log','w') as f:
    for i in range(1_000_000):
        ts=(start+datetime.timedelta(seconds=i)).strftime('%d/%b/%Y:%H:%M:%S +0000')
        f.write(f'{random.choice(ips)} - - [{ts}] "{random.choice(methods)} {random.choice(paths)} HTTP/1.1" {random.choice(statuses)} {random.randint(100,9000)} "-" "Mozilla/5.0"\n')
EOF
ls -lh big.log
time ./parselog.py big.log >/dev/null
```

Now optimize. Try:

1. **Pre-filter cheap before regex**:

```python
for line in f:
    if not line:
        continue
    # Quick path: if no error status anywhere on the line, skip cheaper analysis
    m = LOG_RE.match(line)
    ...
```

2. **Skip the regex when only counting IPs** — split the line on the first space:

```python
ip = line.partition(' ')[0]
ips[ip] += 1
```

3. **Compare with a pure-awk implementation**:

```bash
time awk '{print $1}' big.log | sort | uniq -c | sort -rn | head -10
```

You'll often find `awk | sort | uniq -c | sort -rn` is faster than naive Python — and slower than well-written Python with `Counter`. **Saying that out loud — "let me check whether `awk` would be faster here" — is exactly the senior-engineer instinct interviewers look for.**

### Lab 5: Bash version of the same task (30 min)

Bash with strict mode, the right way:

```bash
#!/bin/bash
set -euo pipefail
IFS=$'\n\t'

LOG="${1:?usage: $0 <logfile>}"

[[ -r "$LOG" ]] || { echo "$LOG: not readable" >&2; exit 1; }

echo "Top 10 IPs:"
awk '{print $1}' "$LOG" | sort | uniq -c | sort -rn | head -10

echo
echo "Error codes:"
awk '$9 ~ /^[45]/ {print $9}' "$LOG" | sort | uniq -c | sort -rn

echo
echo "Top 10 error paths:"
awk '$9 ~ /^[45]/ {print $7}' "$LOG" | sort | uniq -c | sort -rn | head -10
```

Compare with `time` against the Python version:

```bash
time bash parselog.sh big.log >/dev/null
time ./parselog.py big.log >/dev/null
```

For pure aggregation on well-formed logs, the awk pipeline is hard to beat. Python wins when the parsing is more complex, when you need joined data, or when you'd actually use the parsed values in further code.

### Lab 6: A Python production-shape script (30 min)

The version with everything you'd actually want in code review: argparse, logging, gzip support, type hints.

```python
#!/usr/bin/env python3
"""Apache combined-format log analyzer."""
from __future__ import annotations
import argparse
import gzip
import logging
import re
import sys
from collections import Counter
from contextlib import contextmanager
from pathlib import Path
from typing import Iterator, TextIO

LOG_RE = re.compile(
    r'(?P<ip>\S+) \S+ \S+ \[(?P<time>[^\]]+)\] '
    r'"(?P<method>\S+) (?P<path>\S+) \S+" '
    r'(?P<status>\d{3}) (?P<size>\S+)'
)
log = logging.getLogger(__name__)

@contextmanager
def open_log(path: Path) -> Iterator[TextIO]:
    if path.suffix == '.gz':
        with gzip.open(path, 'rt', encoding='utf-8', errors='replace') as f:
            yield f
    else:
        with path.open(encoding='utf-8', errors='replace') as f:
            yield f

def analyze(path: Path) -> dict:
    ips: Counter[str] = Counter()
    codes: Counter[str] = Counter()
    paths: Counter[str] = Counter()
    malformed = 0
    total = 0

    with open_log(path) as f:
        for line in f:
            total += 1
            m = LOG_RE.match(line)
            if not m:
                malformed += 1
                continue
            ips[m['ip']] += 1
            status = m['status']
            if status[0] in ('4', '5'):
                codes[status] += 1
                paths[m['path']] += 1

    return {
        'total': total,
        'malformed': malformed,
        'ips': ips,
        'codes': codes,
        'paths': paths,
    }

def report(r: dict, top: int) -> None:
    print(f"Total lines: {r['total']}, malformed: {r['malformed']}")
    print(f"\nTop {top} IPs:")
    for ip, n in r['ips'].most_common(top):
        print(f"  {n:6d}  {ip}")
    print("\nError codes:")
    for code, n in sorted(r['codes'].items()):
        print(f"  {n:6d}  {code}")
    print(f"\nTop {top} error paths:")
    for p, n in r['paths'].most_common(top):
        print(f"  {n:6d}  {p}")

def main() -> int:
    ap = argparse.ArgumentParser()
    ap.add_argument('logfile', type=Path)
    ap.add_argument('--top', type=int, default=10)
    ap.add_argument('-v', '--verbose', action='store_true')
    args = ap.parse_args()

    logging.basicConfig(
        level=logging.DEBUG if args.verbose else logging.INFO,
        format='%(asctime)s %(levelname)s %(message)s',
    )

    if not args.logfile.is_file():
        log.error("not a file: %s", args.logfile)
        return 1

    report(analyze(args.logfile), args.top)
    return 0

if __name__ == '__main__':
    sys.exit(main())
```

Run it:

```bash
chmod +x parselog2.py
./parselog2.py access.log --top 5
gzip -k big.log               # creates big.log.gz alongside
./parselog2.py big.log.gz --top 5
```

This is the version that survives a code review. Mention each detail you'd add — argparse, gzip support, logging, type hints — even if you wouldn't write all of it on a 25-minute interview.

---

## Afternoon Block (1.5h) — Drills + Story #7

### Self-check (45 min)

1. Walk through what `set -euo pipefail` does, line by line.
2. Why `[[` over `[`?
3. Write a regex for an IPv4 address and explain its limitations.
4. `awk '{print $1}' access.log | sort | uniq -c | sort -rn | head` — explain each stage.
5. The script reads a 50 GB log file. How do you make sure it doesn't OOM?
6. The interviewer says "now also count unique users." Walk me through how you'd modify your Python script.
7. Why would you compile a regex outside the loop?
8. `grep -E` vs `grep -P` — what's the difference, and when does it matter?
9. Production scenario: you need to find which IPs hit your API more than 100 times in the last hour. Walk through how you'd actually do this — not the toy answer.
10. A log line is malformed. What does your script do, and what should it do?
11. The log file is being actively written. How do you read it without missing entries or crashing on rotation?
12. The interviewer says: "your regex matches `999.999.999.999`. Fix it." How?

<details>
<summary>Answers</summary>

1. `-e` exit on any non-zero command (catches errors immediately). `-u` treat unset variables as errors (catches typos). `-o pipefail` pipeline fails if any stage fails (default counts only the last). Together: scripts fail fast on real errors instead of plowing forward producing garbage.
2. `[[ ]]` is a Bash builtin — supports `&&`, `||`, regex `=~`, lexicographic `<`/`>`, no word splitting on unquoted vars (so `[[ -z $x ]]` works without quotes). `[ ]` is `/usr/bin/test`, POSIX, no `&&`, brittle with empty/unquoted variables.
3. `\b(?:\d{1,3}\.){3}\d{1,3}\b`. Matches anything four-octet-shaped, including `999.999.999.999` and `0.0.0.0`. For correctness, validate with Python's `ipaddress.ip_address(s)` — raises ValueError on invalid.
4. Print column 1 (IPs in Apache format) → sort lexically (required for uniq) → count consecutive duplicates → sort by count descending → top N. Standard "histogram of column" pattern.
5. Stream line-by-line with `for line in f:`. Don't `f.read()` or `f.readlines()`. Counters scale with cardinality (number of unique IPs), not line count, so they're bounded.
6. Add `users: Counter[str] = Counter()` (or whichever field), increment in the loop, report it. The shape of the script — match → extract → count — doesn't change.
7. `re.compile()` parses the pattern once. Without it, Python uses an internal cache, but the cache is bounded and explicit compilation is cleaner and slightly faster, especially when the pattern is complex.
8. `-E` is ERE (Extended Regex) — POSIX standard, no `\d`, no lookarounds. `-P` is PCRE — Perl-compatible, supports `\d`, `\b`, lookahead/lookbehind, but is non-portable (not on all systems, slower). Use `-E` by default; reach for `-P` when you need PCRE features.
9. Real production: query the centralized log store (CloudWatch Logs Insights: `stats count() by clientIp | filter @timestamp > ago(1h) | sort count() desc`). If you must do it on box: `awk -v cutoff="$(date -u -d '1 hour ago' +%s)" '...'` is fragile; better to ship logs somewhere queryable.
10. Currently it counts and continues. Should: log a sample of malformed lines (with line number) for debugging, expose a malformed counter to monitoring, fail loud if malformed > some threshold (could indicate format change upstream).
11. For a static-but-rotating file: track inode (`os.stat(...).st_ino`); if it changed, file rotated, reopen. For tailing: use `tail -F` (capital F follows by name across rotations). For production: agents like Vector / Fluent Bit / CloudWatch Agent handle this correctly.
12. Use `ipaddress.ip_address(s)` after the regex match. Or use a stricter regex with bounded octets: `(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]?\d)` per octet — but the validation-after approach is more readable and equally correct.

</details>

### Behavioral (45 min) — Story #7: Deliver Results

End-of-week-one LP: **Deliver Results**. Story prompt:

> "Tell me about a project that ran into serious obstacles. How did you push through and deliver?"

Deliver Results stories work when they show:

- A meaningful goal with a real deadline or stake.
- An obstacle that was not trivial — you had to do something different.
- *You* (not the team in general) made specific choices that drove the outcome.
- A measurable result, ideally with magnitude (% improvement, $ saved, hours reduced).

Avoid: "we worked hard and shipped on time." Deliver Results requires *adversity*. The strongest stories have a moment of "this might fail" and the specific decision you made that turned it around.

STAR. 2-3 minutes. Out loud. Time it.

---

**End of Week 1.** Story bank: 7 stories covering Ownership, Dive Deep, Insist on the Highest Standards, Bias for Action, Customer Obsession, Earn Trust, Deliver Results — 7 of 16 LPs, with high-frequency LPs already in.
