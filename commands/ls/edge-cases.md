# ls — Edge Cases & Gotchas

> Behaviors that surprise even experienced users.
> Know these before writing scripts or trusting ls output.

---

## Table of Contents

- [Never Parse ls in Scripts](#never-parse-ls-in-scripts)
- [Filename Traps](#filename-traps)
- [Hidden File Surprises](#hidden-file-surprises)
- [Alias Interference](#alias-interference)
- [Symlink Behavior](#symlink-behavior)
- [Sorting Surprises](#sorting-surprises)
- [Size Reporting Quirks](#size-reporting-quirks)
- [Timestamp Gotchas](#timestamp-gotchas)
- [Permissions Edge Cases](#permissions-edge-cases)
- [Recursive & Large Directories](#recursive--large-directories)
- [GNU vs BSD Differences](#gnu-vs-bsd-differences)
- [Color & Terminal Quirks](#color--terminal-quirks)
- [Special Filesystem Files](#special-filesystem-files)

---

## Never Parse ls in Scripts

This is the most important section. **Don't parse `ls` in scripts.**

```bash
# ❌ Dangerous: breaks on filenames with spaces
for file in $(ls /tmp); do
  rm "$file"
done

# ❌ Dangerous: breaks on filenames with newlines
ls | while read f; do echo "$f"; done

# ✅ Safe: use glob directly
for file in /tmp/*; do
  rm "$file"
done

# ✅ Safe: use find with -print0
find /tmp -maxdepth 1 -print0 | xargs -0 rm

# ✅ Safe: use find with -exec
find /tmp -maxdepth 1 -exec rm {} \;
```

**Why ls is dangerous in scripts:**
- Filenames can contain spaces, tabs, newlines, or any byte except `/` and null
- `ls` output is formatted for humans, not machines
- Column alignment, color codes, and sorting add noise
- `ls -l` size column is bytes for files but blocks for dirs

---

## Filename Traps

### Files with spaces
```bash
ls "my file.txt"       # ✅ shows info about this one file
ls my file.txt         # ❌ treats as two args: "my" and "file.txt"
```

### Files with newlines in names (valid on Linux!)
```bash
# Create a file with a newline in name (possible but rare)
touch $'file\nwith\nnewlines'
ls          # appears as 3 separate entries — very confusing
ls -b       # shows escape sequences: file\nwith\nnewlines
ls -q       # replaces non-printable with ?
ls -Q       # quotes the name: "file\nwith\nnewlines"
```

### Files starting with `-`
```bash
ls -file.txt           # ❌ Error: invalid option
ls -- -file.txt        # ✅ Correct
ls ./-file.txt         # ✅ Also correct
```

### Files with special characters
```bash
ls "file with 'quotes'.txt"
ls $'file\twith\ttabs'     # bash $'...' syntax for special chars
ls -b                      # shows C-style escapes for non-printable chars
ls -q                      # replaces non-printable with ?
```

### Files with unicode names
```bash
ls "données.csv"       # works in UTF-8 locale
ls "日本語.txt"         # works if locale supports it
# If locale doesn't match filename encoding:
ls -b                  # shows raw escape sequences
LANG=C ls              # byte-by-byte listing, no locale interpretation
```

---

## Hidden File Surprises

### `ls -a` includes `.` and `..`
```bash
ls -a
# .  ..  .bashrc  .config  Desktop  Documents

ls -A          # ✅ Cleaner: omits . and ..
# .bashrc  .config  Desktop  Documents
```

### Glob `.*` is dangerous
```bash
rm .*          # ❌ DANGEROUS: matches . and .. 
               # Trying to rm .. deletes parent directory contents!

ls -d .*       # safer for viewing, but still shows . and ..
ls -Ad .*      # shows but won't confuse since it's just ls
```

### Hidden files missed by globs
```bash
ls *.txt        # ❌ does NOT show .hidden.txt — bash globs don't match dotfiles by default

# Enable dotglob in bash:
shopt -s dotglob
ls *.txt        # now includes .hidden.txt

# Or use -A:
ls -A | grep '.txt$'
```

### `.git` and similar dirs
```bash
ls              # no .git shown
ls -A           # .git appears
du -sh *        # doesn't include .git contents
du -sh .        # DOES include .git (uses ., not *)
```

---

## Alias Interference

```bash
# Most systems alias ls:
alias ls='ls --color=auto'

# This means:
ls                  # runs with --color=auto
\ls                 # bypasses alias, runs raw ls
command ls          # also bypasses alias
/bin/ls             # runs binary directly

# In scripts, always use the full path or command ls
# to avoid alias side effects:
command ls /path    # safe in scripts
/bin/ls /path       # also safe

# Check what ls really is:
type -a ls
# ls is aliased to 'ls --color=auto'
# ls is /bin/ls
```

---

## Symlink Behavior

### `ls -l` on a directory shows symlink targets
```bash
ls -l /etc/localtime
# lrwxrwxrwx 1 root root 33 Jan 1 /etc/localtime -> /usr/share/zoneinfo/UTC

# The size shown (33) is the length of the target PATH, not the file size
```

### `-l` vs `-lL` for symlinks
```bash
ls -l symlink          # shows symlink info: size = length of target path string
ls -lL symlink         # follows symlink: shows target file's actual info

ls -l /etc/ssl/cert.pem       # symlink: small "size"
ls -lL /etc/ssl/cert.pem      # actual cert: real size in bytes
```

### Broken symlinks
```bash
ls broken_link         # ❌ No such file or directory
ls -l broken_link      # ✅ Shows the symlink even if target is gone
                       # colored in red with --color

# Find all broken symlinks:
find . -type l ! -exec test -e {} \; -print
```

### Symlink to directory
```bash
ls linktodir           # lists CONTENTS of the target directory
ls -d linktodir        # lists the symlink itself
ls -ld linktodir       # long format of the symlink itself
```

---

## Sorting Surprises

### Locale affects alphabetical sort
```bash
ls              # sorts by locale (LC_COLLATE)
# In en_US: uppercase and lowercase may interleave

LANG=C ls       # ASCII order: uppercase BEFORE lowercase (A-Z then a-z)
LC_COLLATE=C ls # same effect, only changes collation
```

### Version sort edge cases
```bash
ls -v
# file1.txt file2.txt file10.txt   ← correct natural order

ls              # without -v:
# file1.txt file10.txt file2.txt   ← lexicographic: 10 before 2

# Useful for log rotation files:
ls -v /var/log/syslog*
# syslog syslog.1 syslog.2 ... syslog.10
```

### `-t` sort uses mtime, not ctime
```bash
ls -lt          # sorts by modification time
                # touching metadata (chmod, chown) does NOT reorder
ls -lct         # sorts by change time (includes metadata changes)
ls -lut         # sorts by access time (when last read)
```

### `-f` disables ALL formatting
```bash
ls -f           # unsorted, enables -a, disables -l formatting and color
                # raw directory order from filesystem
```

---

## Size Reporting Quirks

### Directory size is always small
```bash
ls -lh mydir/    # shows 4.0K (the directory entry itself)
                 # NOT the total size of contents

# Total size of directory contents:
du -sh mydir/
```

### `-s` shows blocks, not bytes
```bash
ls -ls file.txt
# 8 -rw-r--r-- 1 alice staff 1234 Jun 15 file.txt
# ↑ 8 = allocated 512-byte blocks (4096 bytes on disk)
# 1234 = actual file size in bytes

ls -lsh file.txt    # block size in human-readable
```

### Sparse files
```bash
# A sparse file has "holes" — allocated less than its reported size
dd if=/dev/zero of=sparse.img bs=1 count=0 seek=1G   # 1GB sparse file

ls -lh sparse.img     # shows 1.0G (apparent size)
ls -lsh sparse.img    # blocks column shows MUCH less (actual allocation)
du -sh sparse.img     # also shows actual allocation
```

### Hard links share blocks
```bash
ln file.txt hardlink.txt
ls -ls file.txt hardlink.txt
# Both show same block count, but space is shared — not doubled
```

---

## Timestamp Gotchas

### Three different timestamps
```bash
ls -l file.txt      # shows mtime (modification time — content changed)
ls -lu file.txt     # shows atime (access time — file was read)
ls -lc file.txt     # shows ctime (change time — metadata changed)
```

### atime is often disabled
```bash
# Many systems mount with noatime or relatime for performance
# ls -lu may show stale access times
cat /proc/mounts | grep "noatime\|relatime"
```

### Times shown without year for recent files
```bash
ls -l
# Jun 15 10:23  recent.txt       ← current year omitted
# Jan  5  2023  old.txt          ← year shown for older files

ls -l --full-time    # always shows full year + time
ls -l --time-style=long-iso
```

### Copying doesn't preserve mtime by default
```bash
cp file.txt copy.txt
ls -lt              # copy.txt is "newest" — has current time
cp -p file.txt copy.txt    # -p preserves timestamps
```

---

## Permissions Edge Cases

### Execute bit on directories means "enter" not "run"
```bash
ls -ld mydir/
# drwxr-x--  ← x on directory = permission to cd into it and stat files
# Without x: you can't cd into dir even if you can see file names with r
```

### Sticky bit (`t`) on directories
```bash
ls -ld /tmp
# drwxrwxrwt  ← t = sticky bit: only file owner can delete their files
# Shown as 't' if x is also set, 'T' if not
```

### SUID/SGID bits
```bash
ls -l /usr/bin/passwd
# -rwsr-xr-x  ← s in owner execute position = SUID (runs as file owner)

ls -l /usr/bin/wall
# -rwxr-sr-x  ← s in group execute position = SGID
```

### ACLs (Access Control Lists)
```bash
ls -l file.txt
# -rw-r--r--+ alice staff 1234 file.txt
#            ↑ + means ACL is set — use getfacl to see it
getfacl file.txt
```

---

## Recursive & Large Directories

### `ls -R` on / is catastrophic
```bash
ls -R /        # ❌ Millions of files, never stops cleanly
ls -R /etc     # ✅ OK for small subtrees
```

### Directories with millions of files
```bash
ls /huge_dir           # slow — calls stat() on every entry
ls -f /huge_dir        # fast — skips stat() and sorting
ls -U /huge_dir        # same as -f for directory order
find /huge_dir -maxdepth 1 | wc -l   # count without listing
```

### `ls -R` doesn't show the path clearly
```bash
ls -R /etc
# /etc:
# hosts  passwd  ...
# /etc/ssh:
# sshd_config  ...

# Cleaner alternative:
find /etc -ls
tree /etc
```

---

## GNU vs BSD Differences

| Feature | GNU (Linux) | BSD (macOS) |
|---------|------------|-------------|
| `--color` | `--color=auto` | `-G` |
| `--group-directories-first` | ✅ | ❌ |
| `-v` (version sort) | ✅ | ❌ |
| `--full-time` | ✅ | ❌ |
| `--time-style` | ✅ | ❌ |
| `--hide=` | ✅ | ❌ |
| `-Z` (SELinux) | ✅ | ❌ |
| `-@` (xattrs) | ❌ | ✅ |
| `-e` (ACLs) | ❌ | ✅ |
| `-T` (full time) | ❌ | ✅ (on macOS) |
| `-b` (octal escapes) | ✅ | ✅ |

**Cross-platform scripts:**
```bash
# Detect OS and use appropriate flags
if ls --color=auto &>/dev/null 2>&1; then
  alias ls='ls --color=auto'   # GNU
else
  alias ls='ls -G'              # BSD/macOS
fi
```

---

## Color & Terminal Quirks

### Colors break when piping
```bash
ls --color=always | less    # color codes appear as raw escape sequences
ls --color=auto | less      # ✅ no color (auto detects non-terminal)
ls | less -R                # no color from ls, but less handles them
ls --color=always | less -R # ✅ color preserved in less
```

### Terminals that can't handle color
```bash
\ls              # bypass alias, no color
ls --color=never
TERM=dumb ls     # simulate dumb terminal
```

### `LS_COLORS` not set
```bash
# If LS_COLORS is empty, ls uses built-in defaults
# Reinitialize:
eval $(dircolors -b)
eval $(dircolors ~/.dircolors)   # custom config
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
