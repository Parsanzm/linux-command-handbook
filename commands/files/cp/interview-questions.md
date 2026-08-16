# cp — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Directories and Structure](#directories-and-structure)
- [Attributes and Metadata](#attributes-and-metadata)
- [Overwrite Behavior](#overwrite-behavior)
- [Internals](#internals)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does cp do?**
> Copies files (and, with `-r`, entire directories) to create an independent duplicate at the destination — changes to the copy afterward have no effect on the original, and vice versa.

---

**Q2 🔥 Does cp overwrite an existing destination file by default?**
> Yes, silently and immediately, with no confirmation prompt of any kind — this is cp's default behavior, and it's one of the most consequential things to be aware of before running it against anything that might already exist.

---

**Q3. What happens if you run plain `cp` on a directory without any additional flags?**
> It fails, refusing to copy the directory's contents at all — a message like "omitting directory" appears, and `-r` (recursive) is required to actually copy a directory and everything inside it.

---

## Directories and Structure

**Q4 🔥 What's the difference in resulting structure between `cp -r project/ backup/` when backup/ doesn't exist versus when it already exists?**
> If `backup/` doesn't already exist, it's created as a direct copy of `project/` itself — the files end up directly inside `backup/`. If `backup/` already exists as a directory, `project/` is instead copied INSIDE it, resulting in `backup/project/` containing the files — the exact same command produces two structurally different outcomes depending purely on whether the destination pre-existed.

---

**Q5. Does a trailing slash on the source path change cp's behavior, the way it does in rsync?**
> No — unlike rsync, where a trailing slash on the source meaningfully changes whether the directory itself or just its contents get copied, cp treats `project` and `project/` identically as sources. To copy only a directory's contents (not the directory itself) into an existing destination, an explicit `project/.` suffix is needed instead.

---

**Q6 🔥 What's a risk of running `cp -r ./project ./project/backup`?**
> The destination is a subdirectory of the source being copied, which can cause cp to enter the newly-created destination directory as part of the same ongoing recursive operation, attempting to copy files it just created — potentially resulting in runaway recursion or unpredictable behavior. Copying to a destination outside the source tree entirely avoids this.

---

## Attributes and Metadata

**Q7 🔥 Does a plain `cp` (without -p) preserve the original file's modification timestamp?**
> No — without `-p`, the copy gets a brand new modification timestamp reflecting when the copy operation itself happened, not the original file's actual history. `-p` is required to carry the original timestamps (along with ownership and permissions, where permitted) forward onto the copy.

---

**Q8. What does the -a (archive) flag do, and when is it typically used?**
> Shorthand for a full, faithful directory copy: recursive, with all attributes preserved (timestamps, ownership, permissions), and symlinks kept as symlinks rather than followed and flattened. It's the standard choice for genuinely cloning or backing up a directory tree.

---

**Q9 🔥 What happens if you copy a symlink with plain `cp`, without -a or -d?**
> By default, cp follows the symlink and copies the TARGET file's actual content — the resulting copy is a regular file containing a duplicate of whatever the link pointed to, not a symlink itself. The "this was a symlink" information is lost entirely unless `-a` (or `-d` specifically) is used to preserve the link as a link.

---

## Overwrite Behavior

**Q10. What's the difference between -i and -n?**
> `-i` prompts before overwriting an existing destination file, giving a chance to back out. `-n` (no-clobber) never overwrites an existing destination at all — it silently skips the copy instead, with no prompt and no error.

---

**Q11 🔥 If both -n and -i are given on the same command line, which one takes effect?**
> Whichever one appears LAST on the command line — GNU cp resolves conflicting overwrite-behavior flags by letting the later flag win, without any warning that an earlier flag is being superseded. Reordering the same two flags produces genuinely different behavior.

---

## Internals

**Q12. What's the basic mechanism cp uses to copy a file's content?**
> Conceptually, opening the source for reading and the destination for writing, then reading and writing the content across in chunks until complete — modern implementations often use more efficient mechanisms like `copy_file_range(2)` when the filesystem supports it, rather than a naive read/write loop.

---

**Q13 🔥 What does --reflink=auto do, and what filesystems support it?**
> On filesystems supporting copy-on-write (Btrfs, XFS with reflink support, APFS on macOS), it creates a near-instant copy that initially shares the same underlying physical data blocks as the original — only actually duplicating storage for blocks that are later modified in either copy, rather than physically duplicating every byte immediately.

---

**Q14. Why might individual `du` output for two reflink-copied files both show the full file size, even though they're barely using extra disk space?**
> `du` reports each file's logical size independently, but with a copy-on-write reflink copy, both files are currently sharing the same underlying physical blocks — actual distinct disk usage is much lower than the sum of the two reported sizes would suggest, until one of the copies is modified and only the changed blocks get physically duplicated.

---

## Scenario-Based

**Q15 🔥 A teammate runs `cp draft.txt final_report.txt`, not realizing final_report.txt already contained important, different content. What happened to that content, and could it have been prevented?**
> It was immediately and silently overwritten with no backup or confirmation — cp's default behavior offers no protection against this. Using `-i` (prompt before overwrite) or `-n` (never overwrite an existing file) beforehand would have prevented the accidental data loss.

---

**Q16. Someone copies a directory of config files with plain `cp -r` for a backup, intending to preserve exact original timestamps for an audit trail, but later discovers all the copied files show today's date instead. What went wrong?**
> Plain `cp -r` doesn't preserve timestamps by default — the copies get new modification times reflecting when the copy operation ran, not the originals' actual history. `-p` (or `-a` for a full directory copy) is required to carry the original timestamps forward onto the copies.

---

**Q17 🔥 A backup of a directory containing several symlinks is made with plain `cp -r`, and afterward the symlinks have turned into full independent copies of whatever they pointed to, rather than remaining symlinks. Why, and what flag avoids this?**
> Plain cp follows symlinks by default and copies the target's actual content, discarding the "this was a symlink" information entirely. `-a` (or `-d` specifically) preserves symlinks as symlinks rather than following and flattening them during the copy.

---

**Q18. A script copies a file into a directory and fails with "Permission denied," even though the user runs the script as the owner of the source file being copied. What's the actual cause?**
> Ownership of the source file is irrelevant here — what matters is whether the current user has WRITE permission on the DESTINATION directory. This is a common misdiagnosis; checking the destination directory's own permissions (not the source file's) reveals the actual cause.

---

**Q19 🔥 A directory containing thousands of small files takes far longer to copy via `cp -r` than a single large file of the same total byte size. Why, and what's a common workaround?**
> Each individual file copy carries its own per-file overhead (opening, creating, and closing file descriptors, plus metadata operations), which accumulates significantly at scale, independent of the actual total data volume. A common workaround is archiving the directory into a single file first (`tar czf`), copying that one archive, and extracting it at the destination — avoiding the overhead of thousands of small individual copy operations.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
