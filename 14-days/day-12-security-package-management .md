# 🔒 Day 12 — Security & Package Management

> [!NOTE]
> **The Goal:** Two themes today, both very high-frequency interview territory:
> 1. **Security**: SELinux is the topic candidates fear most and where interviewers separate the prepared from the unprepared. Plus PAM (the "I can't log in" tree) and LDAP/Kerberos.
> 2. **Package management**: dnf/RPM is mundane until the database corrupts. The recovery procedure is short, surprising, and they will ask about it.

---

## 🧠 Morning Block (3h) — Concepts

### 12A. SELinux fundamentals (1h)

The mental model that makes everything else click:

> [!TIP]
> SELinux is **Mandatory Access Control** layered on top of standard Unix permissions. Standard perms say "user X can read this file." SELinux additionally says "process labeled with type `httpd_t` can read files labeled with type `httpd_sys_content_t`." Both must allow the access. Either can deny.

Every subject (process) and object (file, socket, port, etc.) has a **security context**:

```text
system_u:object_r:httpd_sys_content_t:s0
   |        |          |               |
   user    role       type            level (MLS)
```

For the **targeted policy** (RHEL/Fedora default), the only field that usually matters is **type**. 

**Modes**:
- **enforcing** — Policy is enforced. Denials block the action and are logged.
- **permissive** — Policy is evaluated. Denials are logged but **not blocked**.
- **disabled** — SELinux is off. Requires reboot to change to/from.

```bash
getenforce                    # enforcing / permissive / disabled
sudo setenforce 0             # switch to permissive (runtime only)
sudo setenforce 1             # back to enforcing
```

**AVC denials — the single most important concept**:
When SELinux blocks something, it writes an Access Vector Cache (AVC) denial. **The denial tells you exactly what to fix.**

```bash
sudo ausearch -m AVC -ts recent
sudo sealert -a /var/log/audit/audit.log   # A friendly translator
```

A denial looks like:
```text
type=AVC msg=audit(...): avc: denied { read } for pid=1234 comm="httpd"
  name="secret.txt" dev="dm-0" ino=12345
  scontext=system_u:system_r:httpd_t:s0
  tcontext=unconfined_u:object_r:user_home_t:s0
  tclass=file permissive=0
```
Read it as: **`httpd_t`** was denied `{ read }` access to a **`user_home_t`** file.

**The fix is almost always one of three things:**
1. **Wrong label** — `restorecon` to fix it back to default.
2. **Need a boolean** — `setsebool` to enable an optional policy.
3. **Custom path** — `semanage fcontext` to teach SELinux a new path pattern, then `restorecon`.

> [!IMPORTANT]
> **The canonical interview answer:**
> "First confirm with `getenforce` and check `ausearch -m AVC -ts recent` for the actual denial. If the file should be served by httpd, it needs the right type label — usually `httpd_sys_content_t`. I'd add a persistent rule with `semanage fcontext -a -t httpd_sys_content_t '/srv/web(/.*)?'` and apply with `restorecon -Rv /srv/web`. 
> 
> If httpd needs a feature like network connect, `setsebool -P httpd_can_network_connect on`. As a last resort, I'd build a policy module with `audit2allow`, but defaults plus booleans plus correct labels cover ~99% of real cases."

**A common interview gotcha**: "I changed the file's context with `chcon` and the change disappeared." Because something ran `restorecon` which reset it to the default. Use `semanage fcontext` for persistence.

### 12B. PAM and authentication (45 min)

PAM = Pluggable Authentication Modules. Every authenticating service consults `/etc/pam.d/<service>` to decide who gets in.

**Module categories**:
- `auth` — Are you who you say you are? (passwords, tokens, keys)
- `account` — Is your account allowed to log in right now? (expiry, time of day, lockouts)
- `password` — Allow changing passwords (complexity, history)
- `session` — Setup/teardown around the session (mount home dir, ulimits)

**Control flags**:
- `required` — Failure → still fails, but continues evaluating (hides which module failed).
- `requisite` — Failure → fails immediately, no further checks.
- `sufficient` — Success → stack passes; failure → continue.

**The classic "I can't log in" tree**:
1. `journalctl -u sshd` and `/var/log/secure` — what does the log say?
2. `id <user>` — does the user exist?
3. `chage -l <user>` — is the account expired?
4. `faillock --user <user>` — is the account locked out?
5. `cat /etc/passwd /etc/shadow | grep <user>` — passwd entry sane? shadow entry not `!` or `*`?
6. `sshd -T | grep -i passwordauth\|pubkey` — is the auth method allowed in sshd config?
7. SELinux — `ausearch -m AVC -ts recent`. (SELinux blocks logins if `~/.ssh` has the wrong context.)
8. Permissions on `~/.ssh` and `~/.ssh/authorized_keys` — sshd refuses world-writable home dirs.

