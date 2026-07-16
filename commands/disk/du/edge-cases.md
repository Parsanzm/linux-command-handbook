# du — Edge Cases & Gotchas

> du's numbers can diverge sharply from what ls, df, or a file manager
> reports — sparse files, hard links, mount boundaries, and permissions all matter.

---

## Table of Contents

- [du vs df Showing Wildly Different "Full" Percentages](#du-vs-df-showing-wildly-different-full-percentages)
- [Deleted-but-Open Files Not Counted by du](#deleted-but-open-files-not-counted-by-du)
- [Sparse Files: du and ls Disagree Dramatically](#sparse-files-du-and-ls-disagree-dramatically)
- [Crossing Mount Points Inflates or Confuses Totals](#crossing-mount-points-inflates-or-confuses-totals)
- [Hard Links Silently Prevent Double-Counting](#hard-links-silently-prevent-double-counting)
- [Permission Denied Errors Silently Undercount](#permission-denied-errors-silently-undercount)
- [Block Size Assumptions Vary by System/Filesystem](#block-size-assumptions-vary-by-systemfilesystem)
- [Symlinks: Counted as Tiny, Unless Told to Follow](#symlinks-counted-as-tiny-unless-told-to-follow)
- [du on a Very Large Tree Can Be Extremely Slow](#du-on-a-very-large-tree-can-be-extremely-slow)
- [--exclude Patterns Not Matching as Expected](#--exclude-patterns-not-matching-as-expected)
- [Directory Entry Overhead Itself Counts Toward Usage](#directory-entry-overhead-itself-counts-toward-usage)
- [Network Filesystems (NFS) Reporting Oddities](#network-filesystems-nfs-reporting-oddities)

---

## du vs df Showing Wildly Different "Full" Percentages

### The single most common "disk full mystery" in system administration
```bash
df -h /
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        100G   95G    5G  95% /
# ⚠️ Filesystem reports 95% full!

du -sh / 2>/dev/null
# 60G     /
# ⚠️ But adding up everything VISIBLE only accounts for 60G — where
# did the other 35G go?

# The classic explanation: a large file was DELETED, but a running
# process still has it OPEN. The filesystem can't actually reclaim
# that space until every process holding it closes the file handle —
# but since the file has no directory entry anymore, du (which only
# walks the visible directory tree) has NO WAY to see or count it.

# Diagnosis: find processes holding deleted-but-open files
sudo lsof +L1 2>/dev/null
# Shows files with a link count of 1 or fewer that are still open —
# a strong signal for exactly this scenario. Look for entries marked
# "(deleted)" in the output.

sudo lsof 2>/dev/null | grep deleted
# Alternative search, directly looking for "(deleted)" markers
```

---

## Deleted-but-Open Files Not Counted by du

### A specific, very common cause of the du/df mismatch above
```bash
# A process opens a large log file:
tail -f /var/log/huge_app.log &

# Someone "cleans up" by deleting the file directly:
rm /var/log/huge_app.log
# The FILE ENTRY is gone — du will never see or count it again

du -sh /var/log
# Reports LESS than the actual disk usage, since huge_app.log's space
# is STILL ALLOCATED (the tail -f process still holds it open) but is
# now invisible to any directory-tree-walking tool like du.

df -h /var
# STILL shows the space as used, since the FILESYSTEM correctly knows
# the blocks aren't free yet.

# Fix: restart/signal the process holding it open, so the kernel
# finally releases the space:
sudo systemctl restart the_service_holding_it_open
# or, if you must reclaim space from a log file WITHOUT restarting
# the process, truncate it in place instead of deleting it:
: > /var/log/huge_app.log
# This actually frees space immediately, since the file still EXISTS
# (same inode), just now with zero content — unlike `rm`, which
# removes the directory entry while the underlying data lingers.
```

---

## Sparse Files: du and ls Disagree Dramatically

### A file can be "10GB" by ls but nearly 0 bytes by du
```bash
truncate -s 10G myimage.img
ls -lh myimage.img
# -rw-r--r-- 1 alice alice 10G ... myimage.img    ← LOGICAL size

du -h myimage.img
# 0       myimage.img                              ← ACTUAL disk usage

# This is EXPECTED for sparse files (common with VM disk images,
# preallocated database files, and some container layer formats) —
# NOT a bug, but frequently confuses people who assume ls and du
# should always agree, or who copy such a file with a tool that
# doesn't preserve sparseness (turning a "nearly free" 10GB sparse
# file into a genuinely full 10GB of real disk usage after copying):

cp myimage.img copy.img              # ⚠️ may "de-sparsify" the file!
du -h copy.img
# 10G     copy.img    ← the naive copy might have ACTUALLY allocated
# all 10GB for real, since plain cp doesn't always preserve holes

cp --sparse=always myimage.img copy2.img   # ✅ explicitly preserve sparseness
du -h copy2.img
# 0       copy2.img
```

---

## Crossing Mount Points Inflates or Confuses Totals

### du recurses into OTHER mounted filesystems by default, which can produce misleading totals
```bash
# /mnt/external is a SEPARATE mounted filesystem (e.g., a USB drive)
# nested inside the path being measured
du -sh /home
# 500G    /home
# ⚠️ If /home/alice/external_backup happens to be a MOUNT POINT for a
# separate 480GB external drive, this total INCLUDES that entire
# external filesystem's usage too — potentially giving a wildly
# misleading picture of how much space "/home itself" (on its OWN
# filesystem) actually consumes.

# Fix: use -x to stay within the STARTING filesystem, not crossing
# into any OTHER mounted filesystem along the way
du -xsh /home
# 20G     /home     ← now correctly excludes the external mount's
# contents, showing only what's genuinely on /home's own filesystem
```

---

## Hard Links Silently Prevent Double-Counting

### A "surprising" total that's actually CORRECT behavior
```bash
# A backup tool creates hard links to avoid duplicating unchanged
# files across multiple backup snapshots (a very common space-saving technique)
ls -li /backups/day1/file.txt /backups/day2/file.txt
# 123456 -rw-r--r-- 2 alice alice ... /backups/day1/file.txt
# 123456 -rw-r--r-- 2 alice alice ... /backups/day2/file.txt
# SAME inode number (123456) — these are hard links to the identical data

du -sh /backups/day1 /backups/day2
# 500M    /backups/day1
# 500M    /backups/day2

du -csh /backups/day1 /backups/day2
# 500M    /backups/day1
# 500M    /backups/day2
# 500M    total          ← ✅ CORRECT — NOT 1000M, because du recognizes
# both paths reference the SAME underlying disk blocks via matching
# inodes, and doesn't double-count real disk usage that's actually shared.

# This is often EXACTLY what backup tools like rsync --link-dest or
# rsnapshot rely on — hard links make many "full" snapshots consume
# barely more real disk space than ONE, and du's default counting
# behavior correctly reflects that genuine space efficiency.
```

---

## Permission Denied Errors Silently Undercount

### du can report an incomplete total without making the omission obvious
```bash
du -sh /home 2>/dev/null
# 45G     /home
# ⚠️ Suppressing stderr with 2>/dev/null HIDES "Permission denied"
# messages for directories du couldn't read into — the reported total
# SILENTLY EXCLUDES whatever was inside those inaccessible directories,
# with no visible indication that the number is actually incomplete.

du -sh /home
# du: cannot read directory '/home/bob/.private': Permission denied
# 45G     /home
# ✅ WITHOUT suppressing stderr, the warning is visible, making clear
# the "45G" total is a LOWER BOUND, not necessarily the complete picture.

# Run with sudo for a genuinely complete picture across other users'
# directories, if that's actually needed/appropriate:
sudo du -sh /home
```

---

## Block Size Assumptions Vary by System/Filesystem

### The SAME logical file size can show different du output across systems
```bash
# On a filesystem with a 4K block size:
du file.txt
# 4       file.txt    (rounds up to one 4K block, in 1K-block units = 4)

# On a filesystem with a DIFFERENT block size (less common, but
# possible depending on filesystem type/configuration):
du file.txt
# 8       file.txt    (same logical file, different underlying block
# size means a different ACTUAL disk footprint, even for identical content)

# Always specify an explicit, consistent unit when comparing sizes
# across DIFFERENT machines/filesystems in scripts, rather than
# assuming du's default block-count output means the same thing everywhere:
du -BK file.txt      # force explicit KB units, more portable/comparable
du --apparent-size -BK file.txt   # or compare LOGICAL size instead,
# which is unaffected by underlying block-size differences entirely
```

---

## Symlinks: Counted as Tiny, Unless Told to Follow

### A directory full of symlinks to huge files can look deceptively small
```bash
ln -s /data/huge_file.dat link_to_huge.dat
du -sh link_to_huge.dat
# 4.0K    link_to_huge.dat
# ⚠️ By default, du reports the SYMLINK's OWN tiny size (just the
# space needed to store the path string it points to), NOT the size
# of whatever it points TO.

du -Lsh link_to_huge.dat
# 2.3G    link_to_huge.dat
# ✅ -L (--dereference) makes du follow the symlink and report the
# TARGET's actual size instead — essential when a directory of
# symlinks might otherwise appear nearly empty despite pointing at
# genuinely large files elsewhere.

# This matters a lot for directories built around symlink-based
# organization schemes (like some media library tools, or certain
# backup/deduplication systems using symlinks internally)
```

---

## du on a Very Large Tree Can Be Extremely Slow

### Millions of small files make du (and any tree walk) inherently slow
```bash
time du -sh /var/lib/docker
# real    2m47.319s
# ⚠️ Directories containing MILLIONS of small files (common with
# Docker's overlay filesystem layers, node_modules-heavy JS projects,
# or large mail-spool directories using one-file-per-message storage)
# can take a genuinely long time, since du must stat() EVERY individual
# file to sum up its actual disk usage — there's no shortcut around
# walking the entire tree for this level of detail.

# If a QUICK filesystem-level answer is acceptable instead of precise
# per-directory detail, df is far faster (metadata-only, no tree walk):
df -h /var/lib/docker
# near-instant, but only tells you about the WHOLE filesystem's usage,
# not specifically what WITHIN /var/lib/docker is responsible

# ncdu (a popular third-party tool) also walks the tree but caches/
# presents results interactively, often feeling faster in PRACTICE
# for repeated exploration, even though the underlying work is similar
```

---

## --exclude Patterns Not Matching as Expected

### Glob patterns in --exclude need to match the FULL path, not just a basename
```bash
du -sh --exclude="node_modules" /home/alice/project
# ⚠️ May NOT exclude anything if "node_modules" appears NESTED deep
# inside subdirectories, depending on du's specific pattern-matching
# behavior — --exclude patterns are matched against the ENTIRE path
# being examined, and a bare "node_modules" without wildcards might
# only match a TOP-LEVEL directory named exactly that, not
# "/home/alice/project/packages/foo/node_modules" nested deeper.

# More reliable: use a wildcard to match "node_modules" appearing
# ANYWHERE in the path, at any depth
du -sh --exclude="*/node_modules/*" --exclude="*/node_modules" /home/alice/project

# Always TEST exclude patterns explicitly before trusting them in an
# important script, since silent under-exclusion (or over-exclusion)
# is easy to miss without verification:
du -ah --exclude="*/node_modules/*" /home/alice/project | grep node_modules
# (should show NOTHING if the exclude pattern is working correctly)
```

---

## Directory Entry Overhead Itself Counts Toward Usage

### An "empty" directory with thousands of former entries can still show non-zero usage
```bash
mkdir emptydir
du -sh emptydir
# 4.0K    emptydir    ← even a genuinely EMPTY directory consumes SOME
# space, since directory entries themselves require disk blocks to
# store (even the "empty" list of zero files still needs a block to
# represent that empty state on most filesystems)

# A directory that ONCE contained many thousands of files, then had
# them all deleted, can sometimes retain a LARGER-than-minimal
# directory entry footprint even after everything inside is gone,
# depending on the specific filesystem's directory-shrinking behavior:
du -sh formerly_huge_dir
# 128K    formerly_huge_dir   ⚠️ larger than the typical 4.0K minimum,
# even though `ls formerly_huge_dir` shows it's now completely empty —
# some filesystems don't automatically shrink directory metadata back
# down after mass deletions, though this varies significantly by
# filesystem type (ext4, XFS, Btrfs all behave somewhat differently here).
```

---

## Network Filesystems (NFS) Reporting Oddities

### du over NFS can be slower and occasionally less precise than on local disks
```bash
du -sh /mnt/nfs_share/bigproject
# Over NFS, EVERY file stat() call involves a network round-trip,
# making du dramatically slower on NFS-mounted directories compared
# to the equivalent LOCAL filesystem path — especially noticeable on
# trees with many small files, where the network latency per-file
# adds up significantly.

# Additionally, depending on the NFS server's own underlying
# filesystem and how it reports block usage, du's numbers over NFS
# might not perfectly match what the SAME directory would show if
# examined directly ON the NFS server itself, due to differences in
# how block size / sparse file information is communicated across
# the NFS protocol layer.

# For a large NFS-mounted tree, consider running du DIRECTLY on the
# NFS SERVER (if you have access) rather than over the network mount,
# for both speed and potentially more accurate reporting.
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
