# touch — Practical Examples

> Real-world patterns for creating files, refreshing timestamps, and
> working with build systems and scripts.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Creating Multiple Files at Once](#creating-multiple-files-at-once)
- [Updating Timestamps Without Creating](#updating-timestamps-without-creating)
- [Setting Specific Timestamps](#setting-specific-timestamps)
- [Scripting with touch](#scripting-with-touch)
- [Combining touch with Other Tools](#combining-touch-with-other-tools)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Create a new empty file
touch notes.txt

# Refresh an existing file's timestamp to "now"
touch notes.txt

# Only update access time, leave modification time alone
touch -a notes.txt

# Only update modification time, leave access time alone
touch -m notes.txt
```

---

## Creating Multiple Files at Once

```bash
# Several files in one command
touch file1.txt file2.txt file3.txt

# A quick set of placeholder files using brace expansion
touch report_{jan,feb,mar}.txt
# creates report_jan.txt, report_feb.txt, report_mar.txt

# Create an entire small project skeleton
mkdir myapp && cd myapp
touch README.md main.py requirements.txt .gitignore
```

---

## Updating Timestamps Without Creating

```bash
# Update timestamp ONLY if the file already exists — do nothing
# (and don't error) if it doesn't
touch -c maybe_exists.txt

# Confirm this behaves as a no-op for a genuinely missing file
touch -c does_not_exist.txt
echo $?
# 0 — no error, and the file was NOT created
```

---

## Setting Specific Timestamps

```bash
# Set an exact date and time
touch -t 202601151430.00 file.txt

# Using a more readable date expression instead
touch -d "2026-01-15 14:30:00" file.txt

# Relative date expressions
touch -d "yesterday" file.txt
touch -d "1 hour ago" file.txt
touch -d "next friday" file.txt

# Copy another file's exact timestamp
touch -r original.txt copy_with_same_timestamp.txt
```

---

## Scripting with touch

```bash
# Ensure a log file exists before appending to it
touch /var/log/myapp/app.log
echo "Starting up..." >> /var/log/myapp/app.log

# Create a "lock file" pattern to signal that something is in progress
touch /tmp/myapp.lock
# ... do work ...
rm /tmp/myapp.lock

# Check for the presence of a lock file before starting a task
if [ -f /tmp/myapp.lock ]; then
  echo "Already running, exiting"
  exit 1
fi
touch /tmp/myapp.lock

# Create a marker file recording when a script last ran successfully
touch /var/run/myapp/last_success
```

---

## Combining touch with Other Tools

```bash
# Force make to rebuild a specific target by updating its source's timestamp
touch main.c
make

# Find all files modified more recently than a reference file
touch -t 202601010000 /tmp/reference_point
find /var/log -newer /tmp/reference_point

# List files sorted by modification time after touching a few
touch -d "2 days ago" old_report.txt
ls -lt

# Preserve a file's original timestamp while editing its content
# (capture timestamp first, then restore it after editing)
touch -r original.txt /tmp/timestamp_backup
vim original.txt
touch -r /tmp/timestamp_backup original.txt
```

---

## Real-World Recipes

```bash
# --- Ensure a Config File Exists Before an App Reads It ---
touch -c /etc/myapp/local_overrides.conf 2>/dev/null || \
  { echo "Cannot access config path"; exit 1; }

# --- Bulk-Create Placeholder Files from a List ---
while read -r name; do
  touch "output/${name}.txt"
done < filenames.txt

# --- Force a Cache Invalidation by Refreshing a Sentinel File ---
touch /var/cache/myapp/.invalidate
# a background process watching this file's mtime knows to reload

# --- Preserve Original File Timestamps After a Bulk Copy/Edit ---
for f in *.txt; do
  touch -r "$f" "/tmp/ts_$f"
done
# ... process files ...
for f in *.txt; do
  touch -r "/tmp/ts_$f" "$f"
done

# --- Create a Fresh, Empty Log File Each Day (Simple Log Rotation) ---
LOGFILE="/var/log/myapp/$(date +%F).log"
touch "$LOGFILE"
ln -sf "$LOGFILE" /var/log/myapp/current.log
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
