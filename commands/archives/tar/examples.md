# tar — Practical Examples

> Real-world patterns from backup, deployment, development, and sysadmin.

---

## Table of Contents

- [Create Archives](#create-archives)
- [Extract Archives](#extract-archives)
- [List Contents](#list-contents)
- [Compression Formats](#compression-formats)
- [Selective Archive & Extract](#selective-archive--extract)
- [Exclude Files & Directories](#exclude-files--directories)
- [Preserve Permissions & Ownership](#preserve-permissions--ownership)
- [Working with stdin & stdout](#working-with-stdin--stdout)
- [Incremental & Differential Backup](#incremental--differential-backup)
- [Split Large Archives](#split-large-archives)
- [Remote Archives over SSH](#remote-archives-over-ssh)
- [Real-World Recipes](#real-world-recipes)

---

## Create Archives

```bash
# Basic: create uncompressed archive
tar -cvf archive.tar dir/
tar -cvf archive.tar file1.txt file2.txt dir/

# gzip (most common)
tar -czf archive.tar.gz dir/
tar -czf archive.tgz dir/       # .tgz = shorthand for .tar.gz

# bzip2 (better compression, slower)
tar -cjf archive.tar.bz2 dir/

# xz (best compression, slowest)
tar -cJf archive.tar.xz dir/

# zstandard (fast + good ratio — modern choice)
tar --zstd -cf archive.tar.zst dir/

# Auto-detect compression from filename (-a flag)
tar -caf archive.tar.gz dir/    # uses gzip
tar -caf archive.tar.xz dir/   # uses xz
tar -caf archive.tar.zst dir/  # uses zstd

# Create from multiple sources
tar -czf backup.tar.gz /etc /var/log /home/alice

# Create with verbose output (shows each file added)
tar -czvf archive.tar.gz dir/

# Create without directory structure (just files)
tar -czf archive.tar.gz -C /path/to/dir .
# The -C changes into dir first, so paths inside archive are relative
```

---

## Extract Archives

```bash
# Extract to current directory
tar -xvf archive.tar
tar -xvzf archive.tar.gz
tar -xvjf archive.tar.bz2
tar -xvJf archive.tar.xz

# Auto-detect compression (GNU tar — no flag needed for extract!)
tar -xvf archive.tar.gz       # GNU tar detects gzip automatically
tar -xvf archive.tar.bz2      # detects bzip2
tar -xvf archive.tar.xz       # detects xz

# Extract to specific directory
tar -xvf archive.tar -C /target/dir/
tar -xvzf archive.tar.gz -C /opt/

# Extract specific files only
tar -xvf archive.tar file.txt
tar -xvf archive.tar dir/subdir/file.txt

# Extract files matching a pattern
tar -xvf archive.tar --wildcards '*.conf'
tar -xvf archive.tar --wildcards '*/config/*'

# Extract without overwriting existing files
tar -xvf archive.tar -k
tar -xvf archive.tar --keep-old-files

# Extract and strip leading path components
tar -xvf archive.tar --strip-components=1
# Archive has: project/src/main.c → extracts as: src/main.c

tar -xvf archive.tar --strip-components=2
# Archive has: project/src/main.c → extracts as: main.c

# Preview before extracting (dry run — list only)
tar -tvf archive.tar
```

---

## List Contents

```bash
# List all files
tar -tf archive.tar
tar -tf archive.tar.gz      # auto-detects gzip

# Verbose list (like ls -l)
tar -tvf archive.tar
# Output: -rw-r--r-- alice/staff 1234 2024-06-15 10:23 dir/file.txt

# Very verbose (shows extra metadata)
tar -tvvf archive.tar

# List specific files
tar -tvf archive.tar dir/subdir/

# List matching pattern
tar -tf archive.tar --wildcards '*.py'

# Count files in archive
tar -tf archive.tar | wc -l

# Check if a specific file exists in archive
tar -tf archive.tar | grep "specific_file.txt"

# Show total size of archive contents
tar -tvf archive.tar | awk '{sum += $3} END {print sum " bytes"}'
```

---

## Compression Formats

```bash
# gzip — best balance of speed and compatibility
tar -czf archive.tar.gz dir/     # compress
tar -xzf archive.tar.gz          # decompress
gzip -l archive.tar.gz           # show compression ratio

# bzip2 — better ratio, ~3x slower than gzip
tar -cjf archive.tar.bz2 dir/
tar -xjf archive.tar.bz2

# xz — best ratio, ~5-10x slower than gzip
tar -cJf archive.tar.xz dir/
tar -xJf archive.tar.xz
# xz with level (1=fast/less, 9=slow/more, default=6)
tar -c dir/ | xz -9 > archive.tar.xz    # maximum compression
tar -c dir/ | xz -1 > archive.tar.xz    # fastest

# zstandard — best of both worlds (fast + good ratio)
tar --zstd -cf archive.tar.zst dir/
tar --zstd -xf archive.tar.zst
# zstd with level (1-22, default=3)
tar -c dir/ | zstd -19 > archive.tar.zst   # high compression
tar -c dir/ | zstd -1  > archive.tar.zst   # fastest

# Multi-threaded compression (for large archives)
tar -c dir/ | pigz -p 8 > archive.tar.gz     # parallel gzip (8 threads)
tar -c dir/ | pbzip2 -p8 > archive.tar.bz2   # parallel bzip2
tar -c dir/ | xz -T 8 > archive.tar.xz       # xz multi-thread
tar -c dir/ | zstd -T8 > archive.tar.zst     # zstd multi-thread

# Decompress multi-threaded
pigz -d < archive.tar.gz | tar -x
pbzip2 -d < archive.tar.bz2 | tar -x
```

---

## Selective Archive & Extract

```bash
# Archive only specific file types
tar -czf pyfiles.tar.gz --include='*.py' -T /dev/null $(find . -name '*.py')
# Better approach:
find . -name "*.py" | tar -czf pyfiles.tar.gz -T -

# Archive files modified in last 24 hours
find . -mtime -1 | tar -czf recent.tar.gz -T -

# Archive files larger than 1MB
find . -size +1M | tar -czf large_files.tar.gz -T -

# Extract only .conf files from archive
tar -xvf archive.tar --wildcards '*.conf'

# Extract only files in a specific directory
tar -xvf archive.tar 'project/config/'

# Extract specific file to stdout (pipe to another command)
tar -xOf archive.tar.gz project/config.yaml    # -O = stdout
tar -xOf backup.tar.gz etc/hosts | grep "192.168"

# Add files to existing (uncompressed) archive
tar -rvf archive.tar newfile.txt

# Update archive (only files newer than existing)
tar -uvf archive.tar dir/
```

---

## Exclude Files & Directories

```bash
# Exclude a specific directory
tar -czf backup.tar.gz project/ --exclude='project/.git'
tar -czf backup.tar.gz project/ --exclude='.git'

# Exclude multiple patterns
tar -czf backup.tar.gz project/ \
  --exclude='.git' \
  --exclude='node_modules' \
  --exclude='__pycache__' \
  --exclude='*.pyc' \
  --exclude='*.log'

# Exclude VCS directories (built-in shortcut)
tar --exclude-vcs -czf backup.tar.gz project/
# Excludes: .git .svn .hg .bzr CVS

# Exclude VCS-ignored files (respects .gitignore)
tar --exclude-vcs-ignores -czf backup.tar.gz project/

# Exclude from a file
cat > exclude.txt << EOF
.git
node_modules
__pycache__
*.pyc
*.log
*.tmp
.DS_Store
EOF
tar -czf backup.tar.gz project/ --exclude-from=exclude.txt
# or:
tar -czf backup.tar.gz project/ -X exclude.txt

# Exclude by pattern (shell globs)
tar -czf backup.tar.gz dir/ --exclude='*.log' --exclude='tmp/*'

# Exclude everything EXCEPT specific types (inverse — needs find)
find . -name "*.py" -o -name "*.yaml" | tar -czf code.tar.gz -T -
```

---

## Preserve Permissions & Ownership

```bash
# Default: tar preserves permissions and timestamps
tar -czf archive.tar.gz dir/

# As root: also preserves ownership (UID/GID)
sudo tar -czf archive.tar.gz /etc/

# Explicitly preserve everything
tar -czpf archive.tar.gz dir/    # -p: preserve permissions

# Restore with ownership (requires root)
sudo tar -xzpf archive.tar.gz -C /restore/

# Extract with numeric UID/GID (no name lookup)
tar -xvf archive.tar --numeric-owner

# Don't restore ownership (extract as current user)
tar -xvf archive.tar --no-same-owner

# Don't restore permissions (apply umask)
tar -xvf archive.tar --no-same-permissions

# Preserve extended attributes (ACLs, SELinux contexts)
tar --xattrs -czf archive.tar.gz dir/
tar --xattrs -xzf archive.tar.gz

# Preserve ACLs specifically
tar --acls -czf archive.tar.gz dir/
tar --acls -xzf archive.tar.gz

# Preserve SELinux context
tar --selinux -czf archive.tar.gz dir/
```

---

## Working with stdin & stdout

```bash
# Write archive to stdout (use - as filename)
tar -czf - dir/ > archive.tar.gz

# Read archive from stdin
tar -xzf - < archive.tar.gz

# Pipe: create and transfer (no temp file)
tar -czf - dir/ | ssh user@remote "cat > /backup/archive.tar.gz"

# Pipe: create and immediately extract elsewhere (copy directory)
tar -cf - source/ | tar -xf - -C /destination/

# Extract from URL directly
curl -sL https://example.com/archive.tar.gz | tar -xzf -
wget -qO- https://example.com/archive.tar.gz | tar -xzf -

# Create archive and show progress
tar -czf - dir/ | pv > archive.tar.gz
# pv shows: 100MiB 0:00:03 [33.3MiB/s]

# Extract with progress
pv archive.tar.gz | tar -xzf -

# Stream through compression and show progress
tar -cf - dir/ | pv | gzip > archive.tar.gz
```

---

## Incremental & Differential Backup

```bash
# Level 0: full backup (snapshot file records all files)
tar -czf full_backup.tar.gz \
  --listed-incremental=snapshot.snar \
  /home/alice/

# Level 1: incremental (only files changed since level 0)
tar -czf incremental_backup.tar.gz \
  --listed-incremental=snapshot.snar \
  /home/alice/

# Each subsequent run creates another incremental

# Restore: apply full, then each incremental in order
tar -xzf full_backup.tar.gz -C /restore/ --listed-incremental=/dev/null
tar -xzf incremental_backup.tar.gz -C /restore/ --listed-incremental=/dev/null

# Automated daily backup script:
#!/bin/bash
DATE=$(date +%Y%m%d)
SNAPSHOT="/backup/snapshot.snar"

if [ ! -f "$SNAPSHOT" ]; then
  # First run: full backup
  tar -czf "/backup/full_${DATE}.tar.gz" \
    --listed-incremental="$SNAPSHOT" \
    /home/
else
  # Subsequent runs: incremental
  tar -czf "/backup/incr_${DATE}.tar.gz" \
    --listed-incremental="$SNAPSHOT" \
    /home/
fi
```

---

## Split Large Archives

```bash
# Split archive into 1GB pieces (using split)
tar -czf - largedir/ | split -b 1G - archive.tar.gz.part
# Creates: archive.tar.gz.partaa, archive.tar.gz.partab, ...

# Reassemble and extract
cat archive.tar.gz.part* | tar -xzf -

# Split with numbered suffixes
tar -czf - largedir/ | split -b 500M -d - archive.tar.gz.part
# Creates: archive.tar.gz.part00, archive.tar.gz.part01, ...

# Native tar multi-volume (for tape/removable media)
tar -cMf /dev/tape largedir/
# tar prompts for new tape when volume is full

# Split into fixed-size pieces with dd
tar -czf - largedir/ | \
  dd bs=1M count=500 > part1.tar.gz
# (manual and cumbersome — use split instead)
```

---

## Remote Archives over SSH

```bash
# Create archive on remote, save locally
ssh user@remote "tar -czf - /var/www/" > website_backup.tar.gz

# Create locally, extract on remote (deploy)
tar -czf - dist/ | ssh user@remote "tar -xzf - -C /var/www/"

# Copy directory from remote to local (no temp file)
ssh user@remote "tar -cf - /etc/" | tar -xf - -C ./remote_etc/

# Copy directory from local to remote
tar -cf - /local/dir/ | ssh user@remote "tar -xf - -C /remote/dir/"

# Backup remote to remote via local (relay)
ssh user@source "tar -czf - /data/" | ssh user@dest "tar -xzf - -C /backup/"

# Use rsync with tar compression (alternative approach)
rsync -avz --rsync-path="sudo rsync" user@remote:/etc/ ./remote_etc/
```

---

## Real-World Recipes

```bash
# --- Development ---

# Package a project (exclude VCS and build artifacts)
tar --exclude-vcs \
    --exclude='node_modules' \
    --exclude='*.pyc' \
    --exclude='__pycache__' \
    --exclude='dist' \
    --exclude='build' \
    -czf project_v1.0.tar.gz \
    project/

# Install software from tarball
curl -sL https://example.com/app-1.0.tar.gz | tar -xzf - -C /opt/
cd /opt/app-1.0
./configure && make && make install

# Quick directory copy (faster than cp -r for many small files)
tar -cf - source/ | tar -xf - -C /destination/


# --- Backup & Restore ---

# Full system backup (exclude non-essential)
sudo tar -czpf /backup/system_$(date +%Y%m%d).tar.gz \
  --exclude=/proc \
  --exclude=/sys \
  --exclude=/dev \
  --exclude=/run \
  --exclude=/tmp \
  --exclude=/mnt \
  --exclude=/media \
  --exclude=/lost+found \
  --exclude=/backup \
  /

# Backup home directory
tar -czf ~/backup/home_$(date +%Y%m%d).tar.gz \
  --exclude=~/.cache \
  --exclude=~/.local/share/Trash \
  ~

# Restore specific files from backup
tar -xzf backup.tar.gz -C /restore/ --wildcards '*/etc/nginx/*'

# Verify backup integrity
tar -tzf backup.tar.gz > /dev/null && echo "OK" || echo "CORRUPT"


# --- Docker & Containers ---

# Export Docker image layers
docker save myimage:latest | gzip > myimage.tar.gz
docker load < myimage.tar.gz

# Export container filesystem
docker export container_name | gzip > container_fs.tar.gz


# --- Database Backup ---

# MySQL dump + compress in one step
mysqldump mydb | gzip > db_backup.sql.gz

# Restore
gunzip < db_backup.sql.gz | mysql mydb

# PostgreSQL
pg_dump mydb | tar -czf db_backup.tar.gz -T -
# Restore
tar -xOzf db_backup.tar.gz | psql mydb


# --- Log Archiving ---

# Archive and compress old logs
find /var/log -name "*.log" -mtime +30 | \
  tar -czf /archive/old_logs_$(date +%Y%m).tar.gz -T -

# List what's in the archive
tar -tzf /archive/old_logs_202401.tar.gz

# Extract specific log
tar -xzf old_logs.tar.gz -C /tmp/ var/log/nginx/access.log
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
