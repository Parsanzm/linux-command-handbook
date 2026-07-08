# sed — Practical Examples

> Real-world patterns for find-and-replace, log filtering, config editing, and text pipelines.

---

## Table of Contents

- [Basic Substitution](#basic-substitution)
- [Deleting Lines](#deleting-lines)
- [Printing Specific Lines](#printing-specific-lines)
- [Inserting, Appending, and Changing Lines](#inserting-appending-and-changing-lines)
- [In-Place Editing](#in-place-editing)
- [Working with Multiple Files](#working-with-multiple-files)
- [Regex Patterns & Capture Groups](#regex-patterns--capture-groups)
- [Multi-line Operations](#multi-line-operations)
- [Combining sed with Other Commands](#combining-sed-with-other-commands)
- [Log Filtering & Analysis](#log-filtering--analysis)
- [Configuration File Editing](#configuration-file-editing)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Substitution

```bash
# Replace the first occurrence per line
sed 's/foo/bar/' file.txt

# Replace ALL occurrences per line
sed 's/foo/bar/g' file.txt

# Case-insensitive replace
sed 's/foo/bar/gi' file.txt

# Replace only on lines matching a condition
sed '/pattern/s/foo/bar/' file.txt

# Use alternate delimiters when the pattern contains slashes (like paths)
sed 's|/old/path|/new/path|' file.txt
sed 's#http://#https://#' urls.txt
```

---

## Deleting Lines

```bash
# Delete blank lines
sed '/^$/d' file.txt

# Delete lines matching a pattern
sed '/^#/d' config.txt          # remove comment lines
sed '/DEBUG/d' app.log           # remove debug log entries

# Delete a specific line number
sed '5d' file.txt

# Delete a range of lines
sed '10,20d' file.txt

# Delete from a pattern to the end of the file
sed '/START_REMOVING/,$d' file.txt

# Delete everything EXCEPT lines matching a pattern (inverse)
sed '/keep_this/!d' file.txt
```

---

## Printing Specific Lines

```bash
# Print only line 5 (suppress default printing with -n)
sed -n '5p' file.txt

# Print a range of lines
sed -n '10,20p' file.txt

# Print the first 10 lines (like head -10)
sed -n '1,10p' file.txt

# Print the last line (like tail -1)
sed -n '$p' file.txt

# Print lines matching a pattern (like grep)
sed -n '/error/p' logfile.txt

# Print lines NOT matching a pattern (like grep -v)
sed -n '/error/!p' logfile.txt

# Print every 3rd line starting from line 1 (GNU sed)
sed -n '1~3p' file.txt
```

---

## Inserting, Appending, and Changing Lines

```bash
# Append text AFTER a matched line
sed '/^\[server\]/a\
port=8080' config.ini

# Insert text BEFORE a matched line
sed '3i\
This line was inserted' file.txt

# Change (replace) an entire matched line
sed '/^DEBUG=/c\
DEBUG=false' config.env

# Append after every line (add a separator)
sed 'a\
---' file.txt

# Insert a header at the very top of the file
sed '1i\
# Auto-generated file — do not edit manually' file.txt
```

---

## In-Place Editing

```bash
# Edit in place with no backup (GNU sed)
sed -i 's/foo/bar/' file.txt

# Edit in place WITH a backup copy
sed -i.bak 's/foo/bar/' file.txt
# Creates file.txt.bak (original) before modifying file.txt

# Edit in place, cross-platform safe (works on both GNU and BSD/macOS sed)
sed -i.bak 's/foo/bar/' file.txt && rm file.txt.bak

# Multiple in-place edits chained with -e
sed -i -e 's/foo/bar/' -e 's/baz/qux/' file.txt

# In-place edit across many files matching a glob
sed -i 's/http:/https:/g' *.html
```

---

## Working with Multiple Files

```bash
# Apply the same substitution across many files
sed -i 's/2023/2024/g' *.txt

# Apply and see which files actually changed
for f in *.conf; do
  grep -q "old_value" "$f" && echo "Would change: $f"
done
sed -i 's/old_value/new_value/g' *.conf

# Treat each file's line addressing independently (not as one combined stream)
sed -s -n '$p' *.txt        # print the LAST line of EACH file separately

# Combine content from multiple files with sed's own tools
sed '$s/$/\n---/' file1.txt file2.txt > combined.txt
```

---

## Regex Patterns & Capture Groups

```bash
# Extract and reorder date components
echo "2024-01-15" | sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/'
# 15/01/2024

# Swap first and last name
echo "Smith, John" | sed -E 's/([A-Za-z]+), ([A-Za-z]+)/\2 \1/'
# John Smith

# Wrap every number in brackets
echo "value is 42 and 7" | sed -E 's/[0-9]+/[&]/g'
# value is [42] and [7]

# Remove everything except digits
echo "Phone: (555) 123-4567" | sed 's/[^0-9]//g'
# 5551234567

# Add thousands separators to a number (simplified example)
echo "1234567" | sed -E ':a;s/([0-9]+)([0-9]{3})/\1,\2/;ta'
# 1,234,567

# Trim leading and trailing whitespace
echo "   hello world   " | sed 's/^[ \t]*//;s/[ \t]*$//'
# "hello world"
```

---

## Multi-line Operations

```bash
# Join every pair of consecutive lines into one
sed 'N;s/\n/ /' file.txt

# Reverse the order of lines (classic sed "tac" trick)
sed -n '1!G;h;$p' file.txt

# Print only the last N lines (rough tail -N equivalent using hold space)
sed -e ':a' -e '$q;N;11,$D;ba' file.txt      # roughly like tail -10

# Double-space a file (blank line after every line)
sed 'G' file.txt

# Remove consecutive blank lines, collapsing to a single blank line
sed '/^$/N;/\n$/D' file.txt

# Delete a block of lines between two markers (inclusive)
sed '/BEGIN_BLOCK/,/END_BLOCK/d' file.txt
```

---

## Combining sed with Other Commands

```bash
# Filter, then transform, then count
grep "ERROR" app.log | sed 's/^.*ERROR: //' | sort | uniq -c | sort -rn

# Extract and clean a specific field from structured text
cat data.csv | sed -E 's/^"([^"]*)".*/\1/'

# Use sed to prepare input for another tool
sed 's/,/\t/g' data.csv | column -t

# Pipe curl output through sed for quick text extraction
curl -s https://example.com | sed -n 's/.*<title>\(.*\)<\/title>.*/\1/p'

# Combine with find for batch file processing
find . -name "*.txt" -exec sed -i 's/foo/bar/g' {} +
```

---

## Log Filtering & Analysis

```bash
# Show only ERROR and WARNING lines
sed -n '/ERROR\|WARNING/p' app.log

# Redact sensitive data (e.g., mask credit card-like numbers)
sed -E 's/[0-9]{4}-[0-9]{4}-[0-9]{4}-[0-9]{4}/XXXX-XXXX-XXXX-XXXX/g' transactions.log

# Extract timestamps only
sed -E -n 's/^\[([0-9-]+ [0-9:]+)\].*/\1/p' app.log

# Remove ANSI color codes from a log file (common when logs were captured
# with terminal color escape sequences embedded)
sed -E 's/\x1b\[[0-9;]*m//g' colored.log > clean.log

# Show lines between two timestamps (assuming sorted/sequential log format)
sed -n '/2024-01-15 10:00/,/2024-01-15 11:00/p' app.log
```

---

## Configuration File Editing

```bash
# Toggle a boolean setting
sed -i 's/^DEBUG=false/DEBUG=true/' .env

# Update a version number
sed -i 's/^VERSION=.*/VERSION=2.5.0/' config.ini

# Comment out a specific line
sed -i '/^enable_feature_x/s/^/#/' config.conf

# Uncomment a specific line
sed -i '/^#enable_feature_x/s/^#//' config.conf

# Replace an entire value for a specific key, preserving the key name
sed -i -E 's/^(port\s*=\s*).*/\18080/' server.conf

# Add a new key=value pair after a specific section header
sed -i '/^\[database\]/a timeout=30' app.conf
```

---

## Real-World Recipes

```bash
# --- Bulk Rename References Across a Codebase ---

grep -rl "oldFunctionName" --include="*.js" . | xargs sed -i 's/oldFunctionName/newFunctionName/g'

# --- Cleaning Up Windows Line Endings (CRLF -> LF) ---

sed -i 's/\r$//' windows_file.txt

# --- Removing Trailing Whitespace From an Entire Project ---

find . -name "*.py" -exec sed -i 's/[ \t]*$//' {} +

# --- Generating a Quick Report from Structured Logs ---

sed -n 's/^.*status=\([0-9]*\).*/\1/p' access.log | sort | uniq -c | sort -rn

# --- Templating a Config File from a Template Version ---

sed -e "s/{{DB_HOST}}/$DB_HOST/" \
    -e "s/{{DB_PORT}}/$DB_PORT/" \
    config.template > config.production

# --- Extracting a Version Number from a File for Use in a Script ---

VERSION=$(sed -n 's/^version = "\(.*\)"/\1/p' pyproject.toml)
echo "Building version $VERSION"

# --- Safely Previewing an In-Place Edit Before Applying It ---

sed 's/foo/bar/g' file.txt | diff file.txt -    # preview the diff first
sed -i 's/foo/bar/g' file.txt                    # then apply once confirmed
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
