# sort — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [Field-Based Sorting](#field-based-sorting)
- [Locale & Collation](#locale--collation)
- [Internals & Large Files](#internals--large-files)
- [Stability](#stability)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does the sort command do by default, and what does "default order" actually mean?**
> `sort` reads lines of input and outputs them rearranged in order — by default, using **lexicographic (text/character-based) comparison**, not numeric value. This means numbers are compared character by character, so `"10"` sorts before `"2"` under the default mode, since `'1'` is a "smaller" character than `'2'`.

---

**Q2 🔥 Why is sort considered essential to pair with uniq, and what happens if you skip it?**
> `uniq` only removes **adjacent** duplicate lines — it has no awareness of duplicates elsewhere in the file. Since real-world data is rarely already grouped by identical value, `sort` must run first to bring all identical lines next to each other; without it, `uniq` silently fails to catch duplicates that aren't already adjacent.

---

## Basic Usage

**Q3 🔥 What's the difference between `sort file.txt` and `sort -n file.txt`?**
> Plain `sort` compares lines as **text**, character by character. `-n` tells sort to interpret and compare the content as **numbers** instead — essential for correct ordering when the data actually represents numeric values, since text comparison of numbers produces nonsensical order (e.g., "10" before "2").

---

**Q4. How would you sort a file in reverse numeric order, showing the largest values first?**
> ```bash
> sort -rn file.txt
> ```
> `-r` reverses whatever comparison mode is active; combined with `-n`, this produces descending numeric order.

---

**Q5 🔥 What does `sort -u` do, and how is it different from `sort | uniq`?**
> `sort -u` sorts the input and removes duplicate lines in a **single pass**, functionally equivalent to piping through `uniq` afterward but slightly more efficient since it's handled internally by sort itself rather than requiring a separate process and an extra pass over the data.

---

## Field-Based Sorting

**Q6 🔥 How do you sort a CSV file by its 3rd column, numerically?**
> ```bash
> sort -t',' -k3,3n data.csv
> ```
> `-t','` sets the field delimiter to a comma, and `-k3,3n` sorts using field 3 as the key (numeric comparison), restricted to that field only.

---

**Q7. What's the difference between `-k3n` and `-k3,3n`, and why does this distinction matter in practice?**
> `-k3n` sorts using field 3 **through the rest of the line** as the comparison key (since no end field was specified), while `-k3,3n` restricts the comparison to **only** field 3. This distinction matters because omitting the end field is a common mistake that can produce subtly incorrect results whenever multiple rows share the same field-3 value but differ in subsequent fields, since that trailing text unexpectedly participates in the comparison.

---

**Q8 🔥 How would you sort by multiple fields, e.g., by department first and then by salary descending within each department?**
> ```bash
> sort -t',' -k1,1 -k3,3nr employees.csv
> ```
> Multiple `-k` options are applied in order — sort uses the first key, and only for entries that are equal under that key does it consult the second key as a tiebreaker.

---

## Locale & Collation

**Q9 🔥 Why might the exact same `sort` command produce different output on two different machines?**
> Sort's default comparison is **locale-aware**, governed by `$LC_COLLATE`/`$LANG`. Different locale configurations can apply different collation rules — affecting how uppercase/lowercase letters, punctuation, and accented characters are ordered relative to each other — so identical input can produce genuinely different sorted output depending on the active locale, even using the exact same sort command.

---

**Q10. How would you force strict, locale-independent (pure byte-value) sort order for reproducibility across different systems?**
> ```bash
> LC_ALL=C sort file.txt
> ```
> This forces the "C"/POSIX locale, which uses strict byte-value comparison rather than any locale-specific collation rules, ensuring identical, predictable output regardless of what locale is configured on the machine actually running the command.

---

## Internals & Large Files

**Q11 🔥 How does sort handle a file too large to fit entirely in memory?**
> GNU sort automatically switches to an **external sort** strategy: it splits the input into smaller chunks that fit in memory, sorts each chunk independently, writes the sorted chunks as temporary files to disk, then performs a merge pass combining them into the final fully-sorted output — the same fundamental approach used by database systems for sorting datasets larger than available RAM.

---

**Q12. What options control sort's behavior when processing very large files?**
> `--buffer-size` controls how much memory is used before spilling to temporary disk files; `--temporary-directory` controls where those temporary files are written (important if the default location, often `/tmp`, has limited space); `--parallel` allows sort to use multiple threads to speed up the sorting process on large inputs.

---

## Stability

**Q13 🔥 What does it mean for a sort to be "stable," and why does this matter?**
> A stable sort guarantees that when two lines are considered equal under the active sort key, their **relative order from the original input is preserved** in the output rather than being rearranged arbitrarily. This matters when a meaningful secondary ordering (e.g., original file order, or the result of a previous sort pass) needs to survive intact for entries that tie on the primary key.

---

**Q14. Does GNU sort guarantee stability by default, or does it require an explicit option?**
> Without `-s`, GNU sort may use additional criteria (often the entire line) as a fallback tiebreaker for equal keys, which can shuffle tied entries away from their original relative order. Explicitly passing `-s` (`--stable`) disables that fallback tiebreak, guaranteeing that equal-key lines retain their original relative order.

---

## Scenario-Based

**Q15 🔥 A script runs `sort data.txt > data.txt` intending to sort a file in place, but afterward the file is empty. What happened, and what's the correct way to do this?**
> The shell sets up the `>` redirection by opening `data.txt` for writing (truncating it to zero bytes) **before** `sort` ever gets a chance to read the original content — by the time sort tries to read the file, it's already been wiped out. The correct, safe approach is sort's own `-o` option, specifically designed for this case: `sort -o data.txt data.txt`, which reads the entire input into memory before writing any output back to the same file.

---

**Q16. A developer sorts a CSV file by a numeric column using `sort -t',' -k3n data.csv`, but notices some rows with the same field-3 value aren't grouped together the way expected, with unrelated trailing data seemingly affecting the order. What's the bug?**
> They omitted the end of the field range — `-k3n` sorts using field 3 **and everything after it** on the line as the comparison key, not field 3 alone. The fix is specifying an explicit end field: `-k3,3n`, restricting the comparison strictly to that single field.

---

**Q17 🔥 A CSV file with a header row is sorted directly with `sort -t',' -k2,2n data.csv`, and the header ends up somewhere in the middle of the sorted output instead of staying at the top. Why, and how do you fix it?**
> `sort` has no inherent concept of "this row is a header" — it treats the header row as just another line of text to be compared and positioned according to the sort key, like any other row. The fix is separating the header before sorting and reattaching it afterward: `(head -n 1 data.csv && tail -n +2 data.csv | sort -t',' -k2,2n) > sorted.csv`.

---

**Q18. Two files need to be joined using the `join` command, but `join` produces incomplete or incorrect results even though both files appear to contain matching keys. What prerequisite might be missing?**
> `join` requires both input files to already be **sorted** on the join field — if either file isn't sorted (or is sorted differently, e.g., under a different locale/collation), `join` can silently miss matches or produce incorrect output, since its merge-based matching algorithm assumes sorted order going in. Sorting both files explicitly on the join key beforehand (`sort -k1,1 file1.txt -o file1_sorted.txt`, and the same for the second file) resolves this.

---

**Q19 🔥 A team's script produces different sorted output on a colleague's machine than on the original developer's machine, despite using identical input data and the identical sort command. What's the most likely cause, and what's the fix?**
> The two machines almost certainly have different locale configurations affecting sort's collation rules (`$LC_COLLATE`/`$LANG`), causing the same lexicographic sort to order case, punctuation, or accented characters differently between environments. The fix is explicitly forcing a consistent locale in the script itself, most commonly `LC_ALL=C sort file.txt`, guaranteeing identical, reproducible byte-value ordering regardless of what locale happens to be configured on whichever machine runs it.

---

**Q20. Sorting an enormous file fails partway through with a "No space left on device" error, even though the final output's destination disk clearly has plenty of free space. What's actually happening?**
> GNU sort's external sort strategy writes **temporary sorted chunks** to disk during processing (typically under `/tmp` by default) before merging them into the final output — if that temporary directory's filesystem is small or otherwise constrained, it can run out of space entirely, independent of how much room exists at the final output's actual destination. The fix is redirecting sort's temporary file location to a filesystem with sufficient free space using `--temporary-directory=/path/to/larger/disk`.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
