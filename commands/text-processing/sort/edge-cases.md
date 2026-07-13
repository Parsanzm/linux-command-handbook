# sort — Edge Cases & Gotchas

> sort's defaults quietly assume text order, not numeric order — and locale,
> field ranges, and in-place redirection each hide their own sharp edges.

---

## Table of Contents

- [Default Sort Is Lexicographic, Not Numeric](#default-sort-is-lexicographic-not-numeric)
- [Forgetting the End of a Field Range in -k](#forgetting-the-end-of-a-field-range-in--k)
- [Locale Changes Sort Order Silently](#locale-changes-sort-order-silently)
- [sort file.txt > file.txt Destroys the File](#sort-filetxt--filetxt-destroys-the-file)
- [uniq Without Sorting First Misses Duplicates](#uniq-without-sorting-first-misses-duplicates)
- [Mixed Tabs and Spaces Break Field-Based Sorting](#mixed-tabs-and-spaces-break-field-based-sorting)
- [Trailing Whitespace Silently Affects Order](#trailing-whitespace-silently-affects-order)
- [Header Rows Get Sorted Into the Data](#header-rows-get-sorted-into-the-data)
- [Negative Numbers Sorted as Text](#negative-numbers-sorted-as-text)
- [Stability Assumptions Without -s](#stability-assumptions-without--s)
- [Sorting Files with Inconsistent Field Counts](#sorting-files-with-inconsistent-field-counts)
- [Very Large Files and Temp Directory Space](#very-large-files-and-temp-directory-space)
- [Sort Order Differing Between Linux and macOS/BSD](#sort-order-differing-between-linux-and-macosbsd)

---

## Default Sort Is Lexicographic, Not Numeric

### Plain sort treats numbers as TEXT, producing surprising order
```bash
printf "10\n2\n33\n4\n" | sort
# 10
# 2
# 33
# 4
# ⚠️ This IS correctly sorted — as TEXT/STRINGS. Character by character,
# '1' < '2' < '3' < '4', so "10" (starting with '1') sorts before "2"
# (starting with '2'), regardless of their actual numeric VALUE.

printf "10\n2\n33\n4\n" | sort -n
# 2
# 4
# 10
# 33
# ✅ Correct numeric order — ALWAYS use -n when sorting actual numbers,
# never rely on the default mode for anything numeric.
```

---

## Forgetting the End of a Field Range in -k

### `-k3n` sorts by field 3 THROUGH THE REST OF THE LINE, not just field 3
```bash
cat data.csv
# Alice,Engineering,50000,Senior
# Bob,Sales,45000,Junior
# Carol,Engineering,60000,Senior

sort -t',' -k3n data.csv
# ⚠️ Without specifying an END field, "-k3n" sorts using field 3 AND
# everything after it as the comparison key (field 3 through the end
# of the line) — this can produce subtly wrong results whenever
# multiple rows share the SAME field-3 value but differ afterward,
# since the trailing text unexpectedly participates in the comparison.

sort -t',' -k3,3n data.csv
# ✅ The "3,3" range explicitly means "ONLY field 3" — this is almost
# always what's actually intended when sorting by a single column,
# and the missing comma is one of sort's most common real-world mistakes.
```

---

## Locale Changes Sort Order Silently

### The exact same command can produce different results on different machines
```bash
echo -e "apple\nBanana\ncherry\nApple" | sort
# Under LC_ALL=C (strict byte/ASCII order):
# Apple
# Banana
# apple
# cherry
# (ALL uppercase letters sort before ALL lowercase letters, since
# uppercase ASCII codes are numerically lower)

LC_ALL=en_US.UTF-8 sort file.txt
# Under a typical locale-aware collation:
# apple
# Apple
# Banana
# cherry
# (case and accents are handled more "naturally," but the exact
# result depends entirely on the SPECIFIC locale rules in effect —
# NOT guaranteed to be identical across different systems/configurations)

# For scripts where DETERMINISTIC, REPRODUCIBLE order matters (e.g.,
# generating a file that will be diffed or hashed later), always pin
# the locale explicitly rather than relying on whatever the current
# environment happens to have configured:
LC_ALL=C sort file.txt
```

---

## sort file.txt > file.txt Destroys the File

### The classic shell redirection trap, sort included
```bash
sort data.txt > data.txt
cat data.txt
# (completely EMPTY, or severely truncated!)
# ⚠️ The shell opens data.txt for WRITING (truncating it to zero bytes)
# as part of setting up the ">" redirection, BEFORE sort ever gets a
# chance to actually READ the original content — by the time sort
# tries to read data.txt, it's already been wiped out.

sort -o data.txt data.txt
# ✅ SAFE — sort's own "-o" option is specifically designed to handle
# this exact "sort a file and save back to itself" case correctly,
# reading the ENTIRE input into memory first before writing any output.
```

---

## uniq Without Sorting First Misses Duplicates

### uniq only collapses ADJACENT identical lines — sort is a REQUIRED prerequisite
```bash
printf "banana\napple\nbanana\napple\n" | uniq
# banana
# apple
# banana
# apple
# ⚠️ NOTHING removed — no two ADJACENT lines are identical, even
# though "banana" and "apple" each appear twice overall in the file.

printf "banana\napple\nbanana\napple\n" | sort | uniq
# apple
# banana
# ✅ Sorting FIRST groups all identical lines together (adjacent),
# so uniq can then correctly identify and collapse them.
```

---

## Mixed Tabs and Spaces Break Field-Based Sorting

### -t expects a CONSISTENT separator — mixed delimiters silently misalign fields
```bash
cat data.txt
# Alice	Engineering    50000
# Bob    Sales	45000
# (some rows use TABS, others use SPACES between fields — a common
# result of manually-edited or poorly-generated data files)

sort -t$'\t' -k3,3n data.txt
# ⚠️ Rows using SPACES instead of the specified tab delimiter won't
# split into fields the way you expect — "field 3" for those rows
# might actually be a completely different chunk of text than intended,
# silently producing incorrect sort results with NO error message at all.

# Fix: normalize the delimiter FIRST, before sorting
cat data.txt | tr -s ' \t' '\t' | sort -t$'\t' -k3,3n
# tr -s squeezes runs of spaces/tabs into a SINGLE consistent tab,
# ensuring -t't' consistently applies across every row
```

---

## Trailing Whitespace Silently Affects Order

### Invisible trailing spaces change comparison results unexpectedly
```bash
printf "apple \napple\nbanana\n" | sort
# apple      ← has a TRAILING SPACE (invisible in most terminal displays)
# apple
# banana
# ⚠️ "apple " (with a trailing space) and "apple" (without) are
# DIFFERENT strings as far as sort is concerned — the space character
# has a specific byte value that participates in the comparison, even
# though visually both lines might look identical in a casual glance
# at terminal output.

# Reveal trailing whitespace explicitly before debugging sort order issues
cat -A file.txt | head
# apple $     ← the trailing "$" marker (from cat -A) reveals the
# apple$       actual end-of-line position, exposing the hidden space
```

---

## Header Rows Get Sorted Into the Data

### A naive sort on a CSV/TSV with a header row scrambles the header's position
```bash
cat data.csv
# name,score
# Bob,85
# Alice,92
# Carol,78

sort -t',' -k2,2n data.csv
# Bob,85
# name,score    ⚠️ the HEADER ROW got sorted right into the middle of
# Carol,78        the data, since "name" and "score" are just TEXT to
# Alice,92        sort — it has no inherent concept of "this row is special"

# Fix: separate the header, sort only the data rows, then reassemble
(head -n 1 data.csv && tail -n +2 data.csv | sort -t',' -k2,2n) > sorted.csv
```

---

## Negative Numbers Sorted as Text

### Even with -n, be aware of how sort interprets the negative sign
```bash
printf -- "-5\n3\n-10\n7\n" | sort -n
# -10
# -5
# 3
# 7
# ✅ Actually correct — GNU sort's -n DOES correctly recognize a
# LEADING minus sign as part of a negative number, producing proper
# numeric order. This is often assumed to be a gotcha, but modern GNU
# sort handles it correctly; the actual pitfall is forgetting -n
# entirely and getting lexicographic order instead:

printf -- "-5\n3\n-10\n7\n" | sort
# -10
# -5
# 3
# 7
# (in THIS particular example the text order happens to coincidentally
# match the numeric order, which can mask the -n mistake until tested
# against a case where it actually diverges — never rely on this
# coincidence; always use -n explicitly for numeric data)
```

---

## Stability Assumptions Without -s

### Relying on "preserved original order for ties" without explicitly requesting it
```bash
# Sorting by field 1 only, expecting rows with the SAME field-1 value
# to retain their ORIGINAL relative order (a common assumption)
sort -k1,1 data.txt
# By default, sort MAY use additional criteria (often the whole line)
# as a tiebreaker for equal keys, which can shuffle "tied" rows into
# an order different from their original position in the file —
# this is NOT guaranteed to preserve original order without -s.

sort -s -k1,1 data.txt
# ✅ -s explicitly disables the "whole line" fallback tiebreak,
# guaranteeing that rows with EQUAL field-1 values retain their
# ORIGINAL relative order from the input — essential when a
# meaningful secondary order (e.g., a PREVIOUS sort pass) must survive.
```

---

## Sorting Files with Inconsistent Field Counts

### -k referencing a field that doesn't exist on every line behaves unpredictably
```bash
cat data.txt
# Alice,Engineering,50000
# Bob,Sales
# Carol,Engineering,60000
# (Bob's row is MISSING the 3rd field entirely — inconsistent data)

sort -t',' -k3,3n data.txt
# Rows missing field 3 entirely are typically treated as having an
# EMPTY value for that field, which sorts before any actual number —
# but this can silently mask a DATA QUALITY problem (missing fields)
# rather than raising any visible error, since sort has no concept of
# "this row is malformed" — it just does its best with whatever's there.

# Always validate field counts BEFORE sorting, if data quality matters:
awk -F',' 'NF != 3 {print "Malformed row: " $0}' data.txt
```

---

## Very Large Files and Temp Directory Space

### External sorting needs disk space for temporary files, which can silently fail
```bash
sort huge_100gb_file.txt > sorted.txt
# sort: write failed: /tmp/sortXXXXXX: No space left on device
# ⚠️ GNU sort's external merge-sort strategy writes TEMPORARY sorted
# chunks to disk (typically under /tmp by default) — if /tmp is on a
# SMALL partition (common on systems where /tmp is a separate,
# limited-size filesystem, or a tmpfs backed by limited RAM), sorting
# a very large file can exhaust that space entirely, even though the
# FINAL output destination might have plenty of room available.

# Fix: redirect sort's temp file location to a filesystem with
# adequate free space
sort --temporary-directory=/mnt/large_disk/tmp huge_100gb_file.txt > sorted.txt
df -h /tmp /mnt/large_disk    # verify available space BEFORE a large sort
```

---

## Sort Order Differing Between Linux and macOS/BSD

### GNU sort and BSD sort don't always agree on edge-case behavior
```bash
# GNU sort (Linux default) and BSD sort (macOS default) both broadly
# follow POSIX, but differ in some GNU-specific EXTENSIONS and subtle
# default behaviors — most notably:
# - GNU sort's -h (human-numeric), -V (version-sort), -R (random),
#   and --parallel flags are GNU EXTENSIONS, not present in BSD sort at all
# - Locale-based collation defaults and exact whitespace-handling
#   edge cases can differ subtly between the two implementations

sort -h sizes.txt
# Works on Linux (GNU sort)
# sort: illegal option -- h   ← FAILS on macOS/BSD sort, unless GNU
# coreutils were separately installed (e.g., via Homebrew, commonly
# aliased as `gsort` to avoid clashing with the system's own BSD sort)

# Portable scripts targeting BOTH platforms should avoid GNU-specific
# extensions, or explicitly check for/require GNU coreutils (gsort) first
which gsort >/dev/null 2>&1 && SORT_CMD=gsort || SORT_CMD=sort
$SORT_CMD -h sizes.txt
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
