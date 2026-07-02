# chown — Edge Cases & Gotchas

> chown mistakes are especially dangerous because they're often IRREVERSIBLE by an
> ordinary user — once you give a file away, you generally can't take it back yourself.

---

## Table of Contents

- [You Can't Take Ownership Back](#you-cant-take-ownership-back)
- [Recursive chown on Files vs Directories](#recursive-chown-on-files-vs-directories)
- [setuid/setgid Silently Dropped](#setuidsetgid-silently-dropped)
- [Symlinks: Target vs Link Itself](#symlinks-target-vs-link-itself)
- [UID Reuse After Account Deletion](#uid-reuse-after-account-deletion)
- [Orphaned Files (nouser / nogroup)](#orphaned-files-nouser--nogroup)
- [Ordinary Users Can Change Group — But Only to Groups They're In](#ordinary-users-can-change-group--but-only-to-groups-theyre-in)
- [NFS and Cross-Machine UID Mismatches](#nfs-and-cross-machine-uid-mismatches)
- [Docker Bind Mounts and Host UID Collisions](#docker-bind-mounts-and-host-uid-collisions)
- [chown Doesn't Change Permission Bits](#chown-doesnt-change-permission-bits)
- [--preserve-root Bypassed by Glob Expansion](#--preserve-root-bypassed-by-glob-expansion)
- [Legacy Dot Separator Ambiguity](#legacy-dot-separator-ambiguity)
- [Disk Quota Surprises](#disk-quota-surprises)
- [chown on Read-Only or Special Filesystems](#chown-on-read-only-or-special-filesystems)

---

## You Can't Take Ownership Back

### Giving away a file as a non-root user is often a one-way door
```bash
# alice owns myfile.txt
whoami
# alice
chown bob myfile.txt
# chown: changing ownership of 'myfile.txt': Operation not permitted
# ✅ Actually — THIS specific action is blocked; ordinary users generally
# CANNOT give files away at all (see POSIX_CHOWN_RESTRICTED in README.md)

# But the risk is real for scripts running as ROOT:
sudo chown bob myfile.txt
# Now alice, even though she still has "write" permission bits set,
# is no longer the OWNER — she may lose the ability to chmod it further,
# and can't chown it back to herself without asking root again.

# Always double-check the target before running chown as root in a script —
# there's no built-in "undo," only a manual re-chown by another root action.
```

---

## Recursive chown on Files vs Directories

### -R applies uniformly, which is not always desired
```bash
sudo chown -R alice:staff /srv/shared
# Every file AND directory underneath, regardless of its current owner,
# gets reassigned — including files another user was actively relying on
# having exclusive ownership of.

# Safer: target only what you mean to change
sudo find /srv/shared -user olduser -exec chown alice:staff {} +
# or, restrict to a specific type:
sudo find /srv/shared -type f -exec chown alice:staff {} +
```

### Recursive chown can be extremely slow on large trees
```bash
# On millions of small files (e.g., a large web cache or node_modules-heavy tree),
# chown -R can take a long time and generate heavy I/O
time sudo chown -R www-data:www-data /var/www/huge_site
# Consider running during low-traffic windows, or use find with -exec {} +
# (batches arguments, fewer process spawns than {} \;)
sudo find /var/www/huge_site -exec chown www-data:www-data {} +
```

---

## setuid/setgid Silently Dropped

### chown on an executable clears its setuid/setgid bit
```bash
sudo chmod u+s /usr/local/bin/special_tool
ls -l /usr/local/bin/special_tool
# -rwsr-xr-x ... special_tool

sudo chown newowner /usr/local/bin/special_tool
ls -l /usr/local/bin/special_tool
# -rwxr-xr-x ... special_tool   ⚠️ setuid GONE

# This is deliberate kernel behavior (not a bug): if setuid survived a
# change of ownership, an attacker who could trick root into a chown
# could hijack a previously-safe setuid binary's "run as" semantics.
# Always verify AND re-apply after any chown on a special-permission file:
ls -l /usr/local/bin/special_tool
sudo chmod u+s /usr/local/bin/special_tool
```

---

## Symlinks: Target vs Link Itself

### The default silently follows the link, changing the wrong thing
```bash
ln -s /etc/important_config actual_config_link
sudo chown alice actual_config_link
stat -c "%U" /etc/important_config
# alice   ⚠️ you probably meant to change the LINK, but the TARGET changed instead

# Correct way to change just the symlink's own metadata:
sudo chown -h alice actual_config_link
```

### Recursive chown and symlinks pointing outside the tree
```bash
ln -s /etc/shadow /home/alice/oops_link
sudo chown -R alice:alice /home/alice
# ✅ Safe by DEFAULT — recursive chown does not follow symlinks during
# tree traversal (equivalent to implicit -P), so /etc/shadow is untouched

sudo chown -R -L alice:alice /home/alice
# ⚠️ WITH -L: chown follows symlinks during traversal and WOULD attempt
# to change /etc/shadow's ownership — a serious, easy-to-trigger mistake
# if -L is used out of habit on an untrusted or unfamiliar directory tree.
```

---

## UID Reuse After Account Deletion

### Deleting a user doesn't touch their old files — and the UID can be reused
```bash
sudo userdel olduser          # deletes the ACCOUNT, but files remain
ls -l /home/olduser
# -rw-r--r-- 1 1005 1005 ... file.txt   ← now shows raw UID, no name

# Time passes... a NEW user is created and happens to get UID 1005 again:
sudo useradd newperson         # system assigns UID 1005 (reused)
ls -l /home/olduser/file.txt
# -rw-r--r-- 1 newperson newperson ... file.txt
# ⚠️ newperson now appears to "own" olduser's leftover files — same UID,
# completely different person, and newperson may not even realize it,
# nor have any legitimate business accessing that data.

# Always clean up (or explicitly reassign) files before/after deleting
# a user account, and be cautious about UID reuse policy in your provisioning:
sudo find / -xdev -uid 1005 -exec chown safe_placeholder:safe_placeholder {} +
sudo userdel olduser
```

---

## Orphaned Files (nouser / nogroup)

### Files can reference a UID/GID that no longer maps to anyone
```bash
sudo find / -xdev -nouser
# Lists every file whose UID doesn't resolve to any current /etc/passwd entry

sudo find / -xdev -nogroup
# Same, for GID and /etc/group

# These aren't necessarily broken or dangerous, but they're worth auditing —
# especially after account deletions, migrations, or restoring old backups —
# since an orphaned UID can be silently "claimed" the next time that
# numeric ID is reassigned to a brand-new user (see previous section).
```

---

## Ordinary Users Can Change Group — But Only to Groups They're In

### A subtle permission boundary that surprises people
```bash
groups alice
# alice : alice staff developers

# alice owns her own file:
chown :developers myfile.txt    # ✅ works — alice IS a member of "developers"
chown :root myfile.txt          # ❌ fails — alice is NOT a member of "root"
# chown: changing group of 'myfile.txt': Operation not permitted

chown bob myfile.txt            # ❌ fails — ordinary users can NEVER change the owner
# chown: changing ownership of 'myfile.txt': Operation not permitted

# Root bypasses all of this:
sudo chown bob:root myfile.txt  # ✅ works, unconditionally
```

---

## NFS and Cross-Machine UID Mismatches

### The same numeric UID can mean different people on different machines
```bash
# On machine A: UID 1000 = alice
# On machine B (NFS server): UID 1000 = bob

# alice on machine A creates a file on an NFS share:
touch /nfs/share/file.txt
ls -l /nfs/share/file.txt
# -rw-r--r-- 1 alice alice ... file.txt   (as seen from machine A)

# The SAME file, viewed from machine B:
ls -l /nfs/share/file.txt
# -rw-r--r-- 1 bob bob ... file.txt   ⚠️ shows as "bob" locally — same UID, different name!

# The kernel/NFS layer only ever sees "1000" — the NAME shown depends
# entirely on each individual machine's own /etc/passwd (or NSS source).
# Fix: use centralized identity (LDAP/NIS/sssd) so UID→name mapping is
# consistent everywhere, or use NFSv4 with proper name-based ID mapping.
```

---

## Docker Bind Mounts and Host UID Collisions

### Files created in a container appear owned by an unrelated host account
```bash
# Container runs internally as UID 1000 (its own "app" user)
docker run -v $(pwd):/app myimage touch /app/newfile.txt

# On the HOST machine, UID 1000 might belong to a completely different
# person, or to no one at all:
ls -l newfile.txt
# -rw-r--r-- 1 1000 1000 ... newfile.txt
# If the host's own primary user isn't UID 1000, `ls -l` may show a
# stranger's username, or just the raw number if unmapped.

# This can also break things the other direction: if the HOST user tries
# to edit a file the CONTAINER later needs to read/write as its own UID,
# permission errors appear inside the container despite looking "fine" on the host.

# Fixes:
sudo chown $(id -u):$(id -g) newfile.txt          # reclaim on host
docker run --user $(id -u):$(id -g) ...           # match UIDs from the start
```

---

## chown Doesn't Change Permission Bits

### A file can look "fixed" by chown but still be completely inaccessible
```bash
sudo chown alice:staff file.txt
ls -l file.txt
# ----------  1 alice staff ... file.txt   ← mode was already 000!

# chown changed WHO owns it, but the PERMISSION BITS (rwx) are untouched.
# alice is now correctly the owner, but with mode 000 she still can't
# read/write/execute the file until a SEPARATE chmod is run.
chmod 644 file.txt   # now alice can actually use it
```

---

## --preserve-root Bypassed by Glob Expansion

### The safeguard only checks the literal argument, not the effective result
```bash
sudo chown -R alice:alice /
# ⚠️ blocked by default:
# chown: it is dangerous to operate recursively on '/'
# chown: use --no-preserve-root to override this failsafe

sudo chown -R alice:alice /*
# ⚠️ NOT blocked — the shell expands /* into /bin /etc /home /var ...
# individually BEFORE chown ever sees a literal "/", so the guard never
# triggers, even though the practical effect is nearly as destructive.
```

---

## Legacy Dot Separator Ambiguity

### The old alice.group syntax breaks with usernames containing a literal dot
```bash
# Historically, chown also accepted a dot as owner:group separator:
chown alice.staff file.txt     # legacy form — still works on GNU chown

# But if a username literally CONTAINS a dot (rare, but valid on some systems):
chown john.doe:staff file.txt
# Ambiguous: is this user "john.doe" with group "staff", or user "john"
# with group "doe:staff" (malformed)? Behavior can vary and is a known
# source of scripting bugs — ALWAYS prefer the colon form:
chown "john.doe":staff file.txt   # ✅ explicit and unambiguous
```

---

## Disk Quota Surprises

### chown moves a file's accounted usage to the new owner's quota
```bash
# alice has a strict disk quota. A large file is chown'd TO her:
sudo chown alice large_file.img
# The file's size now counts against ALICE's quota, even though she
# didn't create it and may not have had room for it —
# potentially pushing her over quota immediately, blocking her from
# writing anything else until cleanup happens.

quota -u alice   # check her usage/limits after any bulk chown operation
```

---

## chown on Read-Only or Special Filesystems

### Some filesystems silently ignore or reject ownership changes
```bash
# FAT32/exFAT mounts (common on USB drives) don't support per-file
# Unix ownership at all — every file APPEARS owned by whatever
# uid/gid the mount options specify, uniformly:
mount | grep /mnt/usb
# /dev/sdb1 on /mnt/usb type vfat (rw,uid=1000,gid=1000,...)

sudo chown alice /mnt/usb/file.txt
# May silently succeed but have NO real effect — ownership is fixed by
# the mount options (uid=, gid=), not stored per-file on FAT-family filesystems.

# Read-only filesystems (ISO images, some container overlay layers) reject outright:
sudo chown alice /mnt/cdrom/file.txt
# chown: changing ownership of '/mnt/cdrom/file.txt': Read-only file system
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
