# Day 12 — Security & Package Management (Sat May 9)

Two themes today, both very high-frequency interview territory:

1. **Security**: SELinux is the topic candidates fear most and where interviewers separate the prepared from the unprepared. Plus PAM (the "I can't log in" tree) and LDAP/Kerberos.
2. **Package management**: dnf/RPM is mundane until the database corrupts. The recovery procedure is short, surprising, and they will ask about it.

LP today: **Have Backbone; Disagree and Commit** — a hard one to land well.

---

## Morning Block (3h) — Concepts

### 12A. SELinux fundamentals (1h)

The mental model that makes everything else click:

> SELinux is **Mandatory Access Control** layered on top of standard Unix permissions. Standard perms say "user X can read this file." SELinux additionally says "process labeled with type httpd_t can read files labeled with type httpd_sys_content_t." Both must allow the access. Either can deny.

Every subject (process) and object (file, socket, port, etc.) has a **security context**:

```
system_u:object_r:httpd_sys_content_t:s0
   |        |          |               |
   user    role       type           level (MLS)
```

For the **targeted policy** (RHEL/Fedora default), the only field that usually matters is **type**. Targeted policy confines the well-known services (httpd, sshd, etc.) and leaves the rest unconfined.

**Modes**:

| Mode | What happens |
|---|---|
| **enforcing** | Policy is enforced. Denials block the action and are logged. |
| **permissive** | Policy is evaluated. Denials are logged but **not blocked**. |
| **disabled** | SELinux is off. Requires reboot to change to/from. |

```bash
getenforce                    # enforcing / permissive / disabled
sudo setenforce 0             # switch to permissive (runtime only)
sudo setenforce 1             # back to enforcing
cat /etc/selinux/config       # persistent mode
```

**Reading contexts**:

```bash
ls -Z /var/www/html           # file contexts
ps auxZ | head                # process contexts
id -Z                         # your process's context
ss -Z                         # socket contexts
```

**AVC denials — the single most important concept**:

When SELinux blocks something, it writes an Access Vector Cache (AVC) denial. **The denial tells you exactly what to fix.**

```bash
sudo ausearch -m AVC -ts recent
sudo ausearch -m AVC,USER_AVC -ts today

# A friendly translator
sudo sealert -a /var/log/audit/audit.log

# Last 10 denials, human-readable
sudo ausearch -m AVC --raw | audit2why
```

A denial looks like:

```
type=AVC msg=audit(...): avc: denied { read } for pid=1234 comm="httpd"
  name="secret.txt" dev="dm-0" ino=12345
  scontext=system_u:system_r:httpd_t:s0
  tcontext=unconfined_u:object_r:user_home_t:s0
  tclass=file permissive=0
```

Read it as:

- **What was denied**: `{ read }` permission
- **Who tried**: `httpd_t` (source context)
- **What they tried to access**: `user_home_t` file (target context)
- **Permissive=0**: it was blocked

The fix is almost always one of three things:

1. **Wrong label** — `restorecon` to fix it back to default
2. **Need a boolean** — `setsebool` to enable an optional policy
3. **Custom path** — `semanage fcontext` to teach SELinux a new path pattern, then `restorecon`

**Tools**:

| Tool | Purpose |
|---|---|
| `getenforce` / `setenforce` | Current mode / switch enforcing\|permissive at runtime |
| `ls -Z` / `ps -Z` / `id -Z` / `ss -Z` | Show context of files/processes/self/sockets |
| `restorecon` | Reset file's context to the policy default |
| `chcon` | Set context temporarily (lost on restorecon) |
| `semanage fcontext` | Persistent file-context rules |
| `semanage port` | Allow non-default ports for a service type |
| `semanage boolean` / `getsebool` / `setsebool` | List/get/set policy booleans |
| `ausearch` | Search audit log for AVCs |
| `audit2allow` | Generate a policy module from a denial (last resort) |
| `audit2why` | Explain why a denial happened |
| `sealert` | Pretty-print denials with suggested fixes |
| `matchpathcon` | What context should this path have? |

