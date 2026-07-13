# sort — Practical Examples

> Real-world patterns for reports, log analysis, CSV processing, and data cleanup.

---

## Table of Contents

- [Basic Sorting](#basic-sorting)
- [Numeric and Special Sorting Modes](#numeric-and-special-sorting-modes)
- [Sorting by Field/Column](#sorting-by-fieldcolumn)
- [Sorting CSV Files](#sorting-csv-files)
- [Combining sort with uniq for Reports](#combining-sort-with-uniq-for-reports)
- [Sorting Log Files](#sorting-log-files)
- [Multi-Key Sorting](#multi-key-sorting)
- [In-Place Sorting](#in-place-sorting)
- [Checking If Data Is Already Sorted](#checking-if-data-is-already-sorted)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Sorting

```bash
# Alphabetical sort of a file
sort names.txt

# Sort piped input
cat names.txt | sort

# Reverse alphabetical order
sort -r names.txt

# Sort multiple files together as one combined, sorted stream
sort file1.txt file2.txt file3.txt

# Case-insensitive sort
sort -f mixed_case_names.txt

# Sort and remove duplicates in one pass
sort -u names.txt
```

---

## Numeric and Special Sorting Modes

```bash
# Numeric sort — essential for actual numbers
sort -n scores.txt

# Reverse numeric (highest first)
sort -rn scores.txt

# Human-readable sizes (K, M, G suffixes) — great for du output
du -sh */ | sort -h

# Version-aware sort — handles "file2" before "file10" correctly
ls | sort -V

# Sort month abbreviations in calendar order, not alphabetical
echo -e "Mar\nJan\nDec\nFeb" | sort -M

# Shuffle into random order (GNU extension)
sort -R playlist.txt
```

---

## Sorting by Field/Column

```bash
# Sort a space-separated file by the 2nd column
sort -k2 data.txt

# Sort by the 2nd column, numerically
sort -k2,2n data.txt

# Sort by the LAST field (using a negative-like trick isn't supported
# directly, but this is the common workaround for "sort by last column")
awk '{print $NF, $0}' data.txt | sort -k1,1 | cut -d' ' -f2-

# Sort a tab-separated file by the 3rd field
sort -t$'\t' -k3,3 data.tsv

# Sort ignoring leading whitespace
sort -b indented_data.txt
```

---

## Sorting CSV Files

```bash
# Sort a CSV by its 2nd column (alphabetically)
sort -t',' -k2,2 data.csv

# Sort a CSV by its 3rd column, NUMERICALLY
sort -t',' -k3,3n data.csv

# Sort a CSV, skipping the header row (common real-world need)
(head -n 1 data.csv && tail -n +2 data.csv | sort -t',' -k2,2) > sorted.csv

# Sort by 2nd column, THEN by 1st column as a tiebreaker
sort -t',' -k2,2 -k1,1 data.csv

# Sort descending by a numeric column (e.g., highest sales first)
sort -t',' -k4,4nr sales.csv
```

---

## Combining sort with uniq for Reports

```bash
# Classic word/value frequency report
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn

# Top 10 most frequent values
cat data.txt | sort | uniq -c | sort -rn | head -10

# Find values that appear EXACTLY once
cat file.txt | sort | uniq -u

# Find TRUE duplicates (appearing 2+ times)
cat file.txt | sort | uniq -d

# Deduplicate a mailing list, case-insensitively
cat emails.txt | sort -f | uniq -i
```

---

## Sorting Log Files

```bash
# Sort log entries chronologically (assuming ISO-style timestamps at
# the start of each line, which sort naturally as text)
sort access.log

# Sort by response time (assuming it's a specific whitespace-delimited field)
cat access.log | awk '{print $NF, $0}' | sort -rn | cut -d' ' -f2- | head -20

# Extract and sort unique IP addresses by request count
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn

# Sort multiple rotated log files together into one chronological stream
sort app.log app.log.1 app.log.2 > combined_sorted.log

# Sort log entries by HTTP status code
cat access.log | awk '{print $9, $0}' | sort -n | cut -d' ' -f2-
```

---

## Multi-Key Sorting

```bash
# Sort by department (field 1), then by salary descending (field 3) within each department
sort -t',' -k1,1 -k3,3nr employees.csv

# Sort by year, then month, then day (assuming separate fields)
sort -t',' -k1,1n -k2,2n -k3,3n dates.csv

# Sort a gradebook: by class (field 1), then by score descending (field 4)
sort -t',' -k1,1 -k4,4nr gradebook.csv

# Three-level sort: region, then city, then population descending
sort -t',' -k1,1 -k2,2 -k3,3nr cities.csv
```

---

## In-Place Sorting

```bash
# Safely sort a file AND overwrite it (unlike using ">" onto the same file)
sort -o data.txt data.txt

# Sort and deduplicate a file in place
sort -u -o data.txt data.txt

# Always consider a backup before in-place operations on important data
cp data.txt data.txt.bak
sort -o data.txt data.txt
```

---

## Checking If Data Is Already Sorted

```bash
# Silently check — exit code 0 if sorted, non-zero if not
sort -c data.txt
echo $?
# 0 = already sorted, 1 = NOT sorted

# Verbose check, showing WHERE the first out-of-order line is
sort -c data.txt
# sort: data.txt:15: disorder: some_line_here

# Use in a script to validate an assumption before running something
# that REQUIRES sorted input (like uniq or join)
if sort -c data.txt 2>/dev/null; then
  uniq data.txt
else
  echo "Input isn't sorted — sorting first"
  sort data.txt | uniq
fi
```

---

## Real-World Recipes

```bash
# --- Top 10 Largest Files in a Directory ---

find . -type f -exec du -h {} \; | sort -rh | head -10

# --- Most Frequent Visitors from an Access Log ---

cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10

# --- Sorting a Gradebook by Score, Highest First ---

sort -t',' -k3,3nr gradebook.csv

# --- Deduplicating and Alphabetizing a Contact List ---

cat contacts.txt | sort -f | uniq -i > clean_contacts.txt

# --- Merging and Sorting Multiple Sales Reports Into One ---

sort -t',' -k1,1 -k4,4nr jan_sales.csv feb_sales.csv mar_sales.csv > q1_sorted.csv

# --- Finding the Oldest and Newest Files by Modification Time ---

find . -type f -printf '%T@ %p\n' | sort -n | head -1    # oldest
find . -type f -printf '%T@ %p\n' | sort -rn | head -1   # newest

# --- Sorting /etc/passwd by UID ---

sort -t':' -k3,3n /etc/passwd

# --- Preparing Two Files for a join Operation (both must be sorted) ---

sort -t',' -k1,1 file1.csv -o file1_sorted.csv
sort -t',' -k1,1 file2.csv -o file2_sorted.csv
join -t',' -1 1 -2 1 file1_sorted.csv file2_sorted.csv

# --- Quick Frequency Report of HTTP Status Codes ---

cat access.log | awk '{print $9}' | sort | uniq -c | sort -rn
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
