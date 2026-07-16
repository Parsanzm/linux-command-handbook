# du — The Complete Reference

> **Disk Usage: see exactly how much space files and directories actually consume**
> Present since Version 1 Unix (1971) — one of the original, foundational Unix utilities.
> The tool for answering "what's actually eating my disk space?" one directory at a time.

---

## Table of Contents

- [What is du?](#what-is-du)
- [Where does du live?](#where-does-du-live)
- [How du works internally](#how-du-works-internally)
- [Syntax](#syntax)
- [Basic Usage](#basic-usage)
- [Human-Readable Sizes](#human-readable-sizes)
- [Summarizing vs Full Detail (-s, --max-depth)](#summarizing-vs-full-detail--s---max-depth)
- [All Key Options](#all-key-options)
- [Apparent Size vs Disk Usage (Sparse Files, Blocks)](#apparent-size-vs-disk-usage-sparse-files-blocks)
- [Hard Links and Double-Counting](#hard-links-and-double-counting)
- [Excluding Files and Directories](#excluding-files-and-directories)
- [du vs df](#du-vs-df)
- [Related Commands](#related-commands)

---

## What is du?

`du` stands for **Disk Usage**. It reports how much disk space files and directories actually occupy, recursively walking a directory tree and summing up the space used by every file within it. It's the standard tool for answering "what is actually taking up all this space?" — whether that's a single directory, an entire home folder, or a whole filesystem.

```bash
du -sh /home/alice
# 4.2G    /home/alice

du -h /var/log
# 12K     /var/log/apt
# 156K    /var/log/nginx
# 2.3M    /var/log
```

**Why du matters beyond the obvious:** disk space problems are one of the most common operational headaches in system administration, and `du` (usually combined with `sort` to find the biggest offenders) is the standard first step in diagnosing "why is this filesystem full?" — a question that comes up constantly on servers, personal machines, and CI runners alike.

---

## Where does du live?

```
/usr/bin/du
```

```bash
which du
du --version
# du (GNU coreutils) 9.4
```

Part of **GNU coreutils** on Linux, and part of the POSIX standard utilities — present on virtually every Unix-like system, though macOS/BSD's `du` has some option differences from GNU's (notably around `-h` behavior in older versions and block-size defaults).

---

## How du works internally

### Walking the directory tree and summing allocated space

`du` recursively traverses a directory, and for **every file** it finds, adds up the space actually **allocated on disk** for that file — not necessarily the same as the file's apparent logical size (see [Apparent Size vs Disk Usage](#apparent-size-vs-disk-usage-sparse-files-blocks) below). For directories, du reports the **sum of all files within them**, recursively, plus the small amount of space the directory entries themselves occupy.

```bash
# du doesn't just ask the filesystem "how big is this directory?"
# (directories themselves are typically tiny, just listing filenames) —
# it actually walks through and sums up every FILE within the tree:
du -sh /var/log
# 2.3M    /var/log
# This 2.3M is the SUM of every individual log file's disk usage
# inside /var/log and all its subdirectories, not some single
# "directory size" value read directly from metadata.
```

### Blocks, not bytes, by default

Traditionally, `du` reports usage in **disk blocks** (historically 512-byte or 1024-byte units, depending on the system and settings) rather than raw bytes, because that's what actually reflects real space **consumed** on disk — a filesystem allocates space in fixed-size blocks, so even a 1-byte file typically consumes one entire block's worth of actual disk space.

```bash
du file.txt
# 4       file.txt
# "4" here means 4 KB-blocks (assuming 1K block size default) — even
# if file.txt contains just a few bytes of actual text content, it
# still consumes at least one full filesystem block on disk.

ls -l file.txt
# -rw-r--r-- 1 alice alice 12 Jan 15 10:00 file.txt
# The file's LOGICAL size is just 12 bytes, but its ACTUAL disk
# footprint (what du reports) is rounded up to the filesystem's block size.
```

---

## Syntax

```bash
du [OPTIONS] [FILE/DIRECTORY...]
```

```bash
du /path/to/directory           # detailed, recursive breakdown
du -h /path/to/directory        # human-readable sizes
du -sh /path/to/directory       # SUMMARY only — one total line, not every subdirectory
du -a /path/to/directory        # include INDIVIDUAL FILES too, not just directories
```

---

## Basic Usage

```bash
# Full recursive breakdown, every subdirectory shown, in blocks (default)
du /var/log

# Human-readable sizes (K, M, G instead of raw block counts)
du -h /var/log

# Just ONE summary line for the whole directory, not every subdirectory
du -sh /var/log

# Include individual FILES in the output, not just directory totals
du -ah /var/log

# Multiple directories/files at once
du -sh /var/log /var/cache /tmp
```

---

## Human-Readable Sizes

```bash
du -h /home/alice
# 4.0K    /home/alice/.bashrc
# 156M    /home/alice/Documents
# 2.1G    /home/alice/Videos
# 2.3G    /home/alice

# Show sizes in a SPECIFIC fixed unit instead of auto-selected units
du -BM /home/alice          # force megabytes
du -BG /home/alice          # force gigabytes
du --block-size=1M /home/alice   # equivalent, explicit form

# --si uses powers of 1000 (matching how storage manufacturers
# advertise capacity) instead of -h's powers of 1024
du -sh --si /home/alice
# 2.4G    /home/alice   (slightly different number than plain -h,
# due to the 1000 vs 1024 base difference)
```

---

## Summarizing vs Full Detail (-s, --max-depth)

```bash
# Full recursive detail — EVERY subdirectory gets its own line, which
# can be overwhelming for a deep tree
du -h /var
# ... hundreds of lines for every subdirectory ...

# -s : summarize — ONLY the grand total for the given path(s)
du -sh /var
# 5.2G    /var

# --max-depth=N : show detail, but only DOWN TO a specific depth level
du -h --max-depth=1 /var
# 45M     /var/cache
# 2.1G    /var/lib
# 1.8G    /var/log
# 1.3G    /var/www
# 5.2G    /var
# ✅ Often the MOST useful middle ground: enough detail to see which
# TOP-LEVEL subdirectory is the biggest offender, without drowning in
# every single nested file/folder several levels deep.

du -h --max-depth=2 /var
# Shows two levels deep instead of just one — useful for narrowing
# down further once max-depth=1 has identified the general problem area
```

---

## All Key Options

| Option | Long | Description |
|--------|------|--------------|
| `-h` | `--human-readable` | Sizes in K/M/G/T instead of raw block counts |
| `-s` | `--summarize` | Show only a grand total, not every subdirectory |
| `-a` | `--all` | Include individual FILES in output, not just directories |
| `-c` | `--total` | Add a grand TOTAL line at the end (useful with multiple arguments) |
| `--max-depth=N` | | Limit directory recursion depth shown in output |
| `-x` | `--one-file-system` | Don't cross into OTHER mounted filesystems during recursion |
| `-L` | `--dereference` | Follow symlinks (count the TARGET's size, not the link itself) |
| `-l` | `--count-links` | Count size of hard-linked files MULTIPLE times (normally counted once) |
| `--exclude=PATTERN` | | Exclude files/directories matching PATTERN |
| `--exclude-from=FILE` | | Read exclude patterns from a file |
| `--time` | | Show the LAST MODIFICATION TIME alongside each entry |
| `--apparent-size` | | Show LOGICAL file size instead of actual disk block usage |
| `-0` | `--null` | NUL-terminate output lines instead of newline (for safe scripting) |
| `--inodes` | | Count INODES used, instead of disk space |

```bash
du -sh --exclude="*.log" /var         # exclude matching files from the total
du -x -sh /                            # don't cross into other mounted filesystems
du -c -sh */                            # per-directory totals PLUS a grand total
du --apparent-size -sh file.txt         # LOGICAL size, ignoring block rounding
```

---

## Apparent Size vs Disk Usage (Sparse Files, Blocks)

### A sparse file can show WILDLY different sizes between du and ls

```bash
# Create a sparse file — a file that LOGICALLY appears huge, but
# doesn't actually consume that much real disk space, because large
# stretches of it are "holes" (unallocated, read as zeros)
truncate -s 10G sparse_file.img

ls -lh sparse_file.img
# -rw-r--r-- 1 alice alice 10G Jan 15 10:00 sparse_file.img
# ⚠️ ls shows the LOGICAL size: 10 GB

du -h sparse_file.img
# 0       sparse_file.img
# ✅ du shows the ACTUAL disk space consumed: essentially nothing,
# since truncate just created a sparse file with no real data blocks
# allocated at all — the "10G" is purely a logical/apparent size that
# the filesystem promises to provide if/when actually written to.

du --apparent-size -h sparse_file.img
# 10G     sparse_file.img
# ✅ --apparent-size makes du report the LOGICAL size instead,
# matching what ls would show, ignoring actual block allocation.
```

### Why this matters practically

```bash
# Virtual machine disk images, database files with preallocated space,
# and certain container/Docker layer files are commonly SPARSE —
# `ls -l` alone can dramatically OVERSTATE how much real disk space
# they're consuming, while `du` gives the TRUE picture:
ls -lh vm_disk.qcow2
# -rw-r--r-- 1 alice alice  50G ... vm_disk.qcow2    ← looks huge
du -h vm_disk.qcow2
# 12G     vm_disk.qcow2                                ← actual usage is much smaller
```

---

## Hard Links and Double-Counting

### du normally counts a hard-linked file's space only ONCE, even if referenced from multiple places

```bash
echo "shared content" > original.txt
ln original.txt hardlink_copy.txt   # a HARD LINK, not a copy — same underlying inode

du -sh original.txt hardlink_copy.txt
# 4.0K    original.txt
# 4.0K    hardlink_copy.txt
# du -c -sh original.txt hardlink_copy.txt
# 4.0K    original.txt
# 4.0K    hardlink_copy.txt
# 4.0K    total          ← ✅ CORRECTLY counted ONCE in the total, since
# both names point to the exact SAME underlying data on disk — du is
# smart enough to recognize this via matching inode numbers and avoid
# double-counting the SAME physical space twice.

# Force du to count EACH hard-linked reference separately anyway
# (rarely needed, mostly for specific auditing purposes)
du -c -l -sh original.txt hardlink_copy.txt
# 4.0K    original.txt
# 4.0K    hardlink_copy.txt
# 8.0K    total          ⚠️ -l forces double-counting, which usually
# does NOT reflect actual real disk space consumed
```

---

## Excluding Files and Directories

```bash
# Exclude a specific pattern
du -sh --exclude="*.log" /var/lib

# Exclude MULTIPLE patterns (repeat the flag)
du -sh --exclude="*.log" --exclude="*.tmp" /var/lib

# Read exclude patterns from a file (one pattern per line)
cat exclude_patterns.txt
# *.log
# *.tmp
# node_modules
du -sh --exclude-from=exclude_patterns.txt /home/alice/project

# Common practical use: measure a project's size WITHOUT its
# dependency/cache directories, which often dwarf the actual source code
du -sh --exclude="node_modules" --exclude=".git" /home/alice/myproject
```

---

## du vs df

| Feature | `du` | `df` |
|---------|------|------|
| What it measures | Space used by SPECIFIC files/directories | Space used/available on an ENTIRE mounted filesystem |
| Typical use | "What's taking up space INSIDE this directory?" | "How full is this DISK/PARTITION overall?" |
| Speed | Can be SLOW on huge directory trees (must walk every file) | Nearly INSTANT (reads filesystem metadata directly, no tree walk) |
| Granularity | File/directory level | Filesystem/mount-point level |

```bash
df -h
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        500G  320G  155G  68% /
# Tells you the FILESYSTEM is 68% full overall — but not WHICH
# specific directory is responsible for that usage

du -sh /var /home /opt
# Tells you WHICH specific top-level directories are consuming space
# WITHIN that filesystem, letting you narrow down the actual cause
```

**A common real-world mismatch:** `df` can report a filesystem as nearly full, while `du` on all visible files seems to add up to much LESS than that — often caused by a large file that's been **deleted** but is still **held open** by a running process (the space isn't actually freed until the process closes the file, and `du` (which only sees the visible directory tree) has no way to "see" this invisible, still-allocated space at all).

---

## Related Commands

| Command | Relation |
|---------|----------|
| `df` | Filesystem-level usage, complementary to du's file/directory-level detail |
| `ls -l` | Shows a file's LOGICAL size, which can differ from du's actual disk usage (sparse files) |
| `find` | Often combined with du for more targeted searches (e.g., find files over a certain size) |
| `ncdu` | Interactive, TUI-based disk usage analyzer — a popular, friendlier alternative to raw `du` |
| `stat` | Shows a single file's exact size/block metadata in detail |
| `sort` | Frequently paired with du to rank directories by size (`du -h | sort -rh`) |
| `lsof` | Reveals files still held open by running processes (relevant to the du/df space mismatch) |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