**The canonical interview answer for "SELinux is blocking httpd from reading my custom directory"**:

> "First confirm with `getenforce` and check `ausearch -m AVC -ts recent` for the actual denial. If the file should be served by httpd, it needs the right type label — usually `httpd_sys_content_t`. I'd add a persistent rule with `semanage fcontext -a -t httpd_sys_content_t '/srv/web(/.*)?'` and apply with `restorecon -Rv /srv/web`. If httpd needs a feature like network connect that's gated by a boolean, `setsebool -P httpd_can_network_connect on`. As a last resort for an exotic case, I'd build a policy module with `audit2allow`, but that's almost always overkill — defaults plus booleans plus correct labels cover ~99% of real cases."

**The two highest-frequency booleans**:

- `httpd_can_network_connect` — let httpd make outbound connections (reverse proxies, S3 uploads)
- `httpd_can_network_relay` — let httpd connect to specific relay services

Always use `setsebool -P` (the `-P` makes it persistent).

**A common interview gotcha**: "I changed the file's context with `chcon` and the change disappeared." Because something ran `restorecon` (sometimes automatically) which reset to the default. Use `semanage fcontext` for persistence.

**Custom ports** — a Node.js app listens on port 9000, sshd custom on port 2222:

```bash
sudo semanage port -a -t http_port_t -p tcp 9000
sudo semanage port -a -t ssh_port_t -p tcp 2222
```

Without this, even with `firewalld` open, SELinux blocks the service from binding the port.

### 12B. PAM and authentication (45 min)

PAM = Pluggable Authentication Modules. Every authenticating service (`login`, `sshd`, `sudo`, `su`, `screen`, etc.) consults `/etc/pam.d/<service>` to decide who gets in.

**Module categories** (the stacks):

| Type | Question it answers |
|---|---|
| `auth` | Are you who you say you are? (passwords, tokens, keys) |
| `account` | Is your account allowed to log in right now? (expiry, time of day, lockouts) |
| `password` | Allow changing passwords (complexity, history) |
| `session` | Setup/teardown around the session (mount home dir, start audit, set ulimits) |

**Control flags**:

| Flag | Behavior on success/failure |
|---|---|
| `required` | Failure → still fails, but continues evaluating (hides which module failed) |
| `requisite` | Failure → fails immediately, no further checks |
| `sufficient` | Success → stack passes (if no `required` already failed); failure → continue |
| `optional` | Result doesn't affect outcome unless it's the only one |

Modern PAM also uses `[default=...]` and `[success=N default=ignore]` syntax for finer control.

**The files**:

- `/etc/pam.d/*` — per-service stacks. `sshd`, `login`, `sudo`, `system-auth`, `password-auth`
- `/etc/pam.d/system-auth` and `/etc/pam.d/password-auth` — included from other stacks; centralize most of the policy
- `/etc/security/limits.conf` — ulimits set by `pam_limits`
- `/etc/security/faillock.conf` — lockout policy for `pam_faillock`

**Common modules**:

- `pam_unix` — classic password check against `/etc/shadow`
- `pam_systemd` — sets up the user session, creates `/run/user/<uid>`
- `pam_faillock` — lock account after N failed attempts
- `pam_sss` — delegate to sssd (LDAP/Kerberos)
- `pam_pkcs11` — smart card auth
- `pam_google_authenticator` — TOTP MFA
- `pam_limits` — apply ulimits from `/etc/security/limits.conf`

**Reading the logs**:

```bash
sudo journalctl -u sshd -f
sudo tail -f /var/log/secure         # auth events (RHEL)
sudo tail -f /var/log/auth.log       # auth events (Debian/Ubuntu)
```

Auth failures show up with the module that decided to fail you. Lockouts show as `pam_faillock(<service>:auth): User <user> (...) has reached the maximum number of attempts...`.

**The classic "I can't log in" tree**:

