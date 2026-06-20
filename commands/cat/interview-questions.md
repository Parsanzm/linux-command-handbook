# cat — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Flags & Options](#flags--options)
- [File Handling](#file-handling)
- [Pipelines & Redirection](#pipelines--redirection)
- [Tricky Behavior](#tricky-behavior)
- [Scenario-Based](#scenario-based)
- [Advanced & Internals](#advanced--internals)

---

## Conceptual

**Q1 🔥 What does `cat` stand for?**
> Concatenate. It reads files sequentially and writes them to standard output.

---

**Q2. Where is `cat` located on a Linux system?**
> `/bin/cat` on most Linux distros. Part of **GNU coreutils**. On macOS it's the BSD version at `/bin/cat` or `/usr/bin/cat`.

---

**Q3. What are the three main uses of `cat`?**
> 1. Display file contents
> 2. Concatenate multiple files into one
> 3. Create files from standard input

---

**Q4. What syscalls does `cat` use internally?**
> `open()` → `read()` → `write()` on stdout. On modern Linux it may use `splice()` or `sendfile()` to avoid copying data into userspace — making it very fast for large files.

---

**Q5. What is the exit code of `cat` when a file doesn't exist?**
> `1`. Success returns `0`. You can check with `echo $?` after running `cat`.

---

## Flags & Options

**Q6 🔥 What is the difference between `cat -n` and `cat -b`?**
> `-n` numbers ALL lines including blank ones. `-b` numbers only non-blank lines. When both are given, `-b` takes precedence.

---

**Q7. What does `cat -A` do?**
> Equivalent to `-vET`. Shows non-printing characters (`-v`), line endings as `$` (`-E`), and tabs as `^I` (`-T`). Useful for diagnosing encoding and CRLF issues.

---

**Q8. What does `cat -s` do?**
> Squeezes multiple consecutive blank lines into a single blank line. Useful for cleaning up files with excessive whitespace.

---

**Q9. What does `cat -v` show?**
> Non-printing characters using `^X` notation for control characters and `M-X` notation for high bytes (non-ASCII). Does NOT interpret newlines or tabs.

---

**Q10. On macOS, `cat --help` gives an error. Why?**
> macOS uses the **BSD version** of `cat`, which doesn't support GNU long options like `--help` or `--version`. Use `man cat` instead.

---

## File Handling

**Q11 🔥 How do you `cat` a file whose name starts with a dash?**
> Two ways:
> ```bash
> cat -- -filename.txt
> cat ./-filename.txt
> ```
> The `--` signals end of options; everything after is treated as a filename.

---

**Q12. How do you `cat` a file with spaces in the name?**
> Quote the filename:
> ```bash
> cat "my file.txt"
> cat my\ file.txt
> ```

---

**Q13. What happens when you run `cat` on a directory?**
> Error: `cat: /path: Is a directory`. cat only works on regular files, symlinks (to files), and special files like `/proc/*`.

---

**Q14. Does `cat` follow symbolic links?**
> Yes, automatically. `cat symlink.txt` reads the file the symlink points to. If the symlink is broken (target doesn't exist), cat returns "No such file or directory".

---

**Q15. What happens if one file in a list doesn't exist?**
> cat processes the other files normally and prints an error to stderr for the missing one. Exit code is `1`.
> ```bash
> cat file1.txt missing.txt file3.txt
> # file1.txt: printed ✅
> # missing.txt: error on stderr ❌
> # file3.txt: printed ✅
> ```

---

## Pipelines & Redirection

**Q16 🔥 What does `-` mean as an argument to `cat`?**
> It means "read from stdin at this position in the file list".
> ```bash
> cat header.txt - footer.txt
> # Reads header.txt, then stdin, then footer.txt
> ```

---

**Q17 🔥 What does this command do?**
```bash
cat /dev/null > important.log
```
> Truncates `important.log` to zero bytes without deleting it. The file still exists and has the same permissions, but is now empty.

---

**Q18. What is the difference between `>` and `>>` when used with `cat`?**
> `>` truncates the file before writing (overwrites). `>>` appends to the end of the file.

---

**Q19 🔥 Why is this dangerous?**
```bash
cat file.txt > file.txt
```
> The shell opens `file.txt` for writing (with `>`) BEFORE `cat` runs, which truncates the file to zero bytes. Then `cat` reads the now-empty file. Result: `file.txt` is empty.

---

**Q20. How would you use `cat` to write to a root-owned file without logging in as root?**
> Use `sudo tee`:
> ```bash
> echo "new content" | sudo tee /etc/protected_file
> # append:
> echo "new line" | sudo tee -a /etc/protected_file
> ```
> `sudo cat > file` fails because the `>` redirection runs as the current user, not root.

---

## Tricky Behavior

**Q21 🔥 What is "Useless Use of Cat"? Give an example and the fix.**
> Using `cat` to feed a single file into a command that can read files directly.
> ```bash
> # Useless:
> cat file.txt | grep "pattern"
>
> # Better:
> grep "pattern" file.txt
> ```
> The fix avoids spawning an extra process and is slightly faster. Cat IS appropriate when combining multiple files or making stdin explicit.

---

**Q22. What happens when you run `cat` with no arguments?**
> It reads from stdin and echoes each line back after you press Enter. Press Ctrl+D to send EOF and exit. This is not a bug — it's the correct POSIX behavior.

---

**Q23. How does `cat` handle files that don't end with a newline?**
> cat prints the file as-is, with no newline at the end. If you concatenate two such files, their last and first lines get joined with no separator:
> ```bash
> printf "abc" > a.txt
> printf "def" > b.txt
> cat a.txt b.txt    # → abcdef   (no separator)
> ```

---

**Q24. What is `tac` and how does it relate to `cat`?**
> `tac` is the reverse of `cat` — it prints a file with lines in reverse order (last line first). The name is "cat" spelled backwards.

---

**Q25. Your terminal shows garbled output after running `cat` on a file. What do you do?**
> You likely ran `cat` on a binary file. Fix with:
> ```bash
> reset            # Full terminal reset (most reliable)
> stty sane        # Restore sane terminal settings
> printf '\033c'   # ANSI escape: reset terminal
> ```

---

## Scenario-Based

**Q26 🔥 You have 500 log files named `access_2024_*.log`. How do you merge them all into one file, in chronological order?**
> ```bash
> cat access_2024_*.log > all_access.log
> # Shell glob expands in alphabetical order, which matches date order
> # for YYYY-MM-DD or YYYYMMDD formats
>
> # If order is uncertain:
> ls -t access_2024_*.log | xargs cat > all_access.log   # by modification time
> ```

---

**Q27. How do you merge multiple CSV files while keeping only one header line?**
> ```bash
> head -1 data_jan.csv > combined.csv
> tail -n +2 -q data_*.csv >> combined.csv
> # -q suppresses the filename headers in tail
> ```

---

**Q28 🔥 How would you use `cat` to create a multi-line config file in a script, with variable substitution?**
> ```bash
> cat > /etc/app/config.conf << EOF
> host = ${APP_HOST}
> port = ${APP_PORT}
> debug = false
> EOF
> ```
> Use `<< 'EOF'` (quoted) to prevent variable expansion.

---

**Q29. A developer needs to view an actively growing log file. Should they use `cat`?**
> No. `cat` reads a file once and exits. For live log monitoring, use:
> ```bash
> tail -f /var/log/app.log          # Follow new content
> tail -F /var/log/app.log          # Follow even if file rotates
> ```

---

**Q30 🔥 What is the difference between `zcat`, `bzcat`, and `xzcat`? When would you use each?**
> They decompress and print compressed files to stdout without creating temp files:
> - `zcat` → gzip (`.gz`) files
> - `bzcat` → bzip2 (`.bz2`) files — slower but better compression
> - `xzcat` → xz/lzma (`.xz`) files — best compression ratio
>
> Use when you want to process a compressed file without decompressing to disk:
> ```bash
> zcat server.log.gz | grep "ERROR" | wc -l
> bzcat huge_dataset.bz2 | awk -F',' '{sum+=$3} END{print sum}'
> ```

---

## Advanced & Internals

**Q31. How does GNU `cat` differ from BSD `cat`?**
> GNU (Linux): supports `-A`, `--version`, `--help`, uses `splice()` for performance.
> BSD (macOS): supports `-l` (lock), no long options, standard read/write.
> Write portable scripts using short flags only (e.g., `-E` not `--show-ends`).

---

**Q32. Can `cat` read from `/proc` filesystem files? What's special about them?**
> Yes. `/proc` files are virtual — the kernel generates their content on-the-fly when read. They always show as 0 bytes in `stat()`, but `cat` can read them normally.
> ```bash
> cat /proc/cpuinfo     # CPU details
> cat /proc/meminfo     # Memory stats
> cat /proc/1/cmdline   # Command line of PID 1
> ```

---

**Q33. How would you use `cat` to safely empty a file without affecting other processes that have it open?**
> ```bash
> > file.txt          # Truncate via shell redirect (no cat needed)
> cat /dev/null > file.txt   # Same result
> ```
> Processes that already have the file open will keep their file descriptor pointing to the old inode — they won't see the truncation until they reopen the file. This is safe for log rotation scenarios.

---

**Q34. Why might `cat file | command` be slower than `command file`?**
> `cat file | command` spawns an extra process (`cat`) and uses a pipe, which adds:
> - Process creation overhead (`fork/exec`)
> - Kernel pipe buffering
> - Context switches between processes
>
> `command file` reads the file directly with one process. For large files the difference can be measurable.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
