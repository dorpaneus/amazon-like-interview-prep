# 📜 Day 7 — Scripting and Log Parsing

> [!NOTE]
> **The Goal:** End of week one. Today is the **live-coding day**. The reported question — "extract IPs and error codes from log files" — is the most likely live-coding scenario for this role. We'll write that code, but the real skill is **what you say while you type**: narrating tradeoffs, asking clarifying questions, recognizing when the toy answer is wrong for production scale.
>
> The interviewer is watching for: Bash fluency? Do you stream or load the whole file? Do you know what regex you wrote? Do you handle malformed input? Do you know when to reach for `awk`?

---

## 🧠 Morning Block — Tools and idioms

### 7a. Choosing the right tool

The first thing to **say out loud** in a live-coding session: "What scale are we at?" The answer drives the tool.

| Scale | Tool |
| :--- | :--- |
| One file, a few MB, one-off | `grep`/`awk`/`sort` one-liner |
| One file, GBs, recurring | Bash streaming with `awk`, or Python |
| Many files across hosts, recurring | Centralized logging (CloudWatch Insights, Athena, Elasticsearch) |
| Real-time analysis | Stream processor (Kinesis, Kafka + consumers), or eBPF for live system data |

> [!TIP]
> **The interview-winning answer** to the log-parsing question is *not* the cleverest regex. It's:
> 1. Ask the scale and frequency.
> 2. Solve the toy version cleanly.
> 3. **Note that at production scale you wouldn't grep files** — logs would already be in CloudWatch Logs Insights / Athena / Elasticsearch, queryable by structured fields. The grep-the-file solution is for ad-hoc debugging on one box.

### 7b. Bash hygiene

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

> [!WARNING]
> Don't use `/tmp/foo.$$` — predictable, race-prone, and not cleaned up on failure.

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

### 7c. The text toolkit — `grep`, `awk`, `sed`, `cut`, `sort`, `uniq`

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

### 7d. Bash — patterns for live-coding

**The skeleton** for any "process a log file" question using `awk`:

```bash
#!/bin/bash
set -euo pipefail
IFS=$'\n\t'

LOG="${1:?usage: $0 <logfile>}"

[[ -r "$LOG" ]] || { echo "$LOG: not readable" >&2; exit 1; }

# Using a single awk script to process the file once
awk '
BEGIN {
    # Define regular expressions
    ip_re = "^[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}$"
    err_re = "^[45][0-9]{2}$"
}
{
    # Basic validation (could be more robust based on log format)
    if (NF >= 9) {
        ip = $1
        status = $9
        path = $7

        if (ip ~ ip_re) {
            ips[ip]++
        }
        if (status ~ err_re) {
            codes[status]++
            paths[path]++
        }
    } else {
        malformed++
    }
}
END {
    print "Top 10 IPs:"
    # Use sort to handle the ordering
    for (ip in ips) {
        printf "%6d  %s\n", ips[ip], ip | "sort -rn | head -n 10"
    }
    close("sort -rn | head -n 10") # Important to close the pipe

    print "\nError codes:"
    for (code in codes) {
        printf "%6d  %s\n", codes[code], code | "sort -rn"
    }
    close("sort -rn")

    print "\nTop 10 error paths:"
    for (path in paths) {
        printf "%6d  %s\n", paths[path], path | "sort -rn | head -n 10"
    }
    close("sort -rn | head -n 10")
    
    if (malformed > 0) {
        printf "\nMalformed lines skipped: %d\n", malformed > "/dev/stderr"
    }
}' "$LOG"
```

**Things to say out loud while writing it**:

- *"I'm using `awk` because it processes the file sequentially and is highly optimized for text processing, allowing me to avoid loading the entire file into memory."*
- *"I'm using associative arrays (like `ips[ip]++`) which act like a dictionary or hash map to keep counts."*
- *"This regex catches anything that *looks* like an IPv4. `0.0.0.0` and `999.999.999.999` would both match. If I needed strict validation, I might write a shell function or reach for Python, but `awk` is sufficient for standard logs."*
- *"The 4xx/5xx regex is anchored. `awk` natively splits on whitespace, making it easy to access the status code (usually column 9 in combined format)."*