1. `journalctl -u sshd` and `/var/log/secure` — what does the log say?
2. `id <user>` — does the user exist?
3. `chage -l <user>` — is the account expired?
4. `faillock --user <user>` — is the account locked out?
5. `cat /etc/passwd /etc/shadow | grep <user>` — passwd entry sane? shadow entry not `!` or `*`?
6. `sshd -T | grep -i passwordauth\|pubkey` — is the auth method allowed in sshd config?
7. SELinux — `getenforce`, `ausearch -m AVC -ts recent`. (Yes, SELinux can block logins, especially when `~/.ssh` has wrong context.)
8. Permissions on `~/.ssh` and `~/.ssh/authorized_keys` — sshd refuses world-writable home dirs and key files.

**Unlocking a locked account**:

```bash
sudo faillock --user alice --reset
```

**Force password expiry**:

```bash
sudo chage -d 0 alice                # password expires immediately
sudo chage -l alice                  # view current status
```

### 12C. LDAP, Kerberos, and SSSD (30 min)

For enterprise environments, users come from LDAP (directory) and authenticate via Kerberos (ticketing). On RHEL the unified gateway is **SSSD** (System Security Services Daemon).

**The architectural picture**:

```
nss-related calls → nsswitch.conf → sss → sssd → LDAP server
auth-related calls → pam_sss → sssd → Kerberos KDC
```

**The high-level flow**:

```bash
# Join a domain (Active Directory or FreeIPA)
sudo realm discover example.com
sudo realm join example.com -U admin

# Verify
realm list
id alice@example.com
getent passwd alice@example.com

# Kerberos ticket lifecycle
kinit alice@EXAMPLE.COM                # get a TGT
klist                                  # show tickets
kdestroy                               # remove

# LDAP read
ldapsearch -x -H ldap://ldap.example.com -b "dc=example,dc=com" "(uid=alice)"
```

**Config files**:

- `/etc/sssd/sssd.conf` (mode 0600, root-only)
- `/etc/krb5.conf` — Kerberos realm config
- `/etc/nsswitch.conf` — `passwd: files sss`, `group: files sss`

**The Kerberos rule that bites everyone**: **time sync is mandatory**. Kerberos tickets are time-stamped; if your host's clock is off by more than ~5 minutes (the default skew), all auth fails with "Clock skew too great." Make sure `chronyd` is running and synced:

```bash
chronyc tracking
chronyc sources
```

**Debugging SSSD**:

```bash
# Logs (verbose)
sudo journalctl -u sssd
ls /var/log/sssd/                       # per-domain logs

# Increase debug level temporarily
sudo sssctl debug-level 9              # 0 (quiet) - 9 (verbose)

# Clear cache (common fix for "user exists in LDAP but id says not found")
sudo systemctl stop sssd
sudo sss_cache -E                      # invalidate
sudo rm -rf /var/lib/sss/db/*          # nuclear: wipe cache
sudo systemctl start sssd
```

**The "I can't log in with my AD account" diagnostic**:

1. `realm list` — is the domain joined?
2. `chronyc tracking` — is the clock synced?
3. `kinit user@DOMAIN` — can you get a Kerberos ticket manually?
4. `getent passwd user@domain` — does nsswitch see the user via sss?
5. `sudo journalctl -u sssd` — what's sssd saying?
6. `sudo sssctl logs-fetch` — collect logs for support.
7. If recently changed: `sudo sss_cache -E` to invalidate cache.

### 12D. Package management and RPM recovery (45 min)

`dnf` (RHEL 8+) is the front-end; `rpm` is the underlying package format and database. Things break in interesting ways.

**Everyday operations**:

```bash
dnf install <pkg>
dnf remove <pkg>
dnf update                      # update everything
dnf upgrade <pkg>               # specific package
dnf list installed | grep <pkg>
dnf info <pkg>
dnf search <keyword>
dnf provides /path/to/file      # which package provides this file
dnf history                     # list of transactions
dnf history info <id>           # what was in transaction N
dnf history undo <id>           # roll back a transaction
dnf history rollback <id>       # roll back to state at this transaction
```

**Repositories**:

```bash
dnf repolist                    # enabled repos
dnf repolist --all              # everything including disabled
dnf config-manager --add-repo <url>
dnf config-manager --enable <reponame>
dnf clean all                   # clear metadata cache (fixes many flaky-mirror issues)
```

