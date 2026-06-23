# passwd — The Complete Reference

> **Change user password and manage password policy**
> One of the oldest security commands in Unix — and one of the best examples
> of SUID in practice. More than just changing passwords.

---

## Table of Contents

- [What is passwd?](#what-is-passwd)
- [Where does passwd live?](#where-does-passwd-live)
- [How passwd works internally](#how-passwd-works-internally)
- [SUID: Why passwd can write /etc/shadow](#suid-why-passwd-can-write-etcshadow)
- [Syntax](#syntax)
- [All Options](#all-options)
- [Password Storage: /etc/passwd and /etc/shadow](#password-storage)
- [Hashing Algorithms](#hashing-algorithms)
- [PAM Integration](#pam-integration)
- [passwd vs chpasswd vs usermod](#passwd-vs-chpasswd-vs-usermod)
- [Related Commands](#related-commands)

---

## What is passwd?

`passwd` changes a user's authentication password. It:
- Prompts for the current password (for regular users)
- Prompts for a new password twice
- Validates the new password against policy (length, complexity, history)
- Updates `/etc/shadow` with the new hashed password

Root can change any user's password without knowing the current one.
Regular users can only change their own password.

`passwd` also manages **password policy**: expiry, locking, unlocking, and forcing password change at next login.

---

## Where does passwd live?

```
/usr/bin/passwd     ← most Linux systems (Debian, Ubuntu, Arch, RHEL)
/bin/passwd         ← some older distros (may be symlink)
```

```bash
which passwd
type passwd
ls -l $(which passwd)   # notice the 's' in permissions — SUID!
passwd --version        # version info
```

`passwd` is part of the **shadow-utils** package on RHEL/Fedora/Arch, and **passwd** package on Debian/Ubuntu.

```bash
# Debian/Ubuntu
dpkg -S $(which passwd)     # passwd: /usr/bin/passwd

# RHEL/Fedora
rpm -qf $(which passwd)     # shadow-utils-...
```

---

## How passwd works internally

```
validate caller → prompt old password → check PAM → prompt new password
→ validate policy → hash password → write to /etc/shadow → update aging info
```

Step by step:

1. **Identify caller**: checks real UID (`getuid()`) — who actually ran the command
2. **Check permissions**: if UID ≠ 0 and target ≠ self → deny
3. **Prompt old password**: for regular users, verifies current password via PAM
4. **Prompt new password**: twice (for confirmation)
5. **Validate via PAM**: `pam_pwquality` or `pam_cracklib` checks:
   - Minimum length
   - Complexity (uppercase, digits, special chars)
   - Dictionary words
   - Similarity to old password
   - Password history
6. **Hash the password**: using the algorithm configured in `/etc/login.defs` or PAM
7. **Write to `/etc/shadow`**: updates the hash and resets aging timestamps
8. **Log to syslog**: records the change (without the password)

---

## SUID: Why passwd can write /etc/shadow

`/etc/shadow` is readable only by root:
```bash
ls -l /etc/shadow
# -rw-r----- 1 root shadow 1234 Jun 15 /etc/shadow
```

A regular user cannot read or write it. Yet `passwd` must update it. How?

**The SUID bit:**
```bash
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root 59976 /usr/bin/passwd
#     ↑ 's' in owner execute = SUID
```

When a file has the SUID bit set:
- The process runs with the **file owner's UID** (root = 0), not the caller's UID
- `passwd` runs as root regardless of who invokes it
- This gives it permission to write `/etc/shadow`

**Security safeguard inside passwd:**
`passwd` checks `getuid()` (real UID, not effective UID) to know who actually called it. If the real UID is not 0, it enforces:
- Can only change own password
- Must provide current password first
- New password is subject to policy checks

Root (real UID = 0) bypasses all these restrictions.

```bash
# Verify SUID:
stat /usr/bin/passwd | grep "Access"
# Access: (4755/-rwsr-xr-x)
#          ↑ 4 = SUID bit set
```

---

## Syntax

```
passwd [OPTIONS] [USERNAME]
```

- With no USERNAME → changes current user's password
- Root can specify any USERNAME
- Regular users cannot specify a different USERNAME

---

## All Options

### Password Change

| Option | Long | Description |
|--------|------|-------------|
| (none) | | Change password interactively |
| `-d` | `--delete` | Delete password (make account passwordless) |
| `-e` | `--expire` | Force password expiry (user must change at next login) |
| `-i N` | `--inactive N` | Days after expiry before account is disabled |
| `-k` | `--keep-tokens` | Keep authentication tokens if not expired |

### Account Locking

| Option | Long | Description |
|--------|------|-------------|
| `-l` | `--lock` | Lock account (prepends `!` to hash in `/etc/shadow`) |
| `-u` | `--unlock` | Unlock account (removes `!` from hash) |

### Password Aging / Policy

| Option | Long | Description |
|--------|------|-------------|
| `-n N` | `--mindays N` | Minimum days between password changes |
| `-x N` | `--maxdays N` | Maximum days before password must be changed |
| `-w N` | `--warndays N` | Days before expiry to warn user |
| `-i N` | `--inactive N` | Days of inactivity before account is disabled |

### Information

| Option | Long | Description |
|--------|------|-------------|
| `-S` | `--status` | Show password status for user |
| `-a` | `--all` | Report status for all accounts (use with `-S`) |

### Non-interactive

| Option | Long | Description |
|--------|------|-------------|
| `--stdin` | | Read password from stdin (RHEL/Fedora only) |

---

## Password Storage

### /etc/passwd — User database (public)

```bash
cat /etc/passwd
# alice:x:1001:1001:Alice Smith:/home/alice:/bin/bash
# │     │ │    │    │           │            │
# │     │ │    │    │           │            └─ login shell
# │     │ │    │    │           └─ home directory
# │     │ │    │    └─ GECOS (full name, info)
# │     │ │    └─ primary GID
# │     │ └─ UID
# │     └─ password field: 'x' means shadow is used
# └─ username
```

The `x` in the password field means the actual hash is in `/etc/shadow`.

```bash
# Permissions: world-readable (programs need to look up UIDs/names)
ls -l /etc/passwd
# -rw-r--r-- 1 root root 2847 /etc/passwd
```

### /etc/shadow — Password hashes (root only)

```bash
sudo cat /etc/shadow
# alice:$6$rounds=5000$salt$hash...:19523:0:99999:7:::
# │     │                           │     │ │     │ │││
# │     │                           │     │ │     │ │││└─ reserved
# │     │                           │     │ │     │ ││└── account expiry (days since epoch)
# │     │                           │     │ │     │ │└─── inactive days after expiry
# │     │                           │     │ │     │ └──── warn days before expiry
# │     │                           │     │ │     └────── max days between changes
# │     │                           │     └────────────── min days between changes
# │     │                           └──────────────────── last change (days since epoch)
# │     └──────────────────────────────────────────────── hashed password
# └────────────────────────────────────────────────────── username
```

```bash
# Permissions: root read, shadow group read
ls -l /etc/shadow
# -rw-r----- 1 root shadow 1456 /etc/shadow
```

**Special values in hash field:**

| Value | Meaning |
|-------|---------|
| `$id$salt$hash` | Normal hashed password |
| `!` or `!!` | Locked account (no login with password) |
| `!$6$...` | Locked but hash preserved |
| `*` | No password (system accounts, SSH-only) |
| (empty) | No password required (dangerous) |

---

## Hashing Algorithms

The `$id$` prefix in `/etc/shadow` identifies the algorithm:

| ID | Algorithm | Example |
|----|-----------|---------|
| `$1$` | MD5 | `$1$salt$hash` (obsolete, insecure) |
| `$2a$` | bcrypt | `$2a$12$salt$hash` (not standard on Linux) |
| `$5$` | SHA-256 | `$5$salt$hash` (acceptable) |
| `$6$` | SHA-512 | `$6$rounds=5000$salt$hash` (current default) |
| `$y$` | yescrypt | `$y$j9T$salt$hash` (modern, Ubuntu 22.04+) |
| `$gy$` | gost-yescrypt | Russian standard |

```bash
# Check current default algorithm
grep "ENCRYPT_METHOD" /etc/login.defs
# ENCRYPT_METHOD SHA512

# Or via PAM:
grep "rounds\|sha512\|yescrypt" /etc/pam.d/common-password

# See algorithm of a specific user's hash:
sudo grep "^alice:" /etc/shadow | cut -d: -f2 | cut -d$ -f2
# 6  ← SHA-512
```

**Manually verify a hash (Python):**
```python
import crypt
hash = '$6$rounds=5000$saltsalt$hashhashhashhash...'
print(crypt.crypt("mypassword", hash))
# If output matches hash, password is correct
```

---

## PAM Integration

`passwd` doesn't implement password policy itself — it delegates to **PAM** (Pluggable Authentication Modules).

```bash
cat /etc/pam.d/passwd
# @include common-password
```

```bash
cat /etc/pam.d/common-password
# password requisite pam_pwquality.so retry=3 minlen=8 ...
# password [success=1 default=ignore] pam_unix.so obscure sha512
```

**Key PAM modules for passwd:**

| Module | Purpose |
|--------|---------|
| `pam_pwquality` | Password quality (replaces pam_cracklib) |
| `pam_cracklib` | Older quality checker |
| `pam_unix` | Standard Unix password (writes to shadow) |
| `pam_pwhistory` | Password history (prevent reuse) |

**pam_pwquality settings** (`/etc/security/pwquality.conf`):
```ini
minlen = 12          # minimum length
dcredit = -1         # require at least 1 digit
ucredit = -1         # require at least 1 uppercase
lcredit = -1         # require at least 1 lowercase
ocredit = -1         # require at least 1 special char
maxrepeat = 3        # max consecutive identical chars
gecoscheck = 1       # check against GECOS field
dictcheck = 1        # check against dictionary
usercheck = 1        # check if password contains username
```

---

## passwd vs chpasswd vs usermod

| Command | Use Case | Requires |
|---------|----------|---------|
| `passwd username` | Interactive change, one user | Root or self |
| `chpasswd` | Non-interactive, bulk, scripted | Root |
| `usermod -p` | Set pre-hashed password | Root |
| `openssl passwd` | Generate hash only (no file write) | Anyone |

```bash
# chpasswd: pipe username:password pairs
echo "alice:newpassword" | chpasswd
echo "alice:newpass\nbob:otherpass" | chpasswd   # multiple users

# usermod -p: set pre-hashed password (dangerous — hash visible in ps)
usermod -p "$(openssl passwd -6 'newpass')" alice

# openssl passwd: generate hash
openssl passwd -6 "mypassword"           # SHA-512
openssl passwd -1 "mypassword"           # MD5 (legacy)
openssl passwd -6 -salt mysalt "mypass"  # with specific salt
```

---

## Related Commands

| Command | Relation |
|---------|---------|
| `chpasswd` | Bulk password change (non-interactive, from stdin) |
| `chage` | Manage password aging policy in detail |
| `usermod` | Modify user account (including `-p` for hash) |
| `useradd` | Create user (with `-p` for initial password) |
| `shadow` | `/etc/shadow` file format |
| `pwck` | Verify integrity of `/etc/passwd` and `/etc/shadow` |
| `grpck` | Verify `/etc/group` integrity |
| `login` | Uses PAM to authenticate, same stack as passwd |
| `su` / `sudo` | Elevate privileges — also use PAM |
| `pam_pwquality` | PAM module that enforces password policy |
| `openssl passwd` | Generate password hashes |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
