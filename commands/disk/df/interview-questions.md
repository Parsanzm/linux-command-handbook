# df — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [Inodes](#inodes)
- [Reserved Blocks & Root Margin](#reserved-blocks--root-margin)
- [df vs du](#df-vs-du)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does df do, and how does it get its information?**
> `df` (**Disk Free**) reports used/available space for mounted filesystems. It retrieves this via the `statfs()`/`statvfs()` system call, which asks the kernel directly for a filesystem's pre-aggregated total/free/used space — information the filesystem maintains continuously, rather than something df has to compute by examining individual files.

---

**Q2 🔥 Why is df almost always faster than du, even on an enormous filesystem?**
> df reads a **single** pre-aggregated statistic per filesystem directly from kernel/filesystem metadata, requiring no traversal of individual files. `du`, by contrast, must recursively walk the entire directory tree and sum up every file's actual disk usage — its speed scales with the number of files, while df's speed does not.

---

## Basic Usage

**Q3. What's the difference between `df` and `df -h`?**
> Plain `df` reports sizes in raw 1K-blocks by default, which is hard to read at a glance for large filesystems. `df -h` (`--human-readable`) automatically converts sizes into readable K/M/G/T units.

---

**Q4 🔥 How would you check disk usage for the filesystem containing a specific file or directory, rather than every mounted filesystem?**
> ```bash
> df -h /path/to/file_or_directory
> ```
> df resolves the given path to whichever filesystem it actually resides on and reports only that one filesystem's usage, rather than listing every mounted filesystem on the system.

---

**Q5. What's the difference between `-h` and `--si` in df?**
> Both produce human-readable output, but `-h` uses powers of **1024** (the traditional binary-prefix convention: 1K = 1024 bytes), while `--si` uses powers of **1000** (matching how storage manufacturers typically advertise capacity) — the same underlying byte count can display as slightly different numbers depending on which convention is used.

---

## Inodes

**Q6 🔥 What is an inode, and why can a filesystem run out of them even with plenty of free byte-space remaining?**
> An inode is a metadata structure required for every file, directory, or symlink on a filesystem. Most traditional filesystems allocate a **fixed number** of inodes at creation time, independent of how large individual files are — so a filesystem containing an enormous number of very small files can exhaust its inode supply entirely while still having abundant raw disk bytes free, since each file (regardless of size) consumes exactly one inode.

---

**Q7. How do you check inode usage specifically, and what would indicate inode exhaustion is the actual problem behind a "No space left on device" error?**
> ```bash
> df -i
> ```
> If `df -h` shows plenty of free byte-space (a low "Use%") but `df -i` shows the "IUse%" column near 100%, inode exhaustion — not actual disk space exhaustion — is the real cause of any "No space left on device" errors, even though that identical error message appears for both scenarios.

---

**Q8 🔥 What kind of workload commonly leads to inode exhaustion specifically?**
> Directories containing enormous numbers of very small files — classic examples include mail spool directories storing one file per message, aggressive filesystem-based caching systems, or session storage writing many tiny files — since inode consumption depends purely on file **count**, not total data size.

---

## Reserved Blocks & Root Margin

**Q9 🔥 Why might `df -h` show "Avail" space that's less than `Size` minus `Used`?**
> Many filesystems (notably ext4) reserve a percentage of total space (historically around 5%) exclusively for the **root** user, ensuring root can still log in, write logs, and take corrective action even when a filesystem appears "full" from a regular user's perspective. This reserved margin is subtracted from what `Avail` reports to ordinary users' calculations, even though it technically still exists on the underlying filesystem.

---

**Q10. How would you check or adjust a filesystem's reserved-block percentage?**
> ```bash
> tune2fs -l /dev/sda1 | grep -i reserved    # check current reserved block count
> sudo tune2fs -m 1 /dev/sda1                # adjust reserved percentage to 1%
> ```
> This should be done cautiously, since the reserved margin exists specifically to prevent scenarios where a completely full disk locks out even root from performing necessary cleanup or repair operations.

---

## df vs du

**Q11 🔥 What's the fundamental difference in what df and du each measure?**
> `df` reports usage for an entire **mounted filesystem**, based on kernel/filesystem-level metadata. `du` reports usage for **specific files or directories**, computed by recursively walking and summing actual file sizes. They answer complementary but distinct questions: "how full is this whole disk?" (df) versus "which directory is responsible for that usage?" (du).

---

**Q12. Describe the classic scenario where df reports a filesystem as significantly more full than what du's totals across all visible files would suggest.**
> A large file was deleted while still being held **open** by a running process. The filesystem correctly still counts that file's disk blocks as used (df accurately reflects this), but since there's no remaining directory entry for it, `du` — which only walks the visible directory tree — has no way to see or count that still-allocated space, leading to a mismatch between df's total and du's sum.

---

**Q13 🔥 What's the standard troubleshooting workflow combining df and du when disk space runs low?**
> Run `df -h` first to quickly identify **which** filesystem is nearly full (fast, since it's just metadata). Then run `du -h --max-depth=N` (often starting broad and progressively narrowing) on directories within that specific filesystem to identify **which** directory or files are actually responsible for the bulk of the usage.

---

## Scenario-Based

**Q14 🔥 A server shows `df -h /` at 95% used, but `du -sh /` only accounts for about 60% of the total. What are the two most likely explanations, and how would you distinguish between them?**
> The two most common explanations are: (1) a large file was deleted while still held open by a running process (space allocated but invisible to du), diagnosable via `lsof +L1` or searching for "(deleted)" entries in lsof's output; or (2) `du` was run without sufficient permissions and silently undercounted directories it couldn't read into (check whether stderr was suppressed, and consider re-running with `sudo du -sh /`). Distinguishing between them: the lsof check specifically reveals open-but-deleted files, while re-running du with elevated privileges and without suppressing errors reveals whether permission issues were the cause instead.

---

**Q15. An application fails with "No space left on device" while writing a small configuration file, but `df -h` on the relevant filesystem shows only 40% used. What should you check next, and why?**
> Check inode usage with `df -i` on that same filesystem — a filesystem can be nearly full on inodes (metadata structures needed for every file) while still having abundant raw byte-space free, and the identical "No space left on device" error appears for both byte-space exhaustion and inode exhaustion. If `df -i` shows the inode usage percentage near 100%, the fix requires reducing the **number** of files (not necessarily their total size) — often by cleaning up a directory containing an enormous number of small files.

---

**Q16 🔥 After running `lvextend` to add 50GB to a logical volume, `df -h` still reports the old, smaller size for that filesystem. What step was missed?**
> Extending the underlying block device (via `lvextend`) only grows the **container** — the filesystem sitting on top of it must be separately and explicitly told to grow into the newly available space, using a filesystem-specific tool like `resize2fs` (for ext4) or `xfs_growfs` (for XFS). Until that step runs, df continues reporting the filesystem's previous, smaller size, since the filesystem itself hasn't yet been informed that more underlying space exists.

---

**Q17. Why might `df -h` seemingly "hang" and produce no output at all on a server with several mounted filesystems?**
> If one of the mounted filesystems is a **network mount** (NFS, CIFS) whose remote server has become unreachable, `df` can block indefinitely waiting on that single problematic mount, even though every other local filesystem would report instantly on its own — df processes filesystems sequentially, and one stuck entry can block the entire command's output. Using `df -l` (local filesystems only) proactively avoids this, or wrapping the call in `timeout` prevents an indefinite hang.

---

**Q18 🔥 A system with many installed snap packages shows dozens of df entries reporting "100% used" for tiny filesystems. Is this a sign of a problem?**
> No — each snap package creates its own fixed-size, read-only squashfs-backed mount, which is always exactly as full as its own designed capacity (100% used by design, since it's a static, read-only image, not something being actively filled up over time). This is completely normal and unrelated to actual disk space concerns; filtering these out with `df -h -x squashfs` gives a cleaner view of the genuinely relevant filesystem usage.

---

**Q19. Why might df report the SAME filesystem multiple times with identical usage numbers, once for `/` and again for some other seemingly unrelated mountpoint?**
> This typically indicates a **bind mount** — a directory mounted to appear at an additional location while still being fundamentally part of the same underlying filesystem, not an independent one with separate capacity. Docker's overlay2 storage driver produces a similar effect, showing one entry per running container, all sharing and reporting against the same underlying host filesystem's actual capacity rather than representing dozens of independent storage pools.

---

**Q20 🔥 Someone checks `df -h /home/alice/some_symlink` expecting to see usage information about the filesystem where the symlink itself lives, but instead sees a completely different filesystem's stats. Why?**
> df follows symlinks by default and reports the filesystem of the symlink's **target**, not the filesystem containing the symlink file itself — if the symlink points to a path on a different mounted filesystem (e.g., an external drive), df reports that target filesystem's usage, which is usually the practically useful behavior but can surprise someone specifically wanting information about the symlink's own containing filesystem instead.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
