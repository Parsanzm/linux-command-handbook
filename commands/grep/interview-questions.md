# grep — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [Regex & Patterns](#regex--patterns)
- [Output & Filtering](#output--filtering)
- [Recursive & Files](#recursive--files)
- [Exit Codes & Scripting](#exit-codes--scripting)
- [Performance](#performance)
- [Scenario-Based](#scenario-based)
- [Advanced](#advanced)

---

## Conceptual

**Q1. What does `grep` stand for?**
> **g/re/p** — from the `ed` line editor command: **g**lobally search for a **r**egular **e**xpression and **p**rint matching lines. Written by Ken Thompson in 1973.

---

**Q2 🔥 What are the three regex modes of grep?**
> - **BRE** (Basic Regular Expressions) — default (`grep` or `grep -G`). Metacharacters like `+`, `?`, `|`, `()` must be escaped with `\`.
> - **ERE** (Extended Regular Expressions) — `grep -E` or `egrep`. Metacharacters work without escaping.
> - **PCRE** (Perl-Compatible) — `grep -P`. Adds `\d`, `\w`, `\s`, lookaheads, lookbehinds, non-greedy, etc.

---

**Q3. What are grep's exit codes and why do they matter?**
> - `0` — match found
> - `1` — no match (NOT an error)
> - `2` — error (bad regex, file not found, etc.)
>
> In scripts with `set -e`, a `grep` with no match exits the script. Fix: `grep "pattern" file || true`

---

**Q4. What is the difference between `grep`, `egrep`, `fgrep`, and `rgrep`?**
> - `egrep` = `grep -E` (ERE mode)
> - `fgrep` = `grep -F` (fixed string, no regex — fastest)
> - `rgrep` = `grep -r` (recursive)
>
> All three (`egrep`, `fgrep`, `rgrep`) are deprecated aliases. Use the flag form.

---

**Q5. How does `grep` process input?**
> Line by line. It reads each line, applies the regex/pattern, and if the line matches, writes it to stdout. It is not aware of file structure beyond lines — it cannot natively match patterns spanning multiple lines.

---

## Basic Usage

**Q6 🔥 How do you search for a pattern case-insensitively?**
> ```bash
> grep -i "error" file.txt
> ```

---

**Q7 🔥 How do you search recursively in all files under a directory?**
> ```bash
> grep -r "pattern" /path/to/dir/
> grep -r --include="*.py" "pattern" /src/   # restrict to Python files
> ```

---

**Q8 🔥 How do you show only the filenames that contain a match?**
> ```bash
> grep -l "pattern" *.txt
> grep -rl "pattern" /src/    # recursive
> ```

---

**Q9. How do you invert a grep search (show lines that do NOT match)?**
> ```bash
> grep -v "pattern" file.txt
> grep -v "^#" config.conf      # exclude comments
> grep -v "^$" file.txt         # exclude empty lines
> ```

---

**Q10. How do you count the number of matching lines?**
> ```bash
> grep -c "pattern" file.txt
> # Note: returns 0 (with exit code 1) when no matches — handle in scripts
> ```

---

**Q11. How do you show line numbers alongside matches?**
> ```bash
> grep -n "pattern" file.txt
> # Output: 42:matching line content
> ```

---

**Q12 🔥 How do you show context lines around a match?**
> ```bash
> grep -A 3 "error" log.txt   # 3 lines After
> grep -B 2 "error" log.txt   # 2 lines Before
> grep -C 3 "error" log.txt   # 3 lines both sides
> ```

---

## Regex & Patterns

**Q13 🔥 What is the difference between `grep "a+"` and `grep -E "a+"`?**
> In BRE (default), `+` is a literal character. `grep "a+"` matches lines containing the literal string `a+`.
> In ERE (`grep -E`), `+` is a metacharacter meaning "one or more". `grep -E "a+"` matches lines with one or more consecutive `a`s.

---

**Q14. How do you match a whole word with grep?**
> ```bash
> grep -w "log" file.txt
> # Matches "log" but not "logger", "syslog", "catalog"
> # Equivalent to word boundary: grep -P "\blog\b" file.txt
> ```

---

**Q15 🔥 How do you match lines that start with a pattern? End with a pattern?**
> ```bash
> grep "^pattern" file.txt    # lines starting with "pattern"
> grep "pattern$" file.txt    # lines ending with "pattern"
> grep "^$" file.txt          # empty lines only
> grep -v "^$" file.txt       # non-empty lines only
> ```

---

**Q16. How do you match multiple patterns (OR logic)?**
> ```bash
> grep -e "error" -e "warning" file.txt    # multiple -e flags
> grep -E "error|warning" file.txt          # ERE alternation
> grep -F -f patterns.txt file.txt          # patterns from file
> ```

---

**Q17. How do you match a literal dot? A literal star?**
> Escape with `\`:
> ```bash
> grep "\." file.txt    # literal dot
> grep "\*" file.txt    # literal star
> grep -F "1.2.3" file.txt  # -F treats everything as literal (safest)
> ```

---

**Q18 🔥 What is the difference between `-o` and normal grep output?**
> Normal grep prints the **entire matching line**. `-o` prints only the **matched portion** of the line.
> ```bash
> echo "The error code is 404" | grep -oE "[0-9]+"
> # Output: 404   (just the number, not the whole line)
> ```

---

**Q19. What does `grep -P "\d+"` do that `grep -E "[0-9]+"` doesn't?**
> `\d` in PCRE is a shorthand for `[0-9]` (or Unicode digits in Unicode mode). Functionally they're the same for ASCII. PCRE (`-P`) also adds lookaheads, lookbehinds, non-greedy quantifiers, `\K`, and other advanced features unavailable in ERE.

---

**Q20. What is a lookahead in grep and how do you use it?**
> A lookahead asserts that a pattern is (or isn't) followed by another pattern, without including the second pattern in the match. Requires `-P` (PCRE).
> ```bash
> grep -P "foo(?=bar)" file.txt    # match "foo" only if followed by "bar"
> grep -P "foo(?!bar)" file.txt    # match "foo" only if NOT followed by "bar"
> grep -Po "\d+(?= dollars)" file.txt   # number before " dollars" (prints just number)
> ```

---

## Output & Filtering

**Q21 🔥 How do you use grep to check if a file contains a pattern (without output)?**
> ```bash
> grep -q "pattern" file.txt
> echo $?   # 0 = found, 1 = not found
>
> if grep -q "error" file.txt; then
>   echo "Found errors"
> fi
> ```

---

**Q22. How do you print only the filename for files that do NOT contain a pattern?**
> ```bash
> grep -rL "Copyright" /src/
> grep -L "pattern" *.txt
> ```

---

**Q23. How do you make grep print the matched text in color even when piping?**
> ```bash
> grep --color=always "error" log | less -R
> # --color=auto (default) disables color when output is not a terminal
> # --color=always forces color codes even in pipes
> ```

---

**Q24. How do you stop grep after the first N matches?**
> ```bash
> grep -m 1 "error" huge.log    # stop after 1 match
> grep -m 10 "error" huge.log   # stop after 10 matches
> ```

---

## Recursive & Files

**Q25 🔥 How do you grep recursively but only in Python files, excluding the venv directory?**
> ```bash
> grep -r --include="*.py" --exclude-dir="venv" "pattern" /src/
> ```

---

**Q26. What is the difference between `grep -r` and `grep -R`?**
> `-r` is recursive but does **not** follow symbolic links. `-R` is recursive and **does** follow symbolic links (can cause infinite loops with circular symlinks).

---

**Q27. How do you search for a pattern in compressed log files?**
> ```bash
> zgrep "pattern" file.gz      # gzip
> bzgrep "pattern" file.bz2    # bzip2
> xzgrep "pattern" file.xz     # xz
> ```

---

**Q28. How do you search for a pattern in a file whose name starts with `-`?**
> ```bash
> grep "pattern" -- -filename.txt
> grep "pattern" ./-filename.txt
> ```

---

## Exit Codes & Scripting

**Q29 🔥 You have `set -e` in your script and `grep` returns 1 (no match). What happens and how do you fix it?**
> The script exits immediately because `set -e` treats any non-zero exit code as failure. Fix:
> ```bash
> grep "pattern" file.txt || true         # ignore exit code
> grep "pattern" file.txt || :            # same (: is a no-op)
>
> if grep -q "pattern" file.txt; then     # explicit check — best practice
>   echo "found"
> fi
> ```

---

**Q30. How do you extract a value from a config file using grep?**
> ```bash
> # Config: version=1.2.3
> version=$(grep -oP "(?<=version=)\S+" config.txt)
> # Or with ERE (less precise):
> version=$(grep -E "^version=" config.txt | cut -d= -f2)
> ```

---

**Q31. How do you validate an email address in a bash script using grep?**
> ```bash
> validate_email() {
>   echo "$1" | grep -qP "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"
> }
>
> if validate_email "user@example.com"; then
>   echo "Valid"
> fi
> ```

---

## Performance

**Q32 🔥 When should you use `grep -F` instead of regular `grep`?**
> When searching for a **fixed string with no regex metacharacters**. `-F` skips the regex engine and uses Boyer-Moore string search — significantly faster on large files.
> ```bash
> grep -F "error: connection refused" huge.log   # faster
> grep "error: connection refused" huge.log       # slower (regex engine)
> ```

---

**Q33. How do you make `grep` output appear in real-time when used in a pipeline?**
> ```bash
> tail -f app.log | grep --line-buffered "error"
> # Without --line-buffered, grep's output is block-buffered in pipes
> # and may not appear until the buffer fills (usually 4096 or 8192 bytes)
> ```

---

**Q34. Why is `grep ".*pattern.*"` slower than `grep "pattern"`?**
> `grep` already searches for the pattern **anywhere** in the line by default. Adding `.*` on both sides forces the regex engine to do extra work matching the wildcards, while `grep "pattern"` uses optimized string search algorithms. The `.*` provides no benefit and only adds overhead.

---

## Scenario-Based

**Q35 🔥 How do you find the top 10 IPs making failed login attempts in an auth log?**
> ```bash
> grep "Failed password" /var/log/auth.log \
>   | grep -oE "from [0-9.]+" \
>   | sort | uniq -c | sort -rn | head -10
> ```

---

**Q36 🔥 How do you find all files containing "TODO" in a git repo, excluding `.git`?**
> ```bash
> grep -r --exclude-dir=".git" "TODO" .
> # Or: git grep "TODO"  (respects .gitignore automatically)
> ```

---

**Q37. A developer accidentally committed a password. How do you find it in all files?**
> ```bash
> grep -rn --include="*.{py,js,yaml,env,conf}" \
>   -E "(password|secret|api_key|token)\s*=\s*['\"][^'\"]{4,}" .
>
> # For git history:
> git log -p | grep -E "(password|secret).*=.*['\"][^'\"]{4,}"
> ```

---

**Q38. How do you count HTTP 500 errors per hour from an nginx access log?**
> ```bash
> grep " 500 " /var/log/nginx/access.log \
>   | grep -oP "\d{2}/\w+/\d{4}:\d{2}" \
>   | sort | uniq -c
> ```

---

**Q39 🔥 What's the difference between these two commands?**
```bash
grep -E "cat|dog" file.txt
grep "cat\|dog" file.txt
```
> Both find lines containing "cat" or "dog", but via different engines:
> - `grep -E "cat|dog"` uses ERE where `|` is a metacharacter for alternation
> - `grep "cat\|dog"` uses BRE where `\|` is a GNU extension for alternation
>
> The BRE version (`\|`) is a GNU extension — not portable to BSD grep on macOS. Use `-E` for portability.

---

**Q40 🔥 You need to replace all occurrences of "foo" with "bar" in multiple files. Can grep do this?**
> No — grep only **reads** and matches; it doesn't modify files. Use `sed`:
> ```bash
> grep -rl "foo" /src/ | xargs sed -i 's/foo/bar/g'
> # grep -rl finds files; xargs sed -i performs in-place replacement
> ```

---

**Q41. How do you search for a multiline pattern with grep?**
> Standard `grep` can't — it's line-based. Options:
> ```bash
> # pcregrep (install separately):
> pcregrep -M "start.*\nend" file.txt
>
> # perl:
> perl -0777 -ne 'print if /start.*end/s' file.txt
>
> # awk (for simple cases):
> awk '/start/{found=1} found{print; if(/end/) found=0}' file.txt
> ```

---

**Q42. How do you extract all unique email addresses from a file?**
> ```bash
> grep -oE "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" emails.txt \
>   | sort -u
> ```

---

**Q43 🔥 Why does this pipeline sometimes lose output?**
```bash
grep "error" huge.log | head -1
```
> When `head` exits after reading its first line, `grep` receives **SIGPIPE** and may print "Broken pipe" to stderr. The output itself is not lost (head already read it), but the error message can be confusing. Fix:
> ```bash
> grep "error" huge.log 2>/dev/null | head -1
> # or:
> grep -m 1 "error" huge.log   # stop grep itself after 1 match
> ```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
