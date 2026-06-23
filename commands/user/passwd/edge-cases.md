# passwd — Edge Cases & Gotchas

> Security-sensitive command with subtle traps that matter in production.

---

## Table of Contents

- [SUID Surprises](#suid-surprises)
- [Locking vs Disabling: Not the Same](#locking-vs-disabling-not-the-same)
- [Passwordless Accounts](#passwordless-accounts)
- [/etc/shadow Field Traps](#etcshadow-field-traps)
- [PAM Policy vs passwd -x](#pam-policy-vs-passwd--x)
- [Non-interactive Scripting Traps](#non-interactive-scripting-traps)
- [Race Conditions & File Locking](#race-conditions--file-locking)
- [Hash Algorithm Pitfalls](#hash-algorithm-pitfalls)
- [Environment & PATH Risks](#environment--path-risks)
- [chpasswd Security Risks](#chpasswd-security-risks)
- [GNU vs BSD vs RHEL Differences](#gnu-vs-bsd-vs-rhel-differences)

---

## SUID Surprises

### passwd runs as root — even when called by alice
```bash
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ...
#     ↑ SUID: effective UID = root

# Inside passwd:
# getuid()  → 1001  (alice — real UID, who actually ran it)
# geteuid() → 0     (root  — effective UID due to SUID)

# passwd uses getuid() to decide what's allowed:
# getuid() != 0 → must provide current password, can only change own
# getuid() == 0 → skip current password check, can change anyone's
```

### Dropping SUID breaks passwd for everyone
```bash
# If SUID bit is accidentally removed:
chmod u-s /usr/bin/passwd

# Now regular users can't change their own password:
passwd
# passwd: Authentication token manipulation error

# Fix:
chmod u+s /usr/bin/passwd
# or:
chmod 4755 /usr/bin/passwd
```

### passwd is not the only SUID tool — audit regularly
```bash
# Find all SUID binaries on system (security audit):
find / -type f -perm -4000 -ls 2>/dev/null

# Expected on a clean system — any extras are suspicious:
# /usr/bin/passwd
# /usr/bin/sudo
# /usr/bin/su
# /usr/bin/newgrp
# /usr/bin/chsh
# /usr/bin/chfn
# /usr/bin/gpasswd
```

---

## Locking vs Disabling: Not the Same

### `-l` (lock) only blocks password login
```bash
sudo passwd -l alice

# alice CANNOT: log in with password
# alice CAN STILL:
#   - log in with SSH key
#   - be su'd to by root: sudo su - alice
#   - run cron jobs
#   - have running processes

# To fully disable an account, combine approaches:
sudo passwd -l alice                          # lock password
sudo usermod -e 1 alice                       # set account expiry to past date
sudo usermod -s /sbin/nologin alice           # disable shell
# Now truly inaccessible
```

### `!` vs `!!` vs `*` in /etc/shadow
```bash
sudo grep "^alice:" /etc/shadow | cut -d: -f2

# $6$salt$hash      → normal, has password
# !$6$salt$hash     → locked (passwd -l), but hash preserved — unlock restores it
# !!                → never had a password set (new account created without passwd)
# !                 → locked with no hash (can't be unlocked with passwd -u alone)
# *                 → system account, no password login (e.g., www-data)
#                   → empty string = no password required (DANGEROUS)
```

### passwd -u fails if no hash exists
```bash
sudo passwd -l alice       # lock: changes $hash to !$hash
sudo passwd -u alice       # unlock: removes ! → works ✅

# But if account was created with !! (no password ever set):
sudo passwd -u newuser     # may fail or warn:
# passwd: unlocking the password would result in a passwordless account.
# You should set a password with usermod -p to unlock this account.

# Fix: set a password first, then unlock is unnecessary
sudo passwd newuser
```

---

## Passwordless Accounts

### Empty password field = login without password
```bash
# If /etc/shadow has empty password field:
# alice::19523:0:99999:7:::
#        ↑ empty = no password needed!

# Anyone can log in as alice without typing anything
# This is a critical security risk

# Check for passwordless accounts:
sudo awk -F: '($2 == "") {print $1}' /etc/shadow

# Fix: set a password or lock
sudo passwd alice
# or:
sudo passwd -l alice   # if account shouldn't log in at all
```

### `-d` (delete password) creates passwordless account
```bash
sudo passwd -d alice
# Removes alice's password — alice can now log in without a password
# on systems that allow this (depends on PAM config)

# PAM may block empty password login:
# /etc/pam.d/sshd: auth required pam_unix.so nullok  ← allows empty
# /etc/pam.d/sshd: auth required pam_unix.so         ← blocks empty

# Only use -d for accounts that will use other auth (SSH keys, etc.)
# and only if PAM blocks password login for empty hash
```

---

## /etc/shadow Field Traps

### Date fields are days since epoch (Jan 1, 1970)
```bash
sudo grep "^alice:" /etc/shadow
# alice:$6$...:19523:0:99999:7:::
#               ↑ last change = day 19523 since epoch

# Convert to human date:
date -d "1970-01-01 + 19523 days"
# Fri Jun 14 00:00:00 UTC 2023

# 0 in last-change = expired immediately (force change at next login)
# This is what passwd -e sets it to
```

### Field 9 (max days) value 99999 = effectively never
```bash
# 99999 days ≈ 273 years
# This is the conventional "never expires" value
# Not the same as -1 (which means disable expiry entirely)

# In /etc/shadow:
# field 5 = 99999  → expires in 273 years (functionally never)
# field 5 = -1     → expiry disabled (not all tools use this)
# field 5 = empty  → no expiry policy set (inherits system default)
```

### Corrupted /etc/shadow = no one can log in
```bash
# /etc/shadow is the single point of failure for login
# Corruption = system lockout

# Always backup before bulk edits:
sudo cp /etc/shadow /etc/shadow.bak

# Use vipw/vigr for safe editing (locks the file):
sudo vipw -s     # edit /etc/shadow safely

# If you break it:
# Boot to recovery mode → mount as rw → restore backup → reboot
```

---

## PAM Policy vs passwd -x

### PAM can override passwd aging settings
```bash
# passwd -x 90 alice  sets /etc/shadow field 5 to 90
# BUT if PAM has its own policy (e.g., pam_pwquality minlen=12)
# and the new password doesn't meet it, the change is rejected

# Also: some systems use sssd or LDAP — /etc/shadow may not be used at all
# passwd may appear to work but changes are ignored

# Check if system uses LDAP/SSSD:
grep "passwd:" /etc/nsswitch.conf
# passwd: files systemd sss   ← 'sss' means sssd is in play
# passwd: files ldap          ← ldap
# passwd: files               ← only /etc/passwd and /etc/shadow
```

### pam_pwhistory blocks reusing old passwords
```bash
# If /etc/pam.d/common-password has:
# password required pam_pwhistory.so remember=5

# User cannot reuse last 5 passwords
# This check is enforced by PAM, not by passwd itself
# History stored in /etc/security/opasswd

sudo cat /etc/security/opasswd
# alice:1001:5:$6$hash1,$6$hash2,$6$hash3,...

# Root can bypass this by using chpasswd or usermod -p (sets hash directly)
```

---

## Non-interactive Scripting Traps

### passwd --stdin is RHEL/Fedora only
```bash
# On RHEL/Fedora/CentOS:
echo "newpassword" | sudo passwd --stdin alice    # ✅ works

# On Debian/Ubuntu:
echo "newpassword" | sudo passwd --stdin alice    # ❌ --stdin not available

# Universal alternative:
echo "alice:newpassword" | sudo chpasswd          # ✅ works everywhere
```

### expect scripts for passwd are fragile
```bash
# Using expect to automate passwd — works but is fragile:
expect << 'EOF'
spawn passwd alice
expect "New password:"
send "newpass123\n"
expect "Retype new password:"
send "newpass123\n"
expect eof
EOF
# ⚠️ passwd output messages differ by distro/locale
# Use chpasswd instead — it's designed for scripting
```

### Plaintext password in command history
```bash
# NEVER do this — appears in ps output and shell history:
echo "mypassword" | passwd --stdin alice    # visible in ps aux!
usermod -p "plaintext" alice                # even worse

# Safe alternatives:
echo "alice:$(generate_password)" | chpasswd
# or read from a secure file:
chpasswd < /root/secure_passwords.txt
shred -u /root/secure_passwords.txt   # securely delete after use
```

### Password in environment variable
```bash
# Risky — environment variables can leak:
PASSWORD="secret" chpasswd   # ❌ not how chpasswd works
export PASSWORD=secret; echo "alice:$PASSWORD" | chpasswd  # env visible in /proc

# Safer: use a file descriptor or heredoc
chpasswd << 'HEREDOC'
alice:newpassword123
HEREDOC
```

---

## Race Conditions & File Locking

### /etc/shadow is locked during write
```bash
# passwd uses file locking (/etc/shadow.lock or fcntl) during update
# Concurrent passwd runs queue up

# If passwd crashes mid-write, /etc/.pwd.lock may remain:
ls /etc/.pwd.lock   # if this exists and no passwd is running, it's stale
sudo rm /etc/.pwd.lock   # remove stale lock (carefully!)

# Always check for running passwd before removing lock:
pgrep passwd        # should be empty if no passwd is running
```

### Simultaneous passwd runs
```bash
# Two admins changing passwords simultaneously:
# passwd 1: reads shadow → modifies alice's entry → writes shadow
# passwd 2: reads shadow → modifies bob's entry → writes shadow
# Last write wins — may overwrite the other's change

# This is handled by shadow file locking:
# passwd acquires an exclusive lock before writing
# Second run waits until first completes
```

---

## Hash Algorithm Pitfalls

### Old MD5 hashes still work but are insecure
```bash
# If /etc/shadow has $1$ (MD5) hashes from old systems:
sudo grep "^\$1\$" /etc/shadow   # check for MD5

# md5 is broken — rainbow tables exist for common passwords
# Fix: force password reset so users get modern SHA-512/yescrypt
sudo passwd -e username    # force change at next login
```

### Hash is visible to root — plan accordingly
```bash
# Root can read /etc/shadow and attempt offline cracking
# Use strong hashing (yescrypt or SHA-512 with high rounds)

# Check rounds for SHA-512:
sudo grep "alice" /etc/shadow | grep -oP "rounds=\d+"
# rounds=5000  ← default (consider increasing to 100000+)

# Set higher rounds in /etc/pam.d/common-password:
# password ... pam_unix.so sha512 rounds=100000

# Or in /etc/login.defs:
# SHA_CRYPT_MIN_ROUNDS 100000
# SHA_CRYPT_MAX_ROUNDS 100000
```

### Algorithm mismatch after migration
```bash
# After upgrading algorithm (e.g., SHA-512 → yescrypt):
# Existing users keep their old hash ($6$) until they change password
# New users get the new algorithm ($y$)

# Check mixed hashes:
sudo awk -F: 'NR>1 {print $1, substr($2,1,3)}' /etc/shadow | grep -v "^#\|^\*\|^!"
# alice $6$    ← old SHA-512
# bob   $y$    ← new yescrypt
```

---

## Environment & PATH Risks

### PAM modules can be replaced
```bash
# If an attacker replaces /lib/security/pam_unix.so:
# passwd would call the malicious module
# All password changes could be intercepted

# Verify PAM module integrity:
dpkg -V passwd      # Debian: check file integrity
rpm -V shadow-utils # RHEL: check file integrity
```

### passwd uses $PATH for some operations
```bash
# On some systems, passwd calls external programs (helper binaries)
# If PATH is manipulated:
PATH="/tmp:$PATH" passwd alice   # ⚠️ could call malicious binary from /tmp

# Modern passwd helpers use absolute paths to avoid this
# But always sanitize environment when running suid programs in scripts
```

---

## GNU vs BSD vs RHEL Differences

| Feature | Debian/Ubuntu | RHEL/Fedora | macOS |
|---------|--------------|-------------|-------|
| `--stdin` | ❌ | ✅ | ❌ |
| `-l` lock | ✅ | ✅ | Limited |
| `-e` expire | ✅ | ✅ | ❌ |
| `-S` status | ✅ | ✅ | ❌ |
| PAM config | `/etc/pam.d/` | `/etc/pam.d/` | `/etc/pam.d/` |
| Shadow file | `/etc/shadow` | `/etc/shadow` | `/etc/master.passwd` |
| Hash default | yescrypt (22.04+) | SHA-512 | SHA-512 |
| `chpasswd` | ✅ | ✅ | ❌ |

```bash
# macOS uses dscl instead of passwd for most operations:
dscl . -passwd /Users/alice newpassword
dscl . -read /Users/alice Password

# On macOS, /etc/passwd and /etc/shadow are not the primary store
# Directory Services (opendirectoryd) manages accounts
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
