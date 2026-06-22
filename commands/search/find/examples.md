# find — Practical Examples

> Real-world usage patterns, from everyday to advanced.
> Every example is a pattern you'll actually use.

---

## Table of Contents

- [Basic Search by Name](#basic-search-by-name)
- [Search by Type](#search-by-type)
- [Search by Size](#search-by-size)
- [Search by Time](#search-by-time)
- [Search by Permissions](#search-by-permissions)
- [Search by Ownership](#search-by-ownership)
- [Combining Conditions](#combining-conditions)
- [Executing Commands on Results](#executing-commands-on-results)
- [Deleting Files](#deleting-files)
- [Pruning Directories](#pruning-directories)
- [Working with Symlinks](#working-with-symlinks)
- [Output Formatting](#output-formatting)
- [Piping to xargs](#piping-to-xargs)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Search by Name

```bash
# Find by exact name
find . -name "config.yaml"

# Wildcard — any .txt file
find . -name "*.txt"

# Case-insensitive
find . -iname "readme*"
find . -iname "*.JPG"          # matches .jpg .JPG .Jpg

# Single character wildcard
find . -name "file?.log"       # matches file1.log, filea.log

# Character class
find . -name "file[0-9].txt"

# Match anywhere in path
find . -path "*/tests/*.py"
find . -path "*/node_modules/*" -prune   # commonly used to skip

# Regex match (full path)
find . -regex ".*/[0-9]{4}-[0-9]{2}-[0-9]{2}\.log"
find . -iregex ".*\.(jpg|png|gif|webp)"

# Multiple name patterns (OR)
find . \( -name "*.jpg" -o -name "*.png" -o -name "*.gif" \)

# Exclude a pattern
find . -name "*.py" ! -name "*test*"
find . -name "*.log" ! -name "access.log"
```

---

## Search by Type

```bash
# Only regular files
find . -type f

# Only directories
find . -type d

# Only symlinks
find . -type l

# Only named pipes
find /run -type p

# Only sockets
find /run -type s

# Only block devices
find /dev -type b

# Only character devices
find /dev -type c

# Combine type with name
find . -type f -name "*.sh"
find . -type d -name "logs"
find . -type l -name "*.conf"    # symlinks to config files

# Directories only, limit depth
find . -maxdepth 2 -type d

# Empty files
find . -type f -empty

# Empty directories
find . -type d -empty
```

---

## Search by Size

```bash
# Exactly 0 bytes
find . -size 0c
find . -empty -type f

# Larger than 100MB
find . -size +100M

# Smaller than 1KB
find . -size -1k

# Between 1MB and 100MB
find . -size +1M -size -100M

# Larger than 1GB
find . -size +1G

# Exactly 4096 bytes
find . -size 4096c

# Large log files
find /var/log -name "*.log" -size +50M

# Find and sort by size (using -printf)
find . -type f -printf "%s\t%p\n" | sort -rn | head -20
```

---

## Search by Time

```bash
# Modified in last 24 hours
find . -mtime -1

# Modified more than 30 days ago
find . -mtime +30

# Modified exactly 7 days ago
find . -mtime 7

# Modified in last 60 minutes
find . -mmin -60

# Modified in last 2 hours
find . -mmin -120

# Accessed in last 24 hours
find . -atime -1

# Not accessed in over a year (cleanup candidates)
find /home -atime +365 -type f

# Metadata changed in last hour (chmod, chown, rename)
find . -cmin -60

# Modified after a reference file
find . -newer /etc/passwd

# Modified after a specific date (GNU)
find . -newermt "2024-06-01"
find . -newermt "2024-06-01 00:00:00"

# Modified within a date range
find . -newermt "2024-01-01" ! -newermt "2024-12-31"

# Log files from today
find /var/log -name "*.log" -mtime -1

# Sort results by modification time (newest first)
find . -type f -printf "%T@ %p\n" | sort -rn | awk '{print $2}'
```

---

## Search by Permissions

```bash
# Exact permissions
find . -perm 644 -type f
find . -perm 755 -type f

# Has SUID bit (security audit)
find / -perm -4000 -type f 2>/dev/null

# Has SGID bit
find / -perm -2000 -type f 2>/dev/null

# Has both SUID and SGID
find / -perm -6000 -type f 2>/dev/null

# World-writable files (security risk)
find / -perm /o+w -type f 2>/dev/null
find / -perm -002 -type f 2>/dev/null

# World-writable directories (security risk)
find / -perm -002 -type d 2>/dev/null

# Any execute bit set
find . -perm /111 -type f

# All execute bits set
find . -perm -111 -type f

# Files with no read permission for owner
find . ! -perm /u+r -type f

# Readable by current user
find . -readable -type f

# Executable by current user
find . -executable -type f
```

---

## Search by Ownership

```bash
# Owned by specific user
find /home -user alice
find /var -user www-data

# Owned by specific group
find . -group developers
find /srv -group nginx

# Owned by UID (numeric)
find . -uid 1001

# Orphaned files (owner no longer exists)
find / -nouser -type f 2>/dev/null
find / -nogroup -type f 2>/dev/null

# Files owned by root in user's home
find /home -user root -type f

# Files not owned by current user
find . ! -user "$(whoami)"

# Combine user and group
find /srv -user www-data -group www-data
```

---

## Combining Conditions

```bash
# Large Python files (implicit -and)
find . -name "*.py" -size +100k

# Files modified recently that are executable
find . -mtime -7 -executable -type f

# Directories that are empty and old
find . -type d -empty -mtime +30

# Large files not accessed recently (cleanup candidates)
find /home -type f -size +100M -atime +90

# OR: find txt OR md files
find . \( -name "*.txt" -o -name "*.md" \)

# NOT: everything except .git contents
find . ! -path "*/.git/*"

# Complex: Python files, not in venv or __pycache__, modified this week
find . -type f -name "*.py" \
  ! -path "*/venv/*" \
  ! -path "*/__pycache__/*" \
  -mtime -7

# Images larger than 1MB not accessed in 6 months
find ~/Pictures \( -name "*.jpg" -o -name "*.png" \) \
  -size +1M -atime +180

# Config files owned by root with wrong permissions
find /etc -type f -name "*.conf" -user root ! -perm 644
```

---

## Executing Commands on Results

```bash
# Run once per file (\; syntax)
find . -name "*.log" -exec gzip {} \;

# Batch all files into one call (+ syntax — faster)
find . -name "*.txt" -exec grep "TODO" {} +

# Run from file's own directory (-execdir — safer)
find . -name "*.py" -execdir python -m py_compile {} \;

# Prompt before each action (-ok)
find . -name "*.tmp" -ok rm {} \;

# chmod all scripts
find . -name "*.sh" -exec chmod +x {} \;

# chown recursively (find is more flexible than chown -R)
find /var/www -exec chown www-data:www-data {} +

# Replace text in multiple files
find . -name "*.html" -exec sed -i 's/old/new/g' {} \;

# Compress old logs
find /var/log -name "*.log" -mtime +7 -exec gzip {} \;

# Copy matched files preserving structure
find . -name "*.conf" -exec cp --parents {} /backup/ \;

# Run a shell command (needs sh -c)
find . -name "*.txt" -exec sh -c 'echo "Processing: $1"; wc -l "$1"' _ {} \;

# Multiple commands per file
find . -type f -name "*.py" -exec sh -c '
  echo "=== $1 ==="
  wc -l "$1"
  grep -c "def " "$1"
' _ {} \;
```

---

## Deleting Files

> ⚠️ Always test with `-print` before using `-delete` or `-exec rm`

```bash
# Test first
find . -name "*.tmp" -print

# Then delete
find . -name "*.tmp" -delete

# Delete empty files
find . -type f -empty -delete

# Delete empty directories (post-order with -depth)
find . -type d -empty -delete
find . -depth -type d -empty -delete    # explicit post-order

# Delete files older than 30 days
find /tmp -mtime +30 -type f -delete

# Delete .DS_Store files (macOS cleanup)
find . -name ".DS_Store" -delete

# Delete Python cache
find . -name "*.pyc" -delete
find . -type d -name "__pycache__" -exec rm -rf {} +

# Delete node_modules (dangerous — test first!)
find . -type d -name "node_modules" -prune -exec rm -rf {} \;

# Delete log files over 100MB
find /var/log -name "*.log" -size +100M -delete
```

---

## Pruning Directories

```bash
# Skip .git directories entirely
find . -name ".git" -prune -o -type f -print

# Skip multiple directories
find . \( -name ".git" -o -name "node_modules" -o -name ".cache" \) \
  -prune -o -type f -name "*.js" -print

# Skip by path pattern
find . -path "*/vendor/*" -prune -o -name "*.php" -print

# Stay on one filesystem (don't cross mounts)
find / -xdev -name "*.conf"
find / -mount -name "*.conf"   # same as -xdev

# Combine: skip hidden dirs and node_modules
find . \( -name ".*" -type d -o -name "node_modules" \) \
  -prune -o -type f -print

# Only search specific filesystem type
find / -fstype ext4 -name "*.log" 2>/dev/null
```

---

## Working with Symlinks

```bash
# Default: don't follow symlinks (-P is default)
find . -type l            # find symlinks themselves
find . -type l -name "*.conf"

# Follow all symlinks (-L)
find -L . -type f -name "*.conf"    # follows into symlinked dirs

# Follow only command-line symlinks (-H)
find -H /path/that/might/be/symlink -name "*.txt"

# Find broken symlinks
find . -type l ! -exec test -e {} \; -print
find -L . -type l     # with -L, broken symlinks still show as -type l

# Find symlinks and their targets
find . -type l -printf "%p -> %l\n"

# Find symlinks pointing to a specific file
find / -type l -lname "/path/to/target" 2>/dev/null
```

---

## Output Formatting

```bash
# Default: one path per line
find . -name "*.txt"

# Null-terminated (for xargs -0 or filenames with spaces)
find . -name "*.txt" -print0

# ls-style output
find . -name "*.py" -ls

# Custom format with -printf
find . -type f -printf "%s\t%p\n"          # size TAB path
find . -type f -printf "%m\t%u\t%p\n"      # perms TAB owner TAB path
find . -type f -printf "%TY-%Tm-%Td\t%p\n" # date TAB path

# Sort by modification time (newest first)
find . -type f -printf "%T@ %p\n" | sort -rn | awk '{print $2}'

# Sort by size (largest first)
find . -type f -printf "%s %p\n" | sort -rn | head -10

# CSV output
find . -type f -printf "%p,%s,%u,%m\n" > inventory.csv

# Find files and show their sizes human-readable (combine with numfmt)
find . -type f -size +1M -printf "%s\t%p\n" \
  | sort -rn \
  | awk '{printf "%-10s %s\n", $1, $2}' \
  | numfmt --to=iec --field=1

# Count files per directory
find . -type f -printf "%h\n" | sort | uniq -c | sort -rn | head -20
```

---

## Piping to xargs

```bash
# Basic: find | xargs command
find . -name "*.txt" | xargs grep "pattern"

# Safe: handle spaces and special chars with -print0 + xargs -0
find . -name "*.txt" -print0 | xargs -0 grep "pattern"

# Parallel execution (-P N for N processes)
find . -name "*.log" -print0 | xargs -0 -P 4 gzip

# Limit args per command (-n N)
find . -name "*.txt" -print0 | xargs -0 -n 10 wc -l

# With placeholder (-I)
find . -name "*.md" -print0 | xargs -0 -I{} cp {} /backup/

# Count total files
find . -type f | xargs wc -l | tail -1

# Grep across found files
find . -name "*.py" -print0 | xargs -0 grep -l "import os"

# Delete found files via xargs
find . -name "*.tmp" -print0 | xargs -0 rm -v
```

---

## Real-World Recipes

```bash
# --- Disk Cleanup ---

# Find top 10 largest files
find / -type f -printf "%s %p\n" 2>/dev/null | sort -rn | head -10

# Find and remove files not modified in 90 days from /tmp
find /tmp -type f -mtime +90 -delete

# Remove empty directories recursively
find . -depth -type d -empty -rmdir {} \;  # or -delete

# Find duplicate filenames (not content — just names)
find . -type f -printf "%f\n" | sort | uniq -d


# --- Development Cleanup ---

# Count lines of code (Python)
find . -name "*.py" ! -path "*/venv/*" | xargs wc -l | tail -1

# Find all TODO/FIXME comments
find . -name "*.py" -exec grep -Hn "TODO\|FIXME" {} \;

# Find files with Windows line endings
find . -type f -name "*.sh" -exec grep -lP "\r" {} \;

# Remove compiled Python files
find . \( -name "*.pyc" -o -name "*.pyo" \) -delete
find . -type d -name "__pycache__" -exec rm -rf {} +


# --- Security Auditing ---

# All SUID/SGID files on system
find / -type f \( -perm -4000 -o -perm -2000 \) -ls 2>/dev/null

# World-writable files (not directories)
find / -type f -perm /o+w -ls 2>/dev/null

# Files with no owner or group
find / \( -nouser -o -nogroup \) -ls 2>/dev/null

# Recently modified system files (last 24h)
find /etc /bin /sbin /usr -mtime -1 -type f -ls 2>/dev/null


# --- Backup & Sync ---

# Files modified since last backup
find /data -newer /var/backups/last_backup.timestamp -type f

# Update the timestamp after backup completes
touch /var/backups/last_backup.timestamp

# Find files to include in backup (exclude cache/tmp)
find /home \
  ! -path "*/.cache/*" \
  ! -path "*/tmp/*" \
  ! -name "*.log" \
  -type f -print0 | tar czf backup.tar.gz --null -T -


# --- System Administration ---

# Find large files across whole system
find / -xdev -type f -size +500M -ls 2>/dev/null

# Find files in /etc changed in last hour
find /etc -cmin -60 -type f

# Find open but deleted files (disk space not freed)
# (use lsof for this — find can't detect it)

# List all config files in /etc
find /etc -type f -name "*.conf" | sort

# Find scripts not marked executable
find . -name "*.sh" ! -executable -type f
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
