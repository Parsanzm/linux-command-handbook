# who / w — Edge Cases & Gotchas

> who and w look simple but have surprising blind spots and reliability issues
> that matter most when you need them: during security incidents.

---

## Table of Contents

- [utmp Can Be Manipulated](#utmp-can-be-manipulated)
- [Screen & tmux: Invisible Sessions](#screen--tmux-invisible-sessions)
- [who Doesn't Show All Processes](#who-doesnt-show-all-processes)
- [Idle Time Is Not What You Think](#idle-time-is-not-what-you-think)
- [w's WHAT Column Limitations](#ws-what-column-limitations)
- [wtmp Rotation Loses History](#wtmp-rotation-loses-history)
- [btmp Fills Up Silently](#btmp-fills-up-silently)
- [lastlog Inaccuracy](#lastlog-inaccuracy)
- [Remote Host Truncation](#remote-host-truncation)
- [Container & Namespace Blind Spots](#container--namespace-blind-spots)
- [Timezone Surprises](#timezone-surprises)
- [Race Conditions in Monitoring Scripts](#race-conditions-in-monitoring-scripts)

---

## utmp Can Be Manipulated

### Root can erase their presence from who/w
```bash
# utmp is a binary file writable by root and login daemons
# An attacker with root can clear their session from who/w:

# Method 1: directly zero out the record
# (attacker tools like 'zap' or 'cloak' do this)
utmpdump /var/run/utmp | grep -v "attacker" | utmpdump -r > /tmp/utmp.new
cp /tmp/utmp.new /var/run/utmp

# Method 2: use specialized rootkit tools
# Many rootkits automatically hide sessions from utmp

# Implication: who/w output is NOT tamper-proof
# A sophisticated attacker who has root WON'T appear in who/w
# For forensics: check /proc/*/loginuid (kernel-maintained, harder to fake)

# More reliable check (requires root):
ls /proc/*/fd | xargs -I{} readlink {} 2>/dev/null | grep pts
# Or:
ps aux | grep -E "sshd|bash|zsh" | grep -v grep
```

### utmp file permissions allow limited manipulation
```bash
ls -l /var/run/utmp
# -rw-rw-r-- 1 root utmp 1920 /var/run/utmp
#                  ↑ group 'utmp' can write!

# Members of 'utmp' group can modify the file
# Check who's in utmp group:
getent group utmp
# utmp:x:43:syslog   ← only syslog on a clean system

# If an attacker gets into utmp group, they can hide from who/w
```

---

## Screen & tmux: Invisible Sessions

### Detached screen/tmux sessions DON'T appear in who
```bash
# User alice starts a screen session and detaches:
screen -S work       # start
# ... does work ...
# Ctrl+A D           # detach

# Now:
who                  # alice NOT shown (if she disconnected SSH)
# empty output

# But alice's processes are still running!
ps aux | grep alice  # ✅ alice's processes visible
screen -list         # ✅ shows detached screen sessions
tmux ls              # ✅ shows tmux sessions

# Real scenario: "no users logged in" but system is busy
who    # empty
w      # empty
ps aux # full of alice's processes from detached screen
```

### tmux/screen reattach shows user in who
```bash
# When alice reattaches:
screen -r work       # reattach

# Now alice appears in who (new pts device)
who
# alice    pts/2    2024-06-15 14:00 (192.168.1.5)
# But her original session (pts/0) before detach is gone from who
```

### Nohup processes also invisible
```bash
# User alice runs a background process and logs out:
nohup long_running_script.sh &
logout

# who shows alice is gone — but her script is still running
who       # alice not shown
ps aux | grep alice   # script still running with alice's UID
```

---

## who Doesn't Show All Processes

### who only shows login sessions, not all user activity
```bash
# Cron jobs run as alice but don't appear in who:
crontab -l -u alice
# */5 * * * * /usr/local/bin/backup.sh

# During cron execution:
who          # alice might not be logged in at all
ps aux | grep alice   # backup.sh running as alice

# At jobs, systemd user services, and setuid processes also invisible to who
```

### sudo and su don't create new utmp entries
```bash
# Bob runs sudo su -:
# who shows:
who
# bob      pts/0    ...   (10.0.0.1)

# Bob is now effectively root, but who still shows "bob"
# w shows:
w
# bob      pts/0    10.0.0.1   ...   sudo su -
# (or whatever command was run after su)

# Check effective user vs login user:
ps aux | grep pts/0   # see actual processes including root shell
```

---

## Idle Time Is Not What You Think

### IDLE measures last keypress on TTY, not last activity
```bash
w
# alice    pts/0   192.168.1.5   10:23   30:00  0.05s  0.00s top

# IDLE shows 30 minutes — but alice is running 'top' which updates continuously!
# 'top' reads from /proc, doesn't generate keystrokes on the terminal
# The IDLE counter measures when the TTY device was last written TO BY THE USER
# → reading /proc, running top, watching logs = doesn't reset IDLE
```

### Copy-pasting doesn't reset idle properly
```bash
# Alice is actively working (copy-pasting in her terminal app)
# but if the paste goes through clipboard (not typed), idle may not reset
# Depends on terminal emulator and how it sends input
```

### SSH keepalive packets don't reset idle
```bash
# ServerAliveInterval in SSH sends keepalives
# These don't appear as keystrokes on the PTY
# → long idle times even for "active" SSH connections

# Alice has SSH keepalive sending every 60s:
w
# alice    pts/0   ...   2:00:00   -bash
# 2 hours idle despite SSH being connected
```

---

## w's WHAT Column Limitations

### WHAT shows only the foreground process name
```bash
w
# alice    pts/0   ...  vim /etc/nginx/nginx.conf

# If alice has multiple processes in background, WHAT doesn't show them
# WHAT = the current foreground process of the terminal session

# For full picture:
ps -u alice     # all of alice's processes
```

### WHAT is truncated at terminal width
```bash
w
# alice    pts/0   ...  python3 /very/long/path/to/script.py --config /another/long

# Full command is cut off — use ps for full command line:
ps -u alice -o pid,cmd   # full command

# Or read from /proc:
cat /proc/$(pgrep -u alice python3)/cmdline | tr '\0' ' '
```

### WHAT can be spoofed (process name)
```bash
# A process can change its own argv[0]:
# attacker's process may show as "-bash" or "sshd" in WHAT

# Verify with proc filesystem:
ls -la /proc/PID/exe   # actual binary being run (harder to fake)
```

---

## wtmp Rotation Loses History

### logrotate deletes old wtmp history
```bash
cat /etc/logrotate.d/wtmp
# /var/log/wtmp {
#     monthly
#     create 0664 root utmp
#     rotate 1          ← only 1 month kept!
# }

# By default: only 1 month of login history!
last | tail -1
# wtmp begins Sat May 15 08:00:00 2024   ← history starts here

# For longer retention, edit logrotate:
# rotate 12   ← keep 12 months
```

### Rotated wtmp still accessible
```bash
# After rotation, last month is at wtmp.1 (or wtmp.1.gz)
last -f /var/log/wtmp.1
last -f /var/log/wtmp.2.gz   # if compressed

# Search all available wtmp files:
for f in /var/log/wtmp*; do
  echo "=== $f ==="
  last -f "$f" | grep "alice" | head -5
done
```

### wtmp can be cleared intentionally
```bash
# Root can clear all login history:
> /var/log/wtmp    # truncate (valid file but empty)
last               # "wtmp begins..." shows very recent time

# Attacker covering tracks:
# Clearing wtmp removes all login evidence
# Check file modification time and size:
ls -lh /var/log/wtmp
stat /var/log/wtmp | grep "Modify"
```

---

## btmp Fills Up Silently

### btmp can grow very large during brute-force attacks
```bash
ls -lh /var/log/btmp
# -rw------- 1 root utmp 2.3G /var/log/btmp   ← MASSIVE

# Each struct utmp record = 384 bytes
# 1 million failed attempts = ~384MB

# High login failure rates fill disk:
sudo lastb | wc -l
# 6000000   ← 6 million failed attempts!

# Monitor btmp size:
ls -lh /var/log/btmp

# Rotate if too large:
> /var/log/btmp    # clear (but loses history)
# or properly rotate:
logrotate -f /etc/logrotate.d/btmp
```

### btmp not always enabled
```bash
# Some minimal systems don't have /var/log/btmp at all
ls /var/log/btmp
# ls: cannot access '/var/log/btmp': No such file or directory

# Create it:
touch /var/log/btmp
chown root:utmp /var/log/btmp
chmod 600 /var/log/btmp
```

---

## lastlog Inaccuracy

### lastlog database can have stale entries
```bash
lastlog
# alice  pts/0  192.168.1.5  Sat Jun 15 10:23:00 2024
# bob    pts/2  10.0.0.2     Mon Jan  1 00:00:00 1970   ← epoch = stale/reset

# If user's UID changes, lastlog shows old entry under new UID
# If lastlog file is cleared, shows epoch time
```

### lastlog indexed by UID, not username
```bash
# lastlog is a sparse file indexed by UID
# /var/log/lastlog position = UID * record_size

# If a user is deleted and UID reused:
# New user at UID 1001 inherits old user's lastlog entry!
lastlog -u newuser
# Shows: login time from the previous user who had UID 1001

# Check lastlog file size and sparse nature:
ls -lhs /var/log/lastlog
# 292K  -rw-r--r-- 1 root root 286M /var/log/lastlog
# ↑ 292K actual, 286M apparent (sparse — most is empty)
```

### lastlog -b can miss recently created users
```bash
# lastlog -b 30: users who haven't logged in for 30+ days
# But newly created users show "Never logged in" — not "30 days ago"
# They appear in -b output even though the account is brand new

# Filter out new accounts:
lastlog -b 30 | awk 'NR>1' | while read line; do
  user=$(echo "$line" | awk '{print $1}')
  created=$(stat -c %Y /home/"$user" 2>/dev/null)
  now=$(date +%s)
  age=$(( (now - created) / 86400 ))
  [ "$age" -gt 30 ] && echo "$line"
done
```

---

## Remote Host Truncation

### Long hostnames are truncated in who/w output
```bash
who
# alice    pts/0    2024-06-15 10:23 (very-long-hostname-that-gets-c)
#                                     ↑ truncated at terminal width!

# Full hostname in utmp record: 256 bytes (ut_host field)
# who/w truncate at terminal width

# Get full hostname:
utmpdump /var/run/utmp | grep alice
# Shows full ut_host field

# Or use last with full width:
last -i     # shows IP instead (never truncated)
```

### IPv6 addresses in who output
```bash
who
# alice    pts/0   2024-06-15 10:23 (::ffff:192.168.1.5)
# or:
# alice    pts/0   2024-06-15 10:23 (2001:db8::1)
# IPv6 addresses are long — may be truncated

# Use last -i or w -i to see IPs cleanly
```

---

## Container & Namespace Blind Spots

### who inside a container shows container sessions only
```bash
# Inside Docker container:
docker exec -it mycontainer who
# root    pts/0   2024-06-15 10:00 (172.17.0.1)
# Only shows sessions inside the container's mount namespace

# On the host:
who
# root    pts/0   2024-06-15 10:00 (192.168.1.5)
# alice   pts/1   ...
# The container's sessions may or may not appear here

# Container processes visible on host via ps:
ps aux | grep mycontainer_process   # ✅ visible
who | grep container_user           # ❌ not visible
```

### Kubernetes pods: who is useless
```bash
kubectl exec -it pod -- who
# may show nothing or just the current exec session
# Pod processes don't have traditional login sessions
# Use: kubectl exec, kubectl logs, metrics server instead
```

---

## Timezone Surprises

### who/w show times in LOCAL timezone of the system
```bash
who
# alice    pts/0    2024-06-15 10:23 (192.168.1.5)
# ↑ local time — if system is UTC and alice is in US/Eastern:
# Alice connected at 6:23 AM her time, shows as 10:23 UTC

# Server is UTC, admin reads time thinking it's their local timezone → confusion

# Always check system timezone when interpreting timestamps:
timedatectl
date
cat /etc/timezone   # or: readlink /etc/localtime

# last -F shows full timestamp including timezone offset:
last -F alice
# alice pts/0  192.168.1.5 Sat Jun 15 10:23:45 2024 +0000
```

---

## Race Conditions in Monitoring Scripts

### Session snapshot becomes stale immediately
```bash
# Checking if user is logged in:
logged_in_users=$(who | awk '{print $1}')  # snapshot
# ... time passes ...
echo "$logged_in_users"   # may be stale!

# User logs in between snapshot and check:
# ❌ False negative: appears not logged in
# User logs out between snapshot and action on their session
# ❌ Stale data: session doesn't exist anymore

# For real-time monitoring: use inotifywait on /var/run/utmp
inotifywait -m /var/run/utmp -e modify |
  while read event; do
    echo "Login change: $(date)"
    who
  done
```

### Writing to a user's terminal after logout
```bash
# Check then write — race condition:
if who | grep -q "^alice"; then
  # alice might have logged out here!
  echo "message" | write alice   # fails silently or errors
fi

# Better: use write and check its exit code
echo "message" | write alice pts/0 2>/dev/null || echo "alice logged out"
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
