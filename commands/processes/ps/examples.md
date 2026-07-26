# ps — Practical Examples

> Real-world patterns for finding, filtering, sorting, and monitoring processes.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Finding a Specific Process](#finding-a-specific-process)
- [Sorting by Resource Usage](#sorting-by-resource-usage)
- [Custom Column Output](#custom-column-output)
- [Viewing the Process Tree](#viewing-the-process-tree)
- [Combining ps with Other Tools](#combining-ps-with-other-tools)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Every process, BSD style
ps aux

# Every process, UNIX style
ps -ef

# Just your own processes attached to the current terminal
ps

# Every process on the system, no terminal restriction
ps -e
```

---

## Finding a Specific Process

```bash
# Classic combo — find a process by name
ps aux | grep firefox

# Cleaner, purpose-built alternative (avoids matching "grep" itself)
pgrep -la firefox

# Show only processes owned by a specific user
ps -u alice

# Show a specific PID
ps -p 1234

# Show a specific PID with full detail
ps -fp 1234
```

---

## Sorting by Resource Usage

```bash
# Top 10 processes by memory usage
ps aux --sort=-%mem | head -11

# Top 10 processes by CPU usage
ps aux --sort=-%cpu | head -11

# Custom columns, sorted by memory, cleanest output
ps -eo pid,%mem,%cpu,cmd --sort=-%mem | head -11

# Sort by how long a process has been running
ps -eo pid,etime,cmd --sort=-etime | head
```

---

## Custom Column Output

```bash
# Just PID, user, and command — nothing else
ps -eo pid,user,cmd

# Include parent PID, useful for tracing process ancestry
ps -eo pid,ppid,user,cmd

# Include elapsed time and start time
ps -eo pid,start,etime,cmd

# Wide output so long command lines aren't truncated
ps auxww
```

---

## Viewing the Process Tree

```bash
# ASCII-art parent/child tree
ps -ef --forest

# Just one process and its descendants
ps --forest -o pid,ppid,cmd -g $(ps -o sid= -p 1234)

# Simpler alternative dedicated to tree views
pstree -p
```

---

## Combining ps with Other Tools

```bash
# Count how many processes a specific user is running
ps -u alice | wc -l

# Find and kill all processes matching a pattern (careful — review first!)
ps aux | grep '[n]ode' | awk '{print $2}' | xargs kill

# Watch a filtered process list refresh every 2 seconds
watch -n 2 'ps aux --sort=-%cpu | head -15'

# Check which files a specific process has open
lsof -p "$(pgrep -f myapp)"
```

---

## Real-World Recipes

```bash
# --- Quick "what's eating my CPU/memory right now" check ---
echo "=== Top CPU ==="
ps aux --sort=-%cpu | head -6
echo "=== Top Memory ==="
ps aux --sort=-%mem | head -6

# --- Find and Report Zombie Processes ---
ps aux | awk '$8 ~ /^Z/ {print}'

# --- Confirm a Service Is Actually Running Before Proceeding ---
if ! pgrep -x nginx > /dev/null; then
  echo "nginx is not running!"
  exit 1
fi

# --- Log a Snapshot of Top Processes Every Minute ---
while true; do
  echo "=== $(date) ===" >> /var/log/process_snapshot.log
  ps aux --sort=-%cpu | head -10 >> /var/log/process_snapshot.log
  sleep 60
done

# --- Check How Long a Specific Process Has Been Running ---
ps -o etime= -p "$(pgrep -f myapp)"

# --- Cleanly Kill a Runaway Process by Name, With a Sanity Check First ---
ps -eo pid,cmd | grep '[r]unaway_script'
# review the output above, THEN:
pkill -f runaway_script
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
