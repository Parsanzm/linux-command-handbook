# du — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [Apparent Size vs Actual Usage](#apparent-size-vs-actual-usage)
- [Hard Links & Symlinks](#hard-links--symlinks)
- [du vs df](#du-vs-df)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does du do, and what specifically does it measure?**
> `du` (**Disk Usage**) recursively walks a directory tree and reports how much disk space is actually **allocated** for the files within it — not necessarily the same as a file's logical/apparent size, since disk space is allocated in fixed-size blocks and actual usage is rounded up to block boundaries.

---

**Q2 🔥 Why does du report sizes in blocks by default rather than raw bytes?**
> Because filesystems allocate space in fixed-size blocks, and du is meant to reflect **actual disk space consumed**, not a file's logical content length — even a tiny file typically consumes at least one full block's worth of real disk space, and du's block-based reporting reflects that real-world allocation rather than the file's exact byte count.

---

## Basic Usage

**Q3 🔥 What's the difference between `du -h /var/log` and `du -sh /var/log`?**
> `du -h` shows a full recursive breakdown, with a separate line for every subdirectory. `du -sh` (`--summarize`) collapses this down to just **one** total line for the entire specified path, without listing every subdirectory individually.

---

**Q4. How would you see disk usage broken down just one level deep, instead of either a single total or an overwhelming full recursive listing?**
> ```bash
> du -h --max-depth=1 /var
> ```
> This shows each immediate subdirectory's total plus the grand total, without descending further into deeper nested subdirectories — often the most useful middle ground for quickly identifying which top-level area is consuming the most space.

---

**Q5 🔥 How do you include individual files (not just directory totals) in du's output?**
> ```bash
> du -ah /path
> ```
> `-a` (`--all`) includes individual files alongside directory totals; by default, du only reports directory-level sums, not each file within them.

---

## Apparent Size vs Actual Usage

**Q6 🔥 What is a sparse file, and why might `ls -lh` and `du -h` report drastically different sizes for the same file?**
> A sparse file is one with unallocated "holes" — regions the filesystem promises to provide as zeros on read, without actually storing real data blocks for them. `ls -lh` reports the file's **logical/apparent size** (including the unallocated holes as if they were real content), while `du -h` reports **actual disk blocks allocated** — for a heavily sparse file (common with VM disk images or preallocated database files), these numbers can differ by orders of magnitude.

---

**Q7. How would you make du report a file's logical size instead of its actual disk usage, matching what `ls -l` would show?**
> ```bash
> du --apparent-size -h file.txt
> ```
> This overrides du's normal actual-disk-usage reporting to instead show the logical/apparent size, ignoring block allocation and sparseness entirely.

---

**Q8 🔥 If you copy a sparse file with a naive `cp`, what might happen to its disk usage, and how do you prevent this?**
> A naive copy may not preserve the sparseness, potentially writing out actual zero-filled data for what were previously unallocated holes — turning a nearly-empty sparse file into one that genuinely consumes its full logical size in real disk space. Using `cp --sparse=always` explicitly preserves the sparse structure during the copy, keeping the copy's actual disk usage low, matching the original.

---

## Hard Links & Symlinks

**Q9 🔥 If two files are hard-linked to each other, does du count their shared space twice when both are included in a total?**
> No — by default, du recognizes hard-linked files (files sharing the same inode) and counts their shared underlying disk space only **once** in a combined total (`du -c`), correctly reflecting that they don't actually consume double the real disk space despite appearing as two separate file names.

---

**Q10. How would you force du to count each hard-linked reference's size separately, even though this doesn't reflect real disk usage?**
> ```bash
> du -l ...
> ```
> `-l` (`--count-links`) disables the default deduplication behavior for hard links, counting each reference's size independently — rarely needed in practice, since it produces a total that overstates actual disk consumption.

---

**Q11 🔥 By default, does du follow symlinks and report the size of what they point to, or the symlink's own (tiny) size?**
> By default, du reports the symlink's own small size (just enough to store the target path string), **not** the size of whatever it points to. Using `-L` (`--dereference`) makes du follow symlinks and report the target's actual size instead.

---

## du vs df

**Q12 🔥 What's the fundamental difference between du and df?**
> `du` measures disk space used by **specific files and directories** you point it at, by walking the directory tree. `df` reports usage/availability for an **entire mounted filesystem**, reading filesystem-level metadata directly without walking any directory tree — making df essentially instantaneous regardless of how many files exist, while du's speed depends on the number of files it must examine.

---

**Q13. A filesystem reported by `df` as 95% full doesn't match the total from summing up `du` across all visible directories, which only accounts for a much smaller percentage. What's a common explanation for this discrepancy?**
> A large file was likely **deleted** while still being held **open** by a running process — the filesystem correctly still considers that space "used" (df reflects this accurately), but since the file has no remaining directory entry, `du` (which only walks the visible directory tree) has no way to see or count that still-allocated space at all. Checking for open-but-deleted files (e.g., via `lsof +L1` or searching for "(deleted)" entries in `lsof` output) typically reveals the culprit.

---

**Q14 🔥 If a large log file is deleted while still being written to by an active process, how do you actually reclaim the disk space without restarting the process?**
> Rather than deleting the file (which leaves its space allocated but invisible), **truncate** it in place while it still exists: `: > /path/to/logfile.log`. Since this operates on the same inode the process still has open, it immediately frees the file's contents while leaving the file (and the process's open handle to it) intact.

---

## Scenario-Based

**Q15 🔥 Running `du -sh /home` inside a directory that has an external drive mounted at a subdirectory (e.g., `/home/alice/backup_drive`) reports a total far larger than expected. What's happening, and how do you fix it?**
> By default, du recurses into **any** mounted filesystem it encounters while walking the tree, including a separate external drive mounted somewhere inside the path being measured — so the reported total includes that entire external filesystem's usage too, not just what's genuinely on `/home`'s own filesystem. The fix is `-x` (`--one-file-system`), which stops du from crossing into other mounted filesystems: `du -xsh /home`.

---

**Q16. A script running `du -sh /home 2>/dev/null` in a scheduled report consistently underestimates disk usage compared to a manual check, with no visible errors. What's the likely cause?**
> Redirecting stderr to `/dev/null` silently suppresses "Permission denied" messages for any subdirectories du couldn't read into (e.g., other users' private directories) — the reported total quietly **excludes** whatever was inside those inaccessible paths, with nothing in the visible output indicating the number is incomplete. Running with appropriate elevated privileges (`sudo du -sh /home`) and reviewing stderr (rather than discarding it) reveals whether this is actually happening.

