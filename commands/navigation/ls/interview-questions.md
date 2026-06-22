# ls — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Flags & Output](#flags--output)
- [Permissions & Ownership](#permissions--ownership)
- [Sorting & Filtering](#sorting--filtering)
- [Symlinks & Special Files](#symlinks--special-files)
- [Scripting & Gotchas](#scripting--gotchas)
- [Scenario-Based](#scenario-based)
- [Advanced & Internals](#advanced--internals)

---

## Conceptual

**Q1. What does `ls` stand for and what does it do?**
> `ls` means **list**. It lists the contents of a directory and displays metadata about files. With no arguments it shows the current directory, non-hidden files, sorted alphabetically.

---

**Q2. Where is `ls` located and what package provides it?**
> On Linux: `/bin/ls`, provided by **GNU coreutils**.
> On macOS: `/bin/ls`, BSD version.
> Check with `which ls` or `type -a ls`.

---

**Q3 🔥 Why does `ls` sometimes behave differently than expected in scripts?**
> Because in interactive shells, `ls` is usually **aliased** to `ls --color=auto` or similar. In scripts the alias doesn't apply, so you get raw `ls` behavior. Use `type -a ls` to see the alias. Use `\ls` or `command ls` to bypass it.

---

**Q4. What are the three timestamps a file has on Linux?**
> - **mtime** (modification time) — last time file content changed
> - **atime** (access time) — last time file was read
> - **ctime** (change time) — last time file metadata changed (permissions, ownership)
>
> `ls -l` shows mtime by default. Use `-u` for atime, `-c` for ctime.

---

**Q5. What syscalls does `ls` use internally?**
> `opendir()` → `readdir()` (loop) → `stat()` or `lstat()` per entry → sort → write to stdout. On systems with large directories, the `stat()` calls per entry are the main bottleneck.

---

## Flags & Output

**Q6 🔥 What is the difference between `ls -l` and `ls -la`?**
> `-l` shows long format for non-hidden files. `-la` (or `-lA`) also includes hidden files (dotfiles). The difference between `-a` and `-A`: `-a` includes `.` and `..`; `-A` does not.

---

**Q7 🔥 Explain the output of `ls -l`. What does each column mean?**
> ```
> -rw-r--r--  1  alice  staff  4096  Jun 15 10:23  file.txt
> ```
> 1. File type + permissions (10 chars)
> 2. Hard link count
> 3. Owner
> 4. Group
> 5. Size in bytes
> 6. Last modification timestamp
> 7. Filename

---

**Q8. What does the first character in `ls -l` output represent?**
> The file type:
> - `-` regular file
> - `d` directory
> - `l` symbolic link
> - `b` block device
> - `c` character device
> - `p` named pipe
> - `s` socket

---

**Q9. What is the difference between `ls -l` and `ls -ln`?**
> `-l` shows owner and group **names** (requires `/etc/passwd` and `/etc/group` lookup). `-ln` shows **numeric** UID and GID — faster on systems with large user databases or when names can't be resolved (e.g., NFS with different user mappings).

---

**Q10. What does `ls -lh` show differently from `ls -l`?**
> `-h` makes sizes human-readable: instead of `1073741824` bytes it shows `1.0G`. Uses powers of 1024 (K=1024, M=1048576...). Use `-H` for powers of 1000.

---

**Q11. What is `ls -1` (digit one)?**
> Forces one entry per line, regardless of terminal width. Useful for piping into scripts. Not the same as `-l` (letter l = long format).

---

## Permissions & Ownership

**Q12 🔥 What do the permission bits `rwxr-xr--` mean?**
> Three groups of three: owner / group / others.
> - `rwx` → owner can read, write, execute
> - `r-x` → group can read and execute, not write
> - `r--` → others can only read
>
> In octal: `754`.

---

**Q13. What does the `+` at the end of permissions mean in `ls -l`?**
> The file has an **ACL** (Access Control List) beyond the standard Unix permissions. Use `getfacl filename` to see the full ACL.

---

**Q14. What does the `s` in `-rwsr-xr-x` mean?**
> The **SUID bit** (Set User ID). When this file is executed, it runs with the permissions of the **file owner**, not the caller. Common example: `/usr/bin/passwd` runs as root regardless of who invokes it.

---

**Q15. What does the `t` in `drwxrwxrwt` mean?**
> The **sticky bit** on a directory. Users can only delete files they own, even if they have write permission on the directory. Classic example: `/tmp`.

---

**Q16. Why does a directory's size show as `4096` even if it's empty?**
> That's the size of the **directory entry** (inode data block) on the filesystem, not the total size of its contents. A directory always uses at least one 4KB block. Use `du -sh dirname/` for actual disk usage including contents.

---

## Sorting & Filtering

**Q17 🔥 How do you list files sorted by modification time, newest first?**
> ```bash
> ls -lt
> ls -lth    # with human-readable sizes
> ```

---

**Q18. How do you list the single most recently modified file?**
> ```bash
> ls -t | head -1
> ls -lt | awk 'NR==2 {print $NF}'   # skip the "total" line
> ```

---

**Q19. What is the difference between `ls -lt` and `ls -lct`?**
> `-lt` sorts by **mtime** (file content changed). `-lct` sorts by **ctime** (metadata changed — includes chmod, chown, rename, etc.). A `chmod` on a file updates ctime but not mtime.

---

**Q20. How do you list files with natural version sorting?**
> ```bash
> ls -v
> # file1.txt file2.txt file10.txt  ← correct
> # (without -v: file1.txt file10.txt file2.txt)
> ```

---

**Q21. How do you list only directories in the current path?**
> ```bash
> ls -d */             # glob approach — only works in current dir
> ls -l | grep '^d'   # filter long format
> find . -maxdepth 1 -type d   # most reliable
> ```

---

## Symlinks & Special Files

**Q22 🔥 What is the difference between `ls -l` and `ls -lL` on a symlink?**
> `-l` shows the symlink itself: `lrwxrwxrwx ... link -> target`. The size is the length of the target path string.
> `-lL` **follows** the symlink and shows the **target file's** metadata (real permissions, real size, real timestamps).

---

**Q23. How does `ls` handle a broken symlink?**
> `ls broken_link` gives "No such file or directory".
> `ls -l broken_link` shows the symlink entry (usually highlighted in red with `--color`), even though the target doesn't exist.

---

**Q24. Why does `ls -l symlink_to_dir` list the directory's contents instead of showing the symlink?**
> When a symlink points to a directory and you pass it as an argument (not from a listing), `ls` follows it and lists the directory. Use `ls -ld symlink_to_dir` to see the symlink itself.

---

## Scripting & Gotchas

**Q25 🔥 Why should you never parse `ls` output in scripts?**
> Filenames can contain any character except `/` and null — including spaces, tabs, and newlines — which breaks word splitting. `ls` also applies formatting (colors, alignment) unsuitable for machine parsing. Use `find`, bash globs, or `stat` instead.

---

**Q26 🔥 What's wrong with this script?**
```bash
for file in $(ls /tmp); do
  rm "$file"
done
```
> Three problems:
> 1. Word-splits on spaces in filenames — `my file.txt` becomes two tokens
> 2. Doesn't handle newlines in filenames
> 3. Path context: `rm` won't find the files without full path
>
> Fix: `for file in /tmp/*; do rm "$file"; done`

---

**Q27. How do you check if a directory is empty using `ls`?**
> ```bash
> if [ -z "$(ls -A /path/to/dir)" ]; then
>   echo "empty"
> fi
> ```
> Better (no subprocess): `if [ ! "$(ls -A dir)" ]; then ...`
> Best (no ls at all): use `find . -maxdepth 0 -empty`

---

**Q28. What happens when you run `ls` and `LS_COLORS` is not set?**
> `ls --color=auto` uses built-in compiled-in defaults for colors. To restore the standard color database: `eval $(dircolors -b)`.

---

**Q29. How do you get `ls` color output to work inside `less`?**
> ```bash
> ls --color=always | less -R
> ```
> `--color=always` forces ANSI codes even to a pipe. `less -R` renders them instead of showing raw escape sequences.

---

## Scenario-Based

**Q30 🔥 You have a directory with files named `report_1.txt` through `report_20.txt`. How do you list them in correct numeric order?**
> ```bash
> ls -v
> # Without -v, ls sorts lexicographically: report_1, report_10, report_11... report_2
> # With -v: report_1, report_2, report_3... report_20
> ```

---

**Q31. A file has permissions `-rwsr-xr-x` and is owned by root. A regular user runs it. What UID does the process run as?**
> **root** (UID 0). The SUID bit causes the process to run with the **file owner's** UID, not the caller's. This is how `passwd`, `sudo`, and `ping` work.

---

**Q32. You suspect a file was recently modified but `ls -lt` doesn't show it at the top. What could explain this?**
> - The file's **mtime was manually reset**: `touch -t 202001010000 file.txt`
> - The filesystem is mounted with `noatime` but that only affects atime
> - You're looking at ctime vs mtime — try `ls -lct`
> - The file is on a different filesystem with clock skew
> - NFS mount with time synchronization issues

---

**Q33. How do you list all files in `/var/log` that are larger than 100MB without using `find`?**
> ```bash
> ls -lSh /var/log | awk '$5 ~ /G/ || ($5+0) > 100'
> ```
> Honestly, `find` is better here:
> ```bash
> find /var/log -maxdepth 1 -size +100M -ls
> ```

---

**Q34. What is the difference between these two commands?**
```bash
ls -l /etc/passwd
ls -lL /etc/passwd
```
> `/etc/passwd` is a regular file, not a symlink, so both commands show identical output. The difference would matter if `/etc/passwd` were a symlink — then `-lL` would follow it and show the target's metadata.

---

**Q35 🔥 On macOS, `ls --color=auto` fails. Why, and what's the fix?**
> macOS uses **BSD ls**, which doesn't support GNU long options like `--color`. The BSD equivalent is `-G`:
> ```bash
> ls -G           # macOS color
> ls --color=auto # Linux only
>
> # Cross-platform alias:
> alias ls='ls --color=auto 2>/dev/null || ls -G'
> ```

---

**Q36. A directory has `r` but not `x` permission. What can and can't you do?**
> - **Can**: `ls dirname/` — you can see filenames (directory entries are readable)
> - **Can't**: `cd dirname/`, `cat dirname/file.txt`, or `stat` any file inside
>
> The `x` bit on a directory means "search/traverse" — without it you can't access anything inside, even if you know the filename.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
