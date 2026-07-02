# chmod — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [Octal Mode](#octal-mode)
- [Symbolic Mode](#symbolic-mode)
- [Special Permissions](#special-permissions)
- [Directories & Traversal](#directories--traversal)
- [Security](#security)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does chmod stand for and what does it do?**
> **CHange MODe.** It changes the permission bits of a file or directory — controlling who (owner/group/other) can read, write, or execute it. It does not change ownership; that's `chown`'s job.

---

**Q2 🔥 What are the three permission classes and three permission types in Unix?**
> Classes (who): **u**ser/owner, **g**roup, **o**ther.
> Types (what): **r**ead, **w**rite, e**x**ecute.
> That's 3×3 = 9 standard bits, plus 3 extra "special" bits (setuid, setgid, sticky).

---

**Q3 🔥 In what order does the kernel check permissions, and why does it matter?**
> It checks in this order and stops at the **first match**:
> 1. Is the process the file's owner? → apply owner bits only.
> 2. Is the process in the file's group? → apply group bits only.
> 3. Otherwise → apply other bits only.
>
> It matters because the owner can end up *more* restricted than "everyone else" if permissions are set unusually — e.g., `chmod 077 file` locks the owner out while group/other have full access. The kernel does not fall through to a more permissive class.

---

**Q4. What do read, write, and execute mean on a directory (as opposed to a file)?**
> - `r` — list the directory's contents (`ls`)
> - `w` — create, delete, or rename entries inside the directory
> - `x` — "traverse" the directory (needed to `cd` into it or access anything inside, even by exact filename)
>
> A directory's own permissions govern the directory *entries*, not the permissions of the files inside it.

---

## Basic Usage

**Q5 🔥 What's the difference between `chmod 755 file` and `chmod u+x file`?**
> `chmod 755` is **absolute octal** — it sets the exact mode `rwxr-xr-x`, overwriting whatever permissions existed before.
> `chmod u+x` is **symbolic and relative** — it only adds the execute bit for the owner, leaving every other bit (group, other, existing owner read/write) untouched.

---

**Q6. How do you make a script executable?**
> ```bash
> chmod +x script.sh
> # equivalent to: chmod a+x script.sh (adds execute for everyone)
> # or, owner-only:
> chmod u+x script.sh
> ```

---

**Q7. What does `chmod -R 755 dir/` do, and why is it often a bad idea?**
> `-R` applies the mode recursively to the directory and everything inside it. The problem: it sets `rwxr-xr-x` on **every file too**, making plain data files (text, images, configs) unnecessarily executable. Better: apply `755` to directories and `644` to files separately, or use `chmod -R a+rX` which only adds execute to directories and files that already had it somewhere.

---

## Octal Mode

**Q8 🔥 Explain how the octal digit in chmod is calculated.**
> Each permission type has a value: read=4, write=2, execute=1. Sum the values you want for a class to get one digit. E.g., read+write = 4+2 = 6 (`rw-`); read+execute = 4+1 = 5 (`r-x`); all three = 7 (`rwx`). Three digits represent owner, group, other in that order — e.g., `754` = owner rwx, group r-x, other r--.

---

**Q9. What's the difference between `644` and `755`? When would you use each?**
> `644` (`rw-r--r--`) — standard for regular files/config/documents: owner can edit, everyone else can only read.
> `755` (`rwxr-xr-x`) — standard for executables and directories: owner has full control, everyone else can read/execute (or traverse, for dirs) but not modify.

---

**Q10 🔥 What does a 4-digit octal mode like `4755` or `2775` mean?**
> The leading digit sets special permission bits: `4`=setuid, `2`=setgid, `1`=sticky (these can be summed, e.g. `3775` = setgid+sticky+775). The remaining 3 digits are the normal owner/group/other permissions. `4755` = setuid + rwxr-xr-x. `2775` = setgid + rwxrwxr-x.

---

## Symbolic Mode

**Q11. What do `+`, `-`, and `=` mean in symbolic chmod syntax?**
> - `+` adds the specified permission(s), leaving others untouched
> - `-` removes the specified permission(s), leaving others untouched
> - `=` sets exactly the specified permission(s), clearing anything else in that class not listed
>
> Example: `chmod o=r file` sets others to read-only, removing any write/execute others previously had.

---

**Q12 🔥 What does the capital `X` permission do, and why is it useful?**
> `X` sets the execute bit **conditionally**: always for directories, but for regular files only if execute is already set for *some* class on that file. It's designed for recursive operations like `chmod -R a+rX dir/`, so directories become traversable while ordinary data files don't get accidentally marked executable.

---

**Q13. What does `chmod g=u file.txt` do?**
> It sets the group's permissions to exactly match the owner's current permissions. Symbolic mode allows one class to copy from another this way (`g=u`, `o=g`, etc.), which octal mode cannot express directly.

---

## Special Permissions

**Q14 🔥 What is setuid and give a real-world example.**
> Setuid, when set on an executable, makes the program run with the **file owner's** privileges rather than the invoking user's. Classic example: `/usr/bin/passwd` is owned by root and has setuid set, so any user can run it to modify `/etc/shadow` (which is root-only) — the program runs as root just long enough to do that restricted write, then exits.

---

**Q15 🔥 What is setgid on a directory used for?**
> When set on a directory, any new file or subdirectory created inside **inherits the directory's group** automatically, instead of the creating user's primary group. This is the standard way to set up a shared team directory where everyone's files end up in the same group without each person manually running `chgrp`.

---

**Q16 🔥 What is the sticky bit and where is it classically used?**
> On a directory, the sticky bit restricts file deletion/renaming so that only the file's **owner** (or the directory owner, or root) can remove or rename it — even if the directory itself is world-writable. The textbook example is `/tmp`, which is `1777` (world-writable, but sticky) so any user can create files there, but can't delete other users' files.

---

**Q17. Why is setuid ignored on shell scripts in modern Linux?**
> The Linux kernel deliberately ignores the setuid bit on files starting with a `#!` shebang. This closes a well-known class of race-condition exploits around interpreted scripts (e.g., swapping the script's contents between the kernel opening it and the interpreter reading it). Setuid still works normally on compiled binaries. To run a script with elevated privileges safely, use `sudo` with a controlled sudoers rule instead.

---

**Q18. What happens to the setuid/setgid bit when the file is edited or ownership changes?**
> Most systems automatically clear setuid/setgid when the file's contents are modified (any write) or when `chown` changes ownership. This is intentional — it prevents a scenario where altering a privileged binary or reassigning it to a new owner would silently retain elevated-privilege behavior. You must re-apply the special bit afterward if it's still needed.

---

## Directories & Traversal

**Q19 🔥 A user has read but not execute permission on a directory. What can they do?**
> They can list the directory's contents with `ls` (the read bit), but they cannot `cd` into it, and they cannot access any file inside it by path — even ones they otherwise have permission to read — because traversal requires the execute bit on every directory in the path.

---

**Q20. Can a directory have execute without read? What's the effect?**
> Yes — this is sometimes called "blind traversal" (e.g., mode `711`). Users can access a file inside the directory *if they already know its exact name*, but they cannot list the directory's contents (`ls` fails with permission denied). It's occasionally used to allow access to known files while hiding the directory listing.

---

## Security

**Q21 🔥 Why is a world-writable directory dangerous without the sticky bit?**
> Directory write permission controls who can delete or rename entries inside it — not who owns the files. Without the sticky bit, any user with write access to a `777` directory could delete or rename **any other user's files** in it, even ones they don't own. The sticky bit (as on `/tmp`) restricts deletion to the file's own owner (or root), closing that hole.

---

**Q22. Does `chmod 000 file` prevent root from reading the file?**
> No. Root bypasses standard Unix permission checks entirely, regardless of the mode bits. `chmod` (and Unix permissions generally) protect against other unprivileged users, not against root or any process with equivalent capabilities. For true confidentiality even from root, you need encryption, not permission bits.

---

**Q23 🔥 What's the security risk of `chmod -R 777 /some/dir` and how would you fix over-permissive files properly?**
> `777` grants read/write/execute to everyone, including other unprivileged users on the system — anyone can read sensitive data, overwrite files, or (for directories) delete/rename entries. Proper fix: apply the minimum permissions actually needed, typically `755`/`644` for public-but-not-writable content, or restrict to a specific group with `750`/`640` plus correct `chown`/`chgrp`, rather than opening access to "other" at all.

---

## Scenario-Based

**Q24 🔥 You run `chmod -R 755 project/` and now every file, including images and text files, shows as executable. How do you fix it properly?**
> Apply directory and file permissions separately instead of one blanket recursive mode:
> ```bash
> find project/ -type d -exec chmod 755 {} \;
> find project/ -type f -exec chmod 644 {} \;
> # Restore execute only where actually needed:
> find project/ -name "*.sh" -exec chmod +x {} \;
> ```

---

**Q25. SSH refuses to use your private key with an error about permissions being "too open." How do you fix it?**
> SSH requires strict permissions on key files:
> ```bash
> chmod 700 ~/.ssh
> chmod 600 ~/.ssh/id_rsa
> chmod 644 ~/.ssh/id_rsa.pub
> chmod 600 ~/.ssh/authorized_keys
> ```
> SSH refuses group/other-readable private keys because a readable private key defeats the purpose of key-based authentication.

---

**Q26 🔥 You want a shared directory where multiple team members can create files, and every new file should automatically belong to the "team" group. What do you set?**
> ```bash
> chgrp team /srv/shared/team_dir
> chmod 2775 /srv/shared/team_dir
> # 2 = setgid: new files/dirs inherit the "team" group
> # 775 = owner/group rwx, others r-x
> ```
> Setgid on the directory is the key piece — without it, each user's files would keep that user's own primary group instead of "team."

---

**Q27. What's wrong with `chmod -R 777 /*` as an attempted fix for permission errors, even though `chmod -R 777 /` gets blocked?**
> `chmod -R 777 /` is blocked by GNU chmod's default `--preserve-root` safeguard. But `chmod -R 777 /*` is **not** blocked, because the shell expands the glob into a list of top-level directories (`/bin /etc /home ...`) *before* chmod ever sees a literal `/` — the safeguard only checks the literal argument, not the resulting effect. This is a classic scripting trap that can still destroy an entire filesystem's permission structure despite the "protection" being enabled.

---

**Q28. A file's group permissions were tightened with `chmod g-w`, but a user who was granted access via `setfacl` says they lost write access unexpectedly. Why?**
> When ACLs are present, `chmod`'s changes to the "group" class actually modify the ACL **mask** entry, which caps the effective permissions of all named ACL users/groups on that file — not just the traditional owning group. Tightening the group bits with `chmod` can silently reduce access granted via `setfacl`. Once ACLs are in use, prefer `setfacl`/`getfacl` for permission changes and verify with `getfacl` after any `chmod`.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
