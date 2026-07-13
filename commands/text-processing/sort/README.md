# sort — The Complete Reference

> **Order lines of text — alphabetically, numerically, by field, or by any custom rule**
> Present since the earliest Unix releases (1971), one of the original coreutils.
> The quiet workhorse behind countless pipelines — `sort | uniq`, `sort | head`, `sort -k` reports.

---

## Table of Contents

- [What is sort?](#what-is-sort)
- [Where does sort live?](#where-does-sort-live)
- [How sort works internally](#how-sort-works-internally)
- [Syntax](#syntax)
- [Basic Sorting Modes](#basic-sorting-modes)
- [Sorting by Field/Column (-k)](#sorting-by-fieldcolumn--k)
- [All Key Options](#all-key-options)
- [Locale and Sort Order (LC_COLLATE)](#locale-and-sort-order-lc_collate)
- [Stable Sort and Ties](#stable-sort-and-ties)
- [sort with Large Files (External Sorting)](#sort-with-large-files-external-sorting)
- [Combining sort with uniq](#combining-sort-with-uniq)
- [sort vs Other Tools](#sort-vs-other-tools)
- [Related Commands](#related-commands)

---

## What is sort?

`sort` reads lines of text (from a file or standard input) and outputs them **rearranged in order** according to a specified comparison rule — alphabetical by default, but configurable to numeric, reverse, by a specific field/column, case-insensitive, and more. It's one of the most frequently used building blocks in Unix text-processing pipelines, almost always appearing right before `uniq` (which requires sorted input to work correctly) or as the final step before `head`/`tail` to find extremes.

```bash
sort names.txt                    # alphabetical order
sort -n numbers.txt               # numeric order
sort -r names.txt                 # reverse order
cat data.csv | sort -t',' -k2,2   # sort by the 2nd comma-separated field
```

**Why sort matters as a distinct, general-purpose tool:** implementing a correct, efficient sorting algorithm — especially one that handles arbitrarily large files, custom comparison rules, locale-aware collation, and stability guarantees — is genuinely nontrivial. Having one well-tested, standard implementation that every other tool and script can rely on (rather than every script author reinventing sorting logic) is a core piece of the Unix philosophy of composable, reusable tools.

---

## Where does sort live?

```
/usr/bin/sort
```

```bash
which sort
sort --version
# sort (GNU coreutils) 9.4
```

Part of **GNU coreutils** on Linux; POSIX-mandated, so a `sort` command (with at least the baseline options) exists on essentially every Unix-like system, including macOS/BSD — though, like many coreutils, GNU sort has additional extensions beyond the POSIX baseline (see option tables below).

---

## How sort works internally

### The general algorithm

For input that fits comfortably in memory, `sort` typically uses an efficient **comparison-based sort** (commonly a merge sort variant) to order all lines according to the active comparison rule, then writes the result to output. Sort **never** modifies the input file directly (unless `-o` targets the same file explicitly) — by default it reads from files/stdin and writes the sorted result to stdout.

```bash
sort file.txt
# reads file.txt fully, sorts in memory, prints result to stdout —
# file.txt itself remains COMPLETELY unchanged unless you explicitly
# redirect/overwrite it yourself
sort file.txt > sorted.txt              # save to a NEW file
sort -o file.txt file.txt               # sort AND overwrite the SAME file safely
                                          # (see edge-cases.md for why plain
                                          # ">" onto the same file is dangerous)
```

### External sorting for files too large for memory

When input is larger than sort can comfortably hold in memory, GNU sort automatically switches to an **external sort** strategy: it splits the input into smaller chunks that DO fit in memory, sorts each chunk independently, writes those sorted chunks as temporary files to disk, then performs a **merge** pass combining the sorted chunks into the final fully-sorted output — the same fundamental technique used by database engines and large-scale data processing systems for sorting datasets far bigger than available RAM.

```bash
# sort handles this transparently — no special flag needed for GNU sort
# to switch strategies based on input size, though you CAN tune the
# memory budget and temp directory used during this process:
sort --buffer-size=1G --temporary-directory=/mnt/scratch huge_file.txt
```

### Comparison order — byte-by-byte vs locale-aware

By default, sort's comparison depends on your active **locale** (`$LC_COLLATE`/`$LANG`), which can produce different, sometimes surprising results compared to a strict byte-value comparison — this is one of sort's most common sources of "why isn't this sorted the way I expect" confusion (see [Locale and Sort Order](#locale-and-sort-order-lc_collate) below).

---

## Syntax

```bash
sort [OPTIONS] [FILE...]
command | sort [OPTIONS]
```

```bash
sort file.txt                   # sort a file, print to stdout
sort file1.txt file2.txt        # sort the COMBINED content of multiple files together
cat file.txt | sort              # sort piped input
sort -o sorted.txt file.txt      # sort and write directly to a specific output file
```

---

## Basic Sorting Modes

```bash
# Default: lexicographic (alphabetical-ish, locale-dependent) order
sort names.txt

# Numeric sort — CRITICAL for numbers, since default sort treats them
# as TEXT, producing wildly wrong-looking order otherwise
sort numbers.txt
# 10
# 2
# 33
# 4
# ⚠️ this is TEXT/lexicographic order ("10" sorts before "2" because
# '1' < '2' as CHARACTERS), almost never what's wanted for actual numbers

sort -n numbers.txt
# 2
# 4
# 10
# 33
# ✅ correct NUMERIC order

# Reverse order (combine with any other mode)
sort -r names.txt
sort -rn numbers.txt

# Human-readable numeric sort — understands suffixes like K, M, G
echo -e "10K\n2M\n500\n1G" | sort -h
# 500
# 10K
# 2M
# 1G

# Sort by month name (Jan, Feb, Mar... in CALENDAR order, not alphabetical)
echo -e "Mar\nJan\nDec\nFeb" | sort -M
# Jan
# Feb
# Mar
# Dec

# Version sort — handles version-number-like strings sensibly
echo -e "file10.txt\nfile2.txt\nfile1.txt" | sort -V
# file1.txt
# file2.txt
# file10.txt
# (plain sort would incorrectly put file10.txt BEFORE file2.txt)

# Case-insensitive sort
sort -f names.txt

# Random order (shuffle) — GNU extension
sort -R names.txt

# Remove duplicate lines WHILE sorting, in a single pass
sort -u names.txt
```

---

## Sorting by Field/Column (-k)

```bash
# Sort a space-delimited file by the 2nd column
sort -k2 data.txt

# Sort a CSV by the 3rd comma-separated field
sort -t',' -k3 data.csv

# Sort NUMERICALLY by a specific field
sort -t',' -k3,3n data.csv
# The "3,3" range means "field 3 ONLY" — without the explicit end
# field, -k3n would sort by field 3 THROUGH THE END OF THE LINE,
# not just that single field (a very common mistake, see edge-cases.md)

# Sort by MULTIPLE fields — first by field 2, THEN by field 1 as a tiebreaker
sort -t',' -k2,2 -k1,1 data.csv

# Sort by a field in reverse
sort -t',' -k3,3nr data.csv

# Sort by a SPECIFIC character range within a field (advanced)
sort -k1.3,1.5 data.txt
# Sorts using only characters 3 through 5 of field 1

# Real-world example: sort an /etc/passwd-style file by UID (field 3)
sort -t':' -k3,3n /etc/passwd
```

---

## All Key Options

| Option | Long | Description |
|--------|------|--------------|
| `-n` | `--numeric-sort` | Compare as numbers, not text |
| `-r` | `--reverse` | Reverse the sort order |
| `-f` | `--ignore-case` | Case-insensitive comparison |
| `-u` | `--unique` | Output only the first of any group of equal lines |
| `-k POS` | `--key=POS` | Sort by a specific field/column |
| `-t SEP` | `--field-separator=SEP` | Specify the field delimiter (default: whitespace) |
| `-h` | `--human-numeric-sort` | Understand suffixes like K, M, G (e.g., "2K" < "1M") |
| `-M` | `--month-sort` | Sort by three-letter month abbreviation, calendar order |
| `-V` | `--version-sort` | Natural sort for version-like strings ("file2" before "file10") |
| `-R` | `--random-sort` | Shuffle lines into random order |
| `-c` | `--check` | Check whether input is ALREADY sorted, without producing output |
| `-s` | `--stable` | Disable last-resort tiebreaking, guaranteeing stable order for equal keys |
| `-o FILE` | `--output=FILE` | Write result to FILE (safe even if FILE is one of the inputs) |
| `-b` | `--ignore-leading-blanks` | Ignore leading whitespace when comparing |
| `-z` | `--zero-terminated` | Use NUL instead of newline as the line terminator (pairs with `find -print0`) |
| `--parallel=N` | | Use N threads for sorting (GNU extension, speeds up large sorts) |
| `--buffer-size=SIZE` | | Set the memory buffer size used before spilling to disk |

```bash
sort -n file.txt                    # numeric
sort -k2,2 -t',' file.csv           # by field 2, comma-delimited
sort -c file.txt && echo "Already sorted"
sort -u file.txt                     # sort AND deduplicate in one pass
sort --parallel=4 huge_file.txt      # use 4 threads for speed on large files
```

---

## Locale and Sort Order (LC_COLLATE)

### Why the SAME sort command can produce DIFFERENT results on different systems
```bash
echo -e "banana\nApple\ncherry" | sort
# Under the "C" / "POSIX" locale (strict byte-value comparison):
# Apple
# banana
# cherry
# (uppercase letters sort BEFORE all lowercase letters in ASCII/byte order)

# Under many locales like en_US.UTF-8 (locale-aware "natural" comparison):
LC_ALL=en_US.UTF-8 sort names.txt
# Apple
# banana
# cherry
# (may LOOK the same here, but locale-aware collation can differ
# significantly for punctuation, accented characters, and case
# handling in less trivial examples)

# Force STRICT byte-value ordering, ignoring locale entirely — often
# the safer, more PREDICTABLE choice for scripts, especially ones that
# might run on different systems with different locale configurations
LC_ALL=C sort file.txt
```

### Practical impact: scripts behaving differently across machines
```bash
# A script tested and working correctly on a machine with LC_ALL=C
# might produce SUBTLY different sort order on a colleague's machine
# with a different locale configured — especially noticeable with:
# - Mixed case sorting (does "Apple" come before or after "banana"?)
# - Punctuation and special characters
# - Accented characters (é, ñ, ü, etc.)

# Best practice for REPRODUCIBLE scripts: explicitly force the locale
export LC_ALL=C
sort file.txt
# or, scoped to just this one command:
LC_ALL=C sort file.txt
```

---

## Stable Sort and Ties

```bash
# By default, GNU sort's underlying algorithm is EFFECTIVELY stable in
# most practical cases, but explicitly requesting -s guarantees it and
# also disables an internal "last resort" comparison that would
# otherwise use the ENTIRE line as an additional tiebreaker
sort -s -k1,1 data.txt
# Guarantees: for lines with an IDENTICAL field 1, their RELATIVE
# ORIGINAL ORDER is preserved in the output — useful when sorting by
# one field but wanting a SECONDARY, already-meaningful order (like
# original file order, or a previous sort pass) to remain intact for ties

# Without -s, sort may use additional criteria (typically the whole
# line) to break ties between otherwise-equal keys, which can produce
# a DIFFERENT relative order for tied entries than -s would guarantee
sort -k1,1 data.txt
```

---

## sort with Large Files (External Sorting)

```bash
# GNU sort handles files far larger than available RAM automatically,
# using temporary files on disk during a multi-pass merge sort
sort huge_50gb_file.txt > sorted_output.txt
# No special flag required — sort detects when input exceeds a
# reasonable in-memory threshold and switches strategy transparently

# Control WHERE temporary files are written (important if /tmp is
# small or on a slow disk, common in constrained environments)
sort --temporary-directory=/mnt/large_scratch_disk huge_file.txt

# Control how much memory sort uses BEFORE spilling to temp files
sort --buffer-size=2G huge_file.txt

# Speed up sorting large files using multiple threads
sort --parallel=8 huge_file.txt

# Clean up if sort is interrupted mid-run (leftover temp files)
ls /tmp/sort*    # GNU sort's temp files are typically named sortXXXXXX
```

---

## Combining sort with uniq

```bash
# THE single most common sort pairing: uniq REQUIRES sorted input,
# since it only collapses ADJACENT identical lines
cat names.txt | sort | uniq

# Count occurrences of each unique line
cat names.txt | sort | uniq -c

# Sort BY that count, descending — an extremely common reporting pattern
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn

# Find lines that appear MORE than once (true duplicates)
cat file.txt | sort | uniq -d

# Find lines that are UNIQUE to the file (appear exactly once)
cat file.txt | sort | uniq -u
```

---

## sort vs Other Tools

| Feature | `sort` | `awk` (with custom logic) | Database `ORDER BY` |
|---------|--------|------------------------------|--------------------------|
| Purpose | General-purpose line ordering | Can sort, but requires writing custom comparison logic | Ordering query RESULTS, not arbitrary text files |
| Handles files larger than RAM | ✅ Yes, automatically (external merge sort) | ⚠️ Depends entirely on implementation | ✅ Yes, database-managed |
| Field-based sorting | ✅ Via `-k`/`-t` | ✅ Native field awareness (`$1`, `$2`) | ✅ Native column awareness |
| Ease of use for simple cases | ✅ Very simple (`sort file.txt`) | ⚠️ More verbose for basic sorting | N/A — different context entirely |
| Best for | Quick, general text/data sorting in a pipeline | Complex custom sort logic combined with other processing | Sorting structured, already-loaded database records |

```bash
sort -t',' -k3n data.csv                    # simple field-based numeric sort
awk -F',' '{print $3, $0}' data.csv | sort -n | cut -d' ' -f2-   # awk + sort combo for complex cases
```

---

## Related Commands

| Command | Relation |
|---------|----------|
| `uniq` | Requires sorted input; almost always paired directly after sort |
| `comm` | Compares two ALREADY-SORTED files line by line |
| `join` | Joins two files on a common field; both inputs must be sorted first |
| `cut` | Often used before/after sort to extract the specific field being sorted on |
| `awk` | Alternative or complement for more complex field-based logic beyond simple sorting |
| `shuf` | Dedicated random-shuffling tool (an alternative to `sort -R` in some contexts) |
| `tsort` | Topological sort — a fundamentally DIFFERENT kind of ordering (dependency resolution, not lexical/numeric) |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
