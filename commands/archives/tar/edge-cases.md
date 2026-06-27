# tar — Edge Cases & Gotchas

> tar has more silent failure modes than almost any command.
> These traps can corrupt backups, lose data, or create security holes.

---

## Table of Contents

- [The Overwrite Trap](#the-overwrite-trap)
- [Absolute Paths: Security Risk](#absolute-paths-security-risk)
- [Compression Detection Traps](#compression-detection-traps)
- [Filename & Path Traps](#filename--path-traps)
- [Permissions & Ownership Surprises](#permissions--ownership-surprises)
- [Symlink Attacks](#symlink-attacks)
- [Backup Verification Failures](#backup-verification-failures)
- [Incremental Backup Pitfalls](#incremental-backup-pitfalls)
- [Sparse Files](#sparse-files)
- [Large Files & Size Limits](#large-files--size-limits)
- [Timestamps](#timestamps)
- [stdin/stdout Pitfalls](#stdinstdout-pitfalls)
- [GNU vs BSD tar Differences](#gnu-vs-bsd-tar-differences)
- [Exit Codes Are Misleading](#exit-codes-are-misleading)

---

## The Overwrite Trap

### tar -x overwrites existing files silently
```bash
tar -xf archive.tar
# ⚠️ Overwrites files that already exist — no warning, no prompt!

# Safe: don't overwrite
tar -xf archive.tar -k
tar -xf archive.tar --keep-old-files

# Safe: keep newer files
tar -xf archive.tar --keep-newer-files

# Preview first
tar -tf archive.tar    # list before extracting
```

### Extracting into wrong directory
```bash
tar -xf archive.tar    # extracts into CURRENT directory
                       # if you're in /, this extracts to root!

# Always specify -C or verify your location first
pwd                        # check where you are
tar -xf archive.tar -C /safe/target/dir/

# Or create a temp dir:
tmpdir=$(mktemp -d)
tar -xf archive.tar -C "$tmpdir"
```

### Archive with relative vs absolute paths
```bash
tar -czf archive.tar.gz /etc/nginx    # absolute path stored
tar -xf archive.tar.gz                # extracts to ./etc/nginx (relative) ✅

# But with leading /:
tar -czf archive.tar.gz /etc/nginx    # GNU tar strips leading /
# GNU tar warns: Removing leading '/' from member names

# Old tar or tar -P (POSIX) keeps absolute paths:
tar -czPf archive.tar.gz /etc/nginx   # -P: keep absolute paths
tar -xPf archive.tar.gz               # ⚠️ extracts to /etc/nginx — DANGER
```

---

## Absolute Paths: Security Risk

### Archives with absolute paths overwrite system files
```bash
# A malicious archive might contain: /etc/passwd, /etc/shadow
tar -tf evil.tar
# /etc/passwd
# /etc/shadow

tar -xf evil.tar    # ⚠️ OVERWRITES /etc/passwd and /etc/shadow!

# GNU tar strips leading / by default and warns:
# tar: Removing leading '/' from member names

# But -P disables this protection:
tar -xPf evil.tar   # ❌ extracts to absolute paths — never use -P on untrusted archives

# Safe: always inspect before extracting untrusted archives
tar -tf untrusted.tar | head -20
# Look for: absolute paths (starting with /)
#           paths with ../ (directory traversal)
```

### Path traversal (../../../../../) attacks
```bash
# Malicious archive might contain:
# ../../etc/cron.d/backdoor

tar -tf evil.tar
# ../../etc/cron.d/backdoor

tar -xf evil.tar    # ⚠️ may escape current directory!

# GNU tar handles this safely:
# tar: Removing leading '../' from member names
# But verify your tar version does this

# Safe: extract to isolated directory, then inspect
mkdir /tmp/safe_extract
tar -xf untrusted.tar -C /tmp/safe_extract
ls -la /tmp/safe_extract
```

---

## Compression Detection Traps

### GNU tar auto-detects compression on extract but NOT on older systems
```bash
# GNU tar (Linux): auto-detects from file magic bytes
tar -xvf archive.tar.gz    # ✅ detects gzip automatically

# BSD tar (macOS): also auto-detects
tar -xvf archive.tar.gz    # ✅

# Old POSIX tar: requires explicit flag
tar -xzvf archive.tar.gz   # must specify -z

# Portable (always explicit):
tar -xzf archive.tar.gz    # explicit gzip
tar -xjf archive.tar.bz2   # explicit bzip2
tar -xJf archive.tar.xz    # explicit xz
```

### Wrong extension doesn't mean wrong format
```bash
# Extension is just a hint — tar uses file magic bytes
file archive.tgz            # shows actual compression type
file archive.tar.gz         # same

# Someone renamed archive.tar.bz2 to archive.tar.gz:
tar -xzf archive.tar.gz    # ❌ gzip: not in gzip format
tar -xjf archive.tar.gz    # ✅ bzip2 actually works
tar -xvf archive.tar.gz    # ✅ GNU tar auto-detects correctly
```

### -a (auto-compress) only works with -c, not -x
```bash
tar -caf archive.tar.gz dir/    # ✅ -a: auto from extension on create
tar -xaf archive.tar.gz         # ❌ -a: invalid/ignored on extract
tar -xvf archive.tar.gz         # ✅ auto-detect on extract (GNU tar)
```

---

## Filename & Path Traps

### Filenames with spaces
```bash
# ✅ tar handles spaces fine with proper quoting:
tar -czf archive.tar.gz "my dir with spaces/"
tar -xvf archive.tar "file with spaces.txt"

# ❌ Danger when using shell expansion:
files=$(find . -name "*.txt")
tar -czf archive.tar.gz $files    # word-splits on spaces!

# ✅ Correct: use find -print0 + --null
find . -name "*.txt" -print0 | tar -czf archive.tar.gz --null -T -
```

### Filenames with special characters
```bash
# Filenames with newlines — rare but valid
tar -czf archive.tar.gz $'file\nwith\nnewline'   # ✅ tar handles it

# When listing, newlines in names look like separate files:
tar -tf archive.tar   # ⚠️ misleading output for newline filenames
tar -tf archive.tar --null | xargs -0 ls   # ✅ null-safe listing
```

### 100-character filename limit (USTAR format)
```bash
# USTAR format (default in some tools) truncates filenames over 100 chars
# GNU tar uses extensions to handle long names — but some tools don't

# Check if any files have long names:
find . | awk 'length > 100 {print NR": "length" chars: "$0}'

# Use PAX format for maximum compatibility:
tar --format=posix -czf archive.tar.gz dir/

# Check what format an archive uses:
file archive.tar
# archive.tar: POSIX tar archive (GNU)  ← GNU extensions for long names
```

### Hardlinks in archives
```bash
# tar preserves hard links — both names point to same data
# But only within the same archive run

ln file.txt hardlink.txt
tar -czf archive.tar.gz file.txt hardlink.txt
# Archive stores file.txt fully, hardlink.txt as reference

# If you archive them in separate commands:
tar -czf a1.tar.gz file.txt
tar -czf a2.tar.gz hardlink.txt
# Both archives store full copies — hard link relationship lost
```

---

## Permissions & Ownership Surprises

### Extracting as non-root loses ownership info
```bash
sudo tar -czf backup.tar.gz /var/www/    # stores owner: www-data
tar -xzf backup.tar.gz                   # extracts as: current user (not www-data)
# ⚠️ ownership not restored — need root to restore ownership

sudo tar -xzpf backup.tar.gz -C /restore/  # ✅ restores ownership
```

### setuid/setgid bits on extracted files (security risk)
```bash
# If archive contains SUID files:
tar -tf archive.tar | xargs stat --format='%a %n' 2>/dev/null | grep "^[46]"

# GNU tar strips SUID/SGID on extract for non-root:
# (this is intentional security behavior)

# To restore SUID/SGID bits (root only):
sudo tar -xpf archive.tar    # -p: preserve all permissions including SUID
```

### umask affects extracted permissions
```bash
# Without -p, extracted permissions are modified by umask
umask 022
tar -xf archive.tar     # files with 666 → 644, dirs with 777 → 755

# With -p:
tar -xpf archive.tar    # exact permissions from archive, umask ignored
```

---

## Symlink Attacks

### Symlink in archive can redirect file writes
```bash
# Malicious archive:
# etc/ → symlink to /etc
# etc/cron.d/backdoor → actual malicious file

# When extracted:
# Creates etc/ symlink to /etc
# Writes /etc/cron.d/backdoor!

# GNU tar protects against this by default
# Always extract untrusted archives in isolated directories:
mkdir /tmp/safe && tar -xf untrusted.tar -C /tmp/safe

# Check for symlinks before extracting:
tar -tvf archive.tar | grep "^l"    # l = symlink
```

### -h / --dereference changes what gets archived
```bash
ln -s /etc/passwd symlink_to_passwd

tar -czf archive.tar.gz symlink_to_passwd
# Archives the SYMLINK itself (default)

tar -czh archive.tar.gz symlink_to_passwd  # -h: dereference
# Archives the TARGET FILE (/etc/passwd content)
# ⚠️ Can accidentally include sensitive files!
```

---

## Backup Verification Failures

### Silent corruption is the biggest tar danger
```bash
# Creating an archive succeeds, but it's corrupt:
tar -czf backup.tar.gz /data/   # exit 0 — looks fine
gzip -t backup.tar.gz           # test fails!

# Always verify after creating critical backups:
tar -czf backup.tar.gz /data/ && \
  tar -tzf backup.tar.gz > /dev/null && \
  echo "Backup verified OK" || echo "BACKUP FAILED"

# More thorough verification:
tar -df backup.tar.gz    # compare archive vs filesystem
# Shows differences if any files changed after backup
```

### Files changed during archiving
```bash
tar -czf backup.tar.gz /var/lib/mysql/
# ⚠️ If MySQL is running, files may change mid-archive:
# tar: /var/lib/mysql/ibdata1: file changed as we read it

# Solution 1: snapshot (LVM, ZFS)
# Solution 2: freeze the application
# Solution 3: use application-aware backup (mysqldump)

# tar warning "file changed as we read it" = exit code 1 (not 0)
# Check exit codes in backup scripts!
```

### Disk full during creation = corrupt archive
```bash
tar -czf /backup/archive.tar.gz /data/
# If disk fills up: archive is truncated but may not report error
# Always check available space first:
df -h /backup/
```

---

## Incremental Backup Pitfalls

### Snapshot file must be preserved
```bash
tar -czf backup_day1.tar.gz --listed-incremental=snapshot.snar /data/
tar -czf backup_day2.tar.gz --listed-incremental=snapshot.snar /data/

# If snapshot.snar is lost or corrupt:
# You can't properly restore from incrementals
# Keep the snapshot file alongside the archives!
cp snapshot.snar /backup/
```

### Deleted files aren't handled automatically
```bash
# If you delete a file, incremental backup records it was deleted
# But restoring requires applying backups in order:
tar -xzf full.tar.gz --listed-incremental=/dev/null
tar -xzf day1.tar.gz --listed-incremental=/dev/null
tar -xzf day2.tar.gz --listed-incremental=/dev/null
# Each layer overwrites/deletes as recorded

# Skipping an incremental = incomplete restore
```

---

## Sparse Files

### tar inflates sparse files by default
```bash
# Sparse file: 1GB apparent, 1MB actual data
dd if=/dev/zero of=sparse.img bs=1 count=0 seek=1G

ls -lh sparse.img     # shows 1.0G
du -sh sparse.img     # shows 4.0K (actual)

# Without --sparse: archive is 1GB!
tar -czf huge.tar.gz sparse.img      # ⚠️ archives all the zeros!

# With --sparse: archive is tiny
tar --sparse -czf small.tar.gz sparse.img   # ✅ handles sparse efficiently
tar -Sczf small.tar.gz sparse.img           # -S is alias for --sparse
```

---

## Large Files & Size Limits

### USTAR format: 8GB file size limit
```bash
# USTAR (old default) can't handle files > 8GiB
tar --format=ustar -czf archive.tar.gz huge_file.img
# tar: huge_file.img: file is the archive; not dumped
# or silently truncates!

# GNU format or POSIX handles unlimited sizes:
tar --format=gnu -czf archive.tar.gz huge_file.img    # ✅
tar --format=posix -czf archive.tar.gz huge_file.img  # ✅
# Modern GNU tar defaults to GNU format — usually fine
```

### Very long paths
```bash
# Path prefix + name must fit in 155+100 = 255 bytes (USTAR)
# GNU format handles unlimited via extension headers
# Check:
find . -type f | awk 'length > 255 {print "TOO LONG:", $0}'
```

---

## Timestamps

### Extracted files get archive's timestamps, not current time
```bash
tar -xf archive.tar
ls -la extracted_file    # shows original timestamp from archive

# To use current time instead:
tar -xf archive.tar -m    # -m: don't restore modification times
# Files get current time (now)

# Useful when you want "just extracted" timestamps for build systems
```

### Sub-second timestamps
```bash
# GNU tar format stores sub-second timestamps
# USTAR format truncates to 1-second precision

tar --format=posix -czf archive.tar.gz dir/  # PAX: stores nanoseconds
tar --format=gnu -czf archive.tar.gz dir/    # GNU: stores sub-seconds
tar --format=ustar -czf archive.tar.gz dir/  # USTAR: truncates to seconds
```

---

## stdin/stdout Pitfalls

### Writing to stdout without -f -
```bash
tar -cz dir/ > archive.tar.gz      # ✅ works (stdout redirect)
tar -czf - dir/ > archive.tar.gz   # ✅ explicit stdout

# Missing > redirect:
tar -czf - dir/     # ❌ dumps binary to terminal — corrupts terminal!
# Press Ctrl+C, then: reset
```

### Mixing stdout tar with verbose
```bash
tar -czvf - dir/ > archive.tar.gz   # ❌ verbose goes to stdout too!
# Both the archive data AND the verbose list go to stdout
# archive.tar.gz will be corrupt

# Fix: verbose goes to stderr
tar -czvf - dir/ 2>filelist.txt > archive.tar.gz  # ✅
# or suppress verbose:
tar -czf - dir/ > archive.tar.gz   # ✅ no verbose
```

### Pipe failure not detected
```bash
tar -czf - dir/ | ssh remote "cat > backup.tar.gz"
# If ssh fails, tar doesn't know — still exits 0
# Use pipefail:
set -o pipefail
tar -czf - dir/ | ssh remote "cat > backup.tar.gz"
# Now exits with error if any stage fails
```

---

## GNU vs BSD tar Differences

| Feature | GNU tar (Linux) | BSD tar (macOS) |
|---------|----------------|----------------|
| Auto-detect compression | ✅ | ✅ |
| `-J` (xz) | ✅ | ✅ (newer) |
| `--zstd` | ✅ | ✅ (newer) |
| `--exclude-vcs` | ✅ | ❌ |
| `--listed-incremental` | ✅ | ❌ |
| `--sparse` / `-S` | ✅ | ❌ |
| `--xattrs` | ✅ | ✅ (different flag) |
| `--acls` | ✅ | ❌ |
| `--strip-components` | ✅ | ✅ |
| `--numeric-owner` | ✅ | ✅ |
| `-a` (auto-compress) | ✅ | ✅ |
| Long options | ✅ | Limited |

```bash
# macOS: install GNU tar for full compatibility
brew install gnu-tar
# Then use 'gtar' instead of 'tar'
gtar -czf archive.tar.gz --exclude-vcs dir/

# Portable script check:
if tar --version 2>&1 | grep -q "GNU"; then
  TAR_OPTS="--exclude-vcs --sparse"
else
  TAR_OPTS=""  # BSD tar
fi
tar -czf archive.tar.gz $TAR_OPTS dir/
```

---

## Exit Codes Are Misleading

```bash
# Exit code 0 = success (but may have warnings)
# Exit code 1 = some files differ or warnings treated as errors
# Exit code 2 = fatal error

# "File changed as we read it" → exit code 1 (not 0!)
tar -czf backup.tar.gz /var/log/
echo $?    # 1 if any log file was written during archiving

# In backup scripts, be careful:
tar -czf backup.tar.gz /data/
if [ $? -eq 2 ]; then
  echo "FATAL backup error"
elif [ $? -eq 1 ]; then
  echo "WARNING: some files may have changed during backup"
fi

# Or allow "changed files" warning but not fatal errors:
tar -czf backup.tar.gz /data/
rc=$?
if [ $rc -gt 1 ]; then
  echo "FATAL ERROR: backup failed"
  exit 1
fi
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
