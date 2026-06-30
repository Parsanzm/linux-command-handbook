# awk — Practical Examples

> Real-world patterns from log analysis, data processing, sysadmin, and reporting.

---

## Table of Contents

- [Field Extraction](#field-extraction)
- [Filtering Rows](#filtering-rows)
- [Reformatting Output](#reformatting-output)
- [Counting & Aggregation](#counting--aggregation)
- [Working with CSV](#working-with-csv)
- [Multiple Files](#multiple-files)
- [Custom Field & Record Separators](#custom-field--record-separators)
- [String Manipulation](#string-manipulation)
- [Math & Calculations](#math--calculations)
- [Log Analysis](#log-analysis)
- [System Administration](#system-administration)
- [Arrays in Practice](#arrays-in-practice)
- [Two-File Patterns (Join-like)](#two-file-patterns-join-like)
- [One-Liners Cheat Sheet](#one-liners-cheat-sheet)

---

## Field Extraction

```bash
# Print first field (like cut -f1)
awk '{ print $1 }' file.txt

# Print specific fields
awk '{ print $1, $3 }' file.txt
awk '{ print $2, $1 }' file.txt    # reorder fields

# Print last field
awk '{ print $NF }' file.txt

# Print second-to-last field
awk '{ print $(NF-1) }' file.txt

# Print all fields except first
awk '{ $1=""; print }' file.txt    # leaves a leading space
awk '{ for(i=2;i<=NF;i++) printf "%s ", $i; print "" }' file.txt  # clean

# Print number of fields per line
awk '{ print NF, $0 }' file.txt

# Print lines with exactly N fields
awk 'NF == 3' file.txt

# Print lines with more than N fields
awk 'NF > 5' file.txt
```

---

## Filtering Rows

```bash
# Lines matching a pattern (like grep)
awk '/error/' file.txt
awk '/^ERROR/' file.txt              # starts with ERROR
awk '!/^#/' file.txt                 # exclude comments

# Lines where a field matches a condition
awk '$3 > 100' file.txt              # third field > 100
awk '$1 == "alice"' file.txt         # first field equals alice
awk '$2 ~ /^192\./' file.txt         # second field starts with 192.

# Multiple conditions
awk '$3 > 100 && $4 < 50' file.txt
awk '$1 == "ERROR" || $1 == "FATAL"' file.txt

# Line number ranges
awk 'NR >= 10 && NR <= 20' file.txt  # lines 10-20
awk 'NR == 5' file.txt               # only line 5
awk 'NR > 1' file.txt                # skip header (all but first line)
awk 'NR % 2 == 0' file.txt           # even lines only
awk 'NR % 2 == 1' file.txt           # odd lines only

# Range pattern (between markers)
awk '/START/,/END/' file.txt         # inclusive range

# Negate any condition
awk '!($3 > 100)' file.txt
awk '$1 != "ERROR"' file.txt
```

---

## Reformatting Output

```bash
# Change field separator on output
awk -F: '{ print $1, $3 }' /etc/passwd               # space-separated
awk -F: 'BEGIN{OFS=","} { print $1, $3 }' /etc/passwd # comma-separated

# Reverse field order
awk '{ for(i=NF;i>=1;i--) printf "%s ", $i; print "" }' file.txt

# Print with line numbers
awk '{ print NR": "$0 }' file.txt

# Add a prefix/suffix to each line
awk '{ print "PREFIX_" $0 }' file.txt
awk '{ print $0 "_SUFFIX" }' file.txt

# Convert space-separated to comma-separated
awk '{ $1=$1; print }' OFS=, file.txt
# Note: $1=$1 forces awk to rebuild $0 using new OFS

# Pad columns to fixed width
awk '{ printf "%-15s %10s\n", $1, $2 }' file.txt

# Convert tabs to spaces (or vice versa)
awk -F'\t' '{ $1=$1; print }' OFS=' ' file.txt

# Trim leading/trailing whitespace
awk '{ gsub(/^[ \t]+|[ \t]+$/, ""); print }' file.txt

# Squeeze multiple spaces into one
awk '{ $1=$1; print }' file.txt   # reassigning rebuilds with single OFS
```

---

## Counting & Aggregation

```bash
# Count lines (like wc -l)
awk 'END { print NR }' file.txt

# Count lines matching pattern
awk '/error/ { count++ } END { print count }' file.txt

# Sum a column
awk '{ sum += $3 } END { print sum }' file.txt

# Average of a column
awk '{ sum += $3; count++ } END { print sum/count }' file.txt

# Min and max
awk 'NR==1 { min=max=$3 } { if($3<min) min=$3; if($3>max) max=$3 } END { print min, max }' file.txt

# Count unique values in a field
awk '{ print $1 }' file.txt | sort -u | wc -l
# Or natively in awk:
awk '{ seen[$1]++ } END { print length(seen) }' file.txt

# Frequency count (like sort | uniq -c)
awk '{ count[$1]++ } END { for (k in count) print count[k], k }' file.txt

# Frequency, sorted descending
awk '{ count[$1]++ } END { for (k in count) print count[k], k }' file.txt | sort -rn

# Sum grouped by category
awk '{ sum[$1] += $2 } END { for (k in sum) print k, sum[k] }' file.txt

# Running total
awk '{ total += $1; print $0, total }' file.txt

# Percentage of total
awk '{ sum += $1; vals[NR] = $1 } END { for (i=1;i<=NR;i++) printf "%.1f%%\n", vals[i]/sum*100 }' file.txt
```

---

## Working with CSV

```bash
# Basic CSV (simple, no embedded commas)
awk -F, '{ print $2 }' data.csv

# CSV with header — skip first line
awk -F, 'NR > 1 { print $1, $3 }' data.csv

# CSV with header — print header + filtered rows
awk -F, 'NR==1 || $3 > 100' data.csv

# Convert CSV to TSV
awk -F, 'BEGIN{OFS="\t"} {$1=$1; print}' data.csv

# Sum a CSV column (skip header)
awk -F, 'NR>1 { sum += $3 } END { print sum }' data.csv

# CSV with quoted fields (simple case — awk doesn't handle embedded commas in quotes natively)
# For proper CSV parsing with quotes, use gawk's FPAT:
gawk 'BEGIN {
  FPAT = "([^,]+)|(\"[^\"]+\")"
}
{ print $2 }' data.csv

# Extract specific columns by header name
awk -F, '
  NR==1 { for(i=1;i<=NF;i++) col[$i]=i; next }
  { print $col["email"], $col["name"] }
' data.csv

# Add column with calculated value
awk -F, 'BEGIN{OFS=","} { print $0, $2*$3 }' data.csv

# Filter and count rows matching condition
awk -F, 'NR>1 && $4=="active" { count++ } END { print count }' data.csv
```

---

## Multiple Files

```bash
# Process multiple files, NR is cumulative, FNR resets per file
awk '{ print FILENAME, FNR, NR, $0 }' file1.txt file2.txt

# Print only filename when first encountered
awk 'FNR==1 { print "=== " FILENAME " ===" } { print }' *.txt

# Count lines per file
awk 'FNR==1 && NR>1 { print prev, count } { count++; prev=FILENAME } END { print prev, count }' *.txt
# Cleaner: use END block per file with ENDFILE (gawk)
awk 'ENDFILE { print FILENAME, FNR }' *.txt

# Skip the header in EVERY file (not just first)
awk 'FNR==1 { next } { print }' *.csv

# Combine multiple files, keeping only one header
awk 'FNR==1 && NR!=1 { next } { print }' file1.csv file2.csv file3.csv

# Sum a column across multiple files, with per-file subtotal
awk '
  FNR==1 && NR>1 { print FILENAME": "subtotal; subtotal=0 }
  { subtotal += $1; total += $1 }
  END { print FILENAME": "subtotal; print "TOTAL:", total }
' file1.txt file2.txt
```

---

## Custom Field & Record Separators

```bash
# Colon-separated (like /etc/passwd)
awk -F: '{ print $1, $7 }' /etc/passwd

# Multiple character separator
awk -F'::' '{ print $2 }' file.txt

# Regex separator
awk -F'[,;]' '{ print $1 }' file.txt    # split on comma OR semicolon
awk -F'[ \t]+' '{ print $2 }' file.txt  # split on any whitespace run

# Tab-separated
awk -F'\t' '{ print $3 }' file.tsv

# Set FS in BEGIN instead of -F
awk 'BEGIN{FS=":"} { print $1 }' /etc/passwd

# Multiple-character OFS for output
awk 'BEGIN{FS=","; OFS=" | "} { print $1, $2, $3 }' data.csv

# Record separator: paragraph mode (blank line = separator)
awk 'BEGIN{RS=""} { print "Record:", NR; print $0; print "---" }' paragraphs.txt

# Record separator: custom character
awk 'BEGIN{RS=";"} { print NR, $0 }' file.txt

# Record separator: regex (gawk)
awk 'BEGIN{RS="[0-9]+\\. "} { print NR, $0 }' numbered_list.txt
```

---

## String Manipulation

```bash
# Uppercase / lowercase
awk '{ print toupper($0) }' file.txt
awk '{ print tolower($1), $2 }' file.txt

# Substring extraction
awk '{ print substr($0, 1, 10) }' file.txt          # first 10 chars
awk '{ print substr($0, 5) }' file.txt              # from char 5 to end
awk '{ print substr($0, length($0)-4) }' file.txt   # last 5 chars

# String length
awk '{ print length($0), $0 }' file.txt
awk 'length($0) > 80' file.txt    # lines longer than 80 chars

# Replace (first occurrence)
awk '{ sub(/foo/, "bar"); print }' file.txt

# Replace (all occurrences)
awk '{ gsub(/foo/, "bar"); print }' file.txt

# Replace in specific field only
awk '{ gsub(/foo/, "bar", $2); print }' file.txt

# Remove all digits
awk '{ gsub(/[0-9]/, ""); print }' file.txt

# Extract pattern using match()
awk '{
  if (match($0, /[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+/))
    print substr($0, RSTART, RLENGTH)
}' file.txt

# Split a field further
awk '{ n = split($1, parts, "-"); print parts[1], parts[2] }' file.txt

# Reverse a string
awk '{
  s = $0; r = ""
  for (i = length(s); i >= 1; i--) r = r substr(s, i, 1)
  print r
}' file.txt

# Trim whitespace
awk '{ gsub(/^[ \t]+|[ \t]+$/, ""); print }' file.txt

# Check if string contains substring
awk '$0 ~ /pattern/' file.txt
awk 'index($0, "pattern") > 0' file.txt
```

---

## Math & Calculations

```bash
# Basic arithmetic on fields
awk '{ print $1 + $2 }' file.txt
awk '{ print $1 * $2 }' file.txt

# Percentage calculation
awk '{ printf "%.1f%%\n", ($1/$2)*100 }' file.txt

# Round to nearest integer
awk '{ print int($1 + 0.5) }' file.txt

# Convert units (e.g., bytes to MB)
awk '{ printf "%.2f MB\n", $1/1024/1024 }' file.txt

# Running statistics: min, max, sum, average, count
awk '
  NR==1 { min=max=$1 }
  { sum+=$1; count++; if($1<min)min=$1; if($1>max)max=$1 }
  END {
    printf "Count: %d\n", count
    printf "Sum: %.2f\n", sum
    printf "Average: %.2f\n", sum/count
    printf "Min: %.2f\n", min
    printf "Max: %.2f\n", max
  }
' numbers.txt

# Standard deviation
awk '
  { sum+=$1; sumsq+=$1*$1; count++ }
  END {
    mean = sum/count
    variance = (sumsq/count) - (mean*mean)
    printf "Mean: %.2f  StdDev: %.2f\n", mean, sqrt(variance)
  }
' numbers.txt

# Convert hex to decimal
awk 'BEGIN { print strtonum("0xff") }'    # gawk only

# Convert decimal to hex
awk '{ printf "%x\n", $1 }' file.txt
```

---

## Log Analysis

```bash
# Count requests per status code (nginx/apache log)
awk '{ print $9 }' access.log | sort | uniq -c | sort -rn
# Or natively in awk:
awk '{ count[$9]++ } END { for (c in count) print count[c], c }' access.log

# Find all 5xx errors
awk '$9 ~ /^5/' access.log

# Sum bytes transferred
awk '{ sum += $10 } END { print sum }' access.log

# Top 10 IPs by request count
awk '{ count[$1]++ } END { for (ip in count) print count[ip], ip }' access.log | sort -rn | head -10

# Requests per hour
awk -F'[:[]' '{ print $3 }' access.log | sort | uniq -c

# Extract specific log fields (Apache combined log format)
awk '{ print $1, $4, $7, $9 }' access.log  # IP, timestamp, URL, status

# Find slow requests (if response time is logged)
awk '$NF > 1000 { print }' access.log    # requests over 1000ms

# Unique user agents
awk -F'"' '{ print $6 }' access.log | sort -u

# Error log: count errors by type
awk '/ERROR/ { 
  match($0, /ERROR: [A-Za-z]+/)
  print substr($0, RSTART+7, RLENGTH-7)
}' app.log | sort | uniq -c | sort -rn

# Syslog: extract by process name
awk '$5 ~ /sshd/ { print }' /var/log/syslog

# Count log entries per day
awk '{ print $1, $2, $3 }' /var/log/syslog | sort | uniq -c
```

---

## System Administration

```bash
# Parse /etc/passwd: list usernames and shells
awk -F: '{ print $1, $7 }' /etc/passwd

# Find users with bash shell
awk -F: '$7 == "/bin/bash" { print $1 }' /etc/passwd

# Find users with UID >= 1000 (regular users)
awk -F: '$3 >= 1000 && $3 < 65534 { print $1, $3 }' /etc/passwd

# Parse /etc/group: members of a specific group
awk -F: '$1 == "sudo" { print $4 }' /etc/group

# ps output: sort by memory usage
ps aux | awk 'NR>1 { print $4, $11 }' | sort -rn | head -10

# ps output: total memory used by a specific process
ps aux | awk '$11 ~ /nginx/ { sum += $6 } END { print sum/1024, "MB" }'

# df output: filesystems over 80% full
df -h | awk 'NR>1 { gsub(/%/,"",$5); if ($5+0 > 80) print $6, $5"%" }'

# free output: available memory in GB
free -m | awk '/^Mem:/ { printf "%.2f GB\n", $7/1024 }'

# netstat/ss: count connections by state
ss -tan | awk 'NR>1 { print $1 }' | sort | uniq -c

# Parse ifconfig/ip output for IP addresses
ip addr | awk '/inet / { print $2 }'

# Process list: kill all processes matching pattern (use carefully!)
ps aux | awk '/myapp/ && !/awk/ { print $2 }' | xargs kill

# Disk usage report sorted by size
du -sh /home/* 2>/dev/null | awk '{ print $1, $2 }' | sort -rh
```

---

## Arrays in Practice

```bash
# Deduplicate lines (preserves order, unlike sort -u)
awk '!seen[$0]++' file.txt

# Deduplicate based on a specific field
awk '!seen[$1]++' file.txt

# Find duplicate lines only
awk 'seen[$0]++ == 1' file.txt

# Count distinct values per field
awk '{ a[$1]++; b[$2]++ } END { print "Field1 unique:", length(a); print "Field2 unique:", length(b) }' file.txt

# Group lines by first field
awk '{ group[$1] = group[$1] $0 "\n" } END { for (k in group) { print "=== " k " ==="; print group[k] } }' file.txt

# Two-dimensional array (matrix-like data)
awk '{ matrix[$1,$2] = $3 } END { print matrix["row1","col2"] }' file.txt

# Build a lookup table from one file, use in another (see next section for full join pattern)
awk '
  NR==FNR { lookup[$1] = $2; next }
  { print $0, lookup[$1] }
' lookup.txt data.txt

# Sort array by value (gawk: asorti/asort)
awk '
  { count[$1]++ }
  END {
    n = asorti(count, sorted, "@val_num_desc")
    for (i=1; i<=n; i++) print count[sorted[i]], sorted[i]
  }
' file.txt
```

---

## Two-File Patterns (Join-like)

```bash
# Classic awk join: NR==FNR trick
# file1.txt: id,name
# file2.txt: id,amount
awk -F, '
  NR==FNR { name[$1]=$2; next }    # process file1 first, build lookup
  { print $1, name[$1], $2 }       # process file2, use lookup
' file1.csv file2.csv

# Inner join: only print if match exists
awk -F, '
  NR==FNR { name[$1]=$2; next }
  $1 in name { print $1, name[$1], $2 }
' file1.csv file2.csv

# Left join: print all from file2, blank if no match
awk -F, '
  NR==FNR { name[$1]=$2; next }
  { print $1, ($1 in name ? name[$1] : "N/A"), $2 }
' file1.csv file2.csv

# Find lines in file2 NOT in file1 (anti-join)
awk -F, '
  NR==FNR { seen[$1]=1; next }
  !($1 in seen) { print }
' file1.csv file2.csv

# Compare two files: find differences in matching keys
awk -F, '
  NR==FNR { old[$1]=$2; next }
  $1 in old && old[$1] != $2 { print $1, "changed from", old[$1], "to", $2 }
' old.csv new.csv
```

---

## One-Liners Cheat Sheet

```bash
# Print line count
awk 'END{print NR}' file.txt

# Print last line (like tail -1)
awk 'END{print}' file.txt

# Print first line (like head -1)
awk 'NR==1{print; exit}' file.txt

# Print specific line number
awk 'NR==42' file.txt

# Reverse line order (like tac)
awk '{a[NR]=$0} END{for(i=NR;i>=1;i--)print a[i]}' file.txt

# Remove blank lines (like grep -v '^$')
awk 'NF' file.txt

# Print longest line
awk '{ if(length($0)>max){max=length($0);line=$0} } END{print line}' file.txt

# Sum all numbers in file (one per line)
awk '{s+=$1} END{print s}' numbers.txt

# Double-space a file
awk '1;{print ""}' file.txt

# Number non-blank lines (like cat -b)
awk 'NF{$0=++n": "$0}1' file.txt

# Print between two patterns (exclusive)
awk '/START/{flag=1;next}/END/{flag=0}flag' file.txt

# Justify text to N columns (left-pad)
awk '{printf "%-20s\n", $0}' file.txt

# Strip HTML tags (basic, not robust)
awk '{ gsub(/<[^>]+>/, ""); print }' file.html

# Print every Nth line
awk 'NR%5==0' file.txt    # every 5th line

# Convert Windows line endings to Unix
awk '{ sub(/\r$/, ""); print }' winfile.txt
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
