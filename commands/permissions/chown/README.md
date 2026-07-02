# chown — The Complete Reference

> **CHange OWNer: control which user and group own a file**
> Present since early Unix (1970s), and — unlike `chmod` — restricted to root on virtually every modern system.
> The other half of the Unix permission model: `chmod` decides *what* the bits allow, `chown` decides *who* they apply to.

---

## Table of Contents

- [What is chown?](#what-is-chown)
- [Where does chown live?](#where-does-chown-live)
- [How ownership works internally](#how-ownership-works-internally)
- [Syntax](#syntax)
- [Changing Owner, Group, or Both](#changing-owner-group-or-both)
- [Numeric UID/GID vs Names](#numeric-uidgid-vs-names)
- [All Key Options](#all-key-options)
- [Recursive chown](#recursive-chown)
- [chown and Symlinks](#chown-and-symlinks)
- [chown and Special Permission Bits](#chown-and-special-permission-bits)
- [Why chown Is Restricted to root (POSIX_CHOWN_RESTRICTED)](#why-chown-is-restricted-to-root-posix_chown_restricted)
- [chown vs chgrp vs chmod](#chown-vs-chgrp-vs-chmod)
- [chown and Networked/Container Filesystems](#chown-and-networkedcontainer-filesystems)
- [Related Commands](#related-commands)

---

## What is chown?

`chown` stands for **CHange OWNer**. It changes the **user** (owner) and/or **group** associated with a file or directory. Every file on a Unix filesystem has exactly one owning user and one owning group, stored as numeric IDs (**UID** and **GID**) in the inode — the names you see in `ls -l` are just a lookup against `/etc/passwd` and `/etc/group` (or NSS sources like LDAP), not stored on disk themselves.

**The division of responsibility:**
- `chmod` — controls **what** the owner/group/others are allowed to do (read/write/execute)
- `chown` — controls **who** the owner and group actually are
- `chgrp` — a narrower tool that only changes the group (a subset of what `chown` can do)

```bash
chown alice file.txt          # change owner to alice
chown alice:staff file.txt    # change owner to alice AND group to staff
chown :staff file.txt         # change ONLY the group (chown's chgrp-equivalent form)
```

---

## Where does chown live?

```
/bin/chown
/usr/bin/chown
```

```bash
which chown
chown --version
# chown (GNU coreutils) 9.4
```

Part of **GNU coreutils** on Linux. Requires elevated privileges to change ownership to another user in almost all configurations (see [POSIX_CHOWN_RESTRICTED](#why-chown-is-restricted-to-root-posix_chown_restricted) below).

---

## How ownership works internally

### UID and GID, not names

Every inode stores two 32-bit integers: `st_uid` and `st_gid`. Usernames and group names are a human-friendly *view* resolved at display time:

```bash
stat -c "%u %g" file.txt
# 1000 1000        ← the actual numbers stored in the inode

ls -l file.txt
# -rw-r--r-- 1 alice staff ... file.txt   ← names looked up from UID/GID
```

If a UID has no corresponding entry in `/etc/passwd` (or your NSS source), `ls -l` simply shows the raw number instead of a name — the file is perfectly valid; only the *display* lacks a name to show.

```bash
ls -l orphaned_file.txt
# -rw-r--r-- 1 1005 1005 ... orphaned_file.txt
# UID 1005 no longer exists in /etc/passwd (user was deleted)
```

### Where usernames resolve from

```bash
getent passwd alice     # resolves via NSS: files, LDAP, sssd, etc.
id alice                 # shows alice's UID and group memberships
cat /etc/nsswitch.conf | grep passwd   # shows resolution order
```

### Permission check independence from chown

`chown` and `chmod` are entirely separate operations at the kernel level — changing ownership does **not** by itself change what the new owner/group are permitted to do; the existing permission *bits* stay exactly as they were, now simply interpreted against the new UID/GID.

```bash
ls -l file.txt
# -rwx------ 1 alice staff ... file.txt   (only alice can access it)
sudo chown bob file.txt
ls -l file.txt
# -rwx------ 1 bob staff ... file.txt     (now only bob can access it — same bits, new owner)
```

---

## Syntax

```bash
chown [OPTIONS] [OWNER][:[GROUP]] FILE...
chown [OPTIONS] --reference=RFILE FILE...
```

The **owner/group specifier** has several valid forms:

| Form | Effect |
|------|--------|
| `alice` | Change owner to alice, leave group untouched |
| `alice:staff` | Change owner to alice AND group to staff |
| `alice:` | Change owner to alice AND group to alice's **login group** |
| `:staff` | Change ONLY the group to staff, leave owner untouched (chgrp-equivalent) |
| `1000:1000` | Same as above but using numeric UID:GID |
| `alice.staff` | Legacy dot-separator (still supported, but discouraged — breaks with usernames containing a literal dot) |

```bash
chown alice file.txt         # owner → alice
chown alice:staff file.txt   # owner → alice, group → staff
chown alice: file.txt        # owner → alice, group → alice's primary group
chown :staff file.txt        # group → staff only
chown 1000:1000 file.txt     # numeric form
```

---

## Changing Owner, Group, or Both

```bash
# Owner only
chown alice document.txt

# Group only (two equivalent ways)
chown :developers document.txt
chgrp developers document.txt

# Both at once (most common in real usage)
chown alice:developers document.txt

# Owner, with group automatically set to owner's primary/login group
chown alice: document.txt

# Multiple files/directories in one call
chown alice:developers file1.txt file2.txt dir/

# Using a glob
chown alice:developers *.log
```

---

## Numeric UID/GID vs Names

```bash
# By name (most common, human-readable)
chown alice:staff file.txt

# By numeric ID — useful when the user/group doesn't exist locally yet,
# or when working across systems with different /etc/passwd entries
chown 1000:1000 file.txt

# Mixed: name for owner, number for group (or vice versa) — both valid
chown alice:1000 file.txt
chown 1000:staff file.txt

# Look up the numeric IDs for a name
id alice
# uid=1000(alice) gid=1000(alice) groups=1000(alice),27(sudo),1001(staff)

# Look up the name for a numeric ID
getent passwd 1000
# alice:x:1000:1000:Alice:/home/alice:/bin/bash
```

**Why numeric IDs matter in practice:** when restoring a backup, migrating a disk image, or working inside a container, the target system may not yet have a matching username in `/etc/passwd` for the UID stored in the file — `chown 1000:1000 file` still works correctly because the kernel only cares about the numbers; names are purely cosmetic.

---

## All Key Options

| Option | Long | Description |
|--------|------|-------------|
| `-R` | `--recursive` | Apply to a directory and everything inside it |
| `-v` | `--verbose` | Print a line for every file processed |
| `-c` | `--changes` | Like verbose, but only reports files that actually changed |
| `-f` | `--silent` / `--quiet` | Suppress most error messages |
| `--reference=RFILE` | | Copy owner:group from another file instead of specifying explicitly |
| `--from=CURRENT_OWNER:CURRENT_GROUP` | | Only change ownership if it currently matches this — safer bulk changes |
| `--preserve-root` | | Refuse to operate recursively on `/` (default) |
| `--no-preserve-root` | | Allow recursive operation on `/` (dangerous) |
| `-h` | `--no-dereference` | Act on a symlink itself, not its target |
| `--dereference` | | Act on the symlink's target (default behavior) |
| `-H` | | With `-R`: if a command-line argument is a symlink to a directory, follow it |
| `-L` | | With `-R`: follow all symbolic links encountered during traversal |
| `-P` | | With `-R`: never follow symlinks (default, safest) |

```bash
chown -R alice:staff /srv/app             # recursive
chown -Rv alice:staff /srv/app            # recursive + verbose
chown --reference=template.txt target.txt # copy owner:group from another file
chown --from=bob:bob alice:staff file.txt # only change if currently bob:bob
chown -h alice symlink.txt                # change the symlink's own owner, not the target's
```

---

## Recursive chown

```bash
# Take ownership of an entire directory tree
sudo chown -R alice:staff /home/alice

# Recursive + verbose, so you can see every file processed
sudo chown -Rv www-data:www-data /var/www/html

# Recursive + only-report-changes (quieter than full verbose)
sudo chown -Rc deploy:deploy /opt/app

# Safer targeted recursive change: only affect files currently owned by "olduser"
sudo chown -R --from=olduser:olduser newuser:newuser /shared/project
# Anything NOT already owned by olduser:olduser is left untouched —
# useful when a directory has mixed ownership and you only want to
# migrate one specific user's files.
```

### Recursive chown vs. symlinks — the `-h`, `-H`, `-L`, `-P` distinction

By default (`-P`), a recursive `chown -R` **never follows symlinks** it encounters while walking the tree — it changes the symlink's target only if the symlink itself was given directly as a top-level argument (and even then, only the target, not the link, unless `-h` is also given). This default exists specifically to prevent a recursive chown from accidentally reaching outside the intended tree via a symlink pointing elsewhere on the filesystem (e.g., to `/etc` or `/`).

```bash
ln -s /etc/passwd /home/alice/link_to_passwd
sudo chown -R alice:alice /home/alice
# Does NOT change ownership of /etc/passwd — symlink is skipped during traversal by default

sudo chown -R -L alice:alice /home/alice
# ⚠️ WITH -L, chown follows the symlink and WOULD change /etc/passwd's ownership!
# -L is rarely what you want for recursive operations on untrusted trees.
```

---

## chown and Symlinks

### Default behavior: chown affects the TARGET, not the link
```bash
ln -s realfile.txt link.txt
chown alice link.txt
ls -l realfile.txt
# realfile.txt is now owned by alice — the ACTUAL target changed

ls -l link.txt
# lrwxrwxrwx ... link.txt -> realfile.txt
# Symlinks conventionally always display as owned by whoever created them,
# and their own "ownership" is largely cosmetic — the kernel doesn't
# enforce permission checks against a symlink's own owner/mode at all.
```

### Changing the symlink's own ownership metadata
```bash
chown -h alice link.txt
# Changes the symlink's OWN owner field (used by lchown() internally),
# without touching the target file at all.

# Useful when you want the symlink's metadata to reflect a specific
# user for bookkeeping/auditing purposes, independent of the target.
```

### Broken symlinks
```bash
ln -s /nonexistent/target broken_link
chown alice broken_link
# chown: cannot dereference 'broken_link': No such file or directory
# (default behavior tries to reach the target, which doesn't exist)

chown -h alice broken_link
# ✅ works — -h changes the symlink's own metadata without needing
# a valid target to dereference
```

---

## chown and Special Permission Bits

### chown can silently clear setuid/setgid
```bash
sudo chmod u+s /usr/local/bin/tool
ls -l /usr/local/bin/tool
# -rwsr-xr-x ... tool    (setuid set)

sudo chown newowner /usr/local/bin/tool
ls -l /usr/local/bin/tool
# -rwxr-xr-x ... tool    ⚠️ setuid silently cleared by many systems!

# This is a deliberate kernel security behavior: reassigning ownership
# of a setuid binary could otherwise let an unprivileged owner retain
# "run as previous owner" behavior in confusing/exploitable ways.
# Always re-apply chmod u+s AFTER chown if the special bit is still needed.
sudo chmod u+s /usr/local/bin/tool
```

### Setgid on directories generally survives chown (it's not the same mechanism)
```bash
chmod g+s /srv/shared/team_dir
chown alice:team_dir /srv/shared/team_dir
ls -ld /srv/shared/team_dir
# Setgid on a DIRECTORY (which controls group inheritance for new files,
# not "run as" semantics) is not cleared by chown the way it is on
# executables — but always verify with `ls -ld` after any bulk chown pass.
```

---

## Why chown Is Restricted to root (POSIX_CHOWN_RESTRICTED)

On essentially every modern Unix/Linux system, **only root can change a file's owner to a different user**. An ordinary user *can* change the **group** of their own files (but only to a group they themselves belong to), yet cannot give a file away to another user's ownership at all.

```bash
# As alice (not root), owning her own file:
chown bob myfile.txt
# chown: changing ownership of 'myfile.txt': Operation not permitted

# alice CAN change the group, but only to a group she belongs to:
chown :staff myfile.txt      # ✅ works, if alice is a member of "staff"
chown :root myfile.txt       # ❌ fails, if alice isn't a member of "root"

sudo chown bob myfile.txt    # ✅ works — root can do anything
```

**Why this restriction exists (POSIX_CHOWN_RESTRICTED):** without it, a user could exploit disk quotas by "giving away" large files to another user's ownership to escape their own quota accounting, or launder ownership of files to evade auditing. This behavior is defined by the POSIX `_POSIX_CHOWN_RESTRICTED` option, and virtually every real-world Unix/Linux system enables it by default.

---

## chown vs chgrp vs chmod

| Command | Changes | Requires root? |
|---------|---------|-----------------|
| `chown user file` | Owner (user) | Yes, for a different owner |
| `chown :group file` | Group | No, if you own the file and belong to the target group |
| `chown user:group file` | Both | Yes (owner change requires root) |
| `chgrp group file` | Group only (equivalent to `chown :group`) | No, if you own the file and belong to the target group |
| `chmod mode file` | Permission bits (rwx), not who owns it | No, if you own the file |

```bash
chgrp developers file.txt      # identical result to:
chown :developers file.txt

# chgrp exists mainly as a clearer, narrower, and historically older
# command for the "group-only" case; chown can fully replace it.
```

---

## chown and Networked/Container Filesystems

### NFS and UID/GID mismatches across machines
```bash
# On an NFS-mounted share, UID 1000 might be "alice" on the client
# but "bob" on the server — chown operates on the NUMBER, and
# the server enforces its OWN mapping of that number to a name/policy.

ls -l /nfs/shared/file.txt
# -rw-r--r-- 1 1000 1000 ... file.txt
# Shows raw numbers if the client can't resolve UID 1000 to a local name

# Fix: align UID/GID numbers across systems (or use NFSv4 with proper
# ID mapping / Kerberos, which maps by NAME instead of raw number)
```

### Docker bind-mount ownership surprises
```bash
# Inside a container running as UID 1000, files created in a bind-mounted
# volume are owned by UID 1000 on the HOST too — even if no user named
# "1000" exists on the host at all.
docker run -v $(pwd):/app myimage touch /app/newfile.txt
ls -l newfile.txt
# -rw-r--r-- 1 1000 1000 ... newfile.txt   (on host, UID 1000 may be unmapped)

# Common fix: match the container's internal UID to your host user's UID,
# or chown the result back afterward:
sudo chown $(id -u):$(id -g) newfile.txt

# Or run the container with an explicit matching UID:
docker run --user $(id -u):$(id -g) -v $(pwd):/app myimage touch /app/newfile.txt
```

---

## Related Commands

| Command | Relation |
|---------|---------|
| `chgrp` | Narrower tool that changes only the group |
| `chmod` | Changes permission bits, not ownership |
| `stat` | Shows detailed ownership info, including numeric UID/GID |
| `ls -l` / `ls -n` | Display ownership (`-n` shows raw numeric IDs instead of names) |
| `id` | Show a user's UID, GID, and group memberships |
| `getent passwd` / `getent group` | Resolve names ↔ numeric IDs |
| `find -user` / `find -uid` | Search for files by owner |
| `useradd` / `groupadd` | Create the users/groups that chown will later reference |
| `usermod` | Modify a user's UID/GID (affects future chown lookups, not existing files) |
| `install` | Copy a file while setting ownership and mode in a single step |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
