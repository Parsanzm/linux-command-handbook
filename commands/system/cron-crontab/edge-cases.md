# cron / crontab — Edge Cases & Gotchas

> cron's five-field syntax looks simple, but environment differences, the day-field
> OR quirk, timezone handling, and silent email-based error reporting cause real headaches.

---

## Table of Contents

- [Day-of-Month AND Day-of-Week Is Actually OR](#day-of-month-and-day-of-week-is-actually-or)
- [Minimal Environment Breaks Scripts That Work Manually](#minimal-environment-breaks-scripts-that-work-manually)
- [Cron Job Output Silently Emailed and Never Read](#cron-job-output-silently-emailed-and-never-read)
- [Percent Signs Need Escaping](#percent-signs-need-escaping)
- [Editing Crontab Files Directly Instead of via crontab -e](#editing-crontab-files-directly-instead-of-via-crontab--e)
- [Timezone Confusion](#timezone-confusion)
- [Overlapping Job Runs When a Job Takes Longer Than Its Interval](#overlapping-job-runs-when-a-job-takes-longer-than-its-interval)
- [System Crontab Requires a Username Field, User Crontab Doesn't](#system-crontab-requires-a-username-field-user-crontab-doesnt)
- [Missed Jobs While the System Was Powered Off](#missed-jobs-while-the-system-was-powered-off)
- [crontab -r Deletes Everything With No Confirmation](#crontab--r-deletes-everything-with-no-confirmation)
- [Daylight Saving Time Transitions](#daylight-saving-time-transitions)
- [Silent Failures From Missing Trailing Newline](#silent-failures-from-missing-trailing-newline)

---

## Day-of-Month AND Day-of-Week Is Actually OR

### One of the most consistently misunderstood cron behaviors
```bash
# Intended (WRONG assumption): "run on the 1st of the month, but ONLY
# if that day happens to be a Monday"
0 9 1 * MON /usr/local/bin/report.sh

# ACTUAL behavior: runs at 9 AM if EITHER condition is true —
# it's the 1st of the month, OR it's a Monday (any Monday, every week)
# This means the job actually runs EVERY Monday, PLUS the 1st of
# every month regardless of what weekday that is — far more often
# than the "only if both match" behavior many people expect.

# If you genuinely need "1st of the month AND it must be a Monday,"
# cron's syntax alone can't express that directly — you need the
# script itself to check the condition:
0 9 1 * * [ "$(date +\%u)" = "1" ] && /usr/local/bin/report.sh
# (checks INSIDE the script/command whether today is ALSO a Monday,
# since cron's fields alone can't express a true AND between these two fields)
```

---

## Minimal Environment Breaks Scripts That Work Manually

### The single most common real-world cron complaint
```bash
# Works perfectly when typed directly into an interactive terminal:
my_custom_tool --backup

# The IDENTICAL command in a crontab:
* * * * * my_custom_tool --backup
# /bin/sh: my_custom_tool: command not found
# ⚠️ Interactive shells load a rich PATH (often including things like
# ~/.local/bin, custom tool directories, language version managers'
# shims) via .bashrc/.bash_profile — cron's environment is DELIBERATELY
# minimal and does NOT source any of these interactive shell startup
# files, since cron jobs are non-interactive, non-login processes.

which my_custom_tool
# /home/alice/.local/bin/my_custom_tool
# ✅ found manually, because YOUR interactive PATH includes this location

# Fix: ALWAYS use full, absolute paths for both the interpreter/command
# AND any files it references, or explicitly set PATH at the crontab's top
PATH=/home/alice/.local/bin:/usr/local/bin:/usr/bin:/bin
* * * * * my_custom_tool --backup
```

---

## Cron Job Output Silently Emailed and Never Read

### A job can be "failing" for months without anyone noticing
```bash
* * * * * /usr/local/bin/my_script.sh
# my_script.sh has been printing an error message every single run
# for the past six months

# By DEFAULT, cron captures a job's stdout/stderr and attempts to
# EMAIL it to the crontab owner's LOCAL system mail account — if
# nothing on this machine is actually configured to READ local mail
# (extremely common on modern cloud servers, which rarely have a
# working local MTA/mail setup at all), these error emails simply
# accumulate invisibly (or fail to send entirely) with NOBODY ever
# seeing them, creating a silent, months-long failure.

# Check for accumulated local mail that was never read
mail                              # (if a local MTA exists at all)
cat /var/mail/$(whoami) 2>/dev/null | tail -50

# MUCH more reliable: ALWAYS explicitly redirect output to a log file
# you actually monitor, rather than relying on cron's default
# email-based reporting:
* * * * * /usr/local/bin/my_script.sh >> /var/log/my_script.log 2>&1
```

---

## Percent Signs Need Escaping

### A literal % in a command has SPECIAL meaning to cron
```bash
0 9 * * * echo "Today is %d-%m-%Y" >> /var/log/date.log
# ⚠️ In a crontab, an UNESCAPED "%" is treated as a NEWLINE character
# by cron itself, and everything AFTER the first "%" becomes the
# command's STANDARD INPUT rather than part of the command line —
# this silently breaks the intended command in a confusing way.

0 9 * * * echo "Today is \%d-\%m-\%Y" >> /var/log/date.log
# ✅ escaping each "%" with a backslash makes cron treat it as a
# LITERAL percent character instead of its special newline meaning

# This commonly bites people trying to use `date`'s own formatting
# directly inside a crontab command line:
0 9 * * * echo "Backup started at $(date +\%H:\%M)" >> /var/log/backup.log
```

---

## Editing Crontab Files Directly Instead of via crontab -e

### Bypassing the crontab command can leave cron unaware of changes
```bash
# Editing the underlying spool file DIRECTLY, instead of using crontab -e
sudo vim /var/spool/cron/crontabs/alice
# ⚠️ Depending on the cron implementation, DIRECTLY edited files may
# not be picked up until cron notices the file's modification time
# has changed, OR in some stricter configurations, permission/format
# validation that "crontab -e" would normally perform is entirely
# skipped, potentially leaving a malformed file that cron silently
# ignores without ANY error message to alert you.

# ALWAYS use the crontab command itself, which handles proper
# notification to the running cron daemon AND validates syntax:
crontab -e
# or, to load a prepared file (still going through proper validation):
crontab /path/to/new_crontab_file.txt
```

---

## Timezone Confusion

### Cron uses the SYSTEM's timezone, which may not be what you assumed
```bash
# A cron job scheduled for "2 AM" assumes the SYSTEM clock/timezone,
# which might be UTC on a cloud server, NOT your own local timezone:
0 2 * * * /usr/local/bin/backup.sh
# If the server's timezone is UTC but you're in UTC+3:30 (Iran
# Standard Time, for example), this job actually runs at 5:30 AM
# YOUR local time, not 2 AM as might have been assumed when writing
# the schedule with your own local time in mind.

# Check the system's actual configured timezone
timedatectl
# or:
cat /etc/timezone

# Some cron implementations (notably newer Vixie cron / cronie
# versions) support a CRON_TZ variable to set the timezone for
# specific jobs, independent of the system's overall timezone setting:
CRON_TZ=Asia/Tehran
30 2 * * * /usr/local/bin/backup.sh
# (support for CRON_TZ varies by distro/cron implementation — verify
# it's actually honored on your specific system before relying on it)
```

---

## Overlapping Job Runs When a Job Takes Longer Than Its Interval

### A slow job scheduled too frequently can stack up multiple simultaneous executions
```bash
*/5 * * * * /usr/local/bin/slow_backup.sh
# If slow_backup.sh sometimes takes LONGER than 5 minutes to complete
# (perhaps due to unusually large data on a particular day), cron
# will happily start a SECOND instance at the next 5-minute mark,
# WHILE the first one is still running — cron has NO built-in
# awareness of whether a previous invocation of the SAME job is still
# in progress, and will start overlapping instances indefinitely if
# the job consistently runs long.

# This can lead to resource exhaustion (multiple simultaneous heavy
# processes), database lock contention, or corrupted output if the
# job isn't designed to handle concurrent execution safely.

# Fix: use flock to prevent overlapping runs
*/5 * * * * flock -n /tmp/slow_backup.lock /usr/local/bin/slow_backup.sh
```

---

## System Crontab Requires a Username Field, User Crontab Doesn't

### Copying a line between /etc/crontab and a personal crontab breaks it
```bash
# A line copied FROM /etc/crontab (which requires a username field)...
0 2 * * * root /usr/local/bin/backup.sh

# ...pasted directly into a PERSONAL crontab via `crontab -e`:
crontab -e
# 0 2 * * * root /usr/local/bin/backup.sh
# ⚠️ cron now interprets "root" as the COMMAND to execute (not a
# username), and "/usr/local/bin/backup.sh" as an ARGUMENT to that
# nonexistent "root" command — this fails, often with a confusing
# "command not found" style error, since user crontabs have NO
# username field at all (the crontab's OWNER is implicitly who it runs as).

# Personal crontab (correct, NO username field):
0 2 * * * /usr/local/bin/backup.sh

# /etc/crontab or /etc/cron.d/ files (correct, WITH username field):
0 2 * * * root /usr/local/bin/backup.sh
```

---

## Missed Jobs While the System Was Powered Off

### Standard cron does NOT catch up on jobs missed during downtime
```bash
# A laptop scheduled for a nightly 2 AM backup is completely powered
# off (not just sleeping) from 11 PM to 8 AM every night
0 2 * * * /usr/local/bin/backup.sh
# ⚠️ Standard cron simply CHECKS the current time once a minute — if
# the system wasn't even running at 2 AM, that check never happened,
# and the job is SIMPLY SKIPPED for that day, with no automatic
# "catch-up" run when the system eventually powers back on.

# anacron is specifically DESIGNED for this scenario (commonly used
# on laptops/desktops that aren't always powered on), tracking the
# LAST time each job successfully ran and executing it on next boot
# if the interval was missed:
cat /etc/anacrontab
# 1    5    daily_backup    /usr/local/bin/backup.sh
# (runs once daily, with up to a 5-minute random delay, catching up
# on the NEXT boot if it was missed due to the system being off)
```

---

## crontab -r Deletes Everything With No Confirmation

### A single mistyped command wipes an entire schedule instantly
```bash
crontab -r
# ⚠️ IMMEDIATELY deletes your ENTIRE crontab, with absolutely NO
# confirmation prompt — a single accidental keystroke (especially
# confusing `-r` with `-e`, which are adjacent on many keyboards)
# instantly and irreversibly wipes every scheduled job you had.

# There's no native "undo" — recovery depends ENTIRELY on whether you
# had a recent backup:
crontab my_crontab_backup.txt
# (only works if you previously ran `crontab -l > backup.txt` at some point)

# Best practice: ALWAYS back up before making significant changes,
# and consider aliasing crontab in your shell to add a safety habit:
alias crontab-backup='crontab -l > ~/crontab_backup_$(date +%Y%m%d_%H%M%S).txt'
# Run this habitually before any `crontab -e` session
```

---

## Daylight Saving Time Transitions

### A job scheduled during a "skipped" or "repeated" hour behaves unpredictably
```bash
# A job scheduled for 2:30 AM, on a day when clocks SPRING FORWARD
# from 2:00 AM directly to 3:00 AM (that hour never technically exists):
30 2 * * * /usr/local/bin/task.sh
# ⚠️ Depending on the specific cron implementation, this job might be
# SKIPPED entirely for that one day (since 2:30 AM never occurred),
# or might run at a slightly different adjusted time — behavior can
# vary and isn't always perfectly predictable across different cron versions.

# Conversely, when clocks FALL BACK (an hour repeats, e.g., 1:00-2:00
# AM occurs TWICE), a job scheduled during that repeated hour might
# run TWICE on that day instead of once.

# For jobs where exact, DST-transition-proof timing genuinely matters,
# scheduling well outside the typical 1-3 AM transition window (many
# regions transition around 2 AM) avoids this ambiguity entirely.
```

---

## Silent Failures From Missing Trailing Newline

### A crontab file without a final newline can cause the LAST job to be silently ignored
```bash
# If a crontab file's VERY LAST line doesn't end with a newline
# character (e.g., created by a script that doesn't add one), some
# cron implementations SILENTLY ignore that final, improperly
# terminated line entirely — no error, no warning, the job simply
# never gets registered or executed.

printf "0 2 * * * /usr/local/bin/backup.sh" | crontab -
# ⚠️ (no trailing newline) — this job might be silently dropped

printf "0 2 * * * /usr/local/bin/backup.sh\n" | crontab -
# ✅ WITH the trailing newline, the job registers correctly

# Always verify with -l after loading a crontab from a script/file,
# to confirm every intended job actually made it in:
crontab -l
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
