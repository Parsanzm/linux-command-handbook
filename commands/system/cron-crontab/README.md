# cron / crontab — The Complete Reference

> **Schedule commands to run automatically, at fixed times, dates, or intervals**
> cron dates back to Version 7 Unix (1979), written by Brian Kernighan and others at Bell Labs.
> The standard, universal way Unix systems have automated recurring tasks for over four decades.

---

## Table of Contents

- [What are cron and crontab?](#what-are-cron-and-crontab)
- [Where do cron and crontab live?](#where-do-cron-and-crontab-live)
- [How cron works internally](#how-cron-works-internally)
- [Syntax — The Crontab Line Format](#syntax--the-crontab-line-format)
- [Crontab Field Reference](#crontab-field-reference)
- [Special Characters (*, /, -, ,)](#special-characters------)
- [Special Strings (@reboot, @daily, etc.)](#special-strings-reboot-daily-etc)
- [Managing Your Crontab](#managing-your-crontab)
- [System-Wide vs User Crontabs](#system-wide-vs-user-crontabs)
- [Environment Variables in Cron Jobs](#environment-variables-in-cron-jobs)
- [Logging and Output Handling](#logging-and-output-handling)
- [cron vs systemd Timers vs at](#cron-vs-systemd-timers-vs-at)
- [Related Commands](#related-commands)

---

## What are cron and crontab?

**cron** is a background daemon (a long-running system process) that wakes up once every minute and checks whether any scheduled job is due to run, executing it if so. **crontab** ("cron table") is both the **file format** used to describe those schedules, and the **command** used to view/edit your own personal schedule.

```bash
crontab -l              # list YOUR current scheduled jobs
crontab -e               # edit YOUR scheduled jobs (opens in $EDITOR)
```

A single line in a crontab describes one recurring job:
```
0 2 * * * /usr/local/bin/backup.sh
# ↑ ↑ ↑ ↑ ↑
# │ │ │ │ └─ day of week
# │ │ │ └─── month
# │ │ └───── day of month
# │ └─────── hour
# └───────── minute
# → runs backup.sh every day at 2:00 AM
```

**Why cron remains essential in 2026 despite newer alternatives:** it's universally present on every Unix-like system with zero setup, its syntax is a shared, widely-understood convention across decades of tooling and documentation, and for straightforward "run this at this time" scheduling, nothing is simpler or more portable.

---

## Where do cron and crontab live?

```bash
which crontab
# /usr/bin/crontab

ps aux | grep cron
# root   1234  ...  /usr/sbin/cron    ← the cron DAEMON, always running in the background
```

Crontab files themselves are stored per-user, typically under:
```
/var/spool/cron/crontabs/USERNAME     (Debian/Ubuntu convention)
/var/spool/cron/USERNAME               (RHEL/CentOS convention)
```

System-wide crontab files live separately:
```
/etc/crontab
/etc/cron.d/*
/etc/cron.hourly/, /etc/cron.daily/, /etc/cron.weekly/, /etc/cron.monthly/
```

**Never edit files under `/var/spool/cron/` directly** — always use the `crontab` command, which validates syntax and properly notifies the cron daemon of changes.

---

## How cron works internally

### The daemon's wake-up cycle

The `cron` daemon runs continuously in the background and, once every minute, checks every user's crontab (and the system-wide crontabs) to see if any job's schedule matches the **current minute**. If a match is found, cron forks a new process to execute that job.

```bash
# Conceptually, cron performs a check like this every 60 seconds:
# for each crontab entry:
#   if (current_minute matches entry's minute field) AND
#      (current_hour matches entry's hour field) AND
#      (current_day_of_month matches entry's day field) AND
#      (current_month matches entry's month field) AND
#      (current_day_of_week matches entry's weekday field):
#     execute(entry's command)
```

### Day-of-month AND day-of-week — the "OR" quirk

When **both** the day-of-month and day-of-week fields are restricted (not `*`), cron treats them as an **OR** condition, not AND — a detail that surprises many people:

```bash
0 9 1 * MON /usr/local/bin/report.sh
# Runs at 9:00 AM if EITHER: it's the 1st of the month, OR it's a Monday
# — NOT "the 1st of the month, but only if that day happens to be a Monday"
# as many people initially assume.
```

### Each job runs in a minimal, non-interactive environment

Cron jobs execute with a **much more limited environment** than an interactive login shell — a stripped-down `$PATH`, no inherited shell aliases/functions, and typically no `$HOME`-dependent config files sourced. This is the single most common source of "it works when I run it manually, but not from cron" bugs (see [Environment Variables in Cron Jobs](#environment-variables-in-cron-jobs) below).

---

## Syntax — The Crontab Line Format

```
* * * * * command_to_execute
│ │ │ │ │
│ │ │ │ └── day of week   (0-7, both 0 and 7 = Sunday)
│ │ │ └──── month         (1-12)
│ │ └────── day of month  (1-31)
│ └──────── hour          (0-23)
└────────── minute        (0-59)
```

```bash
30 4 * * *      /path/to/script.sh    # 4:30 AM, every day
0 */2 * * *     /path/to/script.sh    # every 2 hours, on the hour
0 9 * * 1-5     /path/to/script.sh    # 9:00 AM, weekdays only (Mon-Fri)
0 0 1 * *       /path/to/script.sh    # midnight, on the 1st of every month
*/15 * * * *    /path/to/script.sh    # every 15 minutes
```

---

## Crontab Field Reference

| Field | Allowed Values | Notes |
|-------|------------------|-------|
| Minute | 0-59 | |
| Hour | 0-23 | 24-hour format, no AM/PM |
| Day of month | 1-31 | |
| Month | 1-12 | Names also accepted: `jan`, `feb`, ... `dec` |
| Day of week | 0-7 | 0 AND 7 both mean Sunday; names also accepted: `sun`, `mon`, ... `sat` |

```bash
0 9 * * mon-fri    /path/to/script.sh    # names work as an alternative to numbers
0 0 1 jan,jul *    /path/to/script.sh    # January 1st AND July 1st, at midnight
```

---

## Special Characters (*, /, -, ,)

| Character | Meaning | Example |
|-----------|---------|---------|
| `*` | Any value ("every") | `* * * * *` = every minute |
| `,` | A list of specific values | `0 9,17 * * *` = 9 AM and 5 PM |
| `-` | A range of values | `0 9-17 * * *` = every hour from 9 AM to 5 PM |
| `/` | A step value ("every Nth") | `*/15 * * * *` = every 15 minutes |

```bash
# Combine multiple special characters in one field
0 9-17/2 * * *      # every 2 hours, from 9 AM through 5 PM (9, 11, 13, 15, 17)
0 0,12 1,15 * *      # midnight and noon, on the 1st and 15th of each month
30 8-10,14-16 * * *   # minute 30, during hours 8-10 AND 14-16 (two separate ranges)
```

---

## Special Strings (@reboot, @daily, etc.)

GNU/Vixie cron supports human-readable shortcuts in place of the 5-field schedule:

| String | Equivalent to | Meaning |
|--------|-----------------|---------|
| `@reboot` | (none) | Run once, at system startup |
| `@yearly` / `@annually` | `0 0 1 1 *` | Once a year, midnight Jan 1st |
| `@monthly` | `0 0 1 * *` | Once a month, midnight on the 1st |
| `@weekly` | `0 0 * * 0` | Once a week, midnight on Sunday |
| `@daily` / `@midnight` | `0 0 * * *` | Once a day, at midnight |
| `@hourly` | `0 * * * *` | Once every hour, on the hour |

```bash
@reboot /usr/local/bin/startup_script.sh
@daily  /usr/local/bin/backup.sh
@hourly /usr/local/bin/health_check.sh
```

---

## Managing Your Crontab

```bash
# List your current crontab
crontab -l

# Edit your crontab (opens in $EDITOR, usually vim/nano)
crontab -e

# Remove your ENTIRE crontab (all scheduled jobs, no confirmation!)
crontab -r

# Load a crontab from a file, REPLACING your current one entirely
crontab my_schedule.txt

# Edit ANOTHER user's crontab (requires root/sudo)
sudo crontab -u alice -e
sudo crontab -u alice -l

# Back up your crontab before making risky changes
crontab -l > my_crontab_backup.txt
```

---

## System-Wide vs User Crontabs

### User crontabs — managed via `crontab -e`, no username field
```
0 2 * * * /home/alice/backup.sh
```

### System-wide crontab (`/etc/crontab`) — includes an EXTRA username field
```
# /etc/crontab format INCLUDES which user to run the command as:
0 2 * * * root /usr/local/bin/system_backup.sh
0 3 * * * alice /home/alice/personal_task.sh
```

### `/etc/cron.d/` — drop-in files, same format as `/etc/crontab`
```bash
cat /etc/cron.d/my_app
# 0 4 * * * appuser /opt/myapp/nightly_job.sh
```

### The `/etc/cron.{hourly,daily,weekly,monthly}/` directories
```bash
# Simply place an EXECUTABLE script in the relevant directory —
# no crontab-syntax line needed at all, cron runs everything inside
# these directories on the implied schedule automatically
sudo cp my_daily_task.sh /etc/cron.daily/
sudo chmod +x /etc/cron.daily/my_daily_task.sh
```

---

## Environment Variables in Cron Jobs

### The most common cause of "works manually, fails in cron" bugs

```bash
# Cron jobs run with a MINIMAL environment — often just:
# SHELL=/bin/sh
# PATH=/usr/bin:/bin
# (NOT your interactive shell's full PATH, aliases, or sourced profile files)

# A cron job calling a program NOT in this minimal PATH fails silently:
* * * * * my_custom_tool --do-something
# /bin/sh: my_custom_tool: command not found
# (even though `my_custom_tool` works fine when YOU type it manually,
# because YOUR interactive shell has a much richer PATH)

# Fix #1: use the FULL, absolute path to every command and script
* * * * * /usr/local/bin/my_custom_tool --do-something

# Fix #2: explicitly set PATH (and other needed variables) AT THE TOP
# of the crontab file itself
PATH=/usr/local/bin:/usr/bin:/bin
* * * * * my_custom_tool --do-something

# Fix #3: source your full profile explicitly within the job itself
* * * * * . $HOME/.bashrc && my_custom_tool --do-something
```

---

## Logging and Output Handling

```bash
# By default, cron EMAILS any output (stdout/stderr) to the crontab
# owner's local mail account, UNLESS explicitly redirected
* * * * * /path/to/script.sh
# (if the script produces ANY output, expect a LOCAL email — which
# may go completely unnoticed if nothing reads local mail on this system)

# Redirect output to a log file instead of relying on mail
* * * * * /path/to/script.sh >> /var/log/myscript.log 2>&1

# Silence output entirely (rarely recommended — you lose ALL error visibility)
* * * * * /path/to/script.sh > /dev/null 2>&1

# Redirect ONLY errors to a log, letting normal output go to cron's default mail
* * * * * /path/to/script.sh 2>> /var/log/myscript_errors.log

# Check the system log for cron's OWN activity record (did it even
# ATTEMPT to run the job?)
grep CRON /var/log/syslog        # Debian/Ubuntu
grep CRON /var/log/cron           # RHEL/CentOS
journalctl -u cron                 # systemd-based logging
```

---

## cron vs systemd Timers vs at

| Tool | Best for | Key characteristic |
|------|----------|----------------------|
| `cron` | Recurring schedules (every day, every hour, etc.) | Simple, universal, decades-old convention |
| `systemd timers` | Recurring schedules, with tighter systemd integration | Better logging (via journald), dependency management, can wait for network/other units |
| `at` | ONE-TIME scheduled execution ("run this once, later") | Not recurring — schedules a SINGLE future execution |

```bash
crontab -e                  # recurring job, classic cron syntax
systemctl edit --force my-task.timer   # recurring job, modern systemd approach
echo "backup.sh" | at 2am    # ONE-TIME job, runs once tonight at 2 AM, never repeats
```

**Rule of thumb:** use `cron` for simple, portable, recurring schedules that don't need deep systemd integration; use `systemd timers` on systemd-based systems wanting better logging/dependency awareness; use `at` for genuine one-off future execution rather than a repeating schedule.

---

## Related Commands

| Command | Relation |
|---------|----------|
| `at` | One-time (non-recurring) scheduled execution |
| `systemd-timers` | Modern systemd-native alternative to cron for recurring jobs |
| `anacron` | cron variant designed for machines that aren't always powered on (laptops), catching up on MISSED jobs |
| `run-parts` | Executes every script in a directory, used internally by `/etc/cron.daily` etc. |
| `crontab` | The command AND file format used to manage cron schedules |
| `journalctl` / `syslog` | Where cron's own execution log entries can be reviewed |
| `flock` | Often combined with cron jobs to prevent overlapping runs of a slow job |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
