# ls — Practical Examples

> Real-world usage patterns, from everyday to advanced.

---

## Table of Contents

- [Basic Listing](#basic-listing)
- [Long Format Deep Dive](#long-format-deep-dive)
- [Hidden Files](#hidden-files)
- [Sorting](#sorting)
- [File Sizes](#file-sizes)
- [Timestamps](#timestamps)
- [Special Files & Types](#special-files--types)
- [Recursive Listing](#recursive-listing)
- [Filtering & Patterns](#filtering--patterns)
- [Permissions & Ownership](#permissions--ownership)
- [Symlinks](#symlinks)
- [Scripting with ls](#scripting-with-ls)
- [Useful Aliases](#useful-aliases)

---

## Basic Listing

```bash
# Current directory
ls

# Specific directory
ls /etc
ls /var/log
ls ~              # home directory
ls ~/Downloads

# Multiple directories at once
ls /etc /var /tmp

# Specific file (shows info about it)
ls file.txt

# One item per line
ls -1
ls -1 /etc

# Comma-separated
ls -m
```

---

## Long Format Deep Dive

```bash
# Long format
ls -l

# Long + human-readable sizes
ls -lh

# Long + hidden files
ls -la
ls -lA           # hidden but not . and ..

# Long + full timestamp
ls -l --full-time
# Output: 2024-06-15 10:23:45.123456789 +0000

# Long with custom time format
ls -l --time-style='+%Y-%m-%d %H:%M'

# Long + inode numbers
ls -li

# Long + numeric UID/GID (no name lookup — faster on large dirs)
ls -ln

# Long + file type indicator
ls -lF

# Long without group column
ls -lo

# Long without owner column
ls -lg

# Directories first
ls -l --group-directories-first

# Full example combining flags
ls -lAh --group-directories-first --time-style=long-iso
```

**Reading the long format output:**
```
drwxr-xr-x  2  alice  developers   4096  2024-06-15  projects/
-rw-r--r--  1  alice  developers   1234  2024-06-14  notes.txt
lrwxrwxrwx  1  alice  developers      7  2024-06-13  link -> target
-rwxr-xr-x  1  root   root         8192  2024-06-10  script.sh
```

---

## Hidden Files

```bash
# Show ALL files including . and ..
ls -a

# Show hidden files but NOT . and ..  (cleaner)
ls -A

# Long format + hidden
ls -lA

# Only hidden files (glob trick)
ls -ld .*

# Hidden files in a specific directory
ls -A ~/.config
ls -A ~/.ssh

# Show dotfiles alongside regular files, dirs first
ls -lA --group-directories-first
```

---

## Sorting

```bash
# Alphabetical (default)
ls

# Newest first (modification time)
ls -lt
ls -lth          # + human-readable sizes

# Oldest first
ls -ltr

# Largest file first
ls -lS
ls -lSh

# Smallest file first
ls -lSr

# By extension
ls -lX

# Natural version sort (handles file2.txt before file10.txt)
ls -lv
ls -v

# Unsorted (raw filesystem order — fastest, useful for huge dirs)
ls -f
ls -1f

# Reverse any sort
ls -lr           # reverse alpha
ls -ltr          # reverse time (oldest first)
ls -lSrh         # reverse size (smallest first)

# Directories first, then alphabetical
ls -l --group-directories-first
```

---

## File Sizes

```bash
# Human-readable (powers of 1024: K, M, G, T)
ls -lh

# Human-readable (powers of 1000)
ls -lH

# Show allocated disk blocks
ls -ls

# Specific block size
ls -l --block-size=MB       # megabytes
ls -l --block-size=KB       # kilobytes
ls -l --block-size=1        # raw bytes

# Sort by size + human-readable
ls -lSh

# Find the 10 largest files in current directory
ls -lSh | head -11   # +1 for the "total" line

# Total size of directory contents (not recursive — use du for that)
ls -l | awk '{sum += $5} END {print sum " bytes"}'
```

---

## Timestamps

```bash
# Modification time (default in -l)
ls -l

# Access time (when file was last read)
ls -lu
ls -lu file.txt

# Change time (when metadata/permissions changed)
ls -lc
ls -lc file.txt

# Sort by access time
ls -lut

# Sort by change time
ls -lct

# Full nanosecond timestamp
ls -l --full-time

# Custom format
ls -l --time-style='+%Y-%m-%d %H:%M:%S'

# ISO format
ls -l --time-style=long-iso
ls -l --time-style=iso
```

---

## Special Files & Types

```bash
# Show file type indicator (/ * @ = |)
ls -F

# What each indicator means:
# /  → directory
# *  → executable
# @  → symlink
# =  → socket
# |  → named pipe (FIFO)
# >  → door (Solaris)

ls -lF /etc     # see directories with /

# List only directories
ls -d */         # current dir (glob)
ls -ld /etc/*/   # subdirs of /etc

# List only executables (glob + -F)
ls -F | grep '\*$'

# List only symlinks
ls -lA | grep '^l'

# View device files
ls -l /dev | head -30
ls -l /dev/sd*   # disk devices
ls -l /dev/tty*  # terminal devices

# View named pipes
ls -l /var/run/*.pid
ls -lF | grep '|$'

# View sockets
ls -lF /run | grep '=$'
ls -lA /tmp | grep '^s'

# Kernel virtual filesystems
ls /proc
ls /proc/1/       # init process
ls /sys/class/net # network interfaces

# Files with SELinux context (Linux)
ls -Z /etc/passwd
ls -lZ /etc/
```

---

## Recursive Listing

```bash
# Recursive (can be very verbose)
ls -R

# Recursive + long format
ls -lR

# Recursive + long + human-readable
ls -lRh /var/log

# Recursive with depth limit — ls doesn't support it, use find:
find . -maxdepth 2 -ls

# Or use tree (cleaner for recursive view)
tree
tree -L 2           # max 2 levels
tree -a             # include hidden
tree -h             # human-readable sizes
tree -d             # directories only
```

---

## Filtering & Patterns

```bash
# Files matching a glob pattern
ls *.txt
ls *.log
ls *.{txt,md,sh}
ls file?.txt         # ? matches single char

# Files in a range
ls report_202[34]*.pdf

# Hide files matching pattern (GNU)
ls --hide="*.pyc"
ls --hide="__pycache__"

# Ignore pattern (like hide but stronger)
ls --ignore="*.swp"
ls --ignore=".git"

# Case-insensitive glob (bash: set nocaseglob)
shopt -s nocaseglob
ls *.TXT             # matches .txt, .TXT, .Txt

# List only directories
ls -d */

# List only files (no dirs)
ls -p | grep -v /

# List only hidden files
ls -Ad .*
```

---

## Permissions & Ownership

```bash
# See permissions in long format
ls -l

# See numeric permissions (useful for chmod)
stat -c "%a %n" *        # better than ls for this

# Files readable by others
ls -l | grep '^-..r'

# World-writable files (security risk)
ls -l | grep '......w...'

# SUID files (run as owner)
ls -l | grep '^-..s'
find / -perm -4000 -ls 2>/dev/null   # better

# Show owner + group
ls -l file.txt

# Only numeric UID/GID
ls -ln file.txt

# See SELinux context
ls -Z

# List files owned by specific user
ls -l | awk '$3 == "alice"'
```

---

## Symlinks

```bash
# Show symlink and its target
ls -l link.txt
# lrwxrwxrwx 1 alice staff 7 Jun 15 link.txt -> target.txt

# Follow symlink — show target's info
ls -lL link.txt

# Find broken symlinks
ls -lA | grep '^l'        # all symlinks
# broken ones appear in red with --color

# Better: find broken symlinks
find . -type l ! -exec test -e {} \; -print

# Show symlinks in a directory
ls -lA | grep '^l'

# Dereference on command line only
ls -lH /path/that/is/a/symlink/
```

---

## Scripting with ls

```bash
# Count files in directory
ls | wc -l

# Count only files (not dirs)
ls -p | grep -v / | wc -l

# Count only directories
ls -d */ | wc -l

# Get newest file
ls -t | head -1

# Get oldest file
ls -tr | head -1

# Get largest file
ls -S | head -1

# List files modified in last 24h (better with find)
find . -maxdepth 1 -mtime -1 -ls

# Process each file
for f in *.log; do
  echo "Processing: $f"
  gzip "$f"
done

# Check if directory is empty
if [ -z "$(ls -A /path/to/dir)" ]; then
  echo "Directory is empty"
fi

# Get file size from ls (column 5 in long format)
ls -l file.txt | awk '{print $5}'
# Better alternative:
stat -c%s file.txt
wc -c < file.txt
```

> ⚠️ **Note:** Parsing `ls` output in scripts is fragile — filenames can contain spaces, newlines, special characters. Prefer `find`, `stat`, or bash globbing for serious scripting.

---

## Useful Aliases

Common aliases to add to `~/.bashrc` or `~/.zshrc`:

```bash
# The essentials
alias ll='ls -lh'
alias la='ls -lAh'
alias l='ls -CF'
alias lt='ls -lth'          # newest first
alias ltr='ls -ltrh'        # oldest first
alias lS='ls -lSh'          # largest first

# Directories first
alias lld='ls -lAh --group-directories-first'

# With file type indicators
alias lF='ls -lhF'

# Only directories
alias lsd='ls -d */'

# Tree alternative
alias lst='tree -L 2'

# Full timestamp
alias llt='ls -lh --full-time'

# No color (for piping)
alias lnc='ls --color=never'
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
