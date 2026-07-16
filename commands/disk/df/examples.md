# df — Practical Examples

> Real-world patterns for checking disk space, diagnosing full filesystems, and monitoring.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Checking a Specific Path or File's Filesystem](#checking-a-specific-path-or-files-filesystem)
- [Filtering Out Noise](#filtering-out-noise)
- [Checking Inode Usage](#checking-inode-usage)
- [Scripting and Monitoring](#scripting-and-monitoring)
- [Combining df with du](#combining-df-with-du)
- [Filesystem Type Investigation](#filesystem-type-investigation)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Quick overview of every mounted filesystem, human-readable
df -h

# Include filesystem type in the output
df -hT

# Add a combined total row at the bottom
df -h --total

# Show usage in exact bytes (no rounding, for precise scripting)
df -B1
```

---

## Checking a Specific Path or File's Filesystem

```bash
# Check the filesystem containing the root directory
df -h /

# Check the filesystem containing a specific directory
df -h /var/log

# Check the filesystem containing a SPECIFIC FILE (df resolves it to
# whichever filesystem that file's path actually lives on)
df -h /home/alice/bigfile.iso

# Check the filesystem containing the CURRENT directory
df -h .
```

---

## Filtering Out Noise

```bash
# Hide tmpfs (RAM-backed) and other virtual filesystem entries
df -h -x tmpfs -x devtmpfs -x squashfs

# Show only real disk-backed filesystem types
df -h -t ext4 -t xfs -t btrfs

# Show only LOCAL filesystems, excluding any network mounts (NFS, CIFS)
df -hl

# Combine multiple exclusions for a clean, relevant view on a
# snap-heavy or container-heavy system
df -h -x squashfs -x tmpfs -x overlay
```

---

## Checking Inode Usage

```bash
# Check inode usage across all filesystems
df -i

# Check inode usage for a SPECIFIC filesystem
df -ih /var

# Compare byte usage vs inode usage side by side (two separate calls,
# since df doesn't show both in ONE call)
df -h /var
df -i /var

# Quick check: is inode exhaustion the actual problem behind a
# "No space left on device" error?
df -h /tmp    # if this shows PLENTY of free space...
df -i /tmp    # ...but THIS shows IUse% near 100%, inodes are the culprit
```

---

## Scripting and Monitoring

```bash
# Get just the percentage used for a specific filesystem (script-friendly)
df -h / | awk 'NR==2 {print $5}'
# 71%

# Strip the % sign for numeric comparison
USAGE=$(df / | awk 'NR==2 {print $5}' | tr -d '%')
if [ "$USAGE" -gt 90 ]; then
  echo "WARNING: root filesystem is over 90% full!"
fi

# Check ALL mounted filesystems and alert on any nearing capacity
df -h --output=pcent,target | tail -n +2 | while read -r pcent mount; do
  pct=$(echo "$pcent" | tr -d '% ')
  if [ "$pct" -gt 85 ]; then
    echo "WARNING: $mount is at ${pct}%"
  fi
done

# Simple monitoring loop, logging usage over time
while true; do
  echo "$(date): $(df -h / | awk 'NR==2 {print $5}')" >> /var/log/disk_usage.log
  sleep 3600
done
```

---

## Combining df with du

```bash
# Step 1: identify which filesystem is nearly full
df -h
# /dev/sda1  100G  95G  5G  95% /

# Step 2: narrow down WHICH directory within that filesystem is responsible
du -h --max-depth=1 / 2>/dev/null | sort -rh | head -10

# Step 3: drill further into the biggest offender
du -h --max-depth=1 /var 2>/dev/null | sort -rh | head -10

# Combined one-shot triage script
echo "=== Filesystem Usage ==="
df -h
echo ""
echo "=== Top-Level Directory Sizes ==="
du -h --max-depth=1 / 2>/dev/null | sort -rh | head -10
```

---

## Filesystem Type Investigation

```bash
# See every distinct filesystem type currently mounted
df -T | awk 'NR>1 {print $2}' | sort -u

# Count how many mount entries are squashfs (common with snap-heavy systems)
df -h | grep -c squashfs

# Show detailed type + usage for troubleshooting an unfamiliar mount
df -hT /mnt/network_share

# Distinguish real disk usage from RAM-backed tmpfs usage
df -hT | grep -v tmpfs
df -hT | grep tmpfs
```

---

## Real-World Recipes

```bash
# --- Emergency "Disk Full" First Response ---

df -h
# identify the nearly-full filesystem immediately

df -i
# ALSO check inodes — "no space" can mean either byte or inode exhaustion

# --- Pre-Deployment Capacity Check ---

REQUIRED_GB=10
AVAILABLE_KB=$(df /opt --output=avail | tail -1)
AVAILABLE_GB=$((AVAILABLE_KB / 1024 / 1024))
if [ "$AVAILABLE_GB" -lt "$REQUIRED_GB" ]; then
  echo "Insufficient space: need ${REQUIRED_GB}GB, have ${AVAILABLE_GB}GB"
  exit 1
fi

# --- Cross-Server Disk Usage Audit ---

for server in web1 web2 web3 db1; do
  echo "=== $server ==="
  ssh "$server" "df -h /"
done

# --- Monitoring Script for Cron (Alert on High Usage) ---

#!/bin/bash
THRESHOLD=85
df -h --output=pcent,target | tail -n +2 | while read -r pcent mount; do
  pct=$(echo "$pcent" | tr -d '% ')
  if [ "$pct" -ge "$THRESHOLD" ]; then
    echo "ALERT: $mount at ${pct}% (threshold: ${THRESHOLD}%)" | \
      mail -s "Disk Space Warning on $(hostname)" admin@example.com
  fi
done

# --- Comparing Before/After a Cleanup Operation ---

BEFORE=$(df -h / | awk 'NR==2 {print $4}')
sudo apt clean && sudo journalctl --vacuum-time=7d
AFTER=$(df -h / | awk 'NR==2 {print $4}')
echo "Available space: $BEFORE -> $AFTER"

# --- Checking Docker's Impact on Disk Usage ---

df -h /var/lib/docker
docker system df    # Docker's own more detailed breakdown of images/containers/volumes
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