**Module streams** (RHEL 8+):

```bash
dnf module list
dnf module enable nodejs:18
dnf module install nginx:1.20
dnf module reset nginx
```

**Dependency-hell strategies** (in order of escalation):

```bash
# 1. Refresh metadata
dnf clean all && dnf makecache

# 2. Pick "best" available versions (less strict)
dnf update --best

# 3. Sync all packages to the repo's current state
dnf distro-sync

# 4. Downgrade a specific package
dnf downgrade <pkg>

# 5. Allow removing packages to resolve conflicts (last resort)
dnf install <pkg> --allowerasing

# 6. Skip the problematic package
dnf update --skip-broken
```

**The RPM database recovery procedure** (the one interviewers test):

When the RPM database is corrupt, you see errors like:

```
error: rpmdb: BDB1507 Thread/process /pid/tid failed: BDB1505 Thread died in Berkeley DB library
error: cannot open Packages database in /var/lib/rpm
```

The recovery:

```bash
# Make a backup first — always
sudo cp -a /var/lib/rpm /var/lib/rpm.bak.$(date +%F)

# Remove the corrupt indexes (NOT the Packages file, which has the actual data)
sudo rm -f /var/lib/rpm/__db.*

# Rebuild
sudo rpm --rebuilddb

# Verify
rpm -qa | wc -l                 # should match expected package count
dnf check                       # checks for inconsistencies
```

If `Packages` itself is corrupt, you may have to restore from `/var/lib/rpm.bak` or another snapshot. RPM databases are not designed for arbitrary recovery — backups matter.

**Verifying installed files**:

```bash
rpm -V <pkg>                    # verify package: changes in size, perms, owner, etc.
rpm -Va                         # verify ALL packages — slow, but finds tampered files
rpm -V httpd                    # only check httpd's files

# Verify output: each line one mismatched file
# Columns: S.M5...... = Size, Mode, MD5, ...
# S = size changed
# M = mode (permissions) changed
# 5 = MD5 sum changed (content modified)
# T = modification time changed
# U / G = owner / group changed
# . = unchanged
```

**Find what package owns a file**:

```bash
rpm -qf /etc/ssh/sshd_config           # → openssh-server-X.Y
dnf provides /etc/ssh/sshd_config      # works for not-yet-installed packages
```

**Restore a modified config**:

```bash
# What did we change?
rpm -V openssh-server | grep '^.....T.c'    # config files (c) with changed mtime

# Reinstall to restore originals (saves modified versions as .rpmsave)
sudo dnf reinstall openssh-server
ls /etc/ssh/sshd_config*               # original restored; modified now .rpmsave
```

**GPG and signed packages**:

```bash
rpm -qa gpg-pubkey                     # installed signing keys
sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
rpm -K <package.rpm>                   # verify signature
```

GPG errors usually mean the repo key isn't trusted yet (`--import` it) or the repo is mismirrored.

**Interview answer for "the RPM database is corrupt — recover"**:

> "Back up `/var/lib/rpm` first. The indexes are the `__db.*` files — remove them and run `rpm --rebuilddb`. That rebuilds the indexes from the canonical `Packages` file. If `Packages` itself is corrupt, recovery means restoring from a backup or filesystem snapshot — that's why `/var/lib/rpm` should be part of your backup strategy. After recovery, `dnf check` confirms the database is consistent."

---

## Midday Block (2.5h) — Hands-on labs

### Lab 1: SELinux contexts and AVC debugging (45 min)