### 12C. LDAP, Kerberos, and SSSD (30 min)

For enterprise environments, users come from LDAP (directory) and authenticate via Kerberos (ticketing). On RHEL the unified gateway is **SSSD**.

**The Kerberos rule that bites everyone**: **time sync is mandatory**. Kerberos tickets are time-stamped; if your host's clock is off by more than ~5 minutes, all auth fails with "Clock skew too great." Make sure `chronyd` is running.

**Debugging SSSD**:
```bash
sudo sss_cache -E                      # invalidate cache (fixes "user exists in LDAP but id says not found")
sudo rm -rf /var/lib/sss/db/*          # nuclear: wipe cache
```

### 12D. Package management and RPM recovery (45 min)

**Dependency-hell strategies** (in order of escalation):
1. Refresh metadata: `dnf clean all && dnf makecache`
2. Pick "best" available versions: `dnf update --best`
3. Sync all packages to the repo's current state: `dnf distro-sync`
4. Downgrade a specific package: `dnf downgrade <pkg>`
5. Allow removing packages to resolve conflicts (last resort): `dnf install <pkg> --allowerasing`

> [!WARNING]
> **The RPM database recovery procedure** (the one interviewers test):
> When the RPM database is corrupt, you see errors like `BDB1507 Thread died in Berkeley DB library`.
> 
> **The recovery:**
> 1. Make a backup first: `sudo cp -a /var/lib/rpm /var/lib/rpm.bak`
> 2. Remove the corrupt indexes (NOT the Packages file!): `sudo rm -f /var/lib/rpm/__db.*`
> 3. Rebuild: `sudo rpm --rebuilddb`
> 4. Verify: `dnf check`

**Verifying installed files**:
```bash
rpm -V <pkg>                    # verify package: changes in size, perms, owner, etc.
rpm -qf /etc/ssh/sshd_config    # Find what package owns a file
```

---

## 💻 Midday Block (2.5h) — Hands-on labs

### Lab 1: SELinux contexts and AVC debugging (45 min)

```bash
# Confirm SELinux is active
getenforce

# Install a web server for the experiment
sudo dnf install -y httpd
sudo systemctl enable --now httpd

# Default doc root
ls -Zd /var/www/html             # Should show httpd_sys_content_t

# Try serving from a non-standard location
sudo mkdir -p /srv/web
echo "Hello SELinux" | sudo tee /srv/web/index.html
ls -Z /srv/web/                  # default_t — wrong type
sudo sed -i 's|/var/www/html|/srv/web|g' /etc/httpd/conf/httpd.conf
sudo systemctl restart httpd
curl http://localhost/           # should fail (403 or empty)

# Find the AVC
sudo ausearch -m AVC -ts recent | tail -20
sudo sealert -a /var/log/audit/audit.log 2>/dev/null | head -30

# Fix: re-label /srv/web with httpd_sys_content_t
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/web(/.*)?'
sudo restorecon -Rv /srv/web
sudo systemctl restart httpd
curl http://localhost/           # now works
```

**Now the port variant** — let httpd listen on a non-default port:
```bash
sudo sed -i '/^Listen 80/a Listen 8888' /etc/httpd/conf/httpd.conf
sudo systemctl restart httpd          # likely fails

sudo semanage port -a -t http_port_t -p tcp 8888
sudo systemctl restart httpd          # now starts
```

### Lab 2: SELinux booleans (20 min)

```bash
getsebool -a | grep httpd | head

# Quick toggle (non-persistent)
sudo setsebool httpd_can_network_connect on

# Make it persistent across reboots
sudo setsebool -P httpd_can_network_connect on
```

### Lab 3: PAM debugging — user lockout (30 min)

```bash
# Create a test user
sudo useradd -m testuser
echo 'testuser:Password123!' | sudo chpasswd

# Cause a lockout — try wrong password several times
for i in $(seq 1 5); do
  echo "Attempt $i"
  su - testuser -c exit <<< 'wrongpassword' 2>&1 | tail -1
done

# Check lockout status
sudo faillock --user testuser

# Unlock
sudo faillock --user testuser --reset

# Verify it's unlocked
su - testuser -c 'echo "back in"; exit' <<< 'Password123!'
```

### Lab 4: Package recovery scenarios (30 min)

