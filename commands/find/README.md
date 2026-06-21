# find — The Complete Reference

> **Search for files in a directory hierarchy**
> The most powerful file-searching tool in Unix — not just for finding files,
> but for acting on them. A mini-language of its own.

---

## Table of Contents

- [What is find?](#what-is-find)
- [Where does find live?](#where-does-find-live)
- [How find works internally](#how-find-works-internally)
- [Syntax](#syntax)
- [Starting Points](#starting-points)
- [Tests (Filters)](#tests-filters)
- [Actions](#actions)
- [Operators](#operators)
- [Options (Global Behaviors)](#options-global-behaviors)
- [Output Formatting with -printf](#output-formatting-with--printf)
- [find vs locate vs fd](#find-vs-locate-vs-fd)
- [Related Commands](#related-commands)

---

## What is find?

`find` traverses a directory tree and evaluates expressions against each file it encounters. Every file either **matches** (expression is true) or **doesn't**. Matched files can be printed, deleted, passed to another command, or processed with any shell command.

Unlike `ls` or `grep`, `find` is not just a search tool — it's a **file processing engine**. You describe what you're looking for (name, type, size, time, permissions, ownership...) and what to do with it (print, delete, exec, copy...).

First appeared in Unix Version 7 (1979), written by Dick Haight. The GNU version (`findutils`) is used on Linux; BSD `find` ships with macOS with slightly different syntax.

---

## Where does find live?

```
/usr/bin/find       ← most Linux systems
/bin/find           ← some distros (symlink to /usr/bin/find)
/usr/bin/find       ← macOS (BSD version)
```

```bash
which find
type find
find --version      # GNU findutils version (Linux only)
```

`find` is part of **GNU findutils** on Linux (alongside `locate`, `xargs`, `updatedb`).

Check version:
```bash
find --version
# find (GNU findutils) 4.9.0
```

---

## How find works internally

```
for each starting_point:
  walk directory tree (depth-first by default)
    for each file encountered:
      evaluate expression (tests + operators)
      if expression is TRUE → run action
      if expression is FALSE → skip
```

**Traversal order:**
- Default: **pre-order depth-first** (parent before children)
- `-depth`: **post-order** (children before parent) — required for `rmdir` and some `chmod` operations
- `-breadth-first`: not a standard option; use `find` + sort tricks or `fd`

**What "expression" means:**
Every argument after the starting points is part of the **expression**. Tests return true/false. Actions return true/false and have side effects. Operators combine them (`-and`, `-or`, `-not`).

Default action: if no action is specified and the expression is true → `-print` is implied.

**Inode traversal:** `find` uses `openat()`, `getdents()`, and `fstatat()` system calls — directly reading directory entries from the kernel without going through shell glob expansion.

**Symlinks:** By default find does NOT follow symlinks (`-P` is default). Use `-L` to follow them. Use `-H` to follow only those on the command line.

**Exit codes:**
| Code | Meaning |
|------|---------|
| `0` | All files processed successfully |
| `1` | Some files could not be processed |

---

## Syntax

```
find [-H] [-L] [-P] [-D debugopts] [-Olevel] [starting-point...] [expression]
```

Simplified mental model:
```
find [WHERE] [WHAT] [HOW]
```

- **WHERE**: starting directories (default: `.`)
- **WHAT**: tests that filter files (`-name`, `-type`, `-size`, `-mtime`...)
- **HOW**: actions to take (`-print`, `-exec`, `-delete`...)

---

## Starting Points

```bash
find .                    # current directory (recursive)
find /                    # entire filesystem
find /home /var /tmp      # multiple starting points
find ~                    # home directory
find /etc /var/log        # multiple dirs

# No starting point = current directory (GNU find only)
find -name "*.txt"        # same as: find . -name "*.txt"
```

---

## Tests (Filters)

### By Name

```bash
-name "pattern"           # filename only, case-sensitive, shell glob
-iname "pattern"          # case-insensitive name
-path "pattern"           # full path match (including dirs), case-sensitive
-ipath "pattern"          # case-insensitive path
-regex "pattern"          # full path matches ERE regex
-iregex "pattern"         # case-insensitive regex
-wholename "pattern"      # same as -path (GNU alias)
```

Glob patterns supported in `-name`:
- `*` — any string
- `?` — any single character
- `[abc]` — character class

```bash
find . -name "*.txt"
find . -iname "readme*"
find . -name "file?.log"
find . -path "*/config/*.yaml"
find . -regex ".*/[0-9]+\.log"
```

### By Type

```bash
-type f      # regular file
-type d      # directory
-type l      # symbolic link
-type b      # block device
-type c      # character device
-type p      # named pipe (FIFO)
-type s      # socket
-type D      # door (Solaris only)
```

```bash
find . -type f            # only regular files
find . -type d            # only directories
find . -type l            # only symlinks
find /dev -type b         # block devices
find /run -type s         # sockets
```

### By Size

```bash
-size n[cwbkMG]
```

| Suffix | Unit |
|--------|------|
| `c` | bytes (characters) |
| `w` | 2-byte words |
| `b` | 512-byte blocks (default) |
| `k` | kibibytes (1024 bytes) |
| `M` | mebibytes (1024² bytes) |
| `G` | gibibytes (1024³ bytes) |

```bash
find . -size 100c         # exactly 100 bytes
find . -size +100M        # larger than 100 MiB
find . -size -1k          # smaller than 1 KiB
find . -size +1G          # larger than 1 GiB
find . -size +10M -size -100M   # between 10MB and 100MB
find . -size 0            # empty files (0 blocks)
find . -empty             # empty files AND empty directories
```

### By Time

Three timestamps: **mtime** (content modified), **atime** (accessed), **ctime** (metadata changed).

```bash
# Days ago (integer, rounds down to 24h boundaries)
-mtime n      # modified exactly n days ago
-mtime +n     # modified more than n days ago
-mtime -n     # modified less than n days ago
-atime n/+n/-n
-ctime n/+n/-n

# Minutes ago
-mmin n/+n/-n
-amin n/+n/-n
-cmin n/+n/-n

# Relative to a reference file
-newer file       # mtime newer than file's mtime
-anewer file      # atime newer than file's mtime
-cnewer file      # ctime newer than file's mtime
-newerXY ref      # compare specific timestamps (GNU)
```

```bash
find . -mtime -1          # modified in last 24 hours
find . -mtime +30         # not modified in over 30 days
find . -mtime +7 -mtime -14    # modified 7-14 days ago
find . -mmin -60          # modified in last 60 minutes
find . -newer /etc/passwd # modified more recently than /etc/passwd
find . -newermt "2024-01-01"   # modified after Jan 1 2024 (GNU)
find . -newermt "2024-01-01" ! -newermt "2024-12-31"  # whole year
```

### By Permissions

```bash
-perm mode        # exact permissions match
-perm -mode       # ALL listed bits must be set (AND)
-perm /mode       # ANY listed bit must be set (OR) — GNU
-perm +mode       # old syntax for OR (deprecated, use /)
```

```bash
find . -perm 644             # exactly rw-r--r--
find . -perm -644            # at least rw-r--r-- (may have more)
find . -perm /111            # any execute bit set (owner OR group OR other)
find . -perm -111            # all execute bits set
find . -perm -4000           # SUID bit set
find . -perm -2000           # SGID bit set
find . -perm -1000           # sticky bit set
find . -perm /6000           # SUID OR SGID
find . -perm 777             # world-writable+executable (security audit)
find / -perm -4000 -type f   # all SUID files on system
```

### By Ownership

```bash
-user name        # owned by user (name or UID)
-group name       # owned by group (name or GID)
-uid n            # owned by UID n
-gid n            # owned by GID n
-nouser           # no user matches UID (orphaned files)
-nogroup          # no group matches GID
```

```bash
find /home -user alice
find /tmp -user root
find . -group developers
find . -uid 1001
find / -nouser -type f        # orphaned files (user deleted)
find / -nogroup -type f
```

### By Depth

```bash
-maxdepth n       # descend at most n levels (0 = starting point only)
-mindepth n       # don't apply tests at levels less than n
```

```bash
find . -maxdepth 1            # current directory only (no recursion)
find . -maxdepth 2 -name "*.conf"
find . -mindepth 2 -maxdepth 3   # only between 2-3 levels deep
find / -maxdepth 0            # the root directory itself only
```

### By Inode & Links

```bash
-inum n           # inode number
-samefile file    # same inode as file (hard links)
-links n          # n hard links
-links +1         # more than one hard link (has hard link copies)
```

```bash
find . -inum 123456
find . -samefile original.txt    # find all hard links to this file
find . -links +1 -type f         # files with hard links
```

### Other Tests

```bash
-empty            # empty file or empty directory
-executable       # executable by current user (respects ACLs)
-readable         # readable by current user
-writable         # writable by current user
-fstype type      # filesystem type (ext4, nfs, tmpfs...)
-xdev             # don't cross filesystem boundaries (same as -mount)
-mount            # same as -xdev
```

---

## Actions

### Print Actions

```bash
-print            # print path + newline (default when no action given)
-print0           # print path + null byte (safe for xargs -0)
-ls               # print in ls -dils format
-printf format    # custom formatted output (see next section)
-fprint file      # print to file instead of stdout
-fprint0 file     # print with null separator to file
```

### Execute Actions

```bash
-exec command {} \;       # run command once per file ({} = filename)
-exec command {} +        # run command with multiple files batched
-execdir command {} \;    # run from file's directory (safer)
-execdir command {} +     # batched version
-ok command {} \;         # like -exec but prompts before each
-okdir command {} \;      # like -execdir but prompts
```

**`\;` vs `+`:**
```bash
# \; runs once per file:
find . -name "*.log" -exec gzip {} \;
# → gzip a.log
# → gzip b.log
# → gzip c.log

# + batches all files into one call (like xargs):
find . -name "*.log" -exec gzip {} +
# → gzip a.log b.log c.log
# Faster! Fewer process spawns.
```

### Delete Action

```bash
-delete           # delete matched files (implies -depth)
```

```bash
find . -name "*.tmp" -delete
find . -type d -empty -delete    # delete empty directories
```

> ⚠️ `-delete` is irreversible. Always test with `-print` first.

### Pruning

```bash
-prune            # don't descend into matched directory
```

```bash
# Skip .git directories
find . -name ".git" -prune -o -name "*.py" -print

# Skip multiple directories
find . \( -name ".git" -o -name "node_modules" -o -name "__pycache__" \) -prune \
     -o -name "*.js" -print
```

---

## Operators

```bash
expr1 -and expr2      # both must be true (default when no operator given)
expr1 expr2           # same as -and
expr1 -or expr2       # either must be true
! expr                # negate
-not expr             # same as !
\( expr \)            # grouping (must escape parens in shell)
```

```bash
# Files that are txt OR md
find . \( -name "*.txt" -o -name "*.md" \)

# Large files that are NOT logs
find . -size +100M ! -name "*.log"

# Python files modified in last week, not in venv
find . -name "*.py" -mtime -7 ! -path "*/venv/*"

# Empty files OR empty directories
find . \( -empty -type f \) -o \( -empty -type d \)
```

**Short-circuit evaluation:**
- `-and`: if left is false, right is not evaluated
- `-or`: if left is true, right is not evaluated
- This matters for performance and for `-exec` side effects

---

## Options (Global Behaviors)

These must come **before** the starting point or expression:

```bash
-P        # never follow symlinks (default)
-L        # follow symlinks (affects -type, -size, all tests)
-H        # follow symlinks only on command line, not during traversal
-D flags  # debug output (exec, opt, rates, search, stat, tree, all)
-O0-3     # optimization level (default: 1)
```

```bash
find -L /etc -name "*.conf"     # follow symlinks in /etc
find -H /path/to/symlink -type f   # follow only the starting symlink
```

---

## Output Formatting with -printf

`-printf` is one of find's most powerful and underused features.

**Format directives:**

| Directive | Meaning |
|-----------|---------|
| `%p` | File path |
| `%f` | Filename only (no directory) |
| `%h` | Directory part only |
| `%P` | Path relative to starting point |
| `%s` | File size in bytes |
| `%k` | File size in 1KB blocks |
| `%m` | Permission bits (octal) |
| `%M` | Permission string (like ls -l) |
| `%u` | Owner name |
| `%g` | Group name |
| `%U` | Owner UID |
| `%G` | Group GID |
| `%i` | Inode number |
| `%l` | Symlink target |
| `%t` | mtime (ctime format) |
| `%T@` | mtime (seconds since epoch) |
| `%A@` | atime (seconds since epoch) |
| `%C@` | ctime (seconds since epoch) |
| `%TY-%Tm-%Td` | mtime as YYYY-MM-DD |
| `%n` | Hard link count |
| `%y` | File type letter (f d l b c p s) |
| `%d` | Depth in tree |
| `\n` | Newline |
| `\0` | Null byte |
| `\t` | Tab |

```bash
# Filename and size
find . -type f -printf "%f\t%s\n"

# Full path and octal permissions
find . -type f -printf "%m %p\n"

# CSV output: path, size, mtime
find . -type f -printf "%p,%s,%TY-%Tm-%Td\n"

# Sort files by modification time
find . -type f -printf "%T@ %p\n" | sort -n | awk '{print $2}'

# Find oldest file
find . -type f -printf "%T@ %p\n" | sort -n | head -1 | awk '{print $2}'

# Files with their depth
find . -printf "%d %p\n" | sort -n

# Null-terminated for xargs
find . -name "*.txt" -printf "%p\0" | xargs -0 grep "pattern"
```

---

## find vs locate vs fd

| Feature | `find` | `locate` | `fd` |
|---------|--------|----------|------|
| Source | Live filesystem scan | Pre-built database | Live filesystem scan |
| Speed | Slower (real-time) | Very fast | Fast (parallel) |
| Accuracy | Always current | Depends on `updatedb` run | Always current |
| Syntax | Complex but complete | Simple glob/regex | Simple, sane defaults |
| Respects .gitignore | No | No | Yes |
| Follow symlinks | `-L` flag | Depends on updatedb | `-L` flag |
| Actions (`-exec`) | Yes | No | Yes |
| Available by default | Yes | Often not | No (install separately) |

```bash
# locate: fast but may be stale
locate "*.conf"
locate -i "readme"        # case-insensitive
updatedb                  # refresh database (as root)

# fd: modern alternative
fd "pattern"              # search by name
fd -e txt                 # by extension
fd -t f "pattern"         # only files
fd -H "pattern"           # include hidden
fd --no-ignore "pattern"  # don't respect .gitignore
```

---

## Related Commands

| Command | Relation |
|---------|---------|
| `xargs` | Pass find output as arguments to another command |
| `locate` | Fast file search using pre-built database |
| `fd` | Modern, faster, user-friendly alternative to find |
| `grep -r` | Search file *contents* recursively |
| `stat` | Detailed metadata of a single file |
| `du` | Disk usage — often combined with find |
| `ls` | List directory contents (no recursion by default) |
| `updatedb` | Rebuild locate database |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