```bash
# Confirm SELinux is active
getenforce
sestatus

# Look at contexts everywhere
ls -Z /etc/passwd /var/www/html /var/log/messages
ps -ZC sshd
id -Z

# Install a web server for the experiment
sudo dnf install -y httpd
sudo systemctl enable --now httpd

# Default doc root
ls -Zd /var/www/html
# Should show httpd_sys_content_t

# Try serving from a non-standard location
sudo mkdir -p /srv/web
echo "Hello SELinux" | sudo tee /srv/web/index.html
ls -Z /srv/web/                       # default_t — wrong type
sudo sed -i 's|/var/www/html|/srv/web|g' /etc/httpd/conf/httpd.conf
sudo systemctl restart httpd
curl http://localhost/                # should fail (403 or empty)

# Find the AVC
sudo ausearch -m AVC -ts recent | tail -20
sudo sealert -a /var/log/audit/audit.log 2>/dev/null | head -30

# Diagnose: httpd_t was denied read of /srv/web/index.html which is default_t
# Fix: re-label /srv/web with httpd_sys_content_t

sudo semanage fcontext -a -t httpd_sys_content_t '/srv/web(/.*)?'
sudo restorecon -Rv /srv/web
ls -Z /srv/web/
sudo systemctl restart httpd
curl http://localhost/                # now works

# Cleanup
sudo sed -i 's|/srv/web|/var/www/html|g' /etc/httpd/conf/httpd.conf
sudo systemctl restart httpd
sudo semanage fcontext -d '/srv/web(/.*)?'
sudo rm -rf /srv/web
```

**Now the port variant** — let httpd listen on a non-default port:

```bash
# Add a Listen 8888 directive
sudo sed -i '/^Listen 80/a Listen 8888' /etc/httpd/conf/httpd.conf
sudo systemctl restart httpd          # likely fails

# Why?
sudo ausearch -m AVC -ts recent | grep name_bind | tail
# SELinux blocked binding to TCP/8888

sudo semanage port -a -t http_port_t -p tcp 8888
sudo systemctl restart httpd          # now starts
sudo semanage port -l | grep http_port_t

# Cleanup
sudo sed -i '/^Listen 8888/d' /etc/httpd/conf/httpd.conf
sudo semanage port -d -t http_port_t -p tcp 8888
sudo systemctl restart httpd
```

### Lab 2: SELinux booleans (20 min)

```bash
# List booleans
getsebool -a | head -20
getsebool -a | grep httpd | head

# A specific boolean
getsebool httpd_can_network_connect

# Try a scenario: have httpd talk to an external service (e.g., curl from CGI)
# Quick toggle (non-persistent)
sudo setsebool httpd_can_network_connect on
getsebool httpd_can_network_connect

# Make it persistent across reboots
sudo setsebool -P httpd_can_network_connect on

# View descriptions
semanage boolean -l | grep httpd_can_network_connect

# Reset
sudo setsebool -P httpd_can_network_connect off
```

### Lab 3: PAM debugging — user lockout (30 min)

```bash
# Create a test user
sudo useradd -m testuser
echo 'testuser:Password123!' | sudo chpasswd

# Verify it works
su - testuser -c 'echo "logged in as $(whoami)"; exit'

# Check the password-auth stack
sudo cat /etc/pam.d/password-auth | head -20
# Look for pam_faillock — RHEL 8+ ships it on by default

# Read the policy
sudo cat /etc/security/faillock.conf | grep -vE '^\s*#|^\s*$'

# Cause a lockout — try wrong password several times
for i in $(seq 1 5); do
  echo "Attempt $i"
  su - testuser -c exit <<< 'wrongpassword' 2>&1 | tail -1
done

# Check lockout status
sudo faillock --user testuser
# Should show several failed attempts and possibly "locked"

# Unlock
sudo faillock --user testuser --reset

# Verify it's unlocked
sudo faillock --user testuser
su - testuser -c 'echo "back in"; exit' <<< 'Password123!'

# Logs
sudo grep testuser /var/log/secure | tail -10
sudo journalctl _COMM=sshd --since "5 minutes ago"

# Cleanup
sudo userdel -r testuser
```

### Lab 4: SSSD config inspection (20 min)

Even if you're not joined to a domain, you can inspect the typical files and structure:

