# find — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [Time & Size](#time--size)
- [Permissions & Ownership](#permissions--ownership)
- [Actions & exec](#actions--exec)
- [Operators & Logic](#operators--logic)
- [Symlinks & Pruning](#symlinks--pruning)
- [Scenario-Based](#scenario-based)
- [Advanced & Internals](#advanced--internals)

---

## Conceptual

**Q1. What is `find` and how is it different from `ls` or `grep`?**
> `find` traverses a directory tree and matches files based on metadata (name, type, size, time, permissions). Unlike `ls` (which only lists one directory level) and `grep` (which searches file *contents*), `find` recursively searches the *filesystem structure* and can act on results with `-exec`, `-delete`, etc. It's a file processing engine, not just a search tool.

---

**Q2. What package provides `find` on Linux?**
> **GNU findutils**, which also includes `xargs`, `locate`, and `updatedb`. On macOS it's BSD `find` (part of the base system).

---

**Q3 🔥 What is the default action of `find` when no action is specified?**
> `-print` — it prints each matched path followed by a newline. But this is only implied when the expression contains no action at all. Once you add `-exec`, `-delete`, `-ls`, or `-printf`, the implicit `-print` is no longer added.

---

**Q4. What is the default symlink behavior of `find`?**
> `-P` (never follow symlinks) is the default. `find` will find symlinks themselves but won't follow them into directories. Use `-L` to follow all symlinks, or `-H` to follow only those given on the command line.

---

**Q5. What does `find` return as exit code?**
> `0` if all files were processed successfully. `1` if any errors occurred (e.g., permission denied on some directories). This is true even if no files matched — exit code 0 simply means "no errors," not "found something."

---

## Basic Usage

**Q6 🔥 How do you find all `.log` files under `/var`?**
> ```bash
> find /var -name "*.log"
> find /var -name "*.log" -type f   # only regular files, not dirs named *.log
> ```

---

**Q7. How do you find files by name, case-insensitively?**
> ```bash
> find . -iname "readme*"
> find . -iname "*.JPG"    # matches .jpg, .JPG, .Jpg
> ```

---

**Q8 🔥 How do you limit `find` to only the current directory (no recursion)?**
> ```bash
> find . -maxdepth 1 -name "*.txt"
> find . -maxdepth 1 -type f         # all files, not subdirs
> ```

---

**Q9. What is the difference between `-name` and `-path`?**
> `-name` matches only the **filename** (no directory components). `-path` matches the **full path** from the starting point. Use `-path` when you need to match directory structure, e.g., `find . -path "*/config/*.yaml"`.

---

**Q10. How do you find all empty files? All empty directories?**
> ```bash
> find . -empty -type f    # empty files
> find . -empty -type d    # empty directories
> find . -empty            # both empty files and empty directories
> ```

---

## Time & Size

**Q11 🔥 How do you find files modified in the last 24 hours?**
> ```bash
> find . -mtime -1      # modified less than 1 day ago
> find . -mmin -1440    # more precise: last 1440 minutes (24h)
> ```
> Note: `-mtime -1` is NOT exactly 24 hours — it uses 24-hour boundary math. Use `-mmin -1440` for true last-24-hours.

---

**Q12. What is the difference between `-mtime`, `-atime`, and `-ctime`?**
> - `-mtime`: modification time — when file **content** was last changed
> - `-atime`: access time — when file was last **read**
> - `-ctime`: change time — when file **metadata** (permissions, ownership, name) last changed
>
> `ctime` is NOT creation time (Linux doesn't have a creation time in ext4 traditionally).

---

**Q13 🔥 What does `-mtime +7` mean vs `-mtime 7` vs `-mtime -7`?**
> - `+7`: modified **more than** 7 days ago (older than a week)
> - `7`: modified **exactly** 7 days ago (within that 24h boundary)
> - `-7`: modified **less than** 7 days ago (within the last week)
>
> Common use: `-mtime +30` to find files older than 30 days.

---

**Q14. How do you find files larger than 100MB?**
> ```bash
> find . -size +100M
> find / -xdev -size +100M -type f -ls 2>/dev/null   # full system search
> ```

---

**Q15. What is the default size unit in `find -size`?**
> **512-byte blocks** (`b`). To search by bytes, use the `c` suffix: `-size +100c`. Common suffixes: `c` (bytes), `k` (kibibytes), `M` (mebibytes), `G` (gibibytes).

---

**Q16. How do you find files modified after a specific date?**
> ```bash
> find . -newer /path/to/reference_file    # newer than reference's mtime
> find . -newermt "2024-06-01"             # GNU: after June 1, 2024
> ```

---

## Permissions & Ownership

**Q17 🔥 How do you find all SUID files on the system?**
> ```bash
> find / -type f -perm -4000 2>/dev/null
> find / -type f \( -perm -4000 -o -perm -2000 \) -ls 2>/dev/null  # SUID + SGID
> ```

---

**Q18. What is the difference between `-perm 644`, `-perm -644`, and `-perm /644`?**
> - `-perm 644`: **exact** match — must be exactly `rw-r--r--`
> - `-perm -644`: **all** listed bits must be set — matches 644, 755, 777, etc.
> - `-perm /644`: **any** of the listed bits must be set — very broad
>
> For "at least read/write for owner" use `-perm -600`. For SUID audit use `-perm -4000`.

---

**Q19. How do you find world-writable files?**
> ```bash
> find / -type f -perm /o+w 2>/dev/null
> find / -type f -perm -002 2>/dev/null   # equivalent in octal
> ```

---

**Q20. How do you find orphaned files (owned by deleted users)?**
> ```bash
> find / -nouser -type f 2>/dev/null
> find / -nogroup -type f 2>/dev/null
> ```

---

**Q21. How do you find files owned by a specific user?**
> ```bash
> find /home -user alice
> find . -uid 1001           # by numeric UID
> ```

---

## Actions & exec

**Q22 🔥 What is the difference between `-exec cmd {} \;` and `-exec cmd {} +`?**
> - `\;` runs the command **once per file**: `rm file1`, `rm file2`, `rm file3`
> - `+` **batches** all files into one call: `rm file1 file2 file3` (like xargs)
>
> `+` is faster (fewer process spawns) and preferred when the command accepts multiple arguments.

---

**Q23 🔥 Why is `-execdir` safer than `-exec`?**
> `-exec` passes the absolute path, which is vulnerable to TOCTOU (Time-Of-Check-Time-Of-Use) race conditions — an attacker could replace the file with a symlink between find's check and the exec. `-execdir` changes to the file's directory first and uses a relative path (`./filename`), which prevents symlink attacks.

---

**Q24. How do you run a shell command (with variables, pipes) inside `-exec`?**
> ```bash
> find . -name "*.txt" -exec sh -c 'echo "Processing: $1"; wc -l "$1"' _ {} \;
> # _ is $0 (program name placeholder), {} becomes $1
> ```

---

**Q25. How do you prompt before each `-exec` action?**
> ```bash
> find . -name "*.tmp" -ok rm {} \;
> # Asks: < rm ... ? > before each file
> ```

---

**Q26. What is the danger of `-delete` and how do you use it safely?**
> `-delete` is **irreversible** (no trash/recycle). It also silently implies `-depth` (post-order traversal), which can interact unexpectedly with `-prune`. Safe workflow:
> ```bash
> find . -name "*.tmp" -print   # 1. verify what will be deleted
> find . -name "*.tmp" -delete  # 2. delete
> ```

---

## Operators & Logic

**Q27 🔥 How do you find files that are `.txt` OR `.md`?**
> ```bash
> find . \( -name "*.txt" -o -name "*.md" \)
> # Parentheses must be escaped with \( \) in shell
> ```

---

**Q28. How do you negate a condition in `find`?**
> ```bash
> find . ! -name "*.log"         # NOT *.log
> find . -not -name "*.log"      # same (GNU alias)
> find . ! -user root            # not owned by root
> find . ! -path "*/.git/*"      # exclude .git paths
> ```

---

**Q29. What is the operator precedence in `find` expressions?**
> `!` (NOT) > `-a`/`-and` > `-o`/`-or`
> Just like C: NOT, then AND, then OR. Always use `\( \)` when mixing OR and AND to be explicit and avoid bugs.

---

**Q30. What does this find command do?**
```bash
find . -name ".git" -prune -o -name "*.py" -print
```
> Skips `.git` directories entirely (prunes them from traversal) and prints all `.py` files in the rest of the tree. The `-o` (OR) means: either match `.git` (and prune) OR match `*.py` (and print). This is the standard idiom for excluding directories in find.

---

## Symlinks & Pruning

**Q31 🔥 How do you find broken symlinks?**
> ```bash
> find -L . -type l
> # With -L, working symlinks are dereferenced and appear as f or d
> # Broken symlinks can't be dereferenced, so they remain type l
>
> # Alternative without -L:
> find . -type l ! -exec test -e {} \; -print
> ```

---

**Q32. How do you prevent `find` from crossing filesystem boundaries?**
> ```bash
> find / -xdev -name "*.conf"
> find / -mount -name "*.conf"   # same as -xdev
> ```
> Important when running `find /` to avoid traversing `/proc`, `/sys`, NFS mounts, etc.

---

**Q33. Why doesn't `-prune` work with `-delete`?**
> `-delete` implies `-depth` (post-order traversal), which conflicts with `-prune`. Pruning needs pre-order (parent before children) to prevent descending — but `-depth` processes children first. Workaround: find the files to delete separately, then delete with `xargs rm`.

---

## Scenario-Based

**Q34 🔥 How do you find and delete all `.pyc` files and `__pycache__` directories?**
> ```bash
> find . -name "*.pyc" -delete
> find . -type d -name "__pycache__" -exec rm -rf {} +
> ```

---

**Q35 🔥 How do you find the 10 largest files on the system?**
> ```bash
> find / -xdev -type f -printf "%s %p\n" 2>/dev/null \
>   | sort -rn \
>   | head -10
> ```

---

**Q36. How do you find files modified since the last backup?**
> ```bash
> find /data -newer /var/backups/last_backup.timestamp -type f
> # After backup completes:
> touch /var/backups/last_backup.timestamp
> ```

---

**Q37. You need to `chmod 644` all files and `chmod 755` all directories under `/var/www`. How?**
> ```bash
> find /var/www -type f -exec chmod 644 {} +
> find /var/www -type d -exec chmod 755 {} +
> ```

---

**Q38 🔥 How do you find all files owned by a user who has been deleted from the system?**
> ```bash
> find / -nouser -type f -ls 2>/dev/null
> # -nouser matches files whose UID has no corresponding entry in /etc/passwd
> ```

---

**Q39. How do you count the total number of files in a directory tree?**
> ```bash
> find . -type f | wc -l
> find . -type f -printf ".\n" | wc -l   # safer: handles filenames with newlines
> ```

---

**Q40 🔥 What's wrong with this command and how do you fix it?**
```bash
find . -name "*.log" | xargs rm
```
> It breaks on filenames with **spaces or special characters**. xargs splits on whitespace by default.
> Fix:
> ```bash
> find . -name "*.log" -print0 | xargs -0 rm
> # or:
> find . -name "*.log" -delete
> ```

---

## Advanced & Internals

**Q41. How does `-maxdepth` improve `find` performance?**
> Without `-maxdepth`, find calls `stat()` on every file in every subdirectory recursively. With `-maxdepth 1`, it only reads the top-level directory — one `opendir` + one pass of `readdir` + `stat` for each entry. On deep trees with millions of files, this is orders of magnitude faster.

---

**Q42. Why is `-exec cmd {} +` faster than `-exec cmd {} \;`?**
> `\;` spawns one child process per file: N files = N `fork`/`exec` calls. `+` batches all filenames into one command invocation (like xargs), so N files might need only 1-2 process spawns. For N=10,000 files, this is a 10,000x reduction in process creation overhead.

---

**Q43. What does `-printf "%T@ %p\n"` accomplish and why is it useful?**
> `%T@` prints the modification time as **seconds since epoch** (a floating point number). Combined with `sort -n`, this lets you sort files by modification time numerically — something `ls -t` does but without the ability to pipe safely. Example:
> ```bash
> find . -type f -printf "%T@ %p\n" | sort -rn | awk '{print $2}' | head -10
> # Top 10 most recently modified files
> ```

---

**Q44. How do you make `find` search case-insensitively for file extensions across GNU and BSD?**
> ```bash
> find . -iname "*.jpg"      # works on both GNU and BSD
> # vs:
> find . -regex ".*\.jpg" -regextype posix-egrep  # GNU only
> ```
> `-iname` is the portable solution.

---

**Q45. A developer runs `find / -name "*.conf"` and it hangs. Why, and how do you fix it?**
> It's traversing `/proc` — a virtual filesystem where some files block on read or generate infinite data. Fix:
> ```bash
> find / -xdev -name "*.conf" 2>/dev/null   # stay on one filesystem
> # or explicitly exclude /proc and /sys:
> find / -path /proc -prune -o -path /sys -prune -o -name "*.conf" -print
> ```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
