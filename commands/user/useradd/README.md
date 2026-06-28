# useradd — The Complete Reference

> **Create a new user account**
> The low-level tool for adding users to a Linux system.
> Touches five files, creates a home directory, and sets up the entire
> authentication infrastructure for a new account — in one command.

---

## Table of Contents

- [What is useradd?](#what-is-useradd)
- [Where does useradd live?](#where-does-useradd-live)
- [How useradd works internally](#how-useradd-works-internally)
- [Syntax](#syntax)
- [All Options](#all-options)
- [Files Modified by useradd](#files-modified-by-useradd)
- [Default Values: /etc/default/useradd](#default-values-etcdefaultuseradd)
- [/etc/login.defs — System-wide Policy](#etclogindefs--system-wide-policy)
- [Skeleton Directory: /etc/skel](#skeleton-directory-etcskel)
- [UID & GID Ranges](#uid--gid-ranges)
- [useradd vs adduser vs usermod vs userdel](#useradd-vs-adduser-vs-usermod-vs-userdel)
- [Related Commands](#related-commands)

---

## What is useradd?

`useradd` is the low-level command for creating new user accounts on Linux. It:

1. Adds an entry to `/etc/passwd` (user database)
2. Adds an entry to `/etc/shadow` (password database)
3. Adds an entry to `/etc/group` (primary group)
4. Optionally adds to `/etc/gshadow` (group password database)
5. Creates the home directory (with `-m`)
6. Copies skeleton files from `/etc/skel` into the home directory
7. Sets up initial password aging in `/etc/shadow`

`useradd` is the **plumbing** — it does exactly what you tell it, no more. On Debian/Ubuntu, `adduser` is a friendlier wrapper around `useradd` that prompts interactively.

---

## Where does useradd live?

```
/usr/sbin/useradd       ← most Linux systems
/sbin/useradd           ← older distros (symlink)
```

```bash
which useradd
useradd --version
# useradd 4.13 (shadow-utils)
```

`useradd` is part of **shadow-utils** — the same package that provides `passwd`, `groupadd`, `userdel`, `usermod`, `chage`, and `su`.

```bash
# Debian/Ubuntu:
dpkg -S $(which useradd)    # passwd: /usr/sbin/useradd

# RHEL/Fedora:
rpm -qf $(which useradd)    # shadow-utils-...
```

Requires **root** or **CAP_CHOWN**, **CAP_SETUID** capabilities.

---

## How useradd works internally

```
parse args → validate → find next UID → lock files → update databases
→ create home → copy skel → set password aging → unlock files → log
```

**Step by step:**

1. **Parse and validate** arguments — check username format (no spaces, valid chars), no duplicate username/UID
2. **Find next UID** — scan `/etc/passwd` for highest UID, increment (or use `-u` value)
3. **Lock files** — acquire exclusive lock on `/etc/passwd.lock` and `/etc/shadow.lock` to prevent concurrent modifications
4. **Update `/etc/passwd`** — append new line: `username:x:UID:GID:GECOS:home:shell`
5. **Update `/etc/shadow`** — append new line with `!!` (locked, no password yet)
6. **Update `/etc/group`** — create primary group or add to existing
7. **Update `/etc/gshadow`** — group shadow entry
8. **Create home directory** — `mkdir`, `chown`, `chmod` per `-d` and `-m` flags
9. **Copy skeleton** — recursive copy of `/etc/skel` contents to home
10. **Set ownership** — `chown -R uid:gid homedir`
11. **Apply password aging** from `/etc/login.defs`
12. **Unlock files** — release locks
13. **Log to syslog** — records account creation

**File locking mechanism:**
`useradd` uses `lckpwdf()` which creates `/etc/passwd.lock`. If the lock can't be acquired (another useradd/usermod running), it fails rather than corrupting the files.

**Exit codes:**
| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Can't update password file |
| `2` | Invalid command syntax |
| `3` | Invalid argument to option |
| `4` | UID already in use (with `-u`, without `-o`) |
| `6` | Specified group doesn't exist |
| `9` | Username already in use |
| `10` | Can't update group file |
| `12` | Can't create home directory |
| `14` | Can't update SELinux user mapping |

---

## Syntax

```
useradd [OPTIONS] LOGIN
useradd -D [OPTIONS]       # view/modify defaults
```

- `LOGIN` — the username to create (required)
- `-D` — display or modify default values (stored in `/etc/default/useradd`)

---

## All Options

### Identity

| Option | Long | Description |
|--------|------|-------------|
| `-u UID` | `--uid` | Set specific UID (must be unique unless `-o`) |
| `-o` | `--non-unique` | Allow non-unique UID (use with `-u`) |
| `-g GID` | `--gid` | Primary group (name or GID) |
| `-G groups` | `--groups` | Supplementary groups (comma-separated) |
| `-c comment` | `--comment` | GECOS field (full name, phone, etc.) |
| `-l` | `--no-log-init` | Don't add user to lastlog/faillog databases |

### Home Directory

| Option | Long | Description |
|--------|------|-------------|
| `-m` | `--create-home` | Create home directory (copies `/etc/skel`) |
| `-M` | `--no-create-home` | Do NOT create home directory |
| `-d dir` | `--home-dir` | Set home directory path (default: `/home/LOGIN`) |
| `-k skeldir` | `--skel` | Use alternative skeleton directory |
| `-K KEY=VAL` | `--key` | Override `/etc/login.defs` values |

### Shell & Password

| Option | Long | Description |
|--------|------|-------------|
| `-s shell` | `--shell` | Login shell (default: from `/etc/default/useradd`) |
| `-p hash` | `--password` | Set encrypted password (use `passwd` instead!) |
| `-L` | (via passwd) | Lock account after creation |

### Account Expiry

| Option | Long | Description |
|--------|------|-------------|
| `-e date` | `--expiredate` | Account expiry date (YYYY-MM-DD) |
| `-f days` | `--inactive` | Days after password expiry before disabling |

### System Accounts

| Option | Long | Description |
|--------|------|-------------|
| `-r` | `--system` | Create system account (low UID, no home by default) |
| `-N` | `--no-user-group` | Don't create private group for user |
| `-U` | `--user-group` | Create private group (default behavior) |

### SELinux & Misc

| Option | Long | Description |
|--------|------|-------------|
| `-Z context` | `--selinux-user` | SELinux user mapping |
| `-b basedir` | `--base-dir` | Base directory for home (default: `/home`) |

---

## Files Modified by useradd

### /etc/passwd — User Account Database

```
alice:x:1001:1001:Alice Smith,,,:/home/alice:/bin/bash
│     │ │    │    │             │            │
│     │ │    │    │             │            └─ login shell
│     │ │    │    │             └─ home directory
│     │ │    │    └─ GECOS: full name, room, work phone, home phone, other
│     │ │    └─ primary GID
│     │ └─ UID
│     └─ password: 'x' = look in /etc/shadow
└─ username (LOGIN)
```

```bash
# View the entry created by useradd:
grep "^alice:" /etc/passwd

# Permissions: world-readable (programs need UID/name lookup)
ls -l /etc/passwd
# -rw-r--r-- 1 root root 2847 /etc/passwd
```

### /etc/shadow — Password & Aging Database

```
alice:!!:19523:0:99999:7:::
│     │  │     │ │     │
│     │  │     │ │     └─ warn days before expiry
│     │  │     │ └─ max days between changes (99999 = never)
│     │  │     └─ min days between changes
│     │  └─ last change (days since epoch Jan 1 1970)
│     └─ !! = locked, no password set yet
└─ username
```

```bash
# View (root only):
sudo grep "^alice:" /etc/shadow

# Permissions: root read only
ls -l /etc/shadow
# -rw-r----- 1 root shadow 1456 /etc/shadow
```

### /etc/group — Group Database

```
alice:x:1001:
│     │ │    └─ list of supplementary members (empty = no extras)
│     │ └─ GID
│     └─ group password ('x' = in /etc/gshadow)
└─ group name
```

### /etc/gshadow — Group Shadow

```
alice:!::
│     │ └─ members
│     └─ ! = no group password
└─ group name
```

---

## Default Values: /etc/default/useradd

```bash
cat /etc/default/useradd
```

```ini
# Default values used when no flags are specified:

GROUP=100           # default primary GID (if -N: use this group)
HOME=/home          # base directory for home dirs
INACTIVE=-1         # days after expiry before disabling (-1 = never)
EXPIRE=             # account expiry date (empty = never)
SHELL=/bin/sh       # default shell (often /bin/bash on distros)
SKEL=/etc/skel      # skeleton directory
CREATE_MAIL_SPOOL=no  # create /var/mail/username
```

```bash
# View current defaults:
useradd -D

# Modify a default:
useradd -D -s /bin/bash         # change default shell
useradd -D -b /home             # change base directory
useradd -D -e 2024-12-31        # set default expiry

# These changes are written to /etc/default/useradd
```

---

## /etc/login.defs — System-wide Policy

`/etc/login.defs` controls UID/GID ranges, password aging defaults, and other system-wide security policy applied to all new accounts.

```bash
cat /etc/login.defs | grep -v "^#\|^$"
```

**Key settings:**

```ini
# UID/GID ranges
UID_MIN         1000    # minimum UID for regular users
UID_MAX         60000   # maximum UID for regular users
SYS_UID_MIN     201     # minimum UID for system accounts (-r)
SYS_UID_MAX     999     # maximum UID for system accounts

GID_MIN         1000    # minimum GID for regular groups
GID_MAX         60000   # maximum GID
SYS_GID_MIN     201
SYS_GID_MAX     999

# Password aging defaults (applied to new accounts)
PASS_MAX_DAYS   99999   # max days a password is valid
PASS_MIN_DAYS   0       # min days between password changes
PASS_WARN_AGE   7       # days to warn before expiry
PASS_MIN_LEN    8       # minimum password length

# Home directory
CREATE_HOME     yes     # create home dir by default (Debian-style)

# Hashing algorithm
ENCRYPT_METHOD  SHA512  # or YESCRYPT on modern systems

# umask for home directories
UMASK           022

# Login tracking
LASTLOG_ENAB    yes     # log last login time
FAILLOG_ENAB    yes     # track failed login attempts
LOG_UNKFAIL_ENAB no     # log unknown usernames in failed logins

# Mail
MAIL_DIR        /var/mail  # or /var/spool/mail
```

---

## Skeleton Directory: /etc/skel

When `useradd -m` creates a home directory, it copies all files from `/etc/skel` into it.

```bash
ls -la /etc/skel/
# .bash_logout
# .bashrc
# .profile
# (possibly: .bash_profile, .vimrc, etc.)
```

```bash
# Customize skeleton for all new users:
echo "alias ll='ls -la'" >> /etc/skel/.bashrc
cp company_motd.txt /etc/skel/.motd
mkdir /etc/skel/bin           # every new user gets a ~/bin directory

# Use alternative skeleton:
useradd -m -k /etc/skel.developer alice
useradd -m -k /dev/null alice    # empty home (no skeleton files)
```

---

## UID & GID Ranges

```
0           root (superuser)
1-99        statically allocated system accounts (distro-managed)
100-999     dynamically allocated system accounts (useradd -r)
1000-65533  regular user accounts (useradd without -r)
65534       nobody (nfsnobody — unmapped UIDs)
65535       (historical limit — often avoided)
```

```bash
# These ranges are configurable in /etc/login.defs:
grep "^UID_MIN\|^UID_MAX\|^SYS_UID" /etc/login.defs

# Find next available UID:
awk -F: '$3 >= 1000 && $3 < 65534 {print $3}' /etc/passwd | sort -n | tail -1

# List all users and UIDs:
awk -F: '{printf "%-20s %s\n", $1, $3}' /etc/passwd | sort -t' ' -k2 -n
```

---

## useradd vs adduser vs usermod vs userdel

| Command | Purpose | Interactive? | Distro |
|---------|---------|-------------|--------|
| `useradd` | Create user (low-level) | No | All Linux |
| `adduser` | Create user (high-level, friendly) | Yes (Perl script) | Debian/Ubuntu |
| `usermod` | Modify existing user | No | All Linux |
| `userdel` | Delete user | No | All Linux |
| `deluser` | Delete user (high-level) | Optional | Debian/Ubuntu |

```bash
# useradd: manual, precise, scriptable
useradd -m -s /bin/bash -c "Alice Smith" alice

# adduser: interactive, creates home, sets password, prompts for info
adduser alice
# Asks: password, full name, room, phone, etc.

# usermod: modify after creation
usermod -aG sudo alice          # add to sudo group
usermod -s /bin/zsh alice       # change shell
usermod -l newname oldname      # rename user

# userdel: remove user
userdel alice                   # remove account (keep home)
userdel -r alice                # remove account + home + mail spool
```

---

## Related Commands

| Command | Relation |
|---------|---------|
| `adduser` | Friendly wrapper around useradd (Debian/Ubuntu) |
| `usermod` | Modify existing user account |
| `userdel` | Delete user account |
| `passwd` | Set/change user password |
| `chage` | Manage password aging policy |
| `groupadd` | Create new group |
| `groupmod` | Modify existing group |
| `groupdel` | Delete group |
| `groups` | Show groups a user belongs to |
| `id` | Show UID, GID, and group memberships |
| `finger` | Display user information (GECOS) |
| `chfn` | Change GECOS (finger) information |
| `chsh` | Change login shell |
| `newusers` | Bulk create users from file |
| `pwck` | Verify password file integrity |
| `grpck` | Verify group file integrity |
| `vipw` | Safely edit /etc/passwd (with locking) |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
