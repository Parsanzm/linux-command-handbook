# cat — Practical Examples

> Real-world usage patterns, from everyday to advanced.

---

## Table of Contents

- [Displaying Files](#displaying-files)
- [Line Numbers](#line-numbers)
- [Hidden Characters & Debugging](#hidden-characters--debugging)
- [Creating & Writing Files](#creating--writing-files)
- [Concatenating & Merging](#concatenating--merging)
- [stdin & Pipelines](#stdin--pipelines)
- [Files with Special Names](#files-with-special-names)
- [Heredoc](#heredoc)
- [Compressed Files (zcat, bzcat...)](#compressed-files)
- [Scripting Patterns](#scripting-patterns)

---

## Displaying Files

```bash
# Basic display
cat /etc/os-release

# Multiple files in sequence
cat /etc/hostname /etc/hosts /etc/resolv.conf

# Display with line count
cat -n /etc/nginx/nginx.conf

# Page through a large file
cat file.txt | less
# (better: just use less directly)
```

---

## Line Numbers

```bash
# Number ALL lines (including blank)
cat -n file.txt

# Number only non-blank lines
cat -b file.txt

# -b overrides -n when both given
cat -nb file.txt     # same as -b

# Practical use: reference line numbers while debugging
cat -n script.py | grep -n "def "
```

Output of `cat -n`:
```
     1  #!/bin/bash
     2
     3  echo "hello"
```

Output of `cat -b`:
```
     1  #!/bin/bash

     2  echo "hello"
```

---

## Hidden Characters & Debugging

```bash
# Show ALL special characters (-v -E -T)
cat -A file.txt
# Tabs appear as ^I
# Line endings appear as $
# Windows CRLF appears as ^M$

# Show tabs only
cat -T file.txt

# Show line endings only
cat -E file.txt

# Show non-printing chars only
cat -v file.txt
```

**Real example — debugging a broken script:**
```bash
cat -A myscript.sh
# Output:
# #!/bin/bash^M$      ← Windows CRLF! This will break on Linux
# echo "hello"^M$

# Fix: convert to Unix line endings
sed -i 's/\r//' myscript.sh
# or:
dos2unix myscript.sh
```

**Detecting encoding issues:**
```bash
cat -v data.csv | head -5
# If you see M-... sequences, there are non-ASCII/UTF-8 bytes
```

---

## Creating & Writing Files

```bash
# Create a new file (Ctrl+D to finish)
cat > config.txt
host=localhost
port=8080
^D

# Overwrite existing file
cat > existing.txt
new content here
^D

# Append to file
cat >> logfile.txt
new log entry
^D

# Copy a file (like cp, but explicit)
cat source.txt > destination.txt
```

---

## Concatenating & Merging

```bash
# Merge two files
cat part1.sql part2.sql > full_migration.sql

# Merge many files (glob)
cat *.log > all_logs.txt

# Merge with a separator between files
for f in part*.txt; do
  cat "$f"
  echo "---"
done > merged.txt

# Merge CSV files — keep only first header
head -1 data_jan.csv > combined.csv
tail -n +2 -q data_*.csv >> combined.csv

# Prepend a header to a file
cat header.txt original.txt > with_header.txt

# Append a footer
cat original.txt footer.txt > with_footer.txt

# Sandwich: header + content + footer
cat header.txt - footer.txt < body.txt > full.txt
```

---

## stdin & Pipelines

```bash
# Read from stdin explicitly
echo "hello" | cat

# Mix stdin with files (- represents stdin)
echo "middle" | cat top.txt - bottom.txt

# Classic pipeline
cat access.log | grep "404" | awk '{print $7}' | sort | uniq -c | sort -rn

# But prefer direct input when possible:
grep "404" access.log | awk '{print $7}' | sort | uniq -c | sort -rn

# Send to multiple destinations
cat bigfile.txt | tee copy.txt | wc -l

# Use with xargs
cat urls.txt | xargs curl -O
```

---

## Files with Special Names

```bash
# File starting with a dash (-) — use -- to end options
cat -- -myfile.txt
cat ./-myfile.txt     # also works

# File with spaces in name — use quotes
cat "my file.txt"
cat 'my file.txt'
cat my\ file.txt

# File with newlines in name (rare, but possible)
# Use find + -print0 / xargs -0 instead

# Hidden files (dotfiles) — work normally
cat ~/.bashrc
cat ~/.ssh/config
cat .env
cat ../.gitignore      # parent directory

# File with special characters: @, #, !, spaces, unicode
cat "report @2024.txt"
cat "file#1.txt"
cat "données.csv"

# Path with multiple spaces
cat "path/to/my  file/with  spaces.txt"

# Symlinks — cat follows them automatically
cat /etc/localtime     # symlink → binary file (tzdata)
cat /proc/cpuinfo      # virtual file from kernel
cat /sys/class/net/eth0/address   # MAC address
```

---

## Heredoc

Heredoc lets you write multi-line content inline in scripts.

```bash
# Basic heredoc — variables ARE expanded
cat > config.yaml << EOF
host: $HOSTNAME
user: $USER
date: $(date +%Y-%m-%d)
EOF

# Single-quoted heredoc — NO variable expansion (literal)
cat > template.sh << 'EOF'
#!/bin/bash
echo "Hello, $NAME"   # $NAME is NOT expanded here
EOF

# Indented heredoc (bash 4+, use <<-)
# Leading TABS (not spaces) are stripped
cat > output.txt <<- EOF
	line one
	line two
EOF

# Append with heredoc
cat >> existing.txt << EOF
additional content
EOF

# Heredoc to variable
content=$(cat << EOF
line 1
line 2
EOF
)

# Practical: generate config files in scripts
cat > /etc/app/config.conf << EOF
[server]
host = ${APP_HOST:-localhost}
port = ${APP_PORT:-3000}
debug = false
EOF

# Practical: generate SQL
cat << 'EOF' | psql -U admin mydb
BEGIN;
UPDATE users SET active = false WHERE last_login < NOW() - INTERVAL '90 days';
COMMIT;
EOF
```

---

## Compressed Files

```bash
# gzip
zcat file.txt.gz
gzcat file.txt.gz         # macOS / BSD alias

# bzip2
bzcat file.txt.bz2

# xz / lzma
xzcat file.txt.xz

# zstandard
zstdcat file.txt.zst

# lzip
lzcat file.txt.lz

# Pipe compressed logs directly
zcat /var/log/syslog.1.gz | grep "error"
bzcat backup.sql.bz2 | mysql -u root mydb
xzcat large_dataset.csv.xz | awk -F',' '{sum += $3} END {print sum}'

# Multiple compressed files
zcat archive_jan.gz archive_feb.gz archive_mar.gz > q1.txt

# Check without fully decompressing
zcat file.gz | head -20

# Count lines in compressed file
bzcat data.bz2 | wc -l
```

---

## Scripting Patterns

```bash
# Read a config file and process key=value pairs
while IFS='=' read -r key value; do
  [[ "$key" =~ ^#.*$ ]] && continue   # skip comments
  export "$key"="$value"
done < <(cat config.env)

# Safely empty a log file (preserves the file, clears content)
cat /dev/null > app.log

# Display a file only if it exists
[ -f config.txt ] && cat config.txt

# Concatenate only non-empty files
for f in *.txt; do
  [ -s "$f" ] && cat "$f"
done > combined.txt

# Add line numbers to a file and save
cat -n input.txt > numbered.txt

# Display file to terminal AND save to file
cat important.txt | tee backup.txt

# Process file content into variable
content=$(cat myfile.txt)
echo "File has $(cat myfile.txt | wc -l) lines"
# Better:
echo "File has $(wc -l < myfile.txt) lines"
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
