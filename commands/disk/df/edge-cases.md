# df — Edge Cases & Gotchas

> df's numbers look authoritative, but reserved blocks, inode exhaustion,
> deleted-open files, and virtual filesystems all produce genuine surprises.

---

## Table of Contents

- [Available Space Less Than (Size - Used)](#available-space-less-than-size---used)
- [Inode Exhaustion — "No Space" With Plenty of Bytes Free](#inode-exhaustion--no-space-with-plenty-of-bytes-free)
- [Deleted-but-Open Files Inflate df Beyond What du Shows](#deleted-but-open-files-inflate-df-beyond-what-du-shows)
- [tmpfs Counts Against RAM, Not Disk](#tmpfs-counts-against-ram-not-disk)
- [Bind Mounts and Overlay Filesystems Show Duplicate/Confusing Entries](#bind-mounts-and-overlay-filesystems-show-duplicateconfusing-entries)
- [df on a Path That Doesn't Exist Yet](#df-on-a-path-that-doesnt-exist-yet)
- [Snap/squashfs Mounts Cluttering Output](#snapsquashfs-mounts-cluttering-output)
- [Network Filesystem Mounts Hanging df Entirely](#network-filesystem-mounts-hanging-df-entirely)
- [100% Used But Still Some "Avail" Showing](#100-used-but-still-some-avail-showing)
- [df Reporting Stale Information After Resizing a Filesystem](#df-reporting-stale-information-after-resizing-a-filesystem)
- [Percentage Rounding Hides Genuinely Full Filesystems](#percentage-rounding-hides-genuinely-full-filesystems)
- [Different df Behavior for a Symlink vs Its Target](#different-df-behavior-for-a-symlink-vs-its-target)

---

## Available Space Less Than (Size - Used)

### The math doesn't add up, and it's not a bug
```bash
df -h /
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        100G   90G    5G  95% /
# ⚠️ 100G - 90G = 10G, but "Avail" shows only 5G — where did the
# other 5G go?

# Most Linux filesystems (notably ext4) reserve a percentage of total
# space (historically 5%) EXCLUSIVELY for the root user, ensuring
# root can still log in, write logs, and fix problems even when a
# filesystem is "full" from a regular user's perspective:
tune2fs -l /dev/sda1 | grep -i reserved
# Reserved block count:     5242880
# Reserved GID:             0
# This reserved margin is SUBTRACTED from what "Avail" reports to
# ordinary users, but root can still write into it if genuinely necessary.

# Adjust (or remove) this reserved percentage if truly needed —
# use with caution, as it exists specifically to prevent
# root-lockout scenarios on a completely full disk:
sudo tune2fs -m 1 /dev/sda1    # reduce reserved margin to 1%
```

---

## Inode Exhaustion — "No Space" With Plenty of Bytes Free

### The classic "df -h looks fine but I still can't create files" mystery
```bash
touch /var/spool/mail/newfile
# touch: cannot touch '/var/spool/mail/newfile': No space left on device

df -h /var
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        50G   12G   36G  25% /var
# ⚠️ Plenty of BYTE space free — this looks completely fine!

df -i /var
# Filesystem      Inodes   IUsed   IFree IUse% Mounted on
# /dev/sda1      3276800 3276800       0  100% /var
# ✅ THIS is the real problem — every single inode is used, so the
# filesystem literally cannot create ANY new file, directory, or
# symlink, regardless of how much raw disk space remains.

# Classic cause: enormous numbers of TINY files (a mail spool storing
# one file per message, aggressive caching, session storage) — each
# file consumes exactly ONE inode no matter how small its content is.

# ALWAYS check BOTH df -h and df -i together when diagnosing "No
# space left on device," since the identical error message appears
# for both causes, but the FIX is completely different (deleting a
# few large files won't help inode exhaustion at all; you need to
# reduce the NUMBER of files, not their total size).
```

---

## Deleted-but-Open Files Inflate df Beyond What du Shows

### The classic df/du mismatch, from df's side of the story
```bash
df -h /var
# /dev/sda1   50G   45G    5G  90%   /var
# ⚠️ 90% full according to df

du -sh /var 2>/dev/null
# 20G     /var
# ⚠️ But everything VISIBLE only adds up to 20G — a massive discrepancy!

# df is CORRECT here: it reflects actual disk blocks allocated,
# including space held by files that have been deleted but are still
# OPEN by a running process (no directory entry remains for du to see,
# but the kernel hasn't released the actual disk blocks yet, since
# some process still has the file descriptor open).

sudo lsof +L1 2>/dev/null | grep -i deleted
# Reveals exactly which process is holding onto deleted-but-still-
# allocated file space — restarting or signaling that process
# typically releases the space and reconciles df's number downward.
```

---

## tmpfs Counts Against RAM, Not Disk

### "Disk space" that isn't really disk space at all
```bash
df -h
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        100G   68G   28G  71% /
# tmpfs             7.8G   1.2G   6.6G  16% /dev/shm
# tmpfs             7.8G   45M   7.7G   1% /run

# tmpfs entries represent RAM-backed virtual filesystems — their
# reported "Size" typically scales with total available SYSTEM MEMORY
# (often half of RAM by default), NOT any actual physical disk space.
# Writing heavily to a tmpfs-backed directory (like /dev/shm)
# consumes REAL MEMORY, potentially triggering OOM (out-of-memory)
# conditions rather than a disk-full error, even though df displays
# it in exactly the same table alongside genuine disk filesystems.

free -h
# Compare tmpfs's reported size against actual system RAM to
# understand the relationship — a system with 16GB RAM commonly
# shows tmpfs entries sized around 8GB (half of RAM) by default.
```

---

## Bind Mounts and Overlay Filesystems Show Duplicate/Confusing Entries

### The same underlying storage can appear multiple times, seemingly redundantly
```bash
df -h
# /dev/sda1   100G   68G   28G  71%  /
# /dev/sda1   100G   68G   28G  71%  /var/www/data    ← SAME device,
# listed AGAIN because /var/www/data is a BIND MOUNT of a directory
# that's still fundamentally part of the SAME underlying filesystem —
# not a separate disk with its own independent capacity, even though
# it visually appears as its own row.

# Docker's overlay2 storage driver produces similarly confusing entries:
df -h | grep overlay
# overlay   100G   68G   28G  71%  /var/lib/docker/overlay2/abc.../merged
# overlay   100G   68G   28G  71%  /var/lib/docker/overlay2/def.../merged
# Each RUNNING container gets its OWN overlay mount entry, but they
# all typically share and report against the SAME underlying host
# filesystem's capacity — dozens of running containers can produce
# dozens of seemingly redundant overlay rows, all reflecting the
# SAME actual disk space, not dozens of independent capacity pools.
```

---

## df on a Path That Doesn't Exist Yet

### A typo or a not-yet-created directory produces a confusing error, not a helpful one
```bash
df -h /var/www/nonexistent_directory
# df: /var/www/nonexistent_directory: No such file or directory
# ⚠️ df needs an EXISTING path to determine which filesystem to
# report on — it can't answer "how much space would be available
# for a directory I haven't created yet" directly.

# Check the PARENT directory instead, which reflects the SAME
# filesystem the not-yet-created path would eventually belong to:
df -h /var/www
```

---

## Snap/squashfs Mounts Cluttering Output

### A system with many snap packages installed shows dozens of tiny, read-only entries
```bash
df -h
# /dev/loop0   55M   55M     0 100% /snap/core20/1234
# /dev/loop1   47M   47M     0 100% /snap/snapd/5678
# /dev/loop2   90M   90M     0 100% /snap/lxd/9012
# ... (potentially dozens more) ...
# /dev/sda1   100G   68G   28G  71% /
# ⚠️ Each snap package creates its own read-only squashfs mount,
# always shown as "100% used" (since it's a fixed-size, read-only
# image, filled exactly to its own capacity by design) — this is
# completely normal and doesn't indicate any actual problem, but can
# make the REAL disk usage entry (the actual /dev/sda1 row) harder to
# spot amid the noise.

df -h -x squashfs
# ✅ Filters out ALL squashfs entries, leaving just the genuinely
# relevant filesystem usage information
```

---

## Network Filesystem Mounts Hanging df Entirely

### A single unreachable NFS mount can freeze df for EVERY filesystem
```bash
df -h
# (hangs indefinitely, no output at all)
# ⚠️ If ANY mounted network filesystem (NFS, CIFS) is currently
# UNREACHABLE (server down, network partition), plain `df` can hang
# waiting on that ONE problematic mount, even though every OTHER
# local filesystem would report just fine — df processes filesystems
# sequentially and a single stuck one can block the entire command's output.

# Fix: exclude network filesystem types explicitly, or use a timeout
timeout 5 df -h
# Times out after 5 seconds rather than hanging forever, though this
# may still produce incomplete/truncated output for the run

df -l
# --local restricts df to LOCAL filesystems only from the start,
# proactively avoiding any network mount that might hang
```

---

## 100% Used But Still Some "Avail" Showing

### An apparent contradiction, explained by reserved blocks again
```bash
df -h /var
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        50G   50G  1.2G 100% /var
# ⚠️ "100%" used, yet "Avail" shows 1.2G remaining — seemingly
# contradictory at first glance.

# This is the SAME reserved-blocks mechanism mentioned earlier: the
# percentage calculation is based on space available to a NORMAL
# user (which correctly reads as 100% once that ordinary-user quota
# is exhausted), while "Avail" reflects the ADDITIONAL root-reserved
# margin still technically present on the underlying filesystem —
# not actually usable by regular users, but real space nonetheless
# from root's perspective.
```

---

## df Reporting Stale Information After Resizing a Filesystem

### Growing a partition doesn't automatically update df's view of it
```bash
# After resizing an underlying partition/LVM volume with fdisk/parted/lvextend...
sudo lvextend -L +50G /dev/vg0/data
df -h /data
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/vg0-data    100G   90G   10G  90%   /data
# ⚠️ Still shows the OLD size! Extending the underlying BLOCK DEVICE
# doesn't automatically resize the FILESYSTEM sitting on top of it —
# these are two genuinely separate steps.

# The filesystem itself must ALSO be explicitly grown to recognize
# and use the newly available space:
sudo resize2fs /dev/vg0/data      # for ext4
# or:
sudo xfs_growfs /data              # for XFS

df -h /data
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/vg0-data    150G   90G   60G  60%   /data
# ✅ NOW correctly reflects the larger size, after the filesystem
# itself was explicitly told to grow into the newly available space.
```

---

## Percentage Rounding Hides Genuinely Full Filesystems

### "99%" can mean anywhere from "almost fine" to "one write away from failure"
```bash
df -h /
# /dev/sda1   500G  495G  4.8G  99%   /
# "99%" doesn't distinguish between "4.8GB genuinely free, plenty of
# room for normal operations" and a filesystem that's ACTUALLY
# effectively full for practical purposes (large files, database
# growth, log rotation) — always check the ACTUAL "Avail" column
# value in absolute terms too, not just the percentage, especially
# on very large filesystems where even "99%" can still represent a
# meaningful absolute amount of free space.
```

---

## Different df Behavior for a Symlink vs Its Target

### df follows symlinks to report the TARGET's filesystem, which can surprise people
```bash
ln -s /mnt/external/bigfile.dat ~/bigfile_link.dat
df -h ~/bigfile_link.dat
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sdb1        500G  200G  280G  42%  /mnt/external
# ⚠️ df reports the filesystem of the SYMLINK'S TARGET (the external
# drive), NOT the filesystem where the symlink ITSELF technically
# resides (which might be your home directory's own, separate
# filesystem) — this is usually the intuitively "correct" behavior
# for most purposes, but worth knowing explicitly if you specifically
# needed information about the symlink's own containing filesystem instead.
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
