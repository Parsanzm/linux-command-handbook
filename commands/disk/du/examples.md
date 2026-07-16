# du — Practical Examples

> Real-world patterns for finding what's eating disk space, in scripts and interactively.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Finding the Biggest Directories](#finding-the-biggest-directories)
- [Finding the Biggest Individual Files](#finding-the-biggest-individual-files)
- [Limiting Depth for a Quick Overview](#limiting-depth-for-a-quick-overview)
- [Excluding Directories from the Count](#excluding-directories-from-the-count)
- [Comparing Sizes Across Multiple Directories](#comparing-sizes-across-multiple-directories)
- [Combining du with find](#combining-du-with-find)
- [Scripting and Automation](#scripting-and-automation)
- [Sparse Files and Disk Images](#sparse-files-and-disk-images)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Total size of a directory, human-readable, ONE summary line
du -sh /var/log

# Full breakdown of every subdirectory
du -h /var/log

# Include individual files too, not just directory totals
du -ah /home/alice/Documents

# Multiple paths at once, each with its own total
du -sh /var /home /opt /tmp
```

---

## Finding the Biggest Directories

```bash
# Top-level breakdown of a big directory, sorted by size, largest first
du -h --max-depth=1 /var | sort -rh

# Top 10 biggest subdirectories anywhere under a path
du -ah /home/alice | sort -rh | head -10

# Find the single largest top-level directory on the whole filesystem
du -h --max-depth=1 / 2>/dev/null | sort -rh | head -5

# Narrow down progressively: start broad, then dig into the biggest offender
du -h --max-depth=1 / 2>/dev/null | sort -rh | head -3
# Suppose /var is the biggest — dig one level deeper into JUST /var
du -h --max-depth=1 /var 2>/dev/null | sort -rh | head -5
```

---

## Finding the Biggest Individual Files

```bash
# Find the 10 largest files anywhere under a directory
du -ah /home/alice | sort -rh | head -10

# More targeted: use find for FILES specifically (faster, avoids
# directory entries cluttering the "biggest" results)
find /home/alice -type f -exec du -h {} \; | sort -rh | head -10

# Find files over a specific size threshold
find /home/alice -type f -size +100M -exec du -h {} \; | sort -rh
```

---

## Limiting Depth for a Quick Overview

```bash
# See just the TOP-LEVEL breakdown — the fastest way to identify
# which major area of a filesystem is consuming the most space
du -h --max-depth=1 /

# Two levels deep, for slightly more granularity
du -h --max-depth=2 /var

# Progressive drill-down workflow:
du -h --max-depth=1 / 2>/dev/null | sort -rh
# ... identify /var as the biggest ...
du -h --max-depth=1 /var 2>/dev/null | sort -rh
# ... identify /var/lib as the biggest within THAT ...
du -h --max-depth=1 /var/lib 2>/dev/null | sort -rh
# ... continue narrowing down until the actual culprit is found
```

---

## Excluding Directories from the Count

```bash
# Measure a project's real source size, excluding dependencies
du -sh --exclude="node_modules" --exclude=".git" /home/alice/myproject

# Exclude multiple patterns via a file
cat > /tmp/du_excludes.txt << 'EOF'
node_modules
.git
__pycache__
*.pyc
EOF
du -sh --exclude-from=/tmp/du_excludes.txt /home/alice/myproject

# Measure disk usage of a home directory, excluding large media folders
du -sh --exclude="Videos" --exclude="Downloads" /home/alice
```

---

## Comparing Sizes Across Multiple Directories

```bash
# Compare several candidate directories side by side
du -sh /var/log /var/cache /var/lib /var/www

# Compare backup snapshots to see growth over time
du -sh /backups/2024-01-01 /backups/2024-02-01 /backups/2024-03-01

# Compare sizes of each user's home directory
du -sh /home/*/

# Compare Docker volumes
du -sh /var/lib/docker/volumes/*/
```

---

## Combining du with find

```bash
# Total size of only .log files under a directory tree
find /var/log -name "*.log" -exec du -ch {} + | tail -1

# Total size of files older than 30 days (candidates for cleanup)
find /var/log -mtime +30 -exec du -ch {} + | tail -1

# Find and size only files matching a specific extension
find . -name "*.mp4" -exec du -h {} \; | sort -rh

# Combine with -delete conceptually: FIRST see what would be removed
# and how much space it would free, before actually deleting anything
find /tmp -mtime +7 -exec du -ch {} + | tail -1
# review the total, THEN if acceptable:
find /tmp -mtime +7 -delete
```

---

## Scripting and Automation

```bash
# Capture a directory's size into a variable for a script
SIZE=$(du -sh /var/log | cut -f1)
echo "Log directory size: $SIZE"

# Get just the numeric byte value (no human-readable suffix) for
# arithmetic comparisons in scripts
SIZE_KB=$(du -s /var/log | cut -f1)
if [ "$SIZE_KB" -gt 1000000 ]; then
  echo "Log directory exceeds 1GB!"
fi

# Alert if a specific directory grows beyond a threshold
THRESHOLD_MB=500
CURRENT_MB=$(du -sm /var/lib/myapp | cut -f1)
if [ "$CURRENT_MB" -gt "$THRESHOLD_MB" ]; then
  echo "WARNING: myapp data directory is ${CURRENT_MB}MB, exceeding ${THRESHOLD_MB}MB threshold"
fi

# Daily disk usage report
{
  echo "=== Disk Usage Report: $(date) ==="
  du -sh /var /home /opt 2>/dev/null
} >> /var/log/disk_usage_history.log
```

---

## Sparse Files and Disk Images

```bash
# Compare LOGICAL size (ls) vs ACTUAL disk usage (du) for a VM disk image
ls -lh vm_disk.qcow2
du -h vm_disk.qcow2
# The difference reveals how much of the "virtual" disk is actually allocated

# Check a sparse file's TRUE disk footprint
du -h sparse_file.img
du --apparent-size -h sparse_file.img
# Compare BOTH numbers to understand sparseness

# Find all sparse files in a directory (files where du size is much
# smaller than ls size)
for f in *.img; do
  actual=$(du -k "$f" | cut -f1)
  apparent=$(du --apparent-size -k "$f" | cut -f1)
  echo "$f: actual=${actual}K apparent=${apparent}K"
done
```

---

## Real-World Recipes

```bash
# --- Emergency "Disk Full" Triage ---

df -h                                      # confirm which filesystem is full
du -h --max-depth=1 / 2>/dev/null | sort -rh | head -10
# drill into the biggest offender, repeat max-depth+1 as needed

# --- Cleaning Up Old Log Files ---

du -sh /var/log/*.log.* | sort -rh          # see rotated log sizes
find /var/log -name "*.log.*" -mtime +30 -exec du -ch {} + | tail -1
find /var/log -name "*.log.*" -mtime +30 -delete

# --- Auditing Docker Disk Usage ---

docker system df                            # docker's own built-in summary
du -sh /var/lib/docker/containers/*/ | sort -rh | head -10
du -sh /var/lib/docker/volumes/*/ | sort -rh | head -10

# --- Finding What Changed Between Two Backups ---

du -ah /backups/old > /tmp/old_sizes.txt
du -ah /backups/new > /tmp/new_sizes.txt
diff /tmp/old_sizes.txt /tmp/new_sizes.txt

# --- Reporting Per-User Disk Usage on a Shared Server ---

for user_home in /home/*/; do
  echo "$(du -sh "$user_home" | cut -f1)  $user_home"
done | sort -rh

# --- Verifying Cleanup Actually Freed Space ---

BEFORE=$(du -sh /var/cache | cut -f1)
sudo apt clean
AFTER=$(du -sh /var/cache | cut -f1)
echo "Cache size: $BEFORE -> $AFTER"
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
