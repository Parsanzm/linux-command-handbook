# chmod — The Complete Reference

> **CHange MODe: control who can read, write, and execute a file**
> One of the oldest Unix commands (present since Unix v1, 1971).
> The gatekeeper of the entire Unix permission model.

---

## Table of Contents

- [What is chmod?](#what-is-chmod)
- [Where does chmod live?](#where-does-chmod-live)
- [How permissions work internally](#how-permissions-work-internally)
- [Syntax — Symbolic vs Octal](#syntax--symbolic-vs-octal)
- [Octal (Numeric) Mode](#octal-numeric-mode)
- [Symbolic Mode](#symbolic-mode)
- [Special Permissions — setuid, setgid, sticky bit](#special-permissions--setuid-setgid-sticky-bit)
- [All Key Options](#all-key-options)
- [chmod and umask](#chmod-and-umask)
- [chmod vs ACLs](#chmod-vs-acls)
- [Related Commands](#related-commands)

---

## What is chmod?

`chmod` stands for **CHange MODe**. It changes the permission bits (the "mode") of a file or directory, controlling **who** can read, write, or execute it.

Unix permissions are split into three categories of "who" and three types of "what":

**Who (classes):**
- **u** — owner (user)
- **g** — group
- **o** — others (everyone else)
- **a** — all (u+g+o combined)

**What (permission types):**
- **r** — read
- **w** — write
- **x** — execute (or "search" for directories)

Every file has exactly 9 permission bits (3 classes × 3 types), plus 3 extra special bits (setuid, setgid, sticky) — 12 bits total, commonly written as a 3 or 4-digit octal number.

`chmod` **cannot change ownership** (that's `chown`) — it only changes what the existing owner/group/others are allowed to do.

---

## Where does chmod live?

```
/bin/chmod
/usr/bin/chmod
```

```bash
which chmod
chmod --version
# chmod (GNU coreutils) 9.4
type chmod
# chmod is /usr/bin/chmod
```

Part of **GNU coreutils** on Linux. macOS/BSD ship a slightly different (but largely compatible) implementation.

---

## How permissions work internally

### The permission bits

Each file's mode is stored in its **inode**, as part of the `st_mode` field. Visualized with `ls -l`:

```
-rwxr-xr--  1 alice staff  4096 Jun 15 10:23 script.sh
││││││││└─ others: execute? no
│││││││└── others: write?   no
││││││└─── others: read?    yes
│││││└──── group: execute?  yes
││││└───── group: write?    no
│││└────── group: read?     yes
││└─────── owner: execute?  yes
│└──────── owner: write?    yes
└───────── owner: read?     yes
```

The very first character indicates file **type**, not permission:

| Char | Meaning |
|------|---------|
| `-` | Regular file |
| `d` | Directory |
| `l` | Symbolic link |
| `c` | Character device |
| `b` | Block device |
| `p` | Named pipe (FIFO) |
| `s` | Socket |

### What read/write/execute actually mean

| Bit | On a file | On a directory |
|-----|-----------|-----------------|
| `r` | View file contents | List directory contents (`ls`) |
| `w` | Modify file contents | Create/delete/rename entries inside it |
| `x` | Execute the file as a program/script | `cd` into it / traverse it (needed to access anything inside, even with known filenames) |

**Key insight:** directory permissions govern the directory *entries*, not the files' own permissions. You can delete a file you don't own if you have write+execute on its parent directory (unless the sticky bit is set — see below).

### Permission check order

The kernel checks permissions in this exact order, and **stops at the first match**:
1. Is the process's effective UID the file owner? → apply **owner** bits only
2. Is the process's effective GID (or one of its supplementary groups) the file's group? → apply **group** bits only
3. Otherwise → apply **other** bits only

This means the owner can be *more restricted* than "everyone else" if the bits are set that way — the owner's bits are checked first regardless of whether they're more permissive.

```bash
chmod 047 file.txt   # owner: ---, group: r--, others: rwx
# The owner (even though it's their own file) gets NO access!
# Root, however, always bypasses permission checks.
```

---

## Syntax — Symbolic vs Octal

```bash
chmod [OPTIONS] MODE FILE...
```

`chmod` accepts **two mode formats**, which can't be mixed in one invocation:

```bash
# Octal (numeric) — absolute, sets the exact mode
chmod 755 script.sh

# Symbolic — relative or absolute, more expressive
chmod u+x script.sh
chmod u=rwx,g=rx,o=rx script.sh
```

---

## Octal (Numeric) Mode

Each permission type has a numeric value, summed per class:

| Value | Permission |
|-------|-----------|
| `4` | read (r) |
| `2` | write (w) |
| `1` | execute (x) |
| `0` | none |

Add the values for each class (owner, group, other) to get one digit each:

| Octal | rwx | Meaning |
|-------|-----|---------|
| `0` | `---` | no permissions |
| `1` | `--x` | execute only |
| `2` | `-w-` | write only |
| `3` | `-wx` | write + execute |
| `4` | `r--` | read only |
| `5` | `r-x` | read + execute |
| `6` | `rw-` | read + write |
| `7` | `rwx` | read + write + execute |

**Common combinations:**

| Mode | Meaning | Typical use |
|------|---------|-------------|
| `755` | rwxr-xr-x | executables, directories, scripts |
| `644` | rw-r--r-- | regular text/config files |
| `700` | rwx------ | private executables/directories (SSH keys folder) |
| `600` | rw------- | private files (SSH private keys, credentials) |
| `775` | rwxrwxr-x | shared group-writable executables |
| `664` | rw-rw-r-- | shared group-writable files |
| `444` | r--r--r-- | read-only for everyone, even the owner |
| `000` | --------- | no access to anyone (except root) |

```bash
chmod 755 script.sh     # owner: rwx, group: r-x, other: r-x
chmod 644 notes.txt     # owner: rw-,  group: r--,  other: r--
chmod 600 id_rsa        # owner: rw-,  group: ---,  other: ---
chmod 000 secret.txt    # nobody (except root) can access it
```

### 4-digit octal (special bits)

A leading 4th digit sets **setuid, setgid, sticky**:

| Digit | Bits | Meaning |
|-------|------|---------|
| `4` | setuid | run as file owner |
| `2` | setgid | run as file group / inherit group in dirs |
| `1` | sticky | only owner can delete in shared dirs |

```bash
chmod 4755 /usr/bin/passwd    # setuid + rwxr-xr-x
chmod 2775 /shared/project    # setgid + rwxrwxr-x
chmod 1777 /tmp                # sticky + rwxrwxrwx
```

---

## Symbolic Mode

```
chmod [ugoa][+-=][rwxXst] FILE
```

- **Who**: `u` (owner), `g` (group), `o` (other), `a` (all) — omit to mean `a` (but respects umask, unlike octal)
- **Operator**: `+` (add), `-` (remove), `=` (set exactly, clearing anything not listed)
- **Permission**: `r`, `w`, `x`, plus special letters `X`, `s`, `t`

```bash
chmod u+x script.sh          # add execute for owner
chmod g-w file.txt           # remove write for group
chmod o=r file.txt           # set others to read-only exactly
chmod a+r file.txt           # add read for everyone
chmod +x script.sh           # shorthand for a+x (still umask-aware)

# Multiple changes at once (comma-separated, no spaces)
chmod u=rwx,g=rx,o=rx script.sh
chmod u+x,g+x,o-w file.txt

# Multiple classes, same operation
chmod ug+x script.sh         # add execute for owner AND group

# Copy permissions from another class
chmod g=u file.txt           # group gets same perms as owner
chmod o=g file.txt           # others get same perms as group
```

### The special `X` (capital)

`X` sets execute **only if it's a directory, or already executable for someone**. Prevents accidentally making every plain data file executable during a bulk recursive change.

```bash
chmod -R a+rX /var/www/html
# Directories become traversable (+x)
# Files that were already executable stay executable
# Plain files (html, css, images) do NOT get +x
```

### Special letters `s` and `t` (symbolic special bits)

```bash
chmod u+s program        # setuid
chmod g+s directory      # setgid
chmod +t /shared/dir     # sticky bit
chmod u-s program        # remove setuid
```

---

## Special Permissions — setuid, setgid, sticky bit

### setuid (SUID) — `4000` / `u+s`
When set on an **executable**, the program runs with the **file owner's** privileges, not the invoking user's.

```bash
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/passwd
#     ^ 's' instead of 'x' = setuid is set
chmod u+s myprogram
chmod 4755 myprogram
```
Classic example: `passwd` needs to write to `/etc/shadow` (root-only), so it runs as root via setuid even when a normal user invokes it.

**Ignored on directories** and on scripts on most modern Linux kernels (a well-known security restriction — the kernel silently ignores setuid on shebang scripts).

### setgid (SGID) — `2000` / `g+s`
- **On an executable**: runs with the file's **group** privileges.
- **On a directory**: new files/subdirectories created inside **inherit the directory's group** instead of the creating user's primary group. Extremely useful for shared team directories.

```bash
chmod g+s /shared/team_dir
chmod 2775 /shared/team_dir
# Now: anyone creating a file inside inherits the "team_dir" group automatically
```

### Sticky bit — `1000` / `+t`
On a **directory**, restricts deletion/renaming: a user can only delete or rename files they **own**, even if the directory is world-writable.

```bash
ls -ld /tmp
# drwxrwxrwt 10 root root ... /tmp
#          ^ 't' = sticky bit set
chmod +t /shared/uploads
chmod 1777 /shared/uploads
```
Without the sticky bit, a world-writable directory (`777`) would let any user delete anyone else's files inside it — `/tmp` is the textbook reason sticky bit exists.

### How it displays in `ls -l`

| Position | Set + executable | Set + NOT executable |
|----------|-------------------|------------------------|
| setuid (owner x) | `s` (lowercase) | `S` (uppercase) |
| setgid (group x) | `s` (lowercase) | `S` (uppercase) |
| sticky (other x) | `t` (lowercase) | `T` (uppercase) |

Uppercase (`S` / `T`) means the special bit is set but the underlying execute bit is **not** — usually a misconfiguration.

---

## All Key Options

| Option | Long | Description |
|--------|------|-------------|
| `-R` | `--recursive` | Apply to directory and everything inside it |
| `-v` | `--verbose` | Print a diagnostic for every file processed |
| `-c` | `--changes` | Like verbose, but only reports files actually changed |
| `-f` | `--silent` / `--quiet` | Suppress most error messages |
| `--reference=FILE` | | Copy the mode from another file instead of specifying one |
| `--preserve-root` | | Refuse to operate recursively on `/` (default behavior) |
| `--no-preserve-root` | | Allow recursive operation on `/` (dangerous) |
| `--dereference` | | Act on the target of a symlink (default) |
| `--no-dereference` (`-h` on systems with `lchmod`) | | Act on the symlink itself, if the OS supports it |

```bash
chmod -R 755 /var/www/html          # recursive
chmod -Rv 644 *.txt                 # recursive + verbose
chmod --reference=file1 file2       # copy file1's mode onto file2
chmod -c 644 *.conf                 # report only files that actually changed
```

---

## chmod and umask

`umask` doesn't affect `chmod` when you use **octal** mode — octal mode is absolute and always sets exactly the bits you specify.

`umask` **does** affect symbolic mode with a bare `+`/`-`/`=` and no explicit class (rare, and mostly matters for the default creation mode of `mkdir`/`touch`/`open()`, not for `chmod` itself in practice on most systems).

```bash
umask                  # show current umask, e.g. 0022
umask 0022             # new files: 666 & ~022 = 644; new dirs: 777 & ~022 = 755

# chmod always applies exactly what you tell it, regardless of umask:
chmod 777 file.txt     # file is 777, no matter what umask is
```

`umask` controls the **default** permissions at *creation time*; `chmod` changes permissions on **existing** files at any time. They solve different problems.

---

## chmod vs ACLs

Classic Unix permissions (`chmod`) only support **one owner + one group + everyone else**. For more granular control (e.g. "user bob: read-only, user alice: read-write, group X: no access"), use **ACLs**:

```bash
getfacl file.txt              # view ACLs
setfacl -m u:bob:r file.txt   # give bob read access specifically
setfacl -m g:devs:rwx dir/    # give group "devs" full access
```

`chmod` and ACLs interact: `chmod` on the group bits also modifies the ACL "mask" entry, which can silently affect ACL-granted permissions.

---

## Related Commands

| Command | Relation |
|---------|---------|
| `chown` | Change file owner and/or group |
| `chgrp` | Change only the group |
| `umask` | Set default permission mask for new files |
| `stat` | Show detailed permission/mode info (including octal) |
| `ls -l` | Display permissions in `rwx` form |
| `getfacl` / `setfacl` | View/set POSIX ACLs for finer-grained permissions |
| `find -perm` | Search for files by permission bits |
| `install` | Copy files while setting ownership/mode in one step |
| `umask` | Controls default mode of newly created files/directories |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
