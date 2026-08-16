# mv — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Internals and Atomicity](#internals-and-atomicity)
- [Overwrite Behavior](#overwrite-behavior)
- [Cross-Filesystem Behavior](#cross-filesystem-behavior)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does mv do, and why is there no separate "rename" command in standard Unix?**
> mv relocates a file or directory to a new path — which can look like renaming (same directory, new name) or moving (new directory, same or different name). There's no separate rename command because, at the syscall level, renaming and moving within the same filesystem are literally the same operation: updating a directory entry's path.

---

**Q2 🔥 Does mv overwrite an existing destination file by default?**
> Yes, silently and immediately, with no confirmation prompt — this is mv's default behavior, and it's one of the most important things to be aware of before running it against a destination that might already contain something worth keeping.

---

**Q3. What does mv output when it succeeds?**
> Nothing at all, by default — following the standard Unix "silence means success" convention. `-v` (verbose) prints a confirmation line for each file actually moved.

---

## Internals and Atomicity

**Q4 🔥 What system call does mv use for a same-filesystem move, and why does that make it so fast regardless of file size?**
> `rename(2)` — it simply updates the directory entry (the filename-to-inode mapping) without touching the actual file data on disk at all. A 50GB file "moved" within the same filesystem completes in milliseconds, since nothing is actually being copied.

---

**Q5. What happens when the source and destination are on different filesystems?**
> `rename(2)` isn't possible across filesystem boundaries, so mv transparently falls back to copying the data to the new location and then deleting the original — taking time proportional to the file's actual size, just like running `cp` followed by `rm`.

---

**Q6 🔥 Is a same-filesystem mv operation atomic? What does that guarantee in practice?**
> Yes — on the same filesystem, the underlying `rename(2)` call is a single atomic operation, meaning any other process reading the destination path at any point sees either the complete previous file or the complete new file, never a partially-written intermediate state. This makes "write to a temp file, then mv into place" a reliable pattern for atomically publishing generated content.

---

**Q7. Does that same atomicity guarantee hold for a cross-filesystem move?**
> No — a cross-filesystem move falls back to copy-then-delete, which is not wrapped in any transactional guarantee. If interrupted partway through, this can leave a truncated destination file, or in some failure windows, both the original and a (possibly incomplete) copy existing simultaneously.

---

## Overwrite Behavior

**Q8 🔥 What's the difference between -i and -n for mv?**
> `-i` prompts before overwriting an existing destination file. `-n` (no-clobber) never overwrites an existing destination at all — it silently skips the operation instead, with no prompt and no error.

---

**Q9. What does the -u flag do, and what's a risk associated with relying on it?**
> `-u` only performs the move if the source file is newer than an existing destination (or the destination doesn't exist), based on comparing modification timestamps. The risk: if the clocks or timezone handling between the source and destination locations (e.g., across a network mount) are out of sync, `-u` can make an incorrect decision based on inaccurate timestamp comparisons that have nothing to do with the files' actual content.

---

## Cross-Filesystem Behavior

**Q10 🔥 How would you check whether a move between two paths will be an instant rename or a slower copy+delete?**
> Check whether the source and destination paths are on the same filesystem/mount point — `df` on both paths reveals the underlying device; if they match, `rename(2)` applies and the move is near-instant regardless of size; if they differ, mv falls back to copy+delete instead.

---

**Q11. Why might moving a directory containing thousands of small files across filesystems take dramatically longer than moving a single large file of the same total size?**
> Once source and destination are on different filesystems, `rename(2)` can't be used at all, so every individual file requires its own full copy-then-delete cycle — the per-file overhead compounds significantly at scale, unlike a same-filesystem move where thousands of files would move just as near-instantly as one.

---

## Scenario-Based

**Q12 🔥 A teammate runs `mv new_draft.txt final_report.txt`, not realizing final_report.txt already contained important, different content. What happened to that content?**
> It was immediately and silently overwritten — mv's default behavior offers no protection against this, with no backup or confirmation. Using `-i` (prompt before overwrite) or `-n` (never overwrite) beforehand would have prevented the accidental loss.

---

**Q13. A script runs `mv myfile.txt sometarget` and behaves differently across two different environments — in one, myfile.txt gets renamed to "sometarget"; in the other, it ends up at "sometarget/myfile.txt" instead. Why?**
> The outcome depends entirely on whether "sometarget" already exists as a directory at the time the command runs — if it does, mv moves the source INSIDE it; if it doesn't (or it's an existing file), mv treats it as a plain rename target. The two environments likely differed in whether that destination directory pre-existed. Using `-T` (`--no-target-directory`) removes this ambiguity by always treating the destination as a literal rename target.

---

**Q14 🔥 A process has a log file open via `tail -f`, and someone runs `mv` to rename that same file elsewhere. Does this break the running tail process?**
> No — an open file descriptor references the underlying inode, not the path string used to open it, so the already-running process continues following the same file seamlessly even after its filename changes. A NEW process trying to find the file by its old name afterward, however, would fail, since that name no longer points to it.

---

**Q15. During a large cross-filesystem move of a big dataset, the process is killed unexpectedly. What's a realistic resulting state, and why can't this happen with a same-filesystem move?**
> Since the move isn't atomic across filesystems (it's a copy followed by a delete, not a single `rename()`), the process could be killed after the copy completes but before the original is deleted, leaving BOTH the original and the new copy existing simultaneously — or leaving a truncated destination file if killed mid-copy. A same-filesystem move can't produce this outcome, since `rename()` is a single atomic operation with no intermediate window where both states could coexist.

---

**Q16 🔥 Someone attempts `mv project/ project/backup/` and gets an error refusing the operation. What's mv detecting, and why can't this be done?**
> mv is detecting that the destination would need to exist as a subdirectory of the very directory being moved — a structural impossibility, since you can't move something into a location that only exists as part of itself. mv recognizes this specific case and refuses rather than attempting an operation that can't be coherently completed.

---

**Q17. A generated report is published using the pattern `generate_report > report.html.tmp && mv report.html.tmp report.html`. Why is this considered a safer publishing pattern than writing directly to report.html?**
> Because the same-filesystem `mv` is atomic — any process reading `report.html` at any moment sees either the complete previous version or the complete new version, never a partially-written intermediate state that direct writing could expose a concurrent reader to. This pattern relies specifically on mv's atomicity guarantee, which only holds when source and destination share the same filesystem.

---

**Q18 🔥 A symlink is moved with plain `mv`, and afterward it's still a symlink pointing to the same original target — but a colleague recalls that plain `cp` on a symlink instead copies the target's actual content. Why do the two commands behave differently here?**
> mv relocates the symlink itself by default (updating where the link's directory entry points), while plain `cp` (without `-a`) follows the symlink and copies the target file's content instead, discarding the symlink structure entirely. This is a genuine, worth-knowing asymmetry between the two commands' default behaviors regarding symlinks.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
