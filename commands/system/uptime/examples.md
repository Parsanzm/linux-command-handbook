# uptime — Practical Examples

> Real-world patterns for health checks, monitoring scripts, and load diagnosis.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Pretty and Since Formats](#pretty-and-since-formats)
- [Interpreting Load Relative to CPU Count](#interpreting-load-relative-to-cpu-count)
- [Monitoring Scripts](#monitoring-scripts)
- [Checking Multiple Servers at Once](#checking-multiple-servers-at-once)
- [Combining uptime with Other Diagnostic Tools](#combining-uptime-with-other-diagnostic-tools)
- [Reading /proc/loadavg Directly](#reading-proclogavg-directly)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Standard, full output
uptime
#  14:32:07 up 45 days,  3:21,  2 users,  load average: 0.15, 0.22, 0.18

# Quick health check as part of a login banner or dashboard
uptime
```

---

## Pretty and Since Formats

```bash
# Human-phrased duration, easier to read in reports/dashboards
uptime -p
# up 3 days, 2 hours, 15 minutes

# Exact boot time (useful for correlating with logs/incidents)
uptime -s
# 2024-01-12 12:16:52

# Combine both in a script for a clear summary
echo "System has been running since: $(uptime -s)"
echo "That's: $(uptime -p)"
```

---

## Interpreting Load Relative to CPU Count

```bash
# ALWAYS check core count alongside load average — raw numbers alone
# are meaningless without this context
nproc
# 4

uptime
#  load average: 3.20, 2.80, 2.50

# Quick per-core calculation
LOAD=$(uptime | awk -F'load average: ' '{print $2}' | cut -d',' -f1)
CORES=$(nproc)
echo "Load per core: $(echo "$LOAD / $CORES" | bc -l)"

# A small script that flags whether the system looks overloaded
#!/bin/bash
CORES=$(nproc)
LOAD_1MIN=$(awk '{print $1}' /proc/loadavg)
THRESHOLD=$(echo "$CORES * 0.9" | bc)
if (( $(echo "$LOAD_1MIN > $THRESHOLD" | bc -l) )); then
  echo "WARNING: Load ($LOAD_1MIN) is approaching core count ($CORES)"
fi
```

---

## Monitoring Scripts

```bash
# Log uptime/load to a file every hour, building a simple history
while true; do
  echo "$(date): $(uptime)" >> /var/log/load_history.log
  sleep 3600
done

# Alert if 1-minute load average exceeds a threshold
THRESHOLD=8.0
LOAD_1MIN=$(awk '{print $1}' /proc/loadavg)
if (( $(echo "$LOAD_1MIN > $THRESHOLD" | bc -l) )); then
  echo "High load detected: $LOAD_1MIN" | mail -s "Load Alert on $(hostname)" admin@example.com
fi

# Simple health check endpoint for a monitoring system (e.g., a
# script triggered by a monitoring agent or cron)
#!/bin/bash
LOAD_1MIN=$(awk '{print $1}' /proc/loadavg)
CORES=$(nproc)
RATIO=$(echo "$LOAD_1MIN / $CORES" | bc -l)
if (( $(echo "$RATIO > 1.0" | bc -l) )); then
  echo "CRITICAL: load ratio $RATIO"
  exit 2
elif (( $(echo "$RATIO > 0.7" | bc -l) )); then
  echo "WARNING: load ratio $RATIO"
  exit 1
else
  echo "OK: load ratio $RATIO"
  exit 0
fi
```

---

## Checking Multiple Servers at Once

```bash
# Quick uptime check across a fleet of servers via SSH
for server in web1 web2 web3 db1 db2; do
  echo "=== $server ==="
  ssh "$server" uptime
done

# Parallel version using a tool like pssh/parallel-ssh (if installed)
pssh -h hosts.txt -i "uptime"

# Compile a quick fleet-wide load report
for server in $(cat server_list.txt); do
  LOAD=$(ssh "$server" "awk '{print \$1}' /proc/loadavg")
  echo "$server: $LOAD"
done | sort -t: -k2 -rn
```

---

## Combining uptime with Other Diagnostic Tools

```bash
# Standard escalating diagnostic workflow when load looks high
uptime                          # step 1: quick check
top -bn1 | head -20              # step 2: see which processes are consuming CPU
vmstat 1 5                       # step 3: sampled resource breakdown over 5 seconds
free -h                          # step 4: check memory pressure too

# Full one-shot diagnostic snapshot script
#!/bin/bash
echo "=== Uptime & Load ==="
uptime
echo ""
echo "=== Top Processes by CPU ==="
ps aux --sort=-%cpu | head -6
echo ""
echo "=== Memory ==="
free -h
echo ""
echo "=== I/O Wait Check ==="
vmstat 1 3
```

---

## Reading /proc/loadavg Directly

```bash
# Full raw content
cat /proc/loadavg
# 0.15 0.22 0.18 2/458 12345

# Extract just the 1, 5, 15-minute averages
awk '{print $1, $2, $3}' /proc/loadavg

# Extract the "runnable/total processes" field
awk '{print $4}' /proc/loadavg
# 2/458   ← 2 currently runnable, 458 total processes on the system

# Use directly in a script without invoking uptime at all
read -r one five fifteen rest < /proc/loadavg
echo "1min: $one, 5min: $five, 15min: $fifteen"
```

---

## Real-World Recipes

```bash
# --- Post-Deployment Sanity Check ---

echo "Deployment complete. Server status:"
uptime
free -h
df -h /

# --- Investigating a "Server Feels Slow" Report ---

uptime
# load average: 12.50, 11.80, 9.20   ← clearly elevated on this 4-core box
nproc
# 4
top -bn1 | head -15
# identify the specific process(es) responsible

# --- Correlating a Load Spike with Deployment Time ---

uptime -s                        # confirm boot time (rule out a recent crash/reboot)
grep "deploy" /var/log/deploy_history.log | tail -5
# compare against when the load spike started, via load_history.log

# --- Simple Uptime-Based Alerting in a Cron Job ---

*/5 * * * * /usr/local/bin/check_load.sh >> /var/log/load_check.log 2>&1

# check_load.sh:
#!/bin/bash
LOAD=$(awk '{print $1}' /proc/loadavg)
CORES=$(nproc)
RATIO=$(echo "$LOAD / $CORES" | bc -l)
if (( $(echo "$RATIO > 1.5" | bc -l) )); then
  echo "$(date): HIGH LOAD - ratio $RATIO"
fi

# --- Quick Fleet Health Dashboard (simple text version) ---

printf "%-15s %-10s %-10s\n" "Server" "Uptime" "Load(1m)"
for server in web1 web2 db1; do
  UP=$(ssh "$server" "uptime -p")
  LOAD=$(ssh "$server" "awk '{print \$1}' /proc/loadavg")
  printf "%-15s %-10s %-10s\n" "$server" "$UP" "$LOAD"
done
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
