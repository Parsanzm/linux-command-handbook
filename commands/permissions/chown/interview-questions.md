# chown — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [Numeric IDs & Internals](#numeric-ids--internals)
- [Recursive Operations](#recursive-operations)
- [Symlinks](#symlinks)
- [Permissions vs Ownership](#permissions-vs-ownership)
- [Security](#security)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does chown do, and how does it differ from chmod?**
> `chown` (**CHange OWNer**) changes **which user and/or group** owns a file. `chmod` changes **what** the owner/group/others are permitted to do (read/write/execute) to a file they already own. They're independent operations at the kernel level — changing one never implicitly changes the other.

---

**Q2 🔥 What is actually stored on disk for file ownership — names or numbers?**
> Numbers. Every inode stores a numeric `st_uid` and `st_gid`. Usernames and group names shown by tools like `ls -l` are resolved at **display time** by looking up those numbers against `/etc/passwd` and `/etc/group` (or an NSS source like LDAP/sssd) — nothing about the name itself is stored in the file's metadata.

---

**Q3. What happens when you `ls -l` a file whose UID doesn't correspond to any current user account?**
> `ls -l` simply displays the raw numeric UID instead of a name, because there's nothing to resolve it against. The file itself is completely valid and unaffected — only the human-readable *display* lacks a name. This commonly happens after deleting a user account without cleaning up their files.

---

## Basic Usage

**Q4 🔥 What's the difference between `chown alice file`, `chown :staff file`, and `chown alice:staff file`?**
> - `chown alice file` — changes only the owner to alice, group untouched
> - `chown :staff file` — changes only the group to staff, owner untouched (equivalent to `chgrp staff file`)
> - `chown alice:staff file` — changes both owner (to alice) and group (to staff) in one call

---

**Q5. What does `chown alice: file.txt` do (note the trailing colon, no group name)?**
> Sets the owner to alice **and** sets the group to alice's own primary/login group. The trailing colon with nothing after it is shorthand for "use the new owner's default group."

---

**Q6. How do you find a user's numeric UID and GID before using them with chown?**
> ```bash
> id alice
> # uid=1000(alice) gid=1000(alice) groups=1000(alice),27(sudo)
> ```
> Or to reverse-lookup a name from a number: `getent passwd 1000`.

---

## Numeric IDs & Internals

**Q7 🔥 Why would you ever use `chown 1000:1000 file` instead of `chown alice:alice file`?**
> Because ownership is fundamentally numeric — using raw UID/GID works even when the corresponding username doesn't yet exist locally (e.g., restoring a disk image or backup on a fresh system before user accounts are created), and it avoids any ambiguity when working across systems whose `/etc/passwd` entries don't line up.

---

**Q8. Two different machines have UID 1000 assigned to two different people. What problem does this cause on a shared NFS mount, and how would you fix it long-term?**
> The kernel/NFS layer only tracks the numeric UID — each machine independently resolves that number to whatever name is in its own `/etc/passwd`. The same file can appear to "belong" to two completely different people depending on which machine views it. Long-term fix: centralize identity management (LDAP, NIS, or sssd) so UID-to-name mapping is consistent across every machine, or use NFSv4 with proper name-based ID mapping instead of relying on raw numeric UIDs matching by coincidence.

---

## Recursive Operations

**Q9 🔥 What's the risk of `chown -R alice:alice /some/shared/dir` on a directory with mixed ownership?**
> It reassigns **every** file and subdirectory to alice, regardless of who currently owns them — potentially taking ownership away from other users who were relying on it, with no automatic way to know which files "belonged" to whom afterward unless you had a prior record (backup, snapshot, or log).

---

**Q10. How would you migrate only ONE user's files out of a shared, mixed-ownership directory without affecting anyone else's files?**
> Use `--from` to scope the change to files currently owned by that specific user:
> ```bash
> sudo chown -R --from=olduser:olduser newuser:newuser /shared/project
> ```
> Anything not already owned by `olduser:olduser` is left completely untouched.

---

**Q11 🔥 By default, does `chown -R` follow symlinks it encounters while walking a directory tree?**
> No. The default behavior (`-P`) does **not** follow symlinks during recursive traversal — this is a deliberate safety measure so a symlink pointing outside the intended tree (e.g., to `/etc` or `/`) can't cause a recursive chown to reach and modify ownership of files far outside where you intended. Using `-L` explicitly overrides this and follows all symlinks encountered, which is rarely what you actually want.

---

## Symlinks

**Q12 🔥 If you run `chown alice symlink.txt` where symlink.txt points to realfile.txt, what actually changes ownership — the link or the target?**
> By default, chown **dereferences** the symlink and changes the ownership of the **target** file (`realfile.txt`), not the symlink itself. To change the symlink's own ownership metadata without touching the target, use `-h` (`chown -h alice symlink.txt`).

---

**Q13. What happens if you try to `chown` a broken symlink (pointing to a nonexistent target) without `-h`?**
> It fails with an error like "cannot dereference: No such file or directory," because the default behavior needs to resolve the symlink to a real target to operate on. Using `-h` succeeds instead, since it changes the symlink's own metadata directly without needing to resolve anything.

---

## Permissions vs Ownership

**Q14 🔥 After running `chown alice:staff file.txt`, does alice automatically gain read/write access to the file?**
> Not necessarily. `chown` only changes *who* the owner and group are — it does not touch the existing permission bits at all. If the file's mode is, say, `000` (no access to anyone) or `600` with a different owner previously, alice's actual access depends entirely on those pre-existing bits now being interpreted against her as the new owner. A separate `chmod` may still be required for her to actually use the file.

---

**Q15. Does chown affect a file's setuid or setgid bit?**
> Changing ownership of an executable with the setuid or setgid bit set typically **clears that bit automatically** on most systems — a deliberate security measure to prevent a change of ownership from silently preserving "run as previous owner" behavior in a way that could be exploited. The special bit must be re-applied with `chmod` after any such chown if it's still needed.

---

## Security

**Q16 🔥 Why can't an ordinary (non-root) user give one of their own files to another user with chown?**
> This is governed by the POSIX `_POSIX_CHOWN_RESTRICTED` behavior, enabled by default on essentially all modern Unix/Linux systems. Without this restriction, a user could "give away" ownership of large files to evade their own disk quota accounting, or launder file ownership to obscure who actually created or controls certain data. Only root can change a file's owner to a different user; an ordinary user can only change the *group* of a file they own, and only to a group they themselves already belong to.

---

**Q17. Can a regular user change a file's group to any group they want?**
> No — only to a group they are already a member of. Attempting `chown :somegroup file` fails with "Operation not permitted" if the invoking user isn't in `somegroup`, even though they fully own the file otherwise.

---

**Q18 🔥 A file was deleted user's UID (1005) is later reused by a brand-new account. What security problem does this create?**
> Any files still on disk that were owned by the old UID 1005 now appear to belong to the **new** account sharing that same number — even though it's a completely different person. The new user may unknowingly gain access to data that was never meant for them (if permission bits allow group/owner access), simply because the kernel only tracks the numeric ID, not any notion of "identity" beyond that number. Best practice: audit and reassign or clean up a departing user's files (`find / -uid <old_uid>`) before their UID can be reused, rather than relying on `userdel` alone.

---

## Scenario-Based

**Q19 🔥 You're restoring a home directory from an old backup onto a fresh system, and the files show up owned by raw numbers instead of usernames in `ls -l`. Why, and how do you fix it?**
> The backup preserved the original numeric UID/GID, but the new system's `/etc/passwd` doesn't yet have an account with a matching UID — so there's no name to resolve to, and `ls -l` falls back to displaying the raw number. Fix: either create the user account with the matching UID (`useradd -u <original_uid> username`) before restoring, or chown the restored files to the correct (possibly differently-numbered) account that now exists on this system: `sudo chown -R newuser:newuser /home/newuser`.

---

**Q20. A container running as UID 1000 writes files into a bind-mounted host directory. On the host, `ls -l` shows those files owned by a completely unrelated (or nonexistent) user. Explain what happened and two ways to fix it.**
> Bind mounts share the same underlying filesystem between host and container, and ownership is purely numeric — the container's internal UID 1000 is written directly as UID 1000 in the host's view too, regardless of who (if anyone) that UID actually belongs to on the host. Fixes: (1) reclaim ownership on the host after the fact with `sudo chown $(id -u):$(id -g) file`, or (2) run the container with `--user $(id -u):$(id -g)` from the start so the UID written matches your actual host user, avoiding the mismatch entirely.

---

**Q21 🔥 You need to recursively change ownership of `/srv/shared`, but the directory contains a symlink pointing to `/etc/shadow`. What's the safe way to do this, and what would make it unsafe?**
> The safe (and default) way is a plain recursive chown without `-L`:
> ```bash
> sudo chown -R alice:alice /srv/shared
> ```
> By default, chown does not follow symlinks during recursive traversal, so `/etc/shadow` is left untouched. It becomes unsafe the moment `-L` is added (`chown -R -L alice:alice /srv/shared`), which explicitly tells chown to follow every symlink it encounters — in this scenario, that would actually reassign ownership of `/etc/shadow` itself, a serious and easily overlooked mistake.

---

**Q22. Why might `sudo chown -R alice:alice /*` succeed even though `sudo chown -R alice:alice /` is blocked by a "dangerous to operate recursively on /" error?**
> GNU chown's `--preserve-root` safeguard (on by default) only checks whether the **literal argument** passed to chown is `/`. When you use `/*`, the **shell** expands the glob into a list of top-level directory names (`/bin /etc /home /var ...`) before chown ever runs — so chown never actually sees a literal `/` as an argument, and the safeguard never triggers, even though the practical effect on the filesystem is nearly identical to operating on `/` directly.

---

**Q23. A disk quota alert fires for a user immediately after an administrator ran a bulk `chown` reassigning several large files to them. Why did this happen, and is it expected behavior?**
> Yes, this is expected: disk usage accounting for quota purposes is tied to the file's **current owner**, not who originally created it. The moment ownership is reassigned via `chown`, the file's size is counted against the *new* owner's quota immediately — potentially pushing them over their limit even though they had no part in creating or requesting those files. Administrators should check `quota -u <user>` before and after any bulk chown that moves large files between owners.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