```bash
# Is sssd installed/running?
systemctl status sssd 2>/dev/null | head -5

# Config exists?
sudo ls -la /etc/sssd/ 2>/dev/null
sudo cat /etc/sssd/sssd.conf 2>/dev/null | head -30

# Even without a domain joined, examine these:
cat /etc/nsswitch.conf | grep -E '^passwd|^group|^hosts'
cat /etc/krb5.conf | grep -vE '^\s*#|^\s*$' | head -30

# What does the system think about Kerberos?
which klist kinit kdestroy
klist 2>&1 | head            # "No credentials cache found" is normal if not authenticated

# Time sync (critical for Kerberos)
chronyc tracking
chronyc sources

# If you wanted to join a domain, the command would be:
# sudo realm discover example.com
# sudo realm join example.com -U admin
# Then: id user@example.com
```

### Lab 5: Package recovery scenarios (30 min)

```bash
# Inspect dnf history
sudo dnf history | head

# Pick a recent transaction
sudo dnf history info $(sudo dnf history | awk 'NR==3 {print $1}')

# Verify the installed httpd from Lab 1
rpm -V httpd
# (No output = all files unchanged)

# Modify a config and verify
sudo cp /etc/httpd/conf/httpd.conf /tmp/httpd.conf.bak
echo "# my custom comment" | sudo tee -a /etc/httpd/conf/httpd.conf
rpm -V httpd | grep httpd.conf       # now shows changed
# Columns: S = size, T = time, M = mode, 5 = MD5

# Restore by reinstall
sudo dnf reinstall -y httpd
ls /etc/httpd/conf/httpd.conf*       # original restored; modified as .rpmsave
diff /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf.rpmsave

# Find owner of a file
rpm -qf /etc/httpd/conf/httpd.conf
rpm -qf /bin/ls
dnf provides /bin/ls

# Simulate RPM database "corruption" — practice the recovery
# (We're not actually corrupting; just rebuilding the indexes is harmless)
sudo cp -a /var/lib/rpm /var/lib/rpm.bak.$(date +%F)
sudo rm -f /var/lib/rpm/__db.*
sudo rpm --rebuilddb
rpm -qa | wc -l                       # count of packages
sudo dnf check
# Cleanup the safety backup if all looks good
sudo rm -rf /var/lib/rpm.bak.*

# History rollback example (DO NOT RUN unless you're sure)
# sudo dnf history rollback <id>      # rolls back to that state
sudo dnf history list | head
```

---

## Afternoon Block (1.5h) — Drills + Story #12

### Self-check (45 min)

1. Walk me through the SELinux mental model — what's a context, why does it matter, and what are the three fields you usually look at?
2. SELinux is blocking httpd from reading `/srv/data`. Walk me through the diagnosis and fix end to end.
3. Difference between `chcon` and `semanage fcontext` — which would you use in production and why?
4. What's an AVC denial, where is it logged, and what's the friendliest tool to read it?
5. The boolean `httpd_can_network_connect` — what does `-P` do in `setsebool -P`?
6. PAM stack types — what are the four and what does each control?
7. A user can SSH in normally, but `sudo` fails for them. Where do you look?
8. Walk me through the "I can't log in with my AD account" diagnostic tree.
9. Kerberos auth is failing with "Clock skew too great." Cause and fix?
10. The RPM database is corrupt. Walk me through recovery, step by step.
11. A config file at `/etc/sshd_config` was changed by someone. How do you confirm and restore?
12. `dnf install nginx` fails with a dependency conflict. What's your escalation path?

<details>
<summary>Answers</summary>

