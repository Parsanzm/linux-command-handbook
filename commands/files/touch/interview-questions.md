# touch — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Timestamps](#timestamps)
- [Internals](#internals)
- [Permissions](#permissions)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does touch do?**
> Two related things depending on the target: if the file doesn't exist, it creates a new, empty (zero-byte) file. If the file already exists, it updates its access and modification timestamps to the current time, without changing its content.

---

**Q2 🔥 Why is the command called "touch"?**
> Its core purpose was always about timestamps — "touching" a file to mark it as recently accessed or modified. File creation is essentially a side effect of applying that same timestamp-setting operation to a file that doesn't yet exist.

---

**Q3. Does touch ever modify a file's content?**
> No — touch never writes to or alters a file's actual content in any way, whether the file already existed or was just created (in which case it's simply empty).

---

## Timestamps

**Q4 🔥 What are the two timestamps touch can update, and what's the difference between them?**
> Access time (atime) — when the file's contents were last read — and modification time (mtime) — when the file's contents were last changed. `-a` updates only atime, `-m` updates only mtime, and running touch with neither flag updates both.

---

**Q5. What's ctime, and can touch set it directly?**
> Change time — when the file's metadata (permissions, ownership, or content) was last altered. touch cannot set ctime directly; it's entirely kernel-controlled and gets automatically updated to the current time as a side effect whenever touch itself runs, regardless of what mtime/atime value was explicitly requested.

---

**Q6 🔥 How would you set a file's timestamp to a specific date rather than the current time?**
> `-t` with a `[[CC]YY]MMDDhhmm[.ss]` formatted timestamp, or the more flexible `-d` with a human-readable date string (e.g., `touch -d "2026-01-15 14:30:00" file.txt`).

---

**Q7. What does `-r` do?**
> Copies the timestamp from another existing reference file, instead of using the current time — `touch -r original.txt target.txt` sets target.txt's timestamps to exactly match original.txt's.

---

## Internals

**Q8 🔥 What system call does touch use to update timestamps on an existing file?**
> `utimensat(2)` (or the older `utime(2)` on some systems) — this lets a process set a file's access and modification timestamps directly, without touching its content at all.

---

**Q9. What happens internally when touch is run against a file that doesn't exist yet?**
> It first calls `open(2)` with the `O_CREAT` flag to bring the file into existence as an empty file, then applies the same timestamp-setting logic used for existing files.

---

**Q10 🔥 What default permission mode does a newly created touch'd file typically end up with, and why is it different from a newly created directory?**
> Typically 0644 (rw-r--r--) after the default umask (0022) is applied to the requested 0666 — note the default request has no execute bit at all, unlike `mkdir`'s 0777 request, since a plain new file isn't expected to be executable by default.

---

## Permissions

**Q11. What permission is required to update the timestamp of an already-existing file?**
> Write permission on that specific file — even though touch isn't modifying content, timestamp updates on an existing file are still governed by the file's own write permission bit.

---

**Q12 🔥 What permission is required to create a brand-new file with touch, and how does that differ from the previous question?**
> Write permission on the containing DIRECTORY, not on the file itself (which doesn't exist yet, so it has no permissions of its own to check). This is a genuinely different permission requirement depending on whether touch is creating a new file or updating an existing one.

---

## Scenario-Based

**Q13 🔥 A teammate runs `touch mylink.txt` intending to update a symlink's own timestamp, but later discovers the timestamp of the file it points to changed instead. What happened, and how should the command have been run?**
> By default, touch follows symbolic links and affects the target file, not the link itself. To modify the symlink's own metadata specifically, `-h` (`--no-dereference`) is required: `touch -h mylink.txt`.

---

**Q14. Someone runs `touch -c maybe_missing.txt` on a file that turns out not to exist, and is confused that nothing happened and no error appeared. Is this a bug?**
> No — this is `-c`'s intended, documented behavior. `-c` explicitly disables file creation, so touch simply does nothing (successfully, exit code 0) when the target doesn't already exist, rather than creating it as it normally would without that flag.

---

**Q15 🔥 A build using `make` starts behaving very strangely after someone runs `touch -d "2099-01-01" somefile.c` on a source file. What's the likely cause?**
> touch has no built-in restriction against future timestamps — it happily accepted and set a date far ahead of the current time. Build tools like `make` compare source/target modification times to decide what needs rebuilding, and a far-future timestamp can cause deeply confusing, hard-to-diagnose behavior in that comparison logic until the underlying future-dated file is identified and corrected.

---

**Q16. A colleague insists `touch existing_file.txt` and `> existing_file.txt` are basically the same thing since both can result in an empty file. Why is this a dangerous misconception?**
> `touch` on an existing file only updates its timestamps, leaving content completely untouched. `>` (redirection with no preceding command) immediately TRUNCATES an existing file to zero bytes, destructively erasing any content it had — a fundamentally different, data-destroying operation. They only look similar when applied to a file that's already empty or doesn't yet exist; on a file with real content, confusing the two can cause irreversible data loss.

---

**Q17 🔥 `touch -a somefile.txt` is run, but `stat` afterward shows the access time apparently unchanged. What's a likely explanation that has nothing to do with touch itself being broken?**
> The filesystem may be mounted with `noatime` (or certain `relatime` configurations), which disables or restricts access-time tracking at the filesystem level as a performance optimization — under such a mount, even an explicit `touch -a` request to update atime can be ignored or behave differently than expected, independent of touch working correctly.

---

**Q18. Someone runs `touch *.txt` in an empty directory expecting nothing to happen, but instead finds a strangely named file called `*.txt` actually created. What's going on?**
> When a shell glob pattern matches no files, most shells (in their default configuration) leave the pattern unexpanded and pass it through literally — so touch received the literal string `*.txt` as its argument (not zero arguments) and dutifully created a file with that exact, literal name, rather than doing nothing as might be assumed.

---

**Q19 🔥 Why might `touch -d "01/02/2026" file.txt` set an unexpected date for someone used to a different regional date format?**
> `-d`'s flexible date parser generally interprets ambiguous slash-separated dates in MM/DD/YYYY order by default (US convention) rather than raising an error about the ambiguity — someone expecting DD/MM/YYYY interpretation could end up with the wrong date silently set. Using the unambiguous ISO 8601 format (`YYYY-MM-DD`) avoids this regional ambiguity entirely.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
