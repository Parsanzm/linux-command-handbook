# mkdir — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [The -p Flag](#the--p-flag)
- [Permissions](#permissions)
- [Internals](#internals)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does mkdir do?**
> Creates one or more new, empty directories at the specified path(s).

---

**Q2 🔥 What happens by default if you run mkdir on a path where a directory already exists?**
> It errors — "File exists" — plain `mkdir` treats an already-existing target as a failure by default, rather than silently succeeding or doing nothing.

---

**Q3. What does mkdir output when it succeeds?**
> Nothing at all, by default — following the standard Unix convention of "silence means success." `-v` (verbose) can be added to print a confirmation message for each directory actually created.

---

## The -p Flag

**Q4 🔥 What does the -p flag do, and why is it so commonly used in scripts?**
> It creates any missing parent directories along the given path automatically, and — critically — does NOT error if the target directory already exists. This combination (auto-creating parents, tolerating pre-existence) makes it the standard, safe choice for "ensure this directory exists" logic in scripts.

---

**Q5. What happens if you run `mkdir a/b/c` without -p, and "a" doesn't exist yet?**
> It fails with an error — without `-p`, every parent directory in the given path must already exist; mkdir only ever creates the final path component, not any missing intermediate directories.

---

**Q6 🔥 Is `mkdir -p` idempotent — safe to run multiple times with the same result?**
> Yes — running `mkdir -p` again on a path that already exists (whether created by a prior run or something else) succeeds silently rather than erroring, which is exactly what makes it safe to call repeatedly, including from concurrent or retried script executions.

---

## Permissions

**Q7 🔥 What determines the actual permissions of a newly created directory if you don't specify -m?**
> The requested mode (typically 0777, full permissions) minus whatever bits the process's current umask masks out. A common default umask of 0022 results in 0755 (rwxr-xr-x) permissions on a newly created directory, for example.

---

**Q8. What does the -m flag do, and how does it interact with umask?**
> `-m MODE` sets specific permissions directly at creation time, bypassing the umask-based calculation for the bits explicitly specified — the directory ends up with exactly the requested permissions rather than the default minus umask.

---

**Q9 🔥 When combining -m with -p to create a nested directory chain, which directories actually get the specified mode?**
> Only the final (target) directory in the chain — any intermediate parent directories created along the way as part of the same `-p` operation still use the default umask-based permission calculation, not the value passed to `-m`.

---

## Internals

**Q10. What system call does mkdir rely on?**
> `mkdir(2)` — the kernel handles the actual filesystem-level work (allocating an inode, setting up the initial `.` and `..` entries, applying permissions), and mkdir itself is a thin wrapper making this call once per directory (or once per missing directory in a chain, with `-p`).

---

**Q11 🔥 Is `mkdir -p` fully atomic across an entire nested chain, or can it partially succeed?**
> It can partially succeed — if directory creation fails partway through the chain (for example, due to a permissions issue on an already-created parent, or a disk quota being hit), earlier directories in the chain that were already successfully created remain on disk, while later ones in the chain are never created. It's not guaranteed to be all-or-nothing across the whole operation.

---

**Q12. Why can two mkdir calls for "Reports" and "reports" behave differently depending on the filesystem?**
> Case sensitivity is a filesystem-level property, not something mkdir itself controls — on a case-sensitive filesystem (typical on Linux), they're treated as genuinely separate directories, while on a case-insensitive filesystem (common on macOS by default, or certain network shares), they're treated as the same directory, and the second mkdir call would fail with "File exists."

---

## Scenario-Based

**Q13 🔥 A script uses `if [ ! -d "$DIR" ]; then mkdir "$DIR"; fi` and occasionally still fails with "File exists" even though the check just reported the directory didn't exist. What's happening, and what's the fix?**
> This is a classic time-of-check-to-time-of-use (TOCTOU) race condition — between the existence check and the actual mkdir call, another concurrent process could create the same directory first, causing this call to fail despite the earlier check. Replacing the pattern with `mkdir -p "$DIR"` directly sidesteps the race entirely, since `-p` tolerates the directory already existing regardless of exactly when it was created.

---

**Q14. A directory is created with `mkdir -m 555 somedir`, and the script later fails trying to write a file into it. Why, given that mkdir itself reported success?**
> `555` grants read and execute permissions but NOT write permission (r-xr-xr-x) — the directory itself was created successfully, but nothing can be written into it afterward. Directory creation succeeding and directory writability are separate concerns; a script should account for the actual resulting permissions, not just whether the mkdir call itself succeeded.

---

**Q15 🔥 Someone runs `mkdir config` and gets "File exists," but they're confident no directory named "config" exists in that location. What else could be going on?**
> A plain FILE (not a directory) could already exist at that exact path — mkdir's "File exists" error message is identical whether the existing item is a directory or a regular file, and doesn't disambiguate between the two cases. Running `ls -ld config` would clarify which situation is actually occurring, since the appropriate fix differs (removing a conflicting file vs. simply reusing an existing directory).

---

**Q16. A nested `mkdir -pm 700 a/b/c` call is run with the intent of making the ENTIRE new directory tree private, but a review later finds "a" and "a/b" have normal, more permissive permissions while only "a/b/c" is actually private. What went wrong with this assumption?**
> `-m` only applies its specified mode to the final target directory in the chain — any intermediate parent directories created along the way as part of the same operation still get default, umask-based permissions rather than the explicitly requested mode. To make the entire chain consistently private, a separate `chmod -R` after creation is needed, rather than relying on `-m` alone to cover intermediate directories.

---

**Q17 🔥 A build script runs `mkdir output` and later mkdir is run again in a retry of the same script after a partial failure — the second run now fails with "File exists," halting the retry. What's the simplest fix?**
> Switching to `mkdir -p output` instead of plain `mkdir output` — `-p` doesn't error when the target already exists, making the directory-creation step safely re-runnable across retries, which plain `mkdir` is not designed to tolerate by default.

---

**Q18. A relative-path `mkdir output` inside a longer script creates the directory somewhere unexpected. What's the most likely cause?**
> A relative path (no leading `/`) is always resolved against the CURRENT working directory at the moment that specific `mkdir` call executes — if an earlier part of the script changed directories (`cd`) without it being obvious at that point in the script, the relative path resolves somewhere other than what might have been assumed. Checking `pwd` immediately beforehand, or switching to an explicit absolute path, removes this ambiguity.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