**The thing that separates intermediate from senior** — when the interviewer says "now make it 100x faster" or "this file is 50 GB":

- **Pre-filter** — if you only care about errors, `grep -E 'HTTP/1\.[01]" [45]'` before piping to `awk`.
- **Parallelize** — split the file or use `xargs -P` or GNU `parallel`.
- **Or: stop. The right answer is "this shouldn't be a Bash script. This is a CloudWatch Logs Insights query / Athena query / Elasticsearch search."**

**Edge cases to mention** (even if you don't code them all):

- **Malformed lines** — count them and report; don't crash. My script does a basic `NF >= 9` check.
- **Multi-line entries** (Java stack traces, JSON pretty-printed) — line-by-line breaks. You'd need a more complex `awk` script using custom `RS` (Record Separator) or a different tool entirely.
- **Log rotation** mid-script — if tailing, `tail -F` is needed.
- **Compressed logs** — `zcat` or `zgrep` to read on the fly.
- **IPv6** — The simple regex won't match.

### 7e. Regex essentials

The patterns you'll write under pressure:

| Pattern | Matches |
|---|---|
| `[0-9]` | digit (`\d` works in `grep -P`, but not basic `awk` or `sed`) |
| `[ \t]` | whitespace |
| `[a-zA-Z0-9_]` | word char |
| `\b` | word boundary (support varies by tool) |
| `^` / `$` | start / end of line |
| `.` | any char except newline |
| `*` `+` `?` | 0+ / 1+ / 0 or 1 (`+` and `?` often require Extended Regex `-E`) |
| `{n,m}` | between n and m times (often requires Extended Regex) |
| `[abc]` | character class |
| `[^abc]` | negated class |
| `(...)` | group |

**Anchoring discipline** — the most common bug. `[0-9]{3}` matches `500` *anywhere*, including inside `1500`. Use `^[0-9]{3}$` if checking a specific field, or boundaries if searching a whole string.

---

## 💻 Midday Block — Hands-on labs

### Lab 1: Generate a log file to work on

```bash
mkdir -p ~/day7 && cd ~/day7
# Using a simple python one-liner just to generate data
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

### Lab 2: One-liner ladder

Write each of these without looking at the answer first. Then check.

1. Top 5 IPs by request count.
2. All 5xx responses, only the IP and the path.
3. Count of each status code.
4. Total bytes served (sum of column 10).
5. Requests per minute (the timestamp is column 4).

<details>
<summary><strong>Answers</strong></summary>

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

### Lab 3: The Bash live-coding solution, end to end

Write this from scratch on paper or in a fresh file before looking. **Time yourself — 15 minutes target.**

Requirements:

- Read a log file path from `$1`.
- Print top 10 IPs by request count.
- Print count of each 4xx and 5xx status code.
- Print top 10 paths that returned errors.
- Stream the file (don't load it all).
- Handle malformed lines gracefully (count and report).

Then compare with this reference:

```bash
#!/bin/bash
set -euo pipefail

LOG="${1:-}"

if [[ -z "$LOG" ]]; then
    echo "Usage: $0 <logfile>" >&2
    exit 1
fi

if [[ ! -f "$LOG" || ! -r "$LOG" ]]; then
    echo "Error: Cannot read file $LOG" >&2
    exit 1
fi

awk '
BEGIN {
    err_re = "^[45][0-9]{2}$"
}
{
    if (NF >= 9) {
        ip = $1
        status = $9
        path = $7

        ips[ip]++
        
        if (status ~ err_re) {
            codes[status]++
            paths[path]++
        }
    } else {
        malformed++
    }
}
END {
    print "Top 10 IPs:"
    for (ip in ips) {
        printf "%6d  %s\n", ips[ip], ip | "sort -rn | head -n 10"
    }
    close("sort -rn | head -n 10")

    print "\nError codes:"
    for (code in codes) {
        printf "%6d  %s\n", codes[code], code | "sort -rn"
    }
    close("sort -rn")

    print "\nTop 10 error paths:"
    for (path in paths) {
        printf "%6d  %s\n", paths[path], path | "sort -rn | head -n 10"
    }
    close("sort -rn | head -n 10")
    
    if (malformed > 0) {
        printf "\nMalformed lines skipped: %d\n", malformed > "/dev/stderr"
    }
}' "$LOG"
```

```bash
chmod +x parselog.sh
./parselog.sh access.log
```

**Things to verbalize as you write (rehearse this)**:

- *"I'm using `awk` to process the file in a single pass. This is efficient and keeps memory usage low since we only store the unique counts."*
- *"I'm using `NF >= 9` as a basic check for malformed lines. If a line is cut short, it won't have the status field where we expect it."*
- *"I'm piping the output from the `awk` arrays into `sort` and `head` directly from within `awk` to get the top counts."*
- *"For a real production environment, if I need robust parsing (like handling escaped quotes in URLs), I would switch to Python or use a dedicated log parser. If it's fleet-scale, I'd query CloudWatch."*

### Lab 4: Make it 10x faster

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
time ./parselog.sh big.log >/dev/null
```

Now optimize. Try:

1. **Pre-filter cheap before regex**:
If you only need error codes, filtering with `grep` before `awk` can be faster.

```bash
time grep -E 'HTTP/1\.[01]" [45][0-9]{2}' big.log | awk '{print $7}' | sort | uniq -c | sort -rn | head -10
```

You'll often find pipelines of standard tools (`grep | awk | sort | uniq -c`) are heavily optimized. **Saying that out loud — "let me check if pre-filtering with `grep` is faster here" — is exactly the senior-engineer instinct interviewers look for.**

### Lab 5: A Bash production-shape script

The version with everything you'd actually want in code review: help text, robust options parsing, and handling compressed files.

```bash
#!/bin/bash
set -euo pipefail
IFS=$'\n\t'

show_help() {
    echo "Usage: $0 [OPTIONS] <logfile>"
    echo "Analyzes Apache/Nginx combined-format access logs."
    echo ""
    echo "Options:"
    echo "  -n, --top N    Show top N results (default: 10)"
    echo "  -h, --help     Show this help message"
}

TOP=10

# Parse arguments
while [[ "$#" -gt 0 ]]; do
    case $1 in
        -n|--top) TOP="$2"; shift ;;
        -h|--help) show_help; exit 0 ;;
        -*) echo "Unknown parameter: $1" >&2; exit 1 ;;
        *) LOG="$1" ;;
    esac
    shift
done

if [[ -z "${LOG:-}" ]]; then
    show_help >&2
    exit 1
fi

if [[ ! -f "$LOG" || ! -r "$LOG" ]]; then
    echo "Error: Cannot read file $LOG" >&2
    exit 1
fi

# Determine how to read the file (handle gzip)
READ_CMD="cat"
if [[ "$LOG" == *.gz ]]; then
    READ_CMD="zcat"
fi

$READ_CMD "$LOG" | awk -v top="$TOP" '
BEGIN {
    err_re = "^[45][0-9]{2}$"
}
{
    total++
    if (NF >= 9) {
        ip = $1
        status = $9
        path = $7

        ips[ip]++
        
        if (status ~ err_re) {
            codes[status]++
            paths[path]++
        }
    } else {
        malformed++
    }
}
END {
    print "Total lines: " total ", malformed: " malformed + 0

    print "\nTop " top " IPs:"
    sort_cmd = "sort -rn | head -n " top
    for (ip in ips) {
        printf "%6d  %s\n", ips[ip], ip | sort_cmd
    }
    close(sort_cmd)

    print "\nError codes:"
    for (code in codes) {
        printf "%6d  %s\n", codes[code], code | "sort -rn"
    }
    close("sort -rn")

    print "\nTop " top " error paths:"
    for (path in paths) {
        printf "%6d  %s\n", paths[path], path | sort_cmd
    }
    close(sort_cmd)
}'
```

Run it:

```bash
chmod +x parselog2.sh
./parselog2.sh -n 5 access.log
gzip -k big.log               # creates big.log.gz alongside
./parselog2.sh big.log.gz --top 5
```

This is the version that survives a code review. Mention each detail you'd add — argument parsing, gzip support — even if you wouldn't write all of it on a 25-minute interview.

---

## 🎯 Afternoon Block — Drills + Story #7

### Self-check

1. Walk through what `set -euo pipefail` does, line by line.
2. Why `[[` over `[`?
3. `awk '{print $1}' access.log | sort | uniq -c | sort -rn | head` — explain each stage.
4. The script reads a 50 GB log file. How do you make sure it doesn't OOM?
5. The interviewer says "now also count unique users." Walk me through how you'd modify your `awk` script.
6. `grep -E` vs `grep -P` — what's the difference, and when does it matter?
7. Production scenario: you need to find which IPs hit your API more than 100 times in the last hour. Walk through how you'd actually do this — not the toy answer.
8. A log line is malformed. What does your script do, and what should it do?
9. The log file is being actively written. How do you read it without missing entries or crashing on rotation?
10. The interviewer asks why you didn't use Python for this task. What is your response?

<details>
<summary><strong>Answers</strong> (click to reveal)</summary>

1. `-e` exit on any non-zero command (catches errors immediately). `-u` treat unset variables as errors (catches typos). `-o pipefail` pipeline fails if any stage fails (default counts only the last). Together: scripts fail fast on real errors instead of plowing forward producing garbage.
2. `[[ ]]` is a Bash builtin — supports `&&`, `||`, regex `=~`, lexicographic `<`/`>`, no word splitting on unquoted vars (so `[[ -z $x ]]` works without quotes). `[ ]` is `/usr/bin/test`, POSIX, no `&&`, brittle with empty/unquoted variables.
3. Print column 1 (IPs in Apache format) → sort lexically (required for `uniq`) → count consecutive duplicates → sort by count descending → top N. Standard "histogram of column" pattern.
4. Stream line-by-line with `awk`. Don't load the file into a variable. Counters scale with cardinality (number of unique IPs), not line count, so they're bounded.
5. Add an array for users `users[$3]++` (assuming user is column 3), increment in the loop, and add a loop in the `END` block to print it out.
6. `-E` is ERE (Extended Regex) — POSIX standard, no `\d`, no lookarounds. `-P` is PCRE — Perl-compatible, supports `\d`, `\b`, lookahead/lookbehind, but is non-portable (not on all systems, slower). Use `-E` by default; reach for `-P` when you need PCRE features.
7. Real production: query the centralized log store (CloudWatch Logs Insights: `stats count() by clientIp | filter @timestamp > ago(1h) | sort count() desc`). If you must do it on box: `awk -v cutoff="$(date -u -d '1 hour ago' +%s)" '...'` is fragile; better to ship logs somewhere queryable.
8. Currently it counts and continues. Should: log a sample of malformed lines (with line number) for debugging, expose a malformed counter to monitoring, fail loud if malformed > some threshold (could indicate format change upstream).
9. For a static-but-rotating file: track inode; if it changed, file rotated, reopen. For tailing: use `tail -F` (capital F follows by name across rotations). For production: agents like Vector / Fluent Bit / CloudWatch Agent handle this correctly.
10. "For a simple aggregation task like this, `awk` is highly optimized and often faster than Python for single-pass file reading. However, if the log format was complex (like JSON), or I needed to join data against a database, or the script needed to be maintained by a larger team, I would choose Python for its readability, libraries, and ease of testing."

</details>

### 🤝 Behavioral — Story #7: Deliver Results

End-of-week-one LP: **Deliver Results**. Story prompt:

> "Tell me about a project that ran into serious obstacles. How did you push through and deliver?"

> [!TIP]
> Deliver Results stories work when they show:
> - A meaningful goal with a real deadline or stake.
> - An obstacle that was not trivial — you had to do something different.
> - *You* (not the team in general) made specific choices that drove the outcome.
> - A measurable result, ideally with magnitude (% improvement, $ saved, hours reduced).

Avoid: "We worked hard and shipped on time." Deliver Results requires *adversity*. The strongest stories have a moment of "this might fail" and the specific decision you made that turned it around.

STAR. 2-3 minutes. Out loud. Time it.

---

**End of Week 1.** Story bank: 7 stories covering Ownership, Dive Deep, Insist on the Highest Standards, Bias for Action, Customer Obsession, Earn Trust, Deliver Results — 7 of 16 LPs, with high-frequency LPs already in.
