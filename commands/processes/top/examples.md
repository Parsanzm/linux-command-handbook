# top — Practical Examples

> Real-world patterns for live monitoring, filtering, sorting, and
> scripting with batch mode.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Starting Pre-Filtered or Pre-Sorted](#starting-pre-filtered-or-pre-sorted)
- [Sorting and Viewing Options While Running](#sorting-and-viewing-options-while-running)
- [Killing and Renicing from Inside top](#killing-and-renicing-from-inside-top)
- [Batch Mode for Scripts and Logs](#batch-mode-for-scripts-and-logs)
- [Combining top with Other Tools](#combining-top-with-other-tools)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Launch the live interactive view
top

# Quit at any time
# press q

# Refresh every 1 second instead of the default 3
top -d 1

# Exit automatically after 5 refresh cycles
top -n 5
```

---

## Starting Pre-Filtered or Pre-Sorted

```bash
# Only one user's processes
top -u alice

# Only specific known PIDs
top -p 1234,5678,9012

# Start already sorted by memory usage
top -o %MEM

# Start already sorted by CPU usage
top -o %CPU

# Show individual threads as separate rows from the start
top -H
```

---

## Sorting and Viewing Options While Running

```bash
# Inside top, interactively:
# P — sort by %CPU
# M — sort by %MEM
# T — sort by cumulative TIME+
# R — reverse the current sort direction
# 1 — toggle single combined CPU line vs. one line per core
# c — toggle full command line vs. just the program name
# u — filter to one specific user, prompts for username
```

---

## Killing and Renicing from Inside top

```bash
# Inside top:
# k — prompts for a PID, then a signal (defaults to SIGTERM, 15)
# r — prompts for a PID, then a new nice value

# Typical flow to kill a runaway process without leaving top:
# 1. Press P to sort by CPU, spot the offending PID
# 2. Press k
# 3. Type the PID, press Enter
# 4. Type the signal number (or press Enter for the default, SIGTERM)
```

---

## Batch Mode for Scripts and Logs

```bash
# Single non-interactive snapshot
top -bn1

# Snapshot showing only the summary lines plus the top 10 processes by CPU
top -bn1 -o %CPU | head -17

# Log a snapshot every 5 minutes
while true; do
  echo "=== $(date) ==="
  top -bn1 -o %MEM | head -15
  sleep 300
done >> /var/log/resource_snapshots.log

# Capture exactly N iterations, useful for a bounded diagnostic capture
top -b -n 3 -d 2 > diagnostic_capture.txt
```

---

## Combining top with Other Tools

```bash
# Get just the process table portion, skipping the summary header
top -bn1 | tail -n +8

# Extract only PID and %CPU from batch output
top -bn1 | tail -n +8 | awk '{print $1, $9}'

# Compare top's batch snapshot against a live watch loop of ps instead
watch -n 2 'ps aux --sort=-%cpu | head -15'

# Feed top's batch output into a monitoring pipeline
top -bn1 -o %MEM | head -20 | mail -s "Resource snapshot $(hostname)" admin@example.com
```

---

## Real-World Recipes

```bash
# --- Quick "what's using the most CPU right now" check ---
top -o %CPU
# glance at the top few rows, q to quit when done

# --- Investigate a Specific User's Resource Usage ---
top -u alice -o %MEM

# --- Watch a Deployment's Processes Live ---
top -p "$(pgrep -d, -f myapp)"

# --- Confirm I/O Wait Is the Real Bottleneck, Not CPU ---
top
# check the %Cpu(s) line's "wa" figure specifically

# --- Automated Alert If Any Process Exceeds a Memory Threshold ---
top -bn1 -o %MEM | awk 'NR>7 && $10+0 > 20 {print "High memory:", $0}'

# --- Bounded Diagnostic Capture Before/After a Deployment ---
top -bn1 -o %CPU > before_deploy.txt
./deploy.sh
top -bn1 -o %CPU > after_deploy.txt
diff before_deploy.txt after_deploy.txt
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