```bash
# Verify the installed httpd from Lab 1
rpm -V httpd                     # (No output = all files unchanged)

# Modify a config and verify
sudo cp /etc/httpd/conf/httpd.conf /tmp/httpd.conf.bak
echo "# my custom comment" | sudo tee -a /etc/httpd/conf/httpd.conf
rpm -V httpd | grep httpd.conf       # now shows changed (S, T, 5)

# Restore by reinstall
sudo dnf reinstall -y httpd
ls /etc/httpd/conf/httpd.conf* # original restored; modified as .rpmsave
diff /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf.rpmsave

# Simulate RPM database recovery
sudo cp -a /var/lib/rpm /var/lib/rpm.bak.$(date +%F)
sudo rm -f /var/lib/rpm/__db.*
sudo rpm --rebuilddb
sudo dnf check
sudo rm -rf /var/lib/rpm.bak.* # Cleanup
```

---

## 🎯 Afternoon Block (1.5h) — Drills + Story #12

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
<summary><strong>Answers</strong> (click to reveal)</summary>

1. SELinux is Mandatory Access Control on top of standard Unix perms. Every process and object has a context: `user:role:type:level`. For the **targeted** policy (RHEL default), the **type** is what matters in 95% of cases. Both standard perms AND SELinux must allow.
2. (a) `getenforce` to confirm enforcing. (b) `sudo ausearch -m AVC -ts recent` to see the denial. (c) `sudo sealert -a /var/log/audit/audit.log` for human-friendly explanations. (d) Diagnose: probably wrong type label on `/srv/data`. (e) Fix: `semanage fcontext -a -t httpd_sys_content_t '/srv/data(/.*)?'` then `restorecon -Rv /srv/data`.
3. `chcon` sets the context temporarily — gets reset on the next `restorecon`. `semanage fcontext` adds a persistent rule to the policy. **Production**: always `semanage fcontext` for anything that should survive.
4. AVC = Access Vector Cache denial. Logged in `/var/log/audit/audit.log`. Read with `ausearch -m AVC`. Friendliest: `sealert -a /var/log/audit/audit.log` — translates to plain English with suggested fixes.
5. `-P` makes the boolean change persistent across reboots. Without `-P`, it's runtime only and resets on reboot.
6. `auth` (prove you're you), `account` (account state: locked, expired?), `password` (password change rules), `session` (per-login setup/teardown like home mount, ulimits, audit start).
7. `/var/log/secure` for sudo events. `sudo -lU <user>` to see what they're allowed. `/etc/sudoers.d/*` and `visudo -c` for syntax errors. Group membership (`id <user>`) — they may have been removed from wheel. 
8. `realm list` (joined?) → `chronyc tracking` (clock synced?) → `kinit user@DOMAIN` (Kerberos ticket?) → `getent passwd user@domain` (nsswitch sees them?) → `journalctl -u sssd` (sssd happy?) → `sss_cache -E` to invalidate cache.
9. Host clock differs from KDC by more than the allowed skew (default 5 min). Fix: `systemctl status chronyd`, force sync if needed. 
10. (a) Back up `/var/lib/rpm`: `cp -a /var/lib/rpm /var/lib/rpm.bak`. (b) Remove the indexes: `rm -f /var/lib/rpm/__db.*`. (c) Rebuild: `rpm --rebuilddb`. (d) Verify: `dnf check`. 
11. `rpm -V openssh-server` — output column flags show what changed. Restore by reinstall: `sudo dnf reinstall openssh-server`. The previous version is saved as `.rpmsave`. 
12. `dnf clean all && dnf makecache` → `dnf update --best` → `dnf install nginx --best --allowerasing` → `dnf distro-sync`. 

</details>

### 🤝 Behavioral (45 min) — Story #12: Have Backbone; Disagree and Commit

Today's LP: **Have Backbone; Disagree and Commit**. 

Prompt:
> "Tell me about a time you disagreed with a decision your team or leader made. What did you do, and how did it turn out?"

> [!TIP]
> This LP has **two halves** and most candidates fumble one of them:
> 
> **Half 1 — Backbone**: You raised a real disagreement, in writing or directly, with someone who outranked you or held the consensus. Not a passive-aggressive grumble. Not a thing you brought up only after it failed.
> 
> **Half 2 — Commit**: When the decision went against you, you *committed* — fully, publicly, and effectively. You didn't sandbag, "I told you so" later, or do half-hearted execution.

The single most common failure mode in this story: candidates describe Half 1, then either (a) the decision went their way (so Commit isn't tested), or (b) Half 2 reads like sulking.

STAR. 2-3 minutes. Out loud. Time it.
