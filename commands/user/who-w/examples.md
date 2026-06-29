# who / w — Practical Examples

> Real-world usage for monitoring, security auditing, and system administration.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [who: Variations & Flags](#who-variations--flags)
- [w: Variations & Flags](#w-variations--flags)
- [Monitoring Active Sessions](#monitoring-active-sessions)
- [Security Monitoring](#security-monitoring)
- [Login History with last](#login-history-with-last)
- [Failed Logins with lastb](#failed-logins-with-lastb)
- [lastlog: Per-User Last Login](#lastlog-per-user-last-login)
- [Combining with Other Commands](#combining-with-other-commands)
- [Scripting Patterns](#scripting-patterns)

---

## Basic Usage

```bash
# who: who is logged in right now
who
# alice    pts/0    2024-06-15 10:23 (192.168.1.5)
# bob      pts/1    2024-06-15 09:00 (10.0.0.2)
# carol    tty1     2024-06-15 07:00

# w: who + what they're doing + system stats
w
#  10:45:23 up 5 days, 2:30,  3 users,  load average: 0.15, 0.12, 0.10
# USER     TTY      FROM             LOGIN@   IDLE JCPU   PCPU WHAT
# alice    pts/0    192.168.1.5      10:23    0.00s 0.05s  0.02s vim server.py
# bob      pts/1    10.0.0.2         09:00   45:00  0.00s  0.00s bash
# carol    tty1     -                07:00    2days 0.01s  0.01s -bash

# users: just the names
users
# alice bob carol

# Am I the only one logged in?
who | wc -l
```

---

## who: Variations & Flags

```bash
# With column headers
who -H
# NAME     LINE         TIME             COMMENT
# alice    pts/0        2024-06-15 10:23 (192.168.1.5)

# Show last boot time
who -b
# system boot  2024-06-10 08:15

# Show current runlevel
who -r
# run-level 5  2024-06-10 08:15

# Show message status (can other users write to this terminal?)
who -T
# alice    + pts/0   2024-06-15 10:23 (192.168.1.5)
#            ↑ + = writable, - = not writable, ? = unknown

# Show idle time and PID
who -u
# alice    pts/0   2024-06-15 10:23  00:05  1234  (192.168.1.5)
#                                    ↑idle  ↑PID

# Show everything
who -a
# shows boot, runlevel, login processes, dead processes, active users

# Show count only
who -q
# alice bob carol
# # users=3

# Current session only (who am I)
who am i
# alice    pts/0   2024-06-15 10:23 (192.168.1.5)

who -m                # same result as "who am i"

# Lookup hostnames (reverse DNS for IPs)
who --lookup

# Read from a different file (e.g., historical wtmp)
who /var/log/wtmp | head -20
```

---

## w: Variations & Flags

```bash
# Full output (default)
w

# No header
w -h
# alice    pts/0    192.168.1.5  10:23    0.00s 0.05s 0.02s vim server.py
# bob      pts/1    10.0.0.2     09:00   45:00  0.00s 0.00s bash

# Short format (omit login time and CPU columns)
w -s
# USER     TTY      FROM          IDLE WHAT
# alice    pts/0    192.168.1.5   0s   vim server.py
# bob      pts/1    10.0.0.2      45m  bash

# Show IPs instead of hostnames
w -i
# alice    pts/0    192.168.1.5   ...

# Only show specific user's sessions
w alice
# Shows only alice's sessions

# No header + specific user (useful in scripts)
w -h alice
```

---

## Monitoring Active Sessions

```bash
# How many users are currently logged in?
who | wc -l
users | wc -w

# List unique logged-in users (alice may have multiple sessions)
users | tr ' ' '\n' | sort -u

# Who has been idle the longest? (w output, sort by IDLE)
w -h | sort -k5 -r | head -10

# Find forgotten/zombie sessions (idle > 1 hour)
w -h | awk '$5 ~ /[0-9]+:[0-9]+/ && $5 > "01:00" {print $1, $5, $NF}'

# Find sessions idle for days
w -h | grep "days"
# bob  pts/3  ...  3days  bash   ← forgotten session!

# Who is logged in at the console (physical terminal)?
who | grep "tty[0-9]"

# Who is logged in remotely (SSH)?
who | grep "pts/"

# What's the most common command being run right now?
w -h | awk '{print $NF}' | sort | uniq -c | sort -rn

# Show what alice is doing
w alice

# Watch logins in real-time
watch -n 5 who

# Watch w output in real-time
watch -n 2 w
```

---

## Security Monitoring

```bash
# Alert if new user logs in (compare who output every minute)
previous=$(who)
while true; do
  current=$(who)
  if [ "$current" != "$previous" ]; then
    echo "LOGIN CHANGE DETECTED:"
    diff <(echo "$previous") <(echo "$current")
    previous="$current"
  fi
  sleep 60
done

# Find users logged in from suspicious IPs
who | grep -vE "(192\.168\.|10\.|172\.(1[6-9]|2[0-9]|3[01])\.)"
# Shows logins from non-RFC1918 addresses

# Check for root logins
who | grep "^root"

# Users with writable terminals (security risk — can receive write/wall)
who -T | grep "^.*+ "

# Check for unexpected console logins (should be empty on servers)
who | grep "tty[0-9]"

# Multiple sessions for same user (possible account sharing)
who | awk '{print $1}' | sort | uniq -d

# Alert on root SSH login
if who | grep -q "^root.*pts/"; then
  echo "ALERT: root is logged in via SSH!" | mail -s "Security Alert" admin@example.com
fi

# Count sessions per user
who | awk '{print $1}' | sort | uniq -c | sort -rn
# 3 alice    ← alice has 3 active sessions
# 1 bob
```

---

## Login History with last

```bash
# All login history (newest first)
last

# Last 20 entries
last -n 20

# Specific user's history
last alice
last root

# All reboots
last reboot
# reboot   system boot  5.15.0-112-gene Sat Jun 15 08:15   still running
# reboot   system boot  5.15.0-112-gene Fri Jun 14 23:30 - 08:15  (08:44)

# All shutdowns
last shutdown

# With full timestamps (default omits year and seconds)
last -F alice

# Show IP addresses instead of hostnames
last -i

# Date range
last -s "2024-06-01" -t "2024-06-15"
last --since "2024-06-01" --until "2024-06-15"

# Show who was logged in at a specific time
last -s "2024-06-10 14:00" -t "2024-06-10 14:05"

# How many times did alice log in this month?
last alice | grep -v "^$\|^alice.*still\|^wtmp" | wc -l

# All users who logged in from a specific IP
last -i | grep "192.168.1.100"

# Last time root logged in
last root | head -3

# System uptime history (how often it reboots):
last reboot | awk 'NF>0 && !/^wtmp/ {print $5, $6, $7, $8, $9, $10}'

# Read older wtmp files (after rotation)
last -f /var/log/wtmp.1
last -f /var/log/wtmp.2.gz    # if compressed
```

---

## Failed Logins with lastb

```bash
# All failed login attempts (requires root)
sudo lastb

# Last 50 failed attempts
sudo lastb -n 50

# Failed attempts for root (most common target)
sudo lastb root | head -20

# Failed attempts from a specific IP
sudo lastb -i | grep "192.168.1.100"

# Count failed attempts per username (top targets)
sudo lastb | awk 'NF>0 && !/^btmp/ {print $1}' \
  | sort | uniq -c | sort -rn | head -20

# Count failed attempts per IP
sudo lastb -i | awk 'NF>0 && !/^btmp/ {print $3}' \
  | sort | uniq -c | sort -rn | head -20

# Failed logins in the last 24 hours
sudo lastb --since yesterday | head -30

# Brute force detection: IPs with > 10 failed attempts
sudo lastb -i | awk 'NF>0 && !/^btmp/ {print $3}' \
  | sort | uniq -c | sort -rn \
  | awk '$1 > 10 {print $1, "failed:", $2}'

# Compare successful vs failed logins for a user
echo "=== Successful ==="
last alice | grep -v "^$\|^wtmp" | head -5
echo "=== Failed ==="
sudo lastb alice | grep -v "^$\|^btmp" | head -5
```

---

## lastlog: Per-User Last Login

```bash
# All users' last login (shows "Never logged in" for system accounts)
lastlog

# Specific user
lastlog -u alice
# Username  Port    From            Latest
# alice     pts/0   192.168.1.5     Sat Jun 15 10:23:00 +0000 2024

# Users who logged in within last 7 days
lastlog -t 7

# Users who have NOT logged in for 30+ days (inactive accounts)
lastlog -b 30
# Useful for finding stale accounts to disable

# Never-logged-in accounts (potential dead accounts)
lastlog | grep "Never logged in"

# Combine: find inactive human users (UID >= 1000) not logged in for 60 days
lastlog -b 60 | awk 'NR>1 {print $1}' | while read user; do
  uid=$(id -u "$user" 2>/dev/null)
  [ -n "$uid" ] && [ "$uid" -ge 1000 ] && echo "$user (uid=$uid)"
done
```

---

## Combining with Other Commands

```bash
# who + ps: see full process tree for each logged-in user
for user in $(users | tr ' ' '\n' | sort -u); do
  echo "=== $user ==="
  ps -u "$user" --no-header -o pid,cmd | head -5
done

# who + last: login history for currently logged-in users
for user in $(who | awk '{print $1}' | sort -u); do
  echo "=== $user ==="
  last "$user" | head -3
done

# w + grep: find users running specific command
w -h | grep "vim\|nano\|emacs"     # users editing files
w -h | grep "python\|ruby\|node"   # users running scripts
w -h | grep "rsync\|scp\|sftp"     # users transferring files
w -h | grep "mysql\|psql"          # users in databases

# who + uptime context
echo "System: $(uptime -p)"
echo "Users:"
who -H

# Alert on high load with who context
load=$(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | tr -d ',')
if (( $(echo "$load > 4.0" | bc -l) )); then
  echo "HIGH LOAD: $load"
  echo "Active users:"
  w
fi

# Find who is consuming most CPU right now
w -h | sort -k7 -rn | head -5    # sort by PCPU

# Session count per user in a formatted report
echo "=== Active Session Report $(date) ==="
echo "Total sessions: $(who | wc -l)"
echo ""
who | awk '{print $1}' | sort | uniq -c | sort -rn | \
  awk '{printf "  %-20s %d session(s)\n", $2, $1}'
```

---

## Scripting Patterns

```bash
# Check if specific user is logged in
is_logged_in() {
  who | awk '{print $1}' | grep -q "^$1$"
}

if is_logged_in "alice"; then
  echo "alice is online"
fi

# Wait until a user logs out
wait_for_logout() {
  local user="$1"
  echo "Waiting for $user to log out..."
  while who | awk '{print $1}' | grep -q "^$user$"; do
    sleep 30
  done
  echo "$user has logged out"
}

# Get current user's login time
my_login=$(who am i | awk '{print $3, $4}')
echo "Logged in since: $my_login"

# Get all TTYs for a user
get_user_ttys() {
  who | awk -v user="$1" '$1==user {print $2}'
}

# Send message to specific user's terminal
for tty in $(get_user_ttys alice); do
  echo "System maintenance at 18:00" | write alice "$tty"
done

# Count unique users logged in
unique_users=$(users | tr ' ' '\n' | sort -u | wc -l)
echo "$unique_users unique users logged in"

# Log current sessions to file (for audit)
log_sessions() {
  echo "=== $(date -Iseconds) ===" >> /var/log/session_audit.log
  who >> /var/log/session_audit.log
  echo "" >> /var/log/session_audit.log
}

# Run as cron job:
# */15 * * * * /usr/local/bin/log_sessions.sh

# Notify admin if any user is idle > 2 hours
w -h | awk '$5 ~ /^[2-9]:[0-9][0-9]$/ || $5 ~ /[0-9]+days/ {
  print "IDLE WARNING:", $1, "idle for", $5
}' | mail -s "Idle Sessions" admin@company.com

# Check if system has any active users before maintenance
if who | grep -q "."; then
  echo "Active users found — aborting maintenance:"
  who
  exit 1
fi
echo "No users logged in — proceeding with maintenance"
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
