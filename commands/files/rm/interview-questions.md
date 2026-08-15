# rm — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Directories and Flags](#directories-and-flags)
- [Internals](#internals)
- [Data Recovery and Security](#data-recovery-and-security)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does rm do, and how does it differ from deleting a file through a desktop file manager?**
> Removes files (and, with `-r`, directories) permanently — there's no recycle bin, no confirmation by default, and no built-in undo. A desktop file manager typically moves deleted items to a recoverable trash location first; rm does not.

---

**Q2 🔥 Does rm print anything when it successfully removes a file?**
> No — following the standard Unix convention, silence means success. Only errors (a missing file, a permissions problem) produce visible output, along with a non-zero exit status.

---

**Q3. What happens if you run plain `rm` on a directory, with no additional flags?**
> It fails with "Is a directory" — plain rm refuses to remove directories at all without `-r` (recursive) or `-d` (for an empty directory specifically).

---

## Directories and Flags

**Q4 🔥 What's the difference between rmdir and rm -r?**
> `rmdir` only removes a directory if it's completely empty, failing otherwise. `rm -r` recursively removes a directory and everything inside it, regardless of whether it's empty.

---

**Q5. What does -f do, beyond just suppressing confirmation prompts?**
> It also suppresses errors for files/paths that don't exist — a `rm -f` on a nonexistent file returns success (exit code 0) rather than an error, which is useful for "remove if present" scripting but can also mask other, unrelated failures that would normally have been reported.

---

**Q6 🔥 What's the difference between -i and -I?**
> `-i` prompts individually before removing every single file. `-I` prompts only once for a recursive removal involving three or more files, specifically calling out the scale involved — a middle ground that's less tedious than `-i` while still surfacing a meaningful warning for large deletions.

---

## Internals

**Q7. What system call does rm use to remove a regular file, and what does that call actually do?**
> `unlink(2)` — it removes the directory entry (the filename) pointing to the underlying data. It does not necessarily erase the actual data on disk immediately.

---

**Q8 🔥 If a file has multiple hard links, does removing one of them delete the underlying data?**
> No — the underlying data is only freed once every hard link pointing to it has been removed (and no process still has it open). Removing one hard link just removes that one directory entry; the data remains fully accessible through any remaining links.

---

**Q9. Why might disk usage reported by df not immediately decrease after successfully removing a large file?**
> If a running process still has the file open (a file descriptor referencing it), the underlying data blocks remain allocated until that process closes the file or exits — `unlink` only removes the directory entry, not necessarily the data itself while it's still actively referenced.

---

## Data Recovery and Security

**Q10 🔥 Can a file removed with rm typically be recovered?**
> Often, yes, at least until the underlying disk blocks are overwritten by some future write — rm removes the directory entry, not necessarily the data itself, so forensic recovery tools can sometimes reconstruct deleted content, especially on traditional filesystems. rm provides no cryptographic or forensic guarantee of unrecoverability.

---

**Q11. What does shred do differently from rm, and what's a major limitation of it on modern hardware?**
> `shred` overwrites a file's content (multiple times, by default) before removing it, aiming to make recovery much harder. Its major limitation: on SSDs with wear-leveling, or copy-on-write/journaling filesystems, the original data may end up preserved at a different physical location than the one shred actually overwrote, meaning shred's approach doesn't reliably guarantee unrecoverability the way it does on traditional magnetic hard drives.

---

**Q12 🔥 What does --preserve-root do, and why is it the default on modern coreutils?**
> It refuses to allow a recursive `rm` operation directly on `/` (the root filesystem), specifically to prevent the single most catastrophic possible rm mistake — accidentally wiping the entire filesystem. It's the default behavior precisely because the consequences of accidentally disabling it are so severe.

---

## Scenario-Based

**Q13 🔥 A script runs `rm -rf "$BASE_DIR"/temp` and, due to a bug elsewhere, $BASE_DIR ends up unset/empty. What actually gets removed, and why is this a well-known category of disaster?**
> With `$BASE_DIR` empty, the command effectively becomes `rm -rf /temp` — an entirely different, much broader path than intended. This is a well-known disaster pattern precisely because an unset shell variable silently collapses to nothing rather than raising an error, turning a narrowly-scoped intended deletion into an unintentionally broad one. Quoting alone doesn't prevent this; validating that the variable is actually set and non-empty before use does.

---

**Q14. A colleague relies on `alias rm='rm -i'` as their main safety net against accidental deletions, but they still managed to wipe an important directory without any prompt appearing. How is this possible?**
> Aliases only apply in interactive shell sessions where that specific alias is loaded — they don't apply inside scripts (which typically don't source interactive alias definitions), and they're bypassed entirely if rm is invoked via its full path (`/usr/bin/rm`), which ignores shell aliases altogether. The alias is a reasonable everyday habit but was never a guaranteed safety net in every context rm might be invoked from.

---

**Q15 🔥 A cleanup script does `cd "$TARGET_DIR"` followed by `rm -rf ./*`, and one day it wipes out an unrelated directory entirely. What's the most likely root cause?**
> The `cd` command likely failed silently (a typo in the path, a permissions issue, the directory no longer existing) — without an explicit check (`cd "$TARGET_DIR" || exit 1`), the script simply continues running from whatever directory the shell was already in, and the subsequent `rm -rf ./*` then destroys the contents of that unintended, unrelated location instead.

---

**Q16. Someone removes a large log file with `rm`, but `df` shows disk usage barely changed afterward. A teammate suggests the file might not have really been deleted — is that accurate?**
> Not quite — the file genuinely was unlinked (its directory entry removed, meaning `ls` no longer shows it), but if a running process still has that file open, the underlying disk space isn't actually freed until that process closes the file descriptor or exits. Checking `lsof +L1` on the relevant directory would reveal a "deleted but still open" file consuming space, explaining the discrepancy.

---

**Q17 🔥 Someone is troubleshooting why `rm -rf sensitive_data/` didn't securely erase the data forensically, even though the files no longer appear when listing the directory. What's the misconception here?**
> `rm` (with or without `-rf`) only removes directory entries — it makes no attempt whatsoever to overwrite the actual underlying data blocks on disk, which can often remain recoverable via forensic tools until overwritten by unrelated future writes. `rm` was never designed to provide secure deletion; a tool like `shred` (with its own caveats on SSDs) or full-disk encryption with key destruction is the appropriate approach for genuinely sensitive data.

---

**Q18. During a code review, a deployment script contains the line `rm -rf --no-preserve-root /old_release`. What should immediately stand out as concerning, regardless of what the rest of the script does?**
> The presence of `--no-preserve-root` at all is a serious red flag — this flag exists specifically to disable coreutils' built-in protection against catastrophically destructive recursive operations on `/`, and there is essentially never a legitimate reason to need it for a normal deletion task like removing an old release directory. Its presence should prompt immediate scrutiny of the full command and the surrounding script logic before running it anywhere.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
