# cron / crontab — Practical Examples

> Real-world scheduling patterns for backups, monitoring, maintenance, and automation.

---

## Table of Contents

- [Basic Scheduling Patterns](#basic-scheduling-patterns)
- [Common Time Intervals](#common-time-intervals)
- [Managing Your Crontab](#managing-your-crontab)
- [Logging Cron Job Output](#logging-cron-job-output)
- [Preventing Overlapping Runs](#preventing-overlapping-runs)
- [System-Wide and Drop-in Crontabs](#system-wide-and-drop-in-crontabs)
- [Environment Setup for Reliable Jobs](#environment-setup-for-reliable-jobs)
- [Testing and Debugging Cron Jobs](#testing-and-debugging-cron-jobs)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Scheduling Patterns

```bash
# Every day at 2:30 AM
30 2 * * * /usr/local/bin/backup.sh

# Every Monday at 9:00 AM
0 9 * * 1 /usr/local/bin/weekly_report.sh

# On the 1st of every month, at midnight
0 0 1 * * /usr/local/bin/monthly_invoice.sh

# Every weekday (Mon-Fri) at 6:00 PM
0 18 * * 1-5 /usr/local/bin/end_of_day.sh

# Twice a day: 8 AM and 8 PM
0 8,20 * * * /usr/local/bin/twice_daily.sh
```

---

## Common Time Intervals

```bash
# Every 5 minutes
*/5 * * * * /usr/local/bin/quick_check.sh

# Every 15 minutes
*/15 * * * * /usr/local/bin/poll.sh

# Every hour, on the hour
0 * * * * /usr/local/bin/hourly_task.sh

# Every 2 hours
0 */2 * * * /usr/local/bin/every_2_hours.sh

# Every 6 hours (4 times a day)
0 */6 * * * /usr/local/bin/every_6_hours.sh

# During business hours only (9 AM - 5 PM), every 30 minutes
*/30 9-17 * * 1-5 /usr/local/bin/business_hours_check.sh
```

---

## Managing Your Crontab

```bash
# View your current schedule
crontab -l

# Edit your schedule interactively
crontab -e

# Back up your crontab before making changes
crontab -l > ~/crontab_backup_$(date +%Y%m%d).txt

# Restore from a backup
crontab ~/crontab_backup_20240115.txt

# Edit another user's crontab (requires sudo)
sudo crontab -u www-data -e

# Completely remove all your scheduled jobs (be careful!)
crontab -r
```

---

## Logging Cron Job Output

```bash
# Redirect BOTH stdout and stderr to a log file (recommended default)
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# Separate logs for normal output vs errors
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>> /var/log/backup_errors.log

# Timestamp each log entry (the script itself, or a wrapper, must add this)
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
# Inside backup.sh:
# echo "[$(date)] Starting backup" >> /var/log/backup.log

# Rotate cron job logs using logrotate (recommended for long-running jobs)
cat /etc/logrotate.d/mycronjob
# /var/log/backup.log {
#     weekly
#     rotate 4
#     compress
#     missingok
# }
```

---

## Preventing Overlapping Runs

```bash
# Use flock to ensure only ONE instance of a job runs at a time,
# even if a previous run is still in progress when the next is due
*/5 * * * * flock -n /tmp/backup.lock /usr/local/bin/backup.sh
# -n (non-blocking): if the lock is already held, exit immediately
# instead of waiting — the next scheduled attempt "skips" rather than
# stacking up multiple overlapping executions

# Alternative: a simple PID-file based lock inside the script itself
#!/bin/bash
LOCKFILE=/tmp/myjob.lock
if [ -e "$LOCKFILE" ]; then
  echo "Already running, exiting"
  exit 1
fi
touch "$LOCKFILE"
trap 'rm -f "$LOCKFILE"' EXIT
# ... actual job logic ...
```

---

## System-Wide and Drop-in Crontabs

```bash
# /etc/crontab entry (note the EXTRA username field compared to user crontabs)
cat /etc/crontab
# 0 3 * * * root /usr/local/bin/system_maintenance.sh
# 0 4 * * * appuser /opt/myapp/nightly_cleanup.sh

# A dedicated drop-in file for one application, under /etc/cron.d/
sudo tee /etc/cron.d/myapp << 'EOF'
PATH=/usr/local/bin:/usr/bin:/bin
0 4 * * * appuser /opt/myapp/nightly_job.sh >> /var/log/myapp/cron.log 2>&1
EOF
sudo chmod 644 /etc/cron.d/myapp

# Simple daily task using the /etc/cron.daily/ convention (no crontab
# syntax needed — cron runs everything in this directory once a day)
sudo tee /etc/cron.daily/cleanup_tmp << 'EOF'
#!/bin/bash
find /tmp -type f -mtime +7 -delete
EOF
sudo chmod +x /etc/cron.daily/cleanup_tmp
```

---

## Environment Setup for Reliable Jobs

```bash
# Set PATH explicitly at the top of a crontab to avoid "command not
# found" errors for tools not in cron's minimal default PATH
PATH=/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin
0 2 * * * my_custom_backup_tool --full

# Set MAILTO to control (or disable) cron's default email notifications
MAILTO=admin@example.com
0 2 * * * /usr/local/bin/backup.sh
# Set to empty to disable cron's default emailing behavior entirely:
MAILTO=""
* * * * * /usr/local/bin/quiet_task.sh

# Source a full environment explicitly within the job when a script
# genuinely depends on interactive-shell-style setup (nvm, rbenv, etc.)
0 6 * * * /bin/bash -lc 'source ~/.nvm/nvm.sh && node /home/alice/app/scheduled_task.js'
```

---

## Testing and Debugging Cron Jobs

```bash
# Test a job's exact command manually first, using the SAME minimal
# shell cron would use, to catch environment-related surprises early
/bin/sh -c '/usr/local/bin/my_script.sh'

# Temporarily schedule a job to run in the NEXT minute or two, to
# confirm it actually fires and behaves as expected before trusting
# a longer-term schedule
crontab -l
# add a temporary line like: */1 * * * * /usr/local/bin/my_script.sh >> /tmp/test.log 2>&1
# wait a minute, check /tmp/test.log, then remove the temporary line

# Check whether cron even ATTEMPTED to run a job
grep CRON /var/log/syslog | tail -20        # Debian/Ubuntu
grep CRON /var/log/cron | tail -20           # RHEL/CentOS
journalctl -u cron --since "10 minutes ago"   # systemd-based

# Verify the cron daemon itself is actually running
systemctl status cron        # Debian/Ubuntu
systemctl status crond        # RHEL/CentOS
```

---

## Real-World Recipes

```bash
# --- Nightly Database Backup with Rotation ---

0 2 * * * /usr/local/bin/backup_db.sh >> /var/log/db_backup.log 2>&1
# Inside backup_db.sh: keep only the last 7 daily backups
# find /backups/db -name "*.sql.gz" -mtime +7 -delete

# --- Health Check Every 5 Minutes, Alert on Failure ---

*/5 * * * * /usr/local/bin/health_check.sh || echo "Health check failed at $(date)" | mail -s "ALERT" admin@example.com

# --- Weekly Log Cleanup ---

0 3 * * 0 find /var/log/myapp -name "*.log" -mtime +30 -delete

# --- Certificate Renewal Check (Let's Encrypt style) ---

0 0 * * * /usr/bin/certbot renew --quiet --post-hook "systemctl reload nginx"

# --- Synchronizing Files to a Remote Server Every Hour ---

0 * * * * flock -n /tmp/sync.lock rsync -az /data/ backup-server:/data/ >> /var/log/sync.log 2>&1

# --- Run Once at Every Boot ---

@reboot /usr/local/bin/startup_checks.sh

# --- Staggered Jobs to Avoid Simultaneous Load Spikes ---

5 2 * * * /usr/local/bin/backup_app1.sh
15 2 * * * /usr/local/bin/backup_app2.sh
25 2 * * * /usr/local/bin/backup_app3.sh
# Staggering by 10 minutes each avoids all three heavy jobs
# competing for disk/CPU at the exact same instant
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
