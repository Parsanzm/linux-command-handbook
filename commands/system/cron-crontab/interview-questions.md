# cron / crontab — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Syntax & Fields](#syntax--fields)
- [Environment & Execution](#environment--execution)
- [Managing Crontabs](#managing-crontabs)
- [System vs User Crontabs](#system-vs-user-crontabs)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What's the difference between cron and crontab?**
> `cron` is the background **daemon** that continuously runs, waking up every minute to check whether any scheduled job is due. `crontab` is both the **file format** describing those schedules and the **command** used to view or edit a user's own schedule (`crontab -e`, `crontab -l`).

---

**Q2 🔥 How does the cron daemon determine whether to run a job at any given moment?**
> Once every minute, cron checks every crontab entry to see if the current minute, hour, day-of-month, month, and day-of-week all match that entry's schedule fields — if they do, cron forks a new process to execute the associated command.

---

## Syntax & Fields

**Q3 🔥 List the five fields in a crontab schedule, in order.**
> Minute (0-59), hour (0-23), day of month (1-31), month (1-12), day of week (0-7, where both 0 and 7 mean Sunday).

---

**Q4. What does `*/15 * * * *` mean?**
> "Every 15 minutes" — the `/15` step value applied to the minute field (which is `*`, meaning every possible value) selects every 15th one: 0, 15, 30, 45 minutes past each hour.

---

**Q5 🔥 What does `0 9 * * 1-5` schedule, and what's a real-world use case?**
> 9:00 AM, Monday through Friday (weekdays only) — a common schedule for business-hours-only tasks like sending a daily status report or triggering a business-day-only data sync.

---

**Q6. What's the difference between `0 9 1 * *` and `0 9 * * 1`?**
> `0 9 1 * *` runs at 9 AM on the 1st day of every month. `0 9 * * 1` runs at 9 AM every Monday. The day-of-month field (`1`) versus the day-of-week field (`1`, meaning Monday) refer to entirely different scheduling concepts.

---

## Environment & Execution

**Q7 🔥 Why might a command that works perfectly when typed manually fail when run from cron?**
> Cron jobs execute in a **minimal, non-interactive environment** — typically a stripped-down `$PATH` and no sourced shell startup files (`.bashrc`, `.bash_profile`) — unlike an interactive login shell, which loads a much richer environment. A command relying on a tool only found via a custom PATH entry, or an alias/function defined in a shell config file, will fail under cron even though it works fine when typed directly into a terminal.

---

**Q8. What are two ways to fix a cron job failing with "command not found," given that the command works fine manually?**
> (1) Use the full, absolute path to the command/script instead of relying on `$PATH` resolution (e.g., `/usr/local/bin/mytool` instead of just `mytool`). (2) Explicitly set `PATH` (and any other needed environment variables) at the top of the crontab file itself.

---

**Q9 🔥 What happens to a cron job's stdout/stderr output by default, and why can this cause a job's failures to go unnoticed for a long time?**
> By default, cron attempts to **email** any output a job produces to the crontab owner's local system mail account. On many modern systems (especially cloud servers without a configured local mail transfer agent), these emails may never be read, or may fail to deliver at all — meaning a job that's been silently erroring out on every run could go unnoticed indefinitely unless output is explicitly redirected to a monitored log file instead.

---

## Managing Crontabs

**Q10 🔥 How do you view and edit your own crontab?**
> ```bash
> crontab -l    # view (list)
> crontab -e    # edit (opens in $EDITOR)
> ```

---

**Q11. What does `crontab -r` do, and why is it considered dangerous?**
> It **immediately deletes** the entire crontab for the current user, with no confirmation prompt at all. A single accidental invocation (easily confused with `-e` on some keyboard layouts) irreversibly wipes every scheduled job, with recovery only possible if a backup was previously saved (`crontab -l > backup.txt`).

---

**Q12. How would you edit another user's crontab, and what privilege does this require?**
> ```bash
> sudo crontab -u username -e
> ```
> This requires root privileges (via sudo), since only root can modify another user's scheduled jobs directly.

---

## System vs User Crontabs

**Q13 🔥 What's the key structural difference between a personal crontab (edited via `crontab -e`) and `/etc/crontab`?**
> `/etc/crontab` (and files under `/etc/cron.d/`) include an **extra username field** between the schedule and the command, specifying which user the job should run as — since this file can define jobs for multiple different users. A personal crontab has no such field, since it implicitly runs as whichever user owns that crontab.

---

**Q14. What happens if you paste a line copied from `/etc/crontab` directly into a personal crontab without modification?**
> The extra username field (e.g., `root`) gets misinterpreted as part of the **command** to execute rather than a username, since personal crontabs don't expect that field at all — this typically causes a "command not found" style failure, since cron tries to run a nonexistent command literally named after whatever username was in that field.

---

**Q15 🔥 What's the purpose of the `/etc/cron.daily/`, `/etc/cron.hourly/`, etc. directories, and how do you add a job to one?**
> These directories let you schedule a recurring job without writing any crontab syntax at all — simply place an **executable script** inside the appropriate directory (e.g., `/etc/cron.daily/` for once-a-day execution), and the system (via `run-parts`, itself invoked by a cron entry in `/etc/crontab`) executes every script in that directory on the implied schedule.

---

## Scenario-Based

**Q16 🔥 A job is scheduled with `0 9 1 * MON`, intending to run "on the 1st of the month, but only if it's also a Monday." What will actually happen, and why?**
> When both the day-of-month and day-of-week fields are restricted (not `*`), cron treats them as an **OR** condition, not AND — this job will actually run at 9 AM if it's EITHER the 1st of the month OR a Monday (running every single Monday, plus the 1st of every month regardless of weekday), not just on the rare occasions when both conditions coincide. Achieving genuine AND logic requires an additional check inside the script/command itself (e.g., checking `date +%u` for the actual day of week).

---

**Q17. A cron job scheduled every 5 minutes occasionally seems to run multiple overlapping instances simultaneously, causing resource contention. What's the likely cause, and how do you fix it?**
> The job likely sometimes takes **longer than 5 minutes** to complete — cron has no built-in awareness of whether a previous invocation of the same job is still running, and will happily start a new instance at the next scheduled time regardless. The standard fix is wrapping the job with `flock` to prevent overlapping executions: `*/5 * * * * flock -n /tmp/job.lock /path/to/script.sh`, which skips a scheduled run entirely if the previous one is still holding the lock.

---

**Q18 🔥 A crontab entry containing `echo "Date: %d-%m-%Y"` doesn't behave as expected — part of the command seems to be missing or misinterpreted. What's the issue?**
> Cron treats an unescaped `%` character as a **newline**, splitting the command at that point — everything after the first unescaped `%` becomes the job's standard input rather than part of the command line itself. The fix is escaping every literal percent sign with a backslash: `echo "Date: \%d-\%m-\%Y"`.

---

**Q19. A backup script scheduled for 2 AM on a laptop that's regularly powered off overnight never seems to run. Is this a cron bug, and what's the standard solution?**
> Not a bug — standard cron only checks the current time once a minute while the system is actually **running**; if the machine is powered off at 2 AM, that check simply never happens, and the job is skipped for that day with no automatic catch-up. `anacron` is specifically designed for this scenario (common on laptops/desktops not always powered on), tracking the last successful run of each job and executing it on the next boot if the expected interval was missed.

---

**Q20 🔥 A cron job scheduled for "2 AM" runs at an unexpected time relative to the administrator's own local time zone, even though the server's clock itself appears correct. What's the likely explanation?**
> Cron schedules are evaluated against the **system's configured timezone**, which may differ from the administrator's own local timezone — a cloud server commonly defaults to UTC, so "2 AM" in the crontab means 2 AM UTC, not 2 AM in whatever timezone the administrator personally resides in. Checking the system's actual timezone (`timedatectl` or `cat /etc/timezone`) clarifies this, and some cron implementations support a `CRON_TZ` variable to schedule specific jobs against a different timezone than the system default (support varies by implementation and should be verified before relying on it).

---

**Q21. After editing a crontab by directly modifying the file under `/var/spool/cron/crontabs/` instead of using `crontab -e`, the new job never seems to execute. What went wrong?**
> Directly editing the underlying spool file bypasses `crontab`'s validation and proper notification of the running cron daemon — depending on the specific cron implementation, this can leave a file cron doesn't immediately recognize as changed, or one with a subtle syntax error that would have been caught and reported by `crontab -e`'s validation, but instead is silently ignored. The fix is always using the `crontab` command itself (`crontab -e`, or `crontab file.txt` to load a prepared file) rather than editing spool files directly.

---

**Q22 🔥 A script that reliably works when run manually produces no output and seemingly does nothing when triggered via cron, with no errors visible anywhere. What are the first three things you'd check?**
> (1) Check whether cron even attempted to run the job at all, via the system log (`grep CRON /var/log/syslog` or `journalctl -u cron`) — this distinguishes "cron never triggered it" from "cron triggered it, but it failed silently." (2) Check for accumulated local mail cron may have sent with error output that was never read (`mail`, or `/var/mail/username`). (3) Verify the job doesn't depend on an interactive-shell-only environment (custom PATH entries, sourced profile files, aliases) that wouldn't be present in cron's minimal execution environment, by testing the exact command manually via `/bin/sh -c '...'` rather than your normal interactive shell.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