1. SELinux is Mandatory Access Control on top of standard Unix perms. Every process and object has a context: `user:role:type:level`. For the **targeted** policy (RHEL default), the **type** is what matters in 95% of cases. The check is: "can a process labeled X access an object labeled Y in mode Z?" Both standard perms AND SELinux must allow.
2. (a) `getenforce` to confirm enforcing. (b) `sudo ausearch -m AVC -ts recent` to see the denial. (c) `sudo sealert -a /var/log/audit/audit.log` for human-friendly explanations. (d) Diagnose: probably wrong type label on `/srv/data`. (e) Fix: `semanage fcontext -a -t httpd_sys_content_t '/srv/data(/.*)?'` then `restorecon -Rv /srv/data`. (f) Verify with `curl`.
3. `chcon` sets the context temporarily — gets reset on the next `restorecon` (which can happen automatically, e.g. during package updates or relabel-on-boot). `semanage fcontext` adds a persistent rule to the policy. **Production**: always `semanage fcontext` for anything that should survive.
4. AVC = Access Vector Cache denial. Logged in `/var/log/audit/audit.log`. Read with `ausearch -m AVC`. Friendliest: `sealert -a /var/log/audit/audit.log` — translates to plain English with suggested fixes.
5. `-P` makes the boolean change persistent across reboots (writes to policy store, not just runtime). Without `-P`, it's runtime only and resets on reboot.
6. `auth` (authentication: prove you're you), `account` (account state: locked, expired, allowed?), `password` (password change rules), `session` (per-login setup/teardown like home mount, ulimits, audit start).
7. `/var/log/secure` for sudo events. `sudo -lU <user>` to see what they're allowed. `/etc/sudoers.d/*` and `visudo -c` for syntax errors. Group membership (`id <user>`) — they may have been removed from wheel. PAM stack for sudo (`/etc/pam.d/sudo`) if non-default.
8. `realm list` (joined?) → `chronyc tracking` (clock synced?) → `kinit user@DOMAIN` (Kerberos ticket?) → `getent passwd user@domain` (nsswitch sees them?) → `journalctl -u sssd` (sssd happy?) → `sss_cache -E` to invalidate cache if recently changed.
9. Host clock differs from KDC by more than the allowed skew (default 5 min). Kerberos rejects timestamps outside that window to prevent replay. Fix: `systemctl status chronyd`, `chronyc sources`, force sync if needed. Production hosts should always run NTP.
10. (a) Back up `/var/lib/rpm` first: `cp -a /var/lib/rpm /var/lib/rpm.bak`. (b) Remove the indexes: `rm -f /var/lib/rpm/__db.*`. (c) Rebuild: `rpm --rebuilddb`. (d) Verify: `rpm -qa | wc -l` matches expectations, then `dnf check`. If `Packages` itself is corrupt, restore from backup or snapshot — there's no automatic recovery for the data file.
11. `rpm -V openssh-server` — output column flags show what changed (S=size, T=mtime, 5=MD5, etc.). Restore by reinstall: `sudo dnf reinstall openssh-server`. The previous version is saved as `.rpmsave`. Diff to confirm what the operator added.
12. `dnf clean all && dnf makecache` (refresh metadata) → `dnf update --best` → `dnf install nginx --best --allowerasing` (allow removal to resolve) → `dnf distro-sync` (align everything to current repo state) → as a last resort, check module streams (`dnf module list nginx`) and reset/enable the right one. If a third-party repo is the source of conflict, disable it for the install: `--disablerepo=epel-testing`.

</details>

### Behavioral (45 min) — Story #12: Have Backbone; Disagree and Commit

Today's LP. Story prompt:

> "Tell me about a time you disagreed with a decision your team or leader made. What did you do, and how did it turn out?"

This LP has **two halves** and most candidates fumble one of them:

**Half 1 — Backbone**: You raised a real disagreement, in writing or directly, with someone who outranked you or held the consensus. Not a passive-aggressive grumble. Not a thing you brought up only after it failed.

**Half 2 — Commit**: When the decision went against you, you *committed* — fully, publicly, and effectively. You didn't sandbag, "I told you so" later, or do half-hearted execution.

The single most common failure mode in this story: candidates describe Half 1, then either (a) the decision went their way (so Commit isn't tested), or (b) Half 2 reads like sulking.

**Strong shape**:

- A specific decision with real stakes
- Your disagreement, in concrete terms — what was the alternative you proposed?
- The forum: how you raised it, to whom, with what data
- The decision went the other way
- **Your behavior after**: visible, energetic execution of the decision you'd argued against
- The outcome — sometimes you were right, sometimes you were wrong, sometimes ambiguous. All three can land if you handle them honestly.

**The honest version of "I was wrong"** lands very well — it shows judgment and self-awareness without undermining the LP. Don't manufacture it, but if you have it, use it.

STAR. 2-3 minutes. Out loud. Time it.
