# grep — Practical Examples

> Real-world patterns from log analysis, code search, sysadmin, and security.

---

## Table of Contents

- [Basic Searching](#basic-searching)
- [Case & Whole Word](#case--whole-word)
- [Invert & Count](#invert--count)
- [Line Numbers & Context](#line-numbers--context)
- [Recursive Search](#recursive-search)
- [Regex: BRE & ERE](#regex-bre--ere)
- [PCRE: Advanced Patterns](#pcre-advanced-patterns)
- [Output Control](#output-control)
- [Multiple Patterns](#multiple-patterns)
- [Searching Compressed Files](#searching-compressed-files)
- [Binary Files](#binary-files)
- [Pipelines & Chaining](#pipelines--chaining)
- [Log Analysis](#log-analysis)
- [Code Search](#code-search)
- [Security & Sysadmin](#security--sysadmin)
- [Scripting with grep](#scripting-with-grep)

---

## Basic Searching

```bash
# Search in a file
grep "error" logfile.txt

# Search multiple files
grep "error" *.log

# Search in a directory (recursive)
grep -r "error" /var/log/

# Search stdin
cat file.txt | grep "error"
echo "hello world" | grep "world"

# Read pattern from a file
grep -f patterns.txt logfile.txt

# Multiple patterns (OR)
grep -e "error" -e "warning" -e "critical" app.log
```

---

## Case & Whole Word

```bash
# Case-insensitive
grep -i "error" logfile.txt
grep -i "ERROR\|error\|Error" file.txt   # all covered by -i

# Whole word only (word boundaries)
grep -w "log" file.txt
# Matches: "log" "the log file"
# No match: "logger" "syslog" "catalog"

# Whole line must match
grep -x "exactly this line" file.txt
grep -x "[0-9]\+" numbers.txt   # lines with only digits (BRE)

# Combine: case-insensitive whole word
grep -iw "error" file.txt
```

---

## Invert & Count

```bash
# Invert: lines that do NOT match
grep -v "debug" app.log            # exclude debug lines
grep -v "^#" config.conf           # exclude comment lines
grep -v "^$" file.txt              # exclude empty lines
grep -v "^#\|^$" config.conf       # exclude comments AND empty lines (ERE: use -E)
grep -Ev "^#|^$" config.conf       # same with -E

# Count matching lines
grep -c "error" logfile.txt
grep -c "^$" file.txt              # count empty lines

# Count non-matching lines
grep -cv "pattern" file.txt

# Count matches across multiple files
grep -rc "TODO" /src/              # count per file
grep -r "TODO" /src/ | wc -l      # total count
```

---

## Line Numbers & Context

```bash
# Show line numbers
grep -n "error" logfile.txt
# Output: 42:error: connection refused

# Context: lines after match
grep -A 3 "ERROR" app.log          # 3 lines after
grep -A 5 "Exception" java.log     # 5 lines after (stack trace)

# Context: lines before match
grep -B 2 "FATAL" app.log          # 2 lines before

# Context: lines before AND after
grep -C 3 "error" app.log          # 3 lines each side
grep -C 5 "Traceback" python.log   # Python exception context

# Custom separator between context groups
grep -C 2 --group-separator="===" "error" app.log

# Byte offset of match
grep -b "pattern" file.txt
```

---

## Recursive Search

```bash
# Recursive (no symlink follow)
grep -r "TODO" /src/

# Recursive (follow symlinks)
grep -R "TODO" /src/

# Recursive with filename only
grep -rl "TODO" /src/

# Recursive, specific file type
grep -r --include="*.py" "import os" /src/
grep -r --include="*.{js,ts}" "console.log" /src/
grep -r --include="*.conf" "timeout" /etc/

# Recursive, exclude file types
grep -r --exclude="*.min.js" "function" /src/
grep -r --exclude="*.pyc" "pattern" .

# Exclude directories
grep -r --exclude-dir=".git" "pattern" .
grep -r --exclude-dir={".git","node_modules","venv"} "TODO" .

# Recursive, case-insensitive
grep -ri "password" /etc/
```

---

## Regex: BRE & ERE

```bash
# --- BRE (default grep) ---

# Anchor: start of line
grep "^root" /etc/passwd          # lines starting with "root"

# Anchor: end of line
grep "bash$" /etc/passwd          # lines ending with "bash"

# Any character
grep "c.t" file.txt               # cat, cut, cot, c3t...

# Zero or more
grep "ab*c" file.txt              # ac, abc, abbc, abbbc...

# Character class
grep "[aeiou]" file.txt           # lines with a vowel
grep "[0-9][0-9][0-9]" file.txt   # three consecutive digits

# Negated class
grep "[^0-9]" file.txt            # lines with non-digit chars

# POSIX classes
grep "[[:digit:]]" file.txt
grep "[[:upper:]]" file.txt

# Grouping and backreference (BRE needs \( \))
grep "\(foo\).*\1" file.txt       # "foo" followed by "foo" again

# --- ERE (grep -E or egrep) ---

# One or more
grep -E "ab+c" file.txt           # abc, abbc, abbbc (not ac)

# Zero or one
grep -E "colou?r" file.txt        # color or colour

# Alternation
grep -E "cat|dog|bird" file.txt
grep -E "^(error|warning|fatal)" app.log

# Grouping
grep -E "(foo)+" file.txt         # foo, foofoo, foofoofoo

# Quantifiers
grep -E "[0-9]{3}" file.txt       # exactly 3 digits
grep -E "[0-9]{2,4}" file.txt     # 2 to 4 digits
grep -E "[0-9]{3,}" file.txt      # 3 or more digits

# IP address pattern
grep -E "([0-9]{1,3}\.){3}[0-9]{1,3}" file.txt

# Email-like pattern
grep -E "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" file.txt

# URL pattern
grep -E "https?://[a-zA-Z0-9./?=_%:-]+" file.txt
```

---

## PCRE: Advanced Patterns

```bash
# Shorthand classes
grep -P "\d+" file.txt            # one or more digits
grep -P "\w+" file.txt            # word characters
grep -P "\s+" file.txt            # whitespace
grep -P "\D+" file.txt            # non-digits

# Word boundaries
grep -P "\bword\b" file.txt       # whole word "word"
grep -P "\bfoo\B" file.txt        # "foo" NOT at word end

# Lookahead: match X only if followed by Y
grep -P "foo(?=bar)" file.txt     # "foo" before "bar" (prints "foo")
grep -P "\d+(?= dollars)" file.txt   # number before " dollars"

# Negative lookahead: match X only if NOT followed by Y
grep -P "foo(?!bar)" file.txt     # "foo" not before "bar"

# Lookbehind: match X only if preceded by Y
grep -P "(?<=@)\w+" file.txt      # domain part after @
grep -P "(?<=error: ).+" file.txt # message after "error: "

# Negative lookbehind
grep -P "(?<!un)happy" file.txt   # "happy" not preceded by "un"

# Non-greedy
grep -Po "<.+?>" file.txt         # HTML tags (non-greedy)

# Named groups (with -o to extract)
grep -Po "(?<=name=)\w+" file.txt

# \K: reset match start
grep -Po "prefix\K\w+" file.txt   # print only what follows "prefix"

# Unicode properties (if PCRE has Unicode support)
grep -P "\p{Lu}" file.txt         # uppercase Unicode letter

# Multiline patterns are NOT supported in grep (it's line-based)
# Use pcregrep or perl for multiline
```

---

## Output Control

```bash
# Only print matched part (not whole line)
grep -o "pattern" file.txt
grep -oE "[0-9]+" file.txt        # extract all numbers
grep -oE "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" mail.txt  # extract emails
grep -oP "(?<=error: ).+" app.log # extract error messages

# Quiet mode (just exit code)
grep -q "error" file.txt
echo $?   # 0 = found, 1 = not found

# Suppress errors
grep -s "pattern" /nonexistent/file  # no error output

# Only filenames with matches
grep -rl "TODO" /src/

# Only filenames WITHOUT matches
grep -rL "Copyright" /src/

# Stop after N matches
grep -m 5 "error" huge.log        # first 5 matches only
grep -m 1 "error" *.log           # first match per file

# With null byte after filename (for xargs)
grep -rlZ "TODO" /src/ | xargs -0 sed -i 's/TODO/FIXME/g'

# Force color in pipe
grep --color=always "error" app.log | less -R

# Align with tab (when using -n)
grep -Tn "error" file.txt
```

---

## Multiple Patterns

```bash
# OR: multiple -e flags
grep -e "error" -e "warning" -e "fatal" app.log

# OR with ERE
grep -E "error|warning|fatal" app.log

# AND: chain greps
grep "error" app.log | grep "database"    # error AND database

# AND with PCRE lookahead (single grep)
grep -P "(?=.*error)(?=.*database)" app.log   # both on same line

# Patterns from file
cat patterns.txt
# error
# warning
# fatal
grep -f patterns.txt app.log

# Combine -f with -e
grep -f patterns.txt -e "critical" app.log

# NOT: invert
grep "error" app.log | grep -v "test"    # error but not test

# Complex: error OR warning, but not test, with context
grep -E "error|warning" app.log | grep -v "test"
```

---

## Searching Compressed Files

```bash
# gzip
zgrep "error" /var/log/syslog.1.gz
zgrep -i "exception" app.log.gz

# bzip2
bzgrep "error" archive.bz2

# xz
xzgrep "pattern" data.xz

# Multiple compressed files
zgrep "ERROR" /var/log/nginx/access.log*.gz

# Compressed + context
zgrep -C 3 "FATAL" app.log.gz

# Count matches in compressed file
zgrep -c "error" app.log.gz

# Combine: search all logs including compressed
grep -r "error" /var/log/*.log
zgrep "error" /var/log/*.gz
```

---

## Binary Files

```bash
# By default, grep skips binary files (prints "Binary file X matches")
grep "pattern" binary_file

# Force text treatment
grep -a "pattern" binary_file
grep --text "pattern" binary_file

# Show only filenames for binary matches
grep -l "pattern" *             # works on binaries too

# Ignore binary files (treat as no match)
grep -I "pattern" *
grep --binary-files=without-match "pattern" *

# Search for strings in binary (see also: strings command)
grep -a "password" /bin/program
strings /bin/program | grep "password"   # often cleaner

# NUL-delimited input (for filenames with newlines)
find . -name "*.txt" -print0 | xargs -0 grep -l "pattern"
grep -z "pattern" file_with_null_separators
```

---

## Pipelines & Chaining

```bash
# Classic pipeline
cat /etc/passwd | grep "bash"
# Better (no useless cat):
grep "bash" /etc/passwd

# Chain to sort
grep "error" app.log | sort | uniq -c | sort -rn

# Chain to count
grep -c "ERROR" app.log
grep "ERROR" app.log | wc -l   # same but also counts binary matches

# Chain to awk
grep "^[^#]" /etc/hosts | awk '{print $2}'

# Chain to sed
grep "error" app.log | sed 's/error/ERROR/g'

# Chain to cut
grep "user=" app.log | cut -d'=' -f2

# Tee: search AND save
grep "ERROR" app.log | tee errors.txt | wc -l

# Chain multiple greps (AND logic)
grep "database" app.log | grep "timeout" | grep -v "test"

# Feed to xargs
grep -rl "old_function" /src/ | xargs sed -i 's/old_function/new_function/g'

# Feed find results to grep
find . -name "*.py" -print0 | xargs -0 grep "import requests"

# Colorized output through less
grep --color=always "error" app.log | less -R
```

---

## Log Analysis

```bash
# Find all errors in nginx log
grep " 500 " /var/log/nginx/access.log

# Count HTTP status codes
grep -oE " [0-9]{3} " access.log | sort | uniq -c | sort -rn

# Extract IPs making 404 requests
grep " 404 " access.log | grep -oE "^[0-9.]+"

# Find slow requests (> 1 second)
grep -E '"[0-9.]+" [0-9]+ [0-9]+$' access.log | awk '$NF > 1'

# Today's errors in syslog
grep "$(date '+%b %e')" /var/log/syslog | grep -i "error"

# Find all unique error messages
grep -oP "(?<=ERROR: ).+" app.log | sort -u

# Count errors per hour
grep "ERROR" app.log | grep -oP "\d{2}:\d{2}" | cut -d: -f1 | sort | uniq -c

# Last 100 lines with errors
tail -100 app.log | grep "ERROR"

# Errors in last hour (using date math)
grep -E "$(date -d '1 hour ago' '+%H:%M')" app.log | grep "ERROR"

# Extract stack traces (lines after "Exception")
grep -A 10 "Exception" java.log | grep -v "^--$"

# Apache: top 10 IPs by request count
grep -oE "^[0-9.]+" access.log | sort | uniq -c | sort -rn | head -10

# Find all unique URLs returning 404
grep " 404 " access.log | grep -oE '"GET [^ ]+' | sort -u
```

---

## Code Search

```bash
# Find all TODO/FIXME comments
grep -rn "TODO\|FIXME\|HACK\|XXX" /src/

# Find function definitions (Python)
grep -n "^def " *.py
grep -rn "^def \|^class " /src/ --include="*.py"

# Find all imports of a module
grep -rn "import requests" /src/ --include="*.py"
grep -rn "from requests" /src/ --include="*.py"

# Find all calls to a function
grep -rn "old_function(" /src/

# Find hardcoded IPs
grep -rEn "([0-9]{1,3}\.){3}[0-9]{1,3}" /src/ --include="*.py"

# Find hardcoded passwords (security audit)
grep -riE "password\s*=\s*['\"][^'\"]{3,}" /src/
grep -riE "(api_key|secret|token)\s*=\s*['\"][^'\"]+" /src/

# Find long lines (> 120 chars)
grep -En ".{121,}" file.py

# Find files with Windows line endings
grep -rlP "\r" /src/

# Find missing newline at end of file
grep -rPl "[^\n]\z" /src/    # PCRE: files not ending with newline

# Count lines of code (excluding comments and blanks)
grep -cv "^\s*#\|^\s*$" *.py

# Find all class names (Python)
grep -roh "class [A-Z][a-zA-Z]*" /src/ | grep -oE "[A-Z][a-zA-Z]*" | sort -u
```

---

## Security & Sysadmin

```bash
# Find failed SSH logins
grep "Failed password" /var/log/auth.log

# Count failed logins per IP
grep "Failed password" /var/log/auth.log \
  | grep -oE "from [0-9.]+" \
  | sort | uniq -c | sort -rn | head -20

# Find successful sudo usage
grep "sudo" /var/log/auth.log | grep "COMMAND"

# Find cron jobs run
grep "CRON" /var/log/syslog

# Find all listening ports in config
grep -rE "listen\s+[0-9]+" /etc/nginx/

# Find plaintext passwords in configs
grep -rli "password" /etc/ 2>/dev/null

# Find world-readable sensitive files
find /etc -name "*.conf" -readable | xargs grep -l "password" 2>/dev/null

# Search for specific user in logs
grep "alice" /var/log/auth.log | grep -v "^#"

# Find kernel errors
grep -i "error\|fail\|warn" /var/log/kern.log | tail -50

# Find disk errors
grep -i "I/O error\|disk error\|sector" /var/log/syslog

# Check if a service is mentioned in logs
grep -i "nginx\|apache\|mysql" /var/log/syslog | tail -20
```

---

## Scripting with grep

```bash
# Use exit code in if statement
if grep -q "error" logfile.txt; then
  echo "Errors found!"
  exit 1
fi

# Check if file contains pattern
if grep -qF "needle" haystack.txt; then
  echo "Found"
fi

# Count and act
error_count=$(grep -c "ERROR" app.log)
if [ "$error_count" -gt 10 ]; then
  echo "Too many errors: $error_count"
fi

# Extract values
version=$(grep -oP "(?<=version=)\S+" config.txt)
echo "Version: $version"

# Process each matching line
grep "^ERROR" app.log | while IFS= read -r line; do
  echo "Processing: $line"
done

# Find files and process them
grep -rl "deprecated_function" /src/ | while IFS= read -r file; do
  echo "Updating: $file"
  sed -i 's/deprecated_function/new_function/g' "$file"
done

# Validate input
validate_email() {
  echo "$1" | grep -qP "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"
}

if validate_email "user@example.com"; then
  echo "Valid email"
fi

# Grep with timeout (for hanging processes)
timeout 5 grep -r "pattern" /large/dir/ || echo "Search timed out"
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
