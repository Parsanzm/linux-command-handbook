# df — The Complete Reference

> **Disk Free: see how much space is used and available on mounted filesystems**
> Present since Version 1 Unix (1971) — du's older sibling, answering the complementary question.
> The fastest way to answer "is this server about to run out of disk space?"

---

## Table of Contents

- [What is df?](#what-is-df)
- [Where does df live?](#where-does-df-live)
- [How df works internally](#how-df-works-internally)
- [Syntax](#syntax)
- [Basic Usage](#basic-usage)
- [Understanding the Output Columns](#understanding-the-output-columns)
- [Human-Readable Sizes](#human-readable-sizes)
- [All Key Options](#all-key-options)
- [Inodes — A Second Way to "Run Out of Space"](#inodes--a-second-way-to-run-out-of-space)
- [Filtering by Filesystem Type](#filtering-by-filesystem-type)
- [df and Special/Virtual Filesystems](#df-and-specialvirtual-filesystems)
- [df vs du](#df-vs-du)
- [Related Commands](#related-commands)

---

## What is df?

`df` stands for **Disk Free**. It reports how much space is used and available on mounted filesystems, giving a fast, filesystem-wide summary — the standard first command to run when checking "how full is this disk?" or diagnosing a "no space left on device" error.

```bash
df -h
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        100G   68G   28G  71% /
# tmpfs             3.9G     0  3.9G   0% /dev/shm
# /dev/sdb1         500G  320G  155G  68% /data
```

**Why df is usually the very FIRST command run in a disk-space investigation:** unlike `du`, which must walk an entire directory tree to compute usage (potentially slow on large trees), `df` reads pre-aggregated **filesystem-level metadata** directly, making it nearly instantaneous regardless of how many files exist — the ideal quick health check before diving deeper with `du` to find the specific cause.

---

## Where does df live?

```
/bin/df
/usr/bin/df
```

```bash
which df
df --version
# df (GNU coreutils) 9.4
```

Part of **GNU coreutils** on Linux, and part of the POSIX standard utilities — present on virtually every Unix-like system. macOS/BSD's `df` has some differences in default output columns and flag behavior compared to GNU df.

---

## How df works internally

### Reading filesystem superblock statistics, not walking any tree

`df` retrieves its numbers via the `statfs()` (or `statvfs()`) system call, which asks the kernel directly for a filesystem's **total size**, **free space**, and **used space** — information the filesystem itself maintains and updates continuously as files are created, deleted, or resized. This is fundamentally different from `du`, which must recursively examine every file to compute a sum.

```bash
# df's near-instant speed comes from asking the kernel a SINGLE
# question per filesystem ("how much space total/free?"), rather than
# summing up potentially millions of individual file sizes:
df -h /var/lib/docker
# (0.01 seconds, regardless of how many files exist inside)

du -sh /var/lib/docker
# (could take minutes on a directory with millions of files)
```

### Why df's numbers can seem to "disagree" with what you'd expect

Because df reports the filesystem's own bookkeeping (blocks allocated, whether or not a directory entry currently references them), it accurately reflects space consumed by **deleted-but-still-open files** — space that a directory-tree-walking tool like `du` can never see, since there's no visible directory entry for it anymore. This is the single most common cause of "df says the disk is full, but du says everything only adds up to much less" confusion (see [df vs du](#df-vs-du) below and du's own edge-cases documentation for a full example).

---

## Syntax

```bash
df [OPTIONS] [FILE/FILESYSTEM...]
```

```bash
df                          # show ALL mounted filesystems, default block size (usually 1K)
df -h                        # human-readable sizes (K, M, G)
df -h /                      # show usage for just the filesystem containing /
df -h /path/to/file.txt      # show usage for the filesystem CONTAINING that file
```

---

## Basic Usage

```bash
# Show all mounted filesystems
df

# Human-readable sizes — almost always what you actually want
df -h

# Show usage for the filesystem that a SPECIFIC path lives on
df -h /home/alice
df -h /var/log/syslog

# Show usage for the filesystem containing the CURRENT directory
df -h .

# Show only filesystem type alongside the usual columns
df -hT
```

---

## Understanding the Output Columns

```bash
df -h
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        100G   68G   28G  71% /
tmpfs            3.9G     0  3.9G   0% /dev/shm
/dev/sdb1        500G  320G  155G  68% /data
```

| Column | Meaning |
|--------|---------|
| **Filesystem** | The underlying device or virtual filesystem providing this storage |
| **Size** | Total size of the filesystem |
| **Used** | Space currently in use |
| **Avail** | Space still available (note: this can be LESS than `Size - Used` due to reserved blocks, see edge cases) |
| **Use%** | Percentage of Size currently used |
| **Mounted on** | The mount point — where in the directory tree this filesystem is attached |

---

## Human-Readable Sizes

```bash
# Default (no -h): sizes in 1K blocks — hard to read at a glance
df
# Filesystem     1K-blocks     Used Available Use% Mounted on
# /dev/sda1      104857600 71303168  33030144  69% /

# -h : human-readable, auto-selecting K/M/G/T
df -h
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        100G   68G   28G  69% /

# --si : human-readable using powers of 1000 instead of -h's powers
# of 1024 (matches how storage manufacturers advertise capacity)
df --si
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1       108G   73G   34G  69% /
# (slightly DIFFERENT numbers than -h, due to the 1000 vs 1024 base)

# Force a SPECIFIC unit explicitly
df -BM     # force megabytes
df -BG     # force gigabytes
```

---

## All Key Options

| Option | Long | Description |
|--------|------|--------------|
| `-h` | `--human-readable` | Sizes in K/M/G/T (powers of 1024) |
| `--si` | | Sizes in K/M/G/T (powers of 1000, like storage manufacturers use) |
| `-T` | `--print-type` | Include the filesystem TYPE column (ext4, xfs, tmpfs, etc.) |
| `-i` | `--inodes` | Show INODE usage instead of block/space usage |
| `-a` | `--all` | Include pseudo/virtual filesystems normally hidden (like `/proc`) |
| `-t TYPE` | `--type=TYPE` | Show ONLY filesystems of a specific type |
| `-x TYPE` | `--exclude-type=TYPE` | Exclude filesystems of a specific type |
| `-l` | `--local` | Show only LOCAL filesystems, excluding network mounts (NFS, etc.) |
| `-x` (also) | | Exclude filesystems (differs contextually from `-x TYPE` in some option orderings — check `--exclude-type` explicitly for clarity) |
| `-P` | `--portability` | POSIX-compliant output format (one line per filesystem, no wrapping) |
| `--total` | | Add a grand TOTAL line summing across all listed filesystems |

```bash
df -hT                        # human-readable + filesystem type
df -i                          # inode usage instead of block usage
df -t ext4                      # only ext4 filesystems
df -x tmpfs                      # exclude tmpfs (virtual memory-backed) entries
df -l                            # local filesystems only, skip NFS mounts
df -h --total                    # add a combined total row at the end
```

---

## Inodes — A Second Way to "Run Out of Space"

### A filesystem can report plenty of free BYTES but still be "full"

Every filesystem has a finite number of **inodes** — metadata structures required for every single file or directory, allocated at filesystem-creation time and (on most traditional filesystems) fixed thereafter. It's entirely possible to run out of inodes (and be unable to create any new file) while still having gigabytes of raw byte-space free.

```bash
df -h /var
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        50G   10G   38G  21% /var
# ⚠️ Plenty of BYTE space free (79% available)...

df -i /var
# Filesystem      Inodes   IUsed   IFree IUse% Mounted on
# /dev/sda1      3276800 3270000    6800   99% /var
# ⚠️ ...but only 6,800 INODES remain out of 3.27 million — this
# filesystem is about to become UNABLE to create any new file,
# despite having plenty of raw disk BYTES still free!

# This classically happens with directories containing an enormous
# number of TINY files (mail spools storing one file per message,
# certain cache directories, session storage) — each file, however
# small, consumes exactly one inode regardless of its byte size.

touch /var/somefile
# touch: cannot touch '/var/somefile': No space left on device
# ⚠️ This exact error message appears for BOTH genuine byte-space
# exhaustion AND inode exhaustion — always check BOTH df -h AND df -i
# when diagnosing a "no space left" error, since the fix differs
# significantly depending on which resource is actually exhausted.
```

---

## Filtering by Filesystem Type

```bash
# Show only REAL disk filesystems, common types
df -t ext4 -t xfs -t btrfs

# Exclude common VIRTUAL/pseudo filesystem noise
df -x tmpfs -x devtmpfs -x squashfs

# Show only local filesystems (skip any network-mounted NFS/CIFS shares)
df -l

# See what filesystem TYPES exist on this system at all
df -T | awk '{print $2}' | sort -u
```

---

## df and Special/Virtual Filesystems

```bash
df -h
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        100G   68G   28G  71% /
tmpfs            3.9G     0  3.9G   0% /dev/shm
tmpfs            3.9G  1.2M  3.9G   1% /run
overlay          100G   68G   28G  71% /var/lib/docker/overlay2/abc123/merged
```

- **tmpfs** — a RAM-backed filesystem; "disk space" here is actually system memory, and its reported total typically scales with available RAM, not any physical disk
- **overlay** — the union filesystem type Docker uses for container layers; its numbers typically MIRROR the underlying host filesystem's usage, since it doesn't have independent physical storage of its own
- **squashfs** — a read-only, compressed filesystem type commonly seen for snap packages (each shows as its own tiny "filesystem" entry)

```bash
# Snap packages create MANY squashfs entries, often cluttering df's
# output significantly on a system with many snaps installed
df -h | grep -c squashfs
# 15   ← 15 separate squashfs mount entries, one per installed snap

# Filter these out for a cleaner view
df -h -x squashfs
```

---

## df vs du

| Feature | `df` | `du` |
|---------|------|------|
| What it measures | Entire mounted FILESYSTEM's usage | SPECIFIC files/directories you point it at |
| Speed | Nearly instant (metadata only) | Can be slow (must walk the tree) |
| Granularity | Filesystem/mount-point level | File/directory level |
| Typical first question it answers | "Is any filesystem nearly full?" | "WHICH directory is responsible for that usage?" |

```bash
df -h
# Quickly identifies WHICH filesystem is nearly full

du -sh /var/* 2>/dev/null | sort -rh
# Once df has identified the PROBLEM filesystem (e.g., the one
# containing /var), du narrows down WHICH specific directory within
# it is actually responsible for the bulk of that usage
```

**The classic combined workflow:** run `df -h` first to spot a nearly-full filesystem, then run `du -sh` (often with `--max-depth`) on directories within that filesystem to progressively narrow down the actual cause.

---

## Related Commands

| Command | Relation |
|---------|----------|
| `du` | File/directory-level usage, complementary to df's filesystem-level view |
| `mount` | Shows what's currently mounted and its OPTIONS; df focuses on SPACE usage specifically |
| `lsblk` | Shows block DEVICE structure/hierarchy, a different (though related) view from df's filesystem-usage focus |
| `findmnt` | Modern, flexible tool for querying mount information, can show similar filesystem details |
| `stat -f` | Shows filesystem-level statistics for a SPECIFIC path, similar info to df but for one target |
| `quota` | Per-user/group disk usage LIMITS, a different but related concept from overall filesystem usage |
| `ncdu` | Interactive disk usage EXPLORER (built on du-like tree-walking, not df's metadata approach) |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
