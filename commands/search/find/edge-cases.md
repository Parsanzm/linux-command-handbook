# find — Edge Cases & Gotchas

> find has more surprises than almost any other Unix command.
> These are the traps that catch even experienced sysadmins.

---

## Table of Contents

- [The -delete Trap](#the--delete-trap)
- [Time Arithmetic Surprises](#time-arithmetic-surprises)
- [Size Units Are Not What You Think](#size-units-are-not-what-you-think)
- [Permission Matching Is Subtle](#permission-matching-is-subtle)
- [-exec vs -execdir Security](#-exec-vs--execdir-security)
- [Prune Doesn't Work the Way You Think](#prune-doesnt-work-the-way-you-think)
- [Symlink Behavior Pitfalls](#symlink-behavior-pitfalls)
- [Filename Traps](#filename-traps)
- [Operator Precedence](#operator-precedence)
- [Default Action Confusion](#default-action-confusion)
- [Cross-Filesystem Surprises](#cross-filesystem-surprises)
- [GNU vs BSD Differences](#gnu-vs-bsd-differences)
- [Performance Traps](#performance-traps)
- [stderr Floods](#stderr-floods)

---

## The -delete Trap

### -delete implies -depth (post-order traversal)
```bash
# This changes the ORDER of traversal silently
find . -type d -name "cache" -delete
# find processes children BEFORE parents when -delete is active
# This is necessary so directories are empty before rmdir is attempted

# But it can cause -prune to not work as expected:
find . -name ".git" -prune -o -type f -delete
# ⚠️ -delete overrides -depth behavior; -prune may not fire first
# Safer: separate the steps
find . -name ".git" -prune -o -type f -print0 | xargs -0 rm
```

### -delete on non-empty directory
```bash
find . -type d -name "logs" -delete
# ❌ Error: Directory not empty
# -delete uses rmdir() not rm -rf
# Only works on empty directories

# To delete non-empty dirs:
find . -type d -name "logs" -exec rm -rf {} +
# or use -depth to clear contents first:
find . -depth -name "logs" -delete   # still only works if contents matched too
```

### -delete returns true even if delete fails
```bash
# -delete returns true even on permission denied
# This can cause -and chaining to behave unexpectedly
find . -name "*.tmp" -delete -print
# Prints even if delete failed (print runs because delete "succeeded" in expression)
```

### Always test before -delete
```bash
# Rule: replace -delete with -print first
find . -name "*.tmp" -print    # test
find . -name "*.tmp" -delete   # then delete
```

---

## Time Arithmetic Surprises

### -mtime uses 24-hour boundaries, not "last N days"
```bash
find . -mtime -1
# NOT "files modified in the last 24 hours"
# Actually: files where (current_time - mtime) / 86400 < 1
# A file modified 23 hours ago: (23*3600)/86400 = 0.958 → rounds to 0 → matches -mtime -1 ✅
# A file modified 25 hours ago: (25*3600)/86400 = 1.04 → rounds to 1 → does NOT match -mtime -1

# For true "last 24 hours", use -mmin:
find . -mmin -1440     # exactly 24 * 60 minutes
```

### -mtime 0 means "today" (kind of)
```bash
find . -mtime 0
# Matches files modified between 0 and 24 hours ago (current 24h boundary)
# NOT "midnight to now"
```

### -mtime +N vs -mtime N
```bash
find . -mtime +7    # strictly MORE than 7 days ago (age > 7*86400)
find . -mtime 7     # EXACTLY 7 days ago (rounding included)
find . -mtime -7    # LESS than 7 days ago

# Common mistake: "older than a week" should be -mtime +7 not -mtime 7
```

### -newer uses mtime of reference file
```bash
find . -newer reference.txt
# Compares mtime of found files with MTIME of reference.txt
# NOT with atime or ctime of reference.txt
# Use -newerXY for specific timestamp comparisons (GNU):
find . -newerat reference.txt   # compare mtime to atime of reference
find . -newermt "2024-01-01"    # compare mtime to given date
```

---

## Size Units Are Not What You Think

### Default unit is 512-byte blocks, not bytes
```bash
find . -size 1
# Matches files using 1 block = 512 bytes
# NOT files of 1 byte!

find . -size 1c   # 1 byte ← need 'c' suffix for bytes
find . -size 1k   # 1 kibibyte (1024 bytes)
find . -size 1M   # 1 mebibyte (1024² = 1,048,576 bytes)
```

### -size compares allocated blocks, not apparent size
```bash
# A 1-byte file still uses one 4096-byte block on most filesystems
find . -size +0c -size -1k
# Might not find a 1-byte file! It occupies a full block

# Use -empty for truly empty files (0 bytes)
find . -empty -type f   # files with 0 bytes

# -size 0 finds files in 0 blocks (same as -empty for files)
find . -size 0

# Sparse files report apparent size, not allocated:
dd if=/dev/zero of=sparse bs=1 count=0 seek=1G
find . -size +1G -name "sparse"   # ✅ matches (apparent size = 1G)
du -sh sparse                      # shows much less (actual allocation)
```

### k is kibibytes (1024), not kilobytes (1000)
```bash
find . -size +1k    # > 1024 bytes, not > 1000 bytes
# GNU find uses binary units:
# k = 1024, M = 1024², G = 1024³
```

---

## Permission Matching Is Subtle

### -perm mode (exact) vs -perm -mode (at least) vs -perm /mode (any)
```bash
find . -perm 644      # EXACT match: must be exactly rw-r--r--
                      # Won't match 755, 664, or 755

find . -perm -644     # AT LEAST 644: all bits in 644 must be set
                      # Matches 644, 755, 774, 777 but NOT 600 or 444

find . -perm /644     # ANY of the bits in 644 must be set
                      # Matches if owner-read OR group-read OR other-read is set
                      # Very broad — matches almost everything
```

### SUID/SGID in octal
```bash
# SUID = 4000, SGID = 2000, sticky = 1000
find / -perm -4000    # SUID set (4 in first octal digit)
find / -perm -6000    # BOTH SUID and SGID set

# Common mistake: -perm 4755 requires EXACT match
find / -perm 4755     # only rwsr-xr-x exactly
find / -perm -4000    # any file with SUID set ← usually what you want
```

### Uppercase S and T in permissions
```bash
# -rwSr--r--  ← S means SUID set but execute NOT set for owner
# drwxrwxrwT  ← T means sticky set but execute NOT set for others
# find -perm matches the underlying bits regardless of case display
find . -perm -4000    # matches both s and S cases
```

---

## -exec vs -execdir Security

### -exec with {} is vulnerable to race conditions (TOCTOU)
```bash
find /tmp -name "*.sh" -exec chmod +x {} \;
# Between find identifying the file and exec running, an attacker
# could replace the file with a symlink to /etc/passwd

# Safer: use -execdir (runs from the file's directory)
find /tmp -name "*.sh" -execdir chmod +x {} \;
# chmod +x ./thefile  ← relative path, not absolute
```

### -exec {} + with many files can exceed ARG_MAX
```bash
find . -name "*.log" -exec gzip {} +
# If there are millions of files, this may still hit ARG_MAX
# find handles this by breaking into multiple calls automatically
# But if your command has a fixed overhead, test with a small set first
```

### Shell expansion in -exec
```bash
# ❌ Shell variables and globs don't work in -exec directly
find . -name "*.txt" -exec echo "File: $PWD/{}" \;
# $PWD won't expand! -exec doesn't use a shell

# ✅ Use sh -c:
find . -name "*.txt" -exec sh -c 'echo "File: $PWD/$1"' _ {} \;

# ✅ Or use while read:
find . -name "*.txt" -print0 | while IFS= read -r -d '' f; do
  echo "File: $PWD/$f"
done
```

---

## Prune Doesn't Work the Way You Think

### -prune must be in the right position
```bash
# ❌ Wrong: -name test runs on .git dir too before prune can fire
find . -name "*.py" -o -name ".git" -prune

# ✅ Correct: match .git first, then OR with the real search
find . -name ".git" -prune -o -name "*.py" -print

# The pattern is always:
# find . [EXCLUDE] -prune -o [INCLUDE] -print
```

### -prune only prevents descent, doesn't hide the directory
```bash
find . -name ".git" -prune -o -type f -print
# ⚠️ This STILL prints ".git" itself (as a directory match before pruning)

# To suppress the pruned directory from output:
find . -name ".git" -prune -o -type f -print
# .git appears because it matches before -prune fires and -print is implied

# Truly suppress:
find . -name ".git" -prune -o -type f -print
# Actually this is correct — .git is NOT printed because
# -prune returns FALSE, so the -print action doesn't fire
# The -o means: if -name ".git" -prune, don't do -print for it

# If it's still confusing: test in your specific find version
```

### -prune with -depth (and -delete) is incompatible
```bash
find . -name "node_modules" -prune -o -type f -delete
# ⚠️ -delete implies -depth, which conflicts with -prune
# Result: -prune is ignored, find descends into node_modules anyway

# Safer: two-step
find . -name "node_modules" -prune -o -type f -print0 | xargs -0 rm
```

---

## Symlink Behavior Pitfalls

### Default (-P): symlinks to directories are not traversed
```bash
find . -name "*.conf"
# If /etc/nginx is a symlink to /usr/local/etc/nginx,
# find will NOT descend into it by default

find -L . -name "*.conf"    # -L: follow all symlinks
```

### -L can cause infinite loops
```bash
find -L / -name "*.conf"
# If there are circular symlinks: A → B → A
# find detects cycles and warns, but may still be slow

# find prints: "File system loop detected"
# It won't infinite-loop, but it will be slower
```

### -type with -L changes meaning
```bash
find . -type l              # without -L: finds symlinks
find -L . -type l           # with -L: only finds BROKEN symlinks
                            # because -L follows working ones (they become f or d)

# Find broken symlinks:
find -L . -type l           # ← this is the idiom
```

### -L and -delete on symlinks
```bash
find -L . -type f -name "*.tmp" -delete
# If a .tmp is a symlink to a real file, -delete removes the TARGET
# Not the symlink! Because -L follows it.

# To delete the symlinks themselves:
find . -type l -name "*.tmp" -delete   # no -L
```

---

## Filename Traps

### Filenames with spaces
```bash
find . -name "my file.txt"   # ✅ -name handles spaces fine
find . -name "my file.txt" -exec cat {} \;   # ✅ {} handles spaces

# Danger is in piping:
find . -name "*.txt" | xargs cat    # ❌ breaks on spaces
find . -name "*.txt" -print0 | xargs -0 cat   # ✅ null-terminated
```

### Filenames with newlines
```bash
# Rare but valid on Linux
touch $'file\nwith\nnewline'

find . -name "*newline*"     # ✅ find handles it
find . -name "*newline*" -print    # ❌ output is ambiguous (looks like 3 files)
find . -name "*newline*" -print0   # ✅ null-terminated output is unambiguous
find . -name "*newline*" -print0 | xargs -0 cat   # ✅ safe
```

### Filenames starting with -
```bash
# find doesn't have this problem — {} substitutes the full path
find . -name "-badfile" -exec cat {} \;   # ✅ safe
# {} becomes "./-badfile" — not just "-badfile"
```

---

## Operator Precedence

```bash
# -and binds tighter than -or
find . -name "*.txt" -o -name "*.md" -type f
# Parsed as:
# -name "*.txt" -o (-name "*.md" -and -type f)
# NOT: (-name "*.txt" -o -name "*.md") -and -type f

# ✅ Use parentheses to be explicit:
find . \( -name "*.txt" -o -name "*.md" \) -type f

# Another trap:
find . -name "*.log" -o -name "*.tmp" -delete
# Only -name "*.tmp" files get deleted!
# Parsed as: -name "*.log" -o (-name "*.tmp" -delete)

# ✅ Fix:
find . \( -name "*.log" -o -name "*.tmp" \) -delete
```

---

## Default Action Confusion

### Without an explicit action, -print is implied ONLY when no action exists
```bash
find . -name "*.txt"          # implied -print ✅
find . -name "*.txt" -print   # explicit -print ✅ same result

# BUT: once you add any action, -print is no longer implied
find . -name "*.txt" -exec grep "TODO" {} \;
# Only files where grep succeeds get printed? NO — nothing is printed
# grep output goes to stdout, but find doesn't add -print

# ✅ If you want both exec and print:
find . -name "*.txt" -exec grep "TODO" {} \; -print
```

### -ls vs -print
```bash
find . -name "*.txt" -ls      # ls-style output — does NOT also -print
# Output: inode blocks perms links owner group size date name
```

---

## Cross-Filesystem Surprises

### find crosses filesystem boundaries by default
```bash
find / -name "*.conf"
# Traverses /proc, /sys, /dev, NFS mounts, bind mounts — everything!
# /proc has "infinite" virtual files — can hang or behave oddly

# Stay on one filesystem:
find / -xdev -name "*.conf"    # -xdev = don't cross device boundaries
find / -mount -name "*.conf"   # same as -xdev
```

### /proc files have unusual properties
```bash
find /proc -name "status"
# Returns thousands of results — one per process
# Each "file" in /proc is generated by the kernel on-the-fly
# -size on /proc files always returns 0 (kernel doesn't report sizes)
find /proc -size +1k   # matches nothing in /proc even for large files
```

---

## GNU vs BSD Differences

| Feature | GNU find (Linux) | BSD find (macOS) |
|---------|-----------------|-----------------|
| `-printf` | ✅ Full support | ❌ Not available |
| `-regex` type | ERE (default) | BRE (default) |
| `-regextype` | ✅ (posix-awk, posix-egrep, emacs...) | ❌ |
| `-newerXY` | ✅ | ❌ (use `-newer` only) |
| `-perm /mode` | ✅ | ❌ (use `-perm +mode`) |
| `-maxdepth` | ✅ | ✅ |
| `-delete` | ✅ | ✅ |
| `-execdir` | ✅ | ✅ |
| `-empty` | ✅ | ✅ |
| `-readable/-writable` | ✅ | ❌ |
| `-size` units | c,w,b,k,M,G | c,b,k,M,G |

**Common macOS alternatives:**
```bash
# Instead of -printf on macOS:
find . -name "*.txt" | while read f; do stat -f "%z %N" "$f"; done

# Instead of -perm /mode on macOS (use + which is deprecated on GNU):
find . -perm +111   # works on macOS BSD find
find . -perm /111   # works on GNU find

# Instead of -newermt on macOS:
find . -newer reference_file   # only option on BSD
```

---

## Performance Traps

### stat() on every file is expensive
```bash
# find calls stat() on every file it visits
# On millions of files this is very slow

# Faster: -maxdepth limits traversal
find . -maxdepth 1 -name "*.txt"   # much faster than recursive

# Faster: put cheap tests first (name before size)
find . -name "*.log" -size +100M   # ✅ name test short-circuits
find . -size +100M -name "*.log"   # ❌ stat() runs before name check

# Fastest for name-only search: use locate
locate "*.conf"    # uses pre-built database, no stat()
```

### -exec \; vs -exec +
```bash
find . -name "*.log" -exec gzip {} \;
# Spawns one gzip process per file — slow for millions of files

find . -name "*.log" -exec gzip {} +
# Batches files: gzip a.log b.log c.log ... (like xargs)
# Much faster
```

### Avoid -regex for simple name matching
```bash
find . -regex ".*/.*\.txt"   # slower than:
find . -name "*.txt"         # faster — optimized name check
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