---

**Q17 🔥 A directory of symlinks pointing to large media files reports an almost negligible size with `du -sh`, even though the actual referenced files are clearly enormous. Is this a bug, and how would you get the real total?**
> Not a bug — by default, du reports only the tiny space a symlink itself occupies (the path string it stores), not the size of whatever it points to. Using `-L` (`--dereference`) makes du follow each symlink and count the target file's actual size instead: `du -Lsh symlink_directory/`.

---

**Q18. A backup system using hard links across daily snapshots (common with tools like rsync --link-dest) shows each snapshot as "500MB" individually, but the combined total (`du -c`) for all 30 daily snapshots is nowhere near 30 × 500MB. Is this expected, and why?**
> Yes, this is expected and is precisely the intended benefit of the hard-link-based snapshot strategy: files unchanged between snapshots are hard-linked rather than duplicated, sharing the same underlying disk blocks across multiple snapshot directories. du correctly recognizes these shared inodes and counts the real disk space only once in a combined total, rather than naively multiplying the apparent per-snapshot size by the number of snapshots — reflecting genuine, significant disk space savings from the hard-linking approach.

---

**Q19 🔥 Someone runs `du -sh --exclude="node_modules" /home/alice/project` expecting to exclude all node_modules directories, but the reported total still seems to include their contents. What's likely wrong with the exclude pattern?**
> A bare pattern like `"node_modules"` without wildcards may only match a directory with that EXACT path/name at a specific level, not `node_modules` directories nested more deeply within subdirectories throughout the tree. A more robust pattern uses wildcards to match it appearing at any depth: `--exclude="*/node_modules/*" --exclude="*/node_modules"`. Testing the exclude pattern explicitly (e.g., checking that `du -ah --exclude=PATTERN ... | grep node_modules` returns nothing) confirms whether it's actually working as intended.

---

**Q20. Running `du -sh` on a directory containing millions of small files (e.g., a Docker overlay filesystem or a large mail spool) takes several minutes to complete, while `df -h` on the same filesystem returns almost instantly. Why the dramatic difference in speed?**
> `du` must individually `stat()` every single file in the tree to sum up actual disk usage, and with millions of small files, that per-file overhead accumulates into genuinely significant total time — there's no shortcut around walking the entire tree for this level of per-directory detail. `df`, in contrast, reads pre-aggregated filesystem-level metadata directly (how much space is used/free on the whole filesystem), requiring no per-file traversal at all, which is why it remains near-instant regardless of how many files the filesystem contains.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
