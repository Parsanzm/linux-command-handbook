# | (Pipe) — Practical Examples

> Real-world patterns for filtering, transforming, and analyzing data on the command line.

---

## Table of Contents

- [Basic Filtering](#basic-filtering)
- [Counting and Summarizing](#counting-and-summarizing)
- [Sorting and Deduplication](#sorting-and-deduplication)
- [Combining grep, sed, and awk](#combining-grep-sed-and-awk)
- [Process Management with Pipes](#process-management-with-pipes)
- [Working with Log Files](#working-with-log-files)
- [Building Long Analytical Pipelines](#building-long-analytical-pipelines)
- [Piping Into xargs](#piping-into-xargs)
- [Combining Pipes with Redirection](#combining-pipes-with-redirection)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Filtering

```bash
# Find lines containing a specific word
cat file.txt | grep 'hello'

# Filter a directory listing
ls -la | grep ".txt"

# Find a running process by name
ps aux | grep nginx

# Filter environment variables containing "PATH"
env | grep PATH

# Combine multiple filters in sequence
cat data.txt | grep "2024" | grep -v "test"    # matches 2024, EXCLUDES "test"
```

---

## Counting and Summarizing

```bash
# Count how many lines match a pattern
cat access.log | grep "ERROR" | wc -l

# Count total lines, words, and characters in a file
cat file.txt | wc -l
cat file.txt | wc -w
cat file.txt | wc -c

# Count how many files of a certain type exist
ls | grep ".jpg" | wc -l

# Count unique values in a column
cat data.csv | cut -d',' -f2 | sort | uniq -c
```

---

## Sorting and Deduplication

```bash
# Remove duplicate lines (sort is required FIRST — uniq only removes
# ADJACENT duplicates, so unsorted input won't dedupe correctly)
cat names.txt | sort | uniq

# Count occurrences of each unique line
cat names.txt | sort | uniq -c

# Sort numerically, descending, by the count column (common combo)
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn

# Find lines that appear MORE than once
cat file.txt | sort | uniq -d

# Find lines that appear EXACTLY once (unique to the file)
cat file.txt | sort | uniq -u
```

---

## Combining grep, sed, and awk

```bash
# Filter lines, then transform the matched text
cat log.txt | grep "ERROR" | sed 's/ERROR/CRITICAL/g'

# Filter, then extract just a specific column
cat access.log | grep "404" | awk '{print $1}'    # extract IP addresses of 404s

# Filter, transform, then sort the result
ps aux | grep python | awk '{print $2, $11}' | sort

# Chain grep -> cut -> sort -> uniq for a quick report
cat orders.csv | grep "completed" | cut -d',' -f4 | sort | uniq -c | sort -rn
```

---

## Process Management with Pipes

```bash
# Find and inspect a specific process
ps aux | grep nginx

# Extract just the PID of a matching process (excluding grep's own match)
ps aux | grep '[n]ginx' | awk '{print $2}'
# The [n]ginx trick prevents grep from matching its OWN process line,
# since "grep [n]ginx" itself doesn't literally contain "nginx"

# Kill all processes matching a name (use with caution!)
ps aux | grep '[p]ython script.py' | awk '{print $2}' | xargs kill

# Count how many worker processes are currently running
ps aux | grep '[w]orker' | wc -l

# Watch memory usage of a specific process type continuously
watch "ps aux | grep '[n]ginx' | awk '{sum+=\$6} END {print sum/1024 \" MB\"}'"
```

---

## Working with Log Files

```bash
# Show only ERROR lines from a growing log file in real time
tail -f app.log | grep "ERROR"

# Count error occurrences by type
cat app.log | grep "ERROR" | awk -F':' '{print $2}' | sort | uniq -c | sort -rn

# Extract and count unique IP addresses from an access log
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10

# Find the busiest hour of activity
cat access.log | awk '{print $4}' | cut -d: -f2 | sort | uniq -c | sort -rn

# Filter logs within a specific time range using grep with a pattern
cat app.log | grep "2024-01-15 14:" | grep "ERROR"

# Follow multiple log files simultaneously, filtering across both
tail -f app1.log app2.log | grep --line-buffered "CRITICAL"
```

---

## Building Long Analytical Pipelines

```bash
# Word frequency count from a text file (classic Unix pipeline example)
cat book.txt | tr -s ' \n' '\n' | tr 'A-Z' 'a-z' | sort | uniq -c | sort -rn | head -10

# Find the top 5 largest files in a directory tree
find . -type f -exec du -h {} \; | sort -rh | head -5

# Analyze disk usage by file EXTENSION
find . -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn

# Summarize HTTP status codes from an nginx access log
cat access.log | awk '{print $9}' | sort | uniq -c | sort -rn

# Extract, deduplicate, and count unique email domains from a list
cat emails.txt | cut -d'@' -f2 | sort | uniq -c | sort -rn
```

---

## Piping Into xargs

```bash
# grep finds filenames, xargs turns them into ARGUMENTS for another command
grep -l "TODO" *.py | xargs cat

# Delete all files matching a pattern (careful!)
find . -name "*.tmp" | xargs rm

# Run a command on each line of piped input, one at a time
cat urls.txt | xargs -I {} curl -s -o /dev/null -w "%{http_code} {}\n" {}

# Combine grep + xargs to search WITHIN files whose names match a pattern
find . -name "*.log" | xargs grep -l "ERROR"
```

---

## Combining Pipes with Redirection

```bash
# Save AND continue processing (see tee's own documentation for more)
cat data.csv | tee raw_copy.csv | grep "active" | wc -l

# Merge stderr into the pipe so error messages are ALSO filtered/searched
some_command 2>&1 | grep "warning"

# Pipe output into a file at the VERY end of a long pipeline
cat access.log | grep "ERROR" | sort | uniq -c | sort -rn > error_summary.txt

# Read from a file, pipe through several filters, write the final result
cat input.csv | grep -v "^#" | cut -d',' -f1,3 | sort > cleaned_output.csv
```

---

## Real-World Recipes

```bash
# --- Quick Server Health Check ---

ps aux | grep '[n]ginx' | wc -l                  # is nginx running?
df -h | grep -v "tmpfs"                           # disk usage, excluding temp filesystems
free -h | grep Mem                                 # memory summary

# --- Finding the Most Active Users in an Access Log ---

cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10

# --- Cleaning a Messy CSV Before Import ---

cat messy.csv | grep -v "^$" | grep -v "^#" | tr -d '\r' > clean.csv

# --- Monitoring a Deployment in Real Time ---

kubectl logs -f mypod | grep --line-buffered "ERROR\|WARN"

# --- Quick Git History Analysis ---

git log --pretty=format:"%an" | sort | uniq -c | sort -rn
# Shows commit count per author

# --- Finding Large Files Eating Disk Space ---

find / -xdev -type f -size +100M 2>/dev/null | xargs ls -lh | sort -k5 -rh | head -10

# --- Auditing Open Network Ports ---

sudo netstat -tulnp | grep LISTEN | awk '{print $4, $7}' | sort
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
