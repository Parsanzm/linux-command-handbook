# awk — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Fields & Records](#fields--records)
- [Patterns & Actions](#patterns--actions)
- [Variables & Scope](#variables--scope)
- [String & Regex](#string--regex)
- [Arrays](#arrays)
- [Type Coercion](#type-coercion)
- [Scenario-Based](#scenario-based)
- [Advanced & Internals](#advanced--internals)

---

## Conceptual

**Q1 🔥 What is awk and what does the name stand for?**
> awk is a pattern-scanning and text-processing programming language, not just a command. The name comes from the initials of its creators: **A**ho, **W**einberger, and **K**ernighan (Bell Labs, 1977). Unlike `grep` (search only) or `sed` (line-based substitution), awk treats input as structured records and fields, supporting variables, arrays, functions, and arithmetic — closer to a real programming language than a filter.

---

**Q2 🔥 What is the basic processing model of awk?**
> ```
> BEGIN { } runs once before any input
> for each input record:
>     split into fields
>     evaluate each pattern { action } rule in order
> END { } runs once after all input is processed
> ```
> A record is normally one line; a field is normally a whitespace-delimited token within that line. Every awk program is essentially a list of `pattern { action }` pairs evaluated against every record.

---

**Q3. What is the difference between `awk`, `gawk`, `mawk`, and `nawk`?**
> - `awk` — usually a symlink to one of the implementations below
> - `gawk` — GNU awk, most features (gensub, strftime, BEGINFILE/ENDFILE, FPAT), default on RHEL/Fedora
> - `mawk` — fastest implementation, fewer extensions, default `awk` on Debian/Ubuntu
> - `nawk` — "new awk," the AT&T/POSIX reference implementation, default on macOS/BSD
>
> For portability, stick to POSIX features unless you know `gawk` is available.

---

**Q4. When would you choose awk over sed, grep, or a scripting language like Python?**
> - **grep**: only filters lines, no field splitting, no arithmetic, no output transformation
> - **sed**: great for substitution and simple line editing, but clumsy for field-based logic or math
> - **awk**: ideal when data has columns/fields and you need filtering + transformation + aggregation in one pass
> - **Python**: better for complex logic, but slower to write for quick one-liners and has more overhead for simple text-processing tasks
>
> Rule of thumb: if the task is "extract/filter/sum/count columns from structured text," awk is usually the fastest tool to write and the fastest to run.

---

## Fields & Records

**Q5 🔥 What do `$0`, `$1`, and `NF` represent?**
> - `$0` — the entire current record (line) as read
> - `$1`, `$2`, ... — individual fields, split according to `FS`
> - `NF` — the **N**umber of **F**ields in the current record
> - `$NF` — the last field (since `NF` holds the count, `$NF` dereferences it)

---

**Q6. What is the difference between `NR` and `FNR`?**
> - `NR` — total record count across ALL input files processed so far (cumulative)
> - `FNR` — record count within the CURRENT file only (resets to 1 at the start of each new file)
>
> ```bash
> awk '{ print NR, FNR, FILENAME }' file1.txt file2.txt
> # NR keeps climbing; FNR restarts at 1 when file2.txt begins
> ```

---

**Q7 🔥 How do you change the field separator, and what's the difference between using `-F` and setting `FS` in `BEGIN`?**
> ```bash
> awk -F: '{ print $1 }' /etc/passwd          # set FS via command-line flag
> awk 'BEGIN{FS=":"} { print $1 }' /etc/passwd # set FS in BEGIN block
> ```
> Both achieve the same result for whole-program field separation. `-F` is shorthand and applies before the first record is read — functionally equivalent to setting it in `BEGIN`. The `BEGIN` approach is necessary if you need to compute `FS` dynamically or combine it with other initialization logic.

---

**Q8. What happens if you assign a value to a field, like `$2 = "new"`?**
> awk rebuilds `$0` by joining all fields together using `OFS` (Output Field Separator, default: single space). This means if your input was colon-separated but you never set `OFS`, the rebuilt `$0` will use spaces instead of colons — a common source of bugs.
> ```bash
> echo "a:b:c" | awk -F: '{ $2="X"; print }'
> # Output: a X c   ← colons lost! OFS defaults to space
> echo "a:b:c" | awk -F: 'BEGIN{OFS=":"} { $2="X"; print }'
> # Output: a:X:c   ← correct
> ```

---

**Q9. How do you print only lines with a specific number of fields?**
> ```bash
> awk 'NF == 3' file.txt     # exactly 3 fields
> awk 'NF > 5' file.txt      # more than 5 fields
> awk 'NF == 0' file.txt     # blank lines (no fields)
> ```

---

## Patterns & Actions

**Q10 🔥 What happens if you write a pattern with no action, or an action with no pattern?**
> - **Pattern, no action**: defaults to `{ print }` — prints the whole matching record. `awk '/error/' file` is shorthand for `awk '/error/ { print }' file`.
> - **No pattern, action only**: the action runs on EVERY record. `awk '{ print $1 }' file` runs for every line.
> - You cannot have neither — at least one of pattern or action must be present.

---

**Q11. What is a range pattern and how does it work?**
> A range pattern `/start/,/end/` matches from the first line where `start` matches through the first subsequent line where `end` matches (inclusive). It re-arms after each closing match, so it can match multiple ranges in the same file.
> ```bash
> awk '/BEGIN_BLOCK/,/END_BLOCK/' file.txt
> ```
> If `end` never appears, the range extends to the end of the file.

---

**Q12 🔥 What is the difference between `BEGIN`, the main body, and `END`?**
> - `BEGIN { }` — executes once, before any input is read. Used for initialization (setting `FS`, `OFS`, printing headers).
> - Main pattern-action rules — execute once per input record.
> - `END { }` — executes once, after all input has been processed (or after `exit` is called). Used for summaries, totals, final reports. Note: `$0`, `NF`, `NR` retain their values from the LAST processed record inside `END`.

---

## Variables & Scope

**Q13. How do you pass a shell variable into an awk program safely?**
> Use the `-v` flag — never string-interpolate shell variables directly into the awk program:
> ```bash
> name="alice"
> awk -v user="$name" '$1 == user { print }' file.txt    # ✅ safe
>
> # Avoid:
> awk "{ if (\$1 == \"$name\") print }" file.txt          # ❌ fragile, breaks on special chars
> ```
> `-v` assigns the variable BEFORE the `BEGIN` block runs, making it available everywhere in the program.

---

**Q14 🔥 Are awk variables global or local by default? How do you create local variables in a function?**
> All variables are **global** by default, including inside functions — unless declared as extra (unused) parameters in the function definition. By convention, extra parameters after the ones actually used are local variables, separated by extra whitespace for readability:
> ```awk
> function trim(s,    tmp) {   # 'tmp' is local because it's an unused extra param
>   tmp = s
>   gsub(/^ +| +$/, "", tmp)
>   return tmp
> }
> ```
> This is a quirk specific to awk — there's no explicit `local` keyword.

---

**Q15. What happens if you reference a variable that was never assigned?**
> awk does NOT raise an error. Uninitialized variables default to `""` (empty string) in string context and `0` in numeric context — simultaneously. This means a typo in a variable name (e.g., `totl` instead of `total`) silently produces wrong output instead of an error.
> ```bash
> awk '{ total += $1 } END { print totl }' file.txt   # prints empty/0, no warning!
> ```
> Use `gawk --lint` to catch this class of bug during development.

---

## String & Regex

**Q16 🔥 What is the difference between `sub()` and `gsub()`?**
> - `sub(regex, replacement [, target])` — replaces only the FIRST match in the target (default: `$0`)
> - `gsub(regex, replacement [, target])` — replaces ALL matches (global substitution)
>
> Both modify the target in place and return the number of substitutions made.
> ```bash
> echo "foo foo foo" | awk '{ sub(/foo/, "bar"); print }'    # bar foo foo
> echo "foo foo foo" | awk '{ gsub(/foo/, "bar"); print }'   # bar bar bar
> ```

---

**Q17. How do you extract a substring matching a regex pattern?**
> Use `match()` combined with `substr()`, `RSTART`, and `RLENGTH`:
> ```bash
> awk '{
>   if (match($0, /[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+/))
>     print substr($0, RSTART, RLENGTH)
> }' file.txt
> ```
> `match()` sets `RSTART` (1-indexed start position, or 0 if no match) and `RLENGTH` (length of match, or -1 if no match) as side effects.

---

**Q18 🔥 What's the difference between using a single-character FS like `.` versus a multi-character FS like `..`?**
> - **Single-character** `FS`: treated **literally**, even if it's a regex metacharacter. `awk -F. '...'` splits on a literal dot.
> - **Multi-character** `FS`: treated as a **regular expression**. `awk -F'..' '...'` means "split on any run of two arbitrary characters," not the literal string `".."`.
>
> To split on a literal multi-character string, escape regex metacharacters: `awk -F'\\.\\.' '...'`.

---

**Q19. How would you remove all non-alphanumeric characters from a line?**
> ```bash
> awk '{ gsub(/[^a-zA-Z0-9]/, ""); print }' file.txt
> ```

---

## Arrays

**Q20 🔥 What kind of data structure is an awk array?**
> An **associative array** (hash map) — keys can be any string (or number, which is converted to a string internally), and there's no fixed size or pre-declaration needed. There are no true integer-indexed arrays in standard awk; even `arr[1]` uses the string `"1"` as the key.

---

**Q21. How do you check if a key exists in an array without creating it?**
> Use the `in` operator — checking with `arr[key]` directly would CREATE the key with an empty value as a side effect:
> ```bash
> if ("alice" in arr) { print "exists" }     # ✅ safe, doesn't create the key
>
> if (arr["alice"] != "") { ... }            # ❌ this CREATES arr["alice"] = "" if it didn't exist!
> ```

---

**Q22 🔥 How would you deduplicate lines in a file using awk, preserving original order?**
> ```bash
> awk '!seen[$0]++' file.txt
> ```
> This is a classic awk idiom. For each line, `seen[$0]++` returns the PREVIOUS value of `seen[$0]` (0 if first occurrence) before incrementing. `!0` is true, so the line prints on first occurrence; subsequent occurrences have `seen[$0] >= 1`, so `!seen[$0]` is false and they're skipped. Unlike `sort -u`, this preserves the original line order.

---

**Q23. How do you iterate over an array, and is the order guaranteed?**
> ```awk
> for (key in arr) {
>   print key, arr[key]
> }
> ```
> The iteration order is **implementation-defined** — not guaranteed to be insertion order, numeric order, or alphabetical order in POSIX awk. `gawk` allows controlling the order via `PROCINFO["sorted_in"]`.

---

## Type Coercion

**Q24 🔥 Why does `awk 'BEGIN { print ("10" > "9") }'` print `0` (false)?**
> Because both operands are **string literals** in the program text, awk performs a **string comparison**: `"10" > "9"` compares character by character — `"1"` vs `"9"`, and `"1" < "9"` in ASCII, so the result is false.
>
> Contrast with values read from input, which are treated as "numeric strings" and compared numerically if they look like valid numbers:
> ```bash
> echo "10 9" | awk '{ print ($1 > $2) }'   # 1 (true) — numeric comparison
> ```
> The rule: hardcoded string literals in the program are always compared as strings; field/input values that look numeric are compared numerically.

---

**Q25. How do you force a numeric comparison or a string comparison explicitly?**
> ```bash
> # Force numeric: add zero
> awk '{ print ($1+0 > $2+0) }' file.txt
>
> # Force string: concatenate empty string
> awk '{ print ($1"" > $2"") }' file.txt
> ```

---

## Scenario-Based

**Q26 🔥 How would you sum the third column of a CSV file, skipping the header row?**
> ```bash
> awk -F, 'NR > 1 { sum += $3 } END { print sum }' data.csv
> ```

---

**Q27 🔥 How would you count occurrences of each unique value in the first column and print them sorted by frequency?**
> ```bash
> awk '{ count[$1]++ } END { for (k in count) print count[k], k }' file.txt | sort -rn
> ```
> awk builds the frequency table; `sort -rn` handles the final ordering since awk's array iteration order isn't guaranteed.

---

**Q28. How would you implement a SQL-like JOIN between two files on a common key using awk?**
> ```bash
> awk -F, '
>   NR==FNR { name[$1]=$2; next }   # process first file: build lookup table
>   { print $1, name[$1], $2 }      # process second file: use the lookup
> ' file1.csv file2.csv
> ```
> `NR==FNR` is true only while reading the FIRST file listed (because `NR` and `FNR` are equal only during that file). `next` skips to the next record without falling through, so the second block only executes for the second file.

---

**Q29 🔥 You need to print every line between two markers, "START" and "END", but NOT the marker lines themselves. How?**
> ```bash
> awk '/START/{flag=1;next} /END/{flag=0} flag' file.txt
> ```
> When `START` is found, set `flag=1` and `next` (skip printing this line). While `flag` is 1, the bare pattern `flag` (truthy) triggers the default action (`print`). When `END` is found, `flag` resets to 0 before the default print rule would fire for that line.

---

**Q30. How would you process a log file and alert if more than 100 errors occurred in any single hour?**
> ```bash
> awk '
>   {
>     split($0, parts, ":")
>     hour = parts[1]      # assuming hour is first colon-delimited field
>     if ($0 ~ /ERROR/) errors[hour]++
>   }
>   END {
>     for (h in errors)
>       if (errors[h] > 100)
>         print "ALERT: hour", h, "had", errors[h], "errors"
>   }
> ' app.log
> ```

---

**Q31 🔥 How do you make awk treat blank lines as record separators (i.e., process paragraphs instead of single lines)?**
> ```bash
> awk 'BEGIN{RS=""} { print "Record", NR ":", $0 }' file.txt
> ```
> Setting `RS=""` (empty string) is special-cased in awk to mean "paragraph mode" — records are separated by one or more blank lines, and within a record, `FS` additionally treats newlines as field separators (in addition to whatever `FS` is set to).

---

**Q32. How would you find the longest line in a file and print both its length and content?**
> ```bash
> awk '{ if (length($0) > max) { max = length($0); line = $0 } } END { print max, line }' file.txt
> ```

---

## Advanced & Internals

**Q33 🔥 Explain the `NR==FNR` idiom in detail — why does it work?**
> `NR` is the cumulative record count across all files; `FNR` is the record count within the current file only. While awk is reading the FIRST file given on the command line, both counters increase together and `NR == FNR` is always true. Once awk moves to the second file, `FNR` resets to 1 while `NR` keeps climbing from where it left off — so `NR == FNR` becomes false for the rest of the program.
>
> This makes `NR==FNR { ... ; next }` a reliable way to say "only do this while processing the first file" — typically used to build a lookup table from file 1 before processing file 2 against it.

---

**Q34. How does awk's automatic type detection distinguish a "numeric string" from a plain string?**
> A "numeric string" is a value that came from external input (a field, `getline`, `ARGV`, `ENVIRON`, or `split()` results) and looks like a valid number (optionally signed, with digits, decimal point, or exponent notation). Such values carry an internal flag (STRNUM in POSIX terminology) that makes them eligible for numeric comparison.
>
> Values created by string literals or string operations (concatenation, `substr()`, etc.) in the program are plain strings and are ALWAYS compared as strings, even if they happen to look numeric (`"10"`).

---

**Q35 🔥 Why might `awk '{ $1=$1 }1' file.txt` be used, and what does it actually do?**
> This is a common idiom to "normalize" whitespace. `$1=$1` doesn't change the value of field 1, but the act of ASSIGNING to any field forces awk to rebuild `$0` from the fields using `OFS`. Since the default `OFS` is a single space, this collapses any run of multiple spaces/tabs (the original `FS` default behavior) into single spaces in the output. The trailing `1` is awk shorthand for "always true → run the default action (print)."
> ```bash
> echo "a    b   c" | awk '{ $1=$1 }1'
> # Output: a b c   (squeezed to single spaces)
> ```

---

**Q36. What is the performance difference between `mawk` and `gawk`, and why does it matter for production scripts?**
> `mawk` is typically 2-5x faster than `gawk` for simple field-processing tasks because it uses a more lightweight bytecode interpreter with fewer features to check at runtime. `gawk` includes many extensions (locale handling, gensub, network access, profiling hooks) that add overhead even when unused.
>
> For very large files (gigabytes) processed in tight loops, this difference compounds significantly. Production log-processing pipelines on Debian/Ubuntu often benefit from the default `mawk`, while scripts requiring gawk-specific functions must explicitly invoke `gawk` and accept the performance tradeoff.

---

**Q37. Explain what happens internally when you do `print $0 > "file.txt"` inside a loop processing many records.**
> The file is opened in TRUNCATE mode on the FIRST execution of that statement (clearing any existing content), then kept open as a file descriptor for the remainder of the awk program's execution — subsequent writes APPEND to that same open descriptor rather than re-truncating. The file is only closed when awk exits or when you explicitly call `close("file.txt")`. This is why running the same redirect inside a per-record loop does not erase previous lines written during the same awk invocation — only the very first write of the run truncates the file.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
