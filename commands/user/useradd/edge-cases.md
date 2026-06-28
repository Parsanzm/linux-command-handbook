# useradd — Edge Cases & Gotchas

> useradd is deceptively simple but has many silent failure modes,
> security pitfalls, and distro-specific differences that matter in production.

---

## Table of Contents

- [Distro Differences: useradd vs adduser](#distro-differences-useradd-vs-adduser)
- [Home Directory Not Created by Default](#home-directory-not-created-by-default)
- [Password is NOT Set After useradd](#password-is-not-set-after-useradd)
- [UID/GID Allocation Surprises](#uidgid-allocation-surprises)
- [Group Membership Traps](#group-membership-traps)
- [Shell Validation Pitfalls](#shell-validation-pitfalls)
- [Skeleton Files & Permissions](#skeleton-files--permissions)
- [File Locking & Concurrent Modifications](#file-locking--concurrent-modifications)
- [Username Validation Rules](#username-validation-rules)
- [Security Pitfalls](#security-pitfalls)
- [NFS & UID Consistency](#nfs--uid-consistency)
- [System Account Traps](#system-account-traps)
- [Incomplete Cleanup After Failed useradd](#incomplete-cleanup-after-failed-useradd)

---

## Distro Differences: useradd vs adduser

### Debian/Ubuntu: `-m` is NOT the default for useradd
```bash
# On Debian/Ubuntu:
useradd alice
ls /home/alice    # ❌ directory not created!

# Must use -m explicitly:
useradd -m alice

# OR use adduser (Debian's wrapper — always creates home):
adduser alice     # interactive, always creates home, sets password

# On RHEL/Fedora:
useradd alice     # ✅ home IS created by default (CREATE_HOME yes in login.defs)
ls /home/alice    # exists
```

**Check your distro's default:**
```bash
grep "^CREATE_HOME" /etc/login.defs
useradd -D | grep HOME
```

### Default shell differs
```bash
# Debian/Ubuntu default: /bin/sh (not bash!)
useradd -m alice
grep "^alice:" /etc/passwd | cut -d: -f7
# /bin/sh   ← on Debian

# RHEL/Fedora default: /bin/bash
# /bin/bash  ← on RHEL

# Always specify shell explicitly for portability:
useradd -m -s /bin/bash alice
```

### adduser (Debian) vs useradd differences
```bash
# adduser (Perl script wrapper):
adduser alice
# - Prompts for password interactively
# - Prompts for GECOS (name, room, phone)
# - Always creates home directory
# - Always copies /etc/skel
# - Adds to groups interactively
# - More user-friendly output

# useradd (C program, low-level):
useradd -m -s /bin/bash alice
# - No prompts (fully non-interactive)
# - No password set (must run passwd separately)
# - Precise, scriptable
# - Same binary on all Linux distros
```

---

## Home Directory Not Created by Default

```bash
# Silent failure: user created but no home directory
useradd alice           # no -m flag
su - alice              # login works but:
# su: warning: cannot change directory to /home/alice: No such file or directory
# alice@host:/$ pwd
# /                    ← lands at root!

# Fix for existing user:
mkdir /home/alice
cp -r /etc/skel/. /home/alice/
chown -R alice:alice /home/alice
chmod 755 /home/alice

# OR use mkhomedir (PAM module):
pam_mkhomedir   # creates home on first login if it doesn't exist
# Add to /etc/pam.d/common-session:
# session optional pam_mkhomedir.so skel=/etc/skel umask=0022
```

---

## Password is NOT Set After useradd

```bash
# After useradd, the shadow entry has '!!' = no password set = locked
sudo grep "^alice:" /etc/shadow
# alice:!!:19523:0:99999:7:::
#        ↑↑ no password

# Alice CANNOT log in until a password is set:
passwd alice          # set password (root)
# OR force alice to set it at first login:
passwd -e alice       # expire = force change at next login

# ⚠️ Common mistake: creating a user and forgetting to set a password
# The account exists but is silently unusable
```

### Setting password at creation (risky)
```bash
# -p takes a pre-hashed password — NOT plaintext
useradd -p "plaintext" alice    # ❌ WRONG: "plaintext" stored as hash!
                                # login will never work

# Correct: generate hash first
useradd -p "$(openssl passwd -6 'actualpassword')" -m alice   # ✅
useradd -p "$(python3 -c "import crypt; print(crypt.crypt('pass', crypt.mksalt(crypt.METHOD_SHA512)))")" -m alice

# Security risk: hash visible in ps output during creation!
# Better: useradd -m alice && echo "alice:password" | chpasswd
```

---

## UID/GID Allocation Surprises

### UID recycling risk
```bash
# If you delete a user and create a new one, the new one may get the old UID
userdel alice          # alice was uid=1001
useradd bob            # bob may get uid=1001!

# bob now owns all of alice's old files (if not deleted):
ls -ln /old/alice/files
# -rw-r--r-- 1 1001 1001 ... oldfile   ← now appears owned by bob!

# Prevention: always use userdel -r to remove home, or manually clean
# And audit orphaned files after deletion:
find / -nouser -o -nogroup 2>/dev/null
```

### UID conflicts with NFS/LDAP
```bash
# If using NFS or LDAP, UIDs must be consistent across machines
# useradd auto-assigns next available UID — may conflict with network UIDs

# Check what UIDs are in use from all sources:
getent passwd | awk -F: '{print $3}' | sort -n

# Always specify UID explicitly in multi-machine environments:
useradd -u 1500 alice   # same UID on all machines
```

### System UID range exhaustion
```bash
# System accounts (useradd -r) use SYS_UID_MIN to SYS_UID_MAX (default 201-999)
# If you create many system accounts, you can run out

grep "^SYS_UID" /etc/login.defs
# SYS_UID_MIN  201
# SYS_UID_MAX  999

# Check usage:
getent passwd | awk -F: '$3 >= 201 && $3 <= 999 {print $3, $1}' | sort -n

# Expand range if needed:
# Edit /etc/login.defs: SYS_UID_MAX 1999
```

---

## Group Membership Traps

### -G replaces, usermod -aG appends
```bash
useradd -m -G docker,sudo alice    # creation: alice in docker AND sudo

# Later modification — WRONG:
usermod -G audio alice    # ❌ REMOVES docker and sudo! Only audio remains

# Correct:
usermod -aG audio alice   # ✅ -a = append: adds audio, keeps others
```

### New group membership requires re-login
```bash
usermod -aG docker alice
# alice is now in docker group, but if alice is already logged in:
su - alice
groups    # docker NOT listed yet! (old session)

# alice must log out and back in for group change to take effect
# Or use newgrp to switch in current session:
newgrp docker    # activate new group membership in current shell
```

### Primary group vs supplementary groups
```bash
# Files created by a user belong to their PRIMARY group
useradd -m -g staff alice      # primary group = staff
touch /home/alice/newfile
ls -l /home/alice/newfile
# -rw-r--r-- 1 alice staff ...   ← group is 'staff' not 'alice'

# Default: useradd creates a private group (alice:alice)
useradd -m alice               # primary group = alice (private group)
touch /tmp/test
ls -l /tmp/test
# -rw-r--r-- 1 alice alice ...   ← group is 'alice'
```

---

## Shell Validation Pitfalls

### useradd doesn't validate shell exists
```bash
useradd -s /bin/zsh alice    # ✅ created even if zsh not installed!
# alice tries to login → "No such file or directory"

# Better: validate first
if [ -x "/bin/zsh" ]; then
  useradd -s /bin/zsh alice
else
  echo "zsh not installed" >&2
  exit 1
fi

# Or check /etc/shells:
grep -q "/bin/zsh" /etc/shells || { echo "zsh not in /etc/shells"; exit 1; }
```

### /etc/shells and PAM
```bash
# PAM's pam_shells module rejects login if shell not in /etc/shells
cat /etc/shells
# /bin/sh
# /bin/bash
# /usr/bin/bash
# ...

# If you set a shell not in /etc/shells:
useradd -s /usr/local/bin/myshell alice
# alice can't log in via PAM (ssh, login, etc.)

# Fix:
echo "/usr/local/bin/myshell" >> /etc/shells
```

### /sbin/nologin vs /bin/false
```bash
# /sbin/nologin: prints a message, exits 1
useradd -s /sbin/nologin service_account
# Login attempt: "This account is currently not available."

# /bin/false: exits 1 immediately, no message
useradd -s /bin/false service_account

# Both prevent interactive login
# /sbin/nologin is friendlier for debugging
# Some SFTP configs require /sbin/nologin, not /bin/false
```

---

## Skeleton Files & Permissions

### Skeleton files copied with permissions from /etc/skel
```bash
# If /etc/skel has world-readable sensitive files:
echo "SECRET_KEY=abc123" > /etc/skel/.env

useradd -m alice
ls -l /home/alice/.env
# -rw-r--r-- alice alice ...    ← WORLD READABLE!

# Fix: ensure skel files have correct permissions
chmod 600 /etc/skel/.env
# Or set UMASK in /etc/login.defs to 077

# Check skel permissions:
ls -la /etc/skel/
```

### Home directory permissions depend on UMASK
```bash
grep "^UMASK" /etc/login.defs
# UMASK  022   → home dir mode = 755 (others can read!)
# UMASK  027   → home dir mode = 750
# UMASK  077   → home dir mode = 700 (most private)

# With UMASK 022:
useradd -m alice
ls -ld /home/alice
# drwxr-xr-x   ← 755: other users can read alice's files!

# Harden: use UMASK 077 or chmod after creation
chmod 700 /home/alice

# Debian default: 022 (permissive)
# RHEL default: 022
# Some security benchmarks require 750 or 700
```

---

## File Locking & Concurrent Modifications

### Concurrent useradd can corrupt /etc/passwd
```bash
# useradd uses lckpwdf() to lock /etc/passwd
# If two useradd run simultaneously:
useradd alice &   # background
useradd bob       # foreground
# One will wait for the lock, the other proceeds
# If lock fails: "cannot lock /etc/passwd; try again later."

# Check for stale lock:
ls /etc/.pwd.lock
# If file exists and no useradd is running, it's stale
pgrep useradd     # should be empty if no useradd running
rm /etc/.pwd.lock  # remove stale lock (carefully!)
```

### Never edit /etc/passwd directly — use vipw
```bash
# ❌ Direct edit — no locking, corrupt on crash mid-write:
vim /etc/passwd

# ✅ vipw — locks file, validates syntax before saving:
vipw           # edit /etc/passwd safely
vipw -s        # edit /etc/shadow safely
vigr           # edit /etc/group safely
vigr -s        # edit /etc/gshadow safely

# vipw checks syntax on exit — won't save a broken file
```

---

## Username Validation Rules

### POSIX username constraints
```bash
# Valid: letters, digits, underscore, hyphen, dot
# Must start with letter or underscore
# Max length: 32 chars (Linux), 8 chars (POSIX strict)

useradd "alice"          # ✅
useradd "alice_smith"    # ✅
useradd "alice.smith"    # ✅ (but some tools dislike dots)
useradd "alice-smith"    # ✅
useradd "1alice"         # ❌ starts with digit (rejected on most distros)
useradd "alice smith"    # ❌ spaces not allowed
useradd "alice@corp"     # ❌ @ not valid
useradd "ALICE"          # ✅ technically valid, but convention is lowercase
useradd "a" * 33         # ❌ too long (> 32 chars on Linux)

# Check NAME_REGEX in /etc/adduser.conf (Debian):
grep NAME_REGEX /etc/adduser.conf
# NAME_REGEX="^[a-z][-a-z0-9]*$"
```

### Username collisions with system names
```bash
# Dangerous: creating a user with the same name as a system account
useradd mail     # ❌ 'mail' already exists as system account!
# useradd: user 'mail' already exists

# Always check first:
id username 2>&1 | grep -q "no such user" || echo "already exists"
getent passwd username
```

---

## Security Pitfalls

### -p flag exposes password hash in process list
```bash
# During the brief window when useradd runs:
ps aux | grep useradd
# useradd -p $6$salt$hash... -m alice   ← hash visible to all users!

# ⚠️ Even a hashed password in ps output is a security risk:
# attacker can grab the hash and crack it offline

# Safer alternatives:
useradd -m alice && echo "alice:password" | chpasswd    # chpasswd reads from pipe
useradd -m alice && passwd alice                         # interactive
```

### useradd with no expiry in regulated environments
```bash
# CIS Benchmark / PCI-DSS: all accounts must have password expiry
useradd -m alice
# Default: PASS_MAX_DAYS 99999 (effectively never)

# Compliance requires:
useradd -m alice
chage -M 90 -W 14 alice   # 90-day max, 14-day warning

# Or set system-wide in /etc/login.defs:
# PASS_MAX_DAYS  90
# PASS_WARN_AGE  14
```

### Forgetting to lock unused system accounts
```bash
# System accounts created without -s /sbin/nologin can be used for login
useradd -r myservice     # shell defaults to /bin/sh or /bin/bash!

# If someone sets a password, they can log in as myservice
passwd myservice

# Always specify no-login shell for system accounts:
useradd -r -s /sbin/nologin myservice
```

---

## NFS & UID Consistency

### Different UIDs on different machines = wrong file ownership
```bash
# Machine A: useradd alice → uid=1001
# Machine B: useradd alice → uid=1003 (different users existed first)

# When Machine B mounts Machine A's NFS share:
ls -l /nfs/home/alice/
# -rw-r--r-- 1 1001 1001 ...  ← appears as uid 1001, not alice!

# Fix: use consistent UIDs across all machines
useradd -u 1001 alice    # specify same UID everywhere

# Or use centralized auth: LDAP, FreeIPA, Active Directory
# These assign UIDs centrally, ensuring consistency
```

---

## System Account Traps

### -r doesn't disable login by itself
```bash
# Common misconception: -r creates a "locked" or "non-login" account
useradd -r myservice      # WRONG assumption: this is just a low-UID account
                          # the shell is still /bin/sh!

# If someone sets a password:
passwd myservice          # now myservice can log in!

# Always combine -r with -s /sbin/nologin:
useradd -r -s /sbin/nologin -d /var/lib/myservice myservice
```

### Home directory for system accounts
```bash
# useradd -r: by default does NOT create home directory
useradd -r myservice
ls /home/myservice   # doesn't exist

# Service needs a data directory? Use -m with -d:
useradd -r -m -d /var/lib/myservice myservice
# Creates /var/lib/myservice, NOT /home/myservice

# Without -m, the -d value is recorded in /etc/passwd but directory not created:
useradd -r -d /var/lib/myservice myservice   # directory not created!
ls /var/lib/myservice   # doesn't exist
mkdir /var/lib/myservice   # must create manually
chown myservice:myservice /var/lib/myservice
```

---

## Incomplete Cleanup After Failed useradd

### Partial state after interrupted useradd
```bash
# If useradd is interrupted mid-run (power loss, kill signal):
# /etc/passwd may have the entry but /etc/shadow may not
# or home directory created but ownership wrong

# Check for inconsistency:
pwck    # verify /etc/passwd and /etc/shadow consistency
grpck   # verify /etc/group and /etc/gshadow

# If user is in passwd but not shadow:
# Add missing shadow entry:
pwconv  # convert passwd to shadow (fixes missing shadow entries)

# Manual cleanup of partial useradd:
# Remove from /etc/passwd:
vipw    # delete the line manually

# Remove from /etc/shadow:
vipw -s

# Remove from /etc/group:
vigr

# Remove home:
rm -rf /home/username

# Check for orphaned files:
find / -user username 2>/dev/null     # if UID was assigned
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
