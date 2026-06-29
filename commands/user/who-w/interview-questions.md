# who / w — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [Output Interpretation](#output-interpretation)
- [Security & Auditing](#security--auditing)
- [Login History](#login-history)
- [Scenario-Based](#scenario-based)
- [Advanced & Internals](#advanced--internals)

---

## Conceptual

**Q1 🔥 What is the difference between `who` and `w`?**
> Both show currently logged-in users and both read from `/var/run/utmp`. The difference:
>
> `who` — shows: username, TTY, login time, remote host. Simple, minimal output.
>
> `w` — shows everything `who` shows, plus: system uptime, load average, idle time per user, CPU usage (JCPU/PCPU), and the current foreground process (WHAT). It also reads `/proc` for process and system info.
>
> Use `who` when you only need to know who's logged in. Use `w` when you need to understand what they're doing and what's happening on the system.

---

**Q2 🔥 What file do `who` and `w` read from?**
> `/var/run/utmp` — a binary file in `struct utmp` format, maintained by the login subsystem (login, sshd, gdm, etc.). It contains records for all current login sessions with type `USER_PROCESS`.
>
> Related files:
> - `/var/log/wtmp` — same format, historical log (all logins + logouts + boots)
> - `/var/log/btmp` — failed login attempts only

---

**Q3. What is the `struct utmp` and what does it store?**
> A C structure (defined in `<utmp.h>`) representing one login record:
> - `ut_type` — record type (USER_PROCESS for active logins)
> - `ut_pid` — PID of the login process
> - `ut_line` — TTY device name (e.g., "pts/0")
> - `ut_user` — username (up to 32 chars)
> - `ut_host` — remote hostname or display (up to 256 chars)
> - `ut_tv` — timestamp (timeval: seconds + microseconds)
> - `ut_addr_v6` — remote IP address (4×int32 = IPv6-capable)

---

**Q4. What does `who am i` do and how is it different from `whoami`?**
> - `who am i` — shows the session info of the **terminal you're currently using**: username, TTY, login time, and source IP. Uses utmp data. Shows who you logged in as (even if you've `su`'d to another user).
> - `whoami` — shows the **effective username**: who you are RIGHT NOW (changes if you `sudo` or `su`).
>
> ```bash
> # Logged in as alice, then: sudo su - root
> who am i      # alice    pts/0   ... (shows original login)
> whoami        # root     (shows current effective user)
> ```

---

**Q5. What is `/var/log/wtmp` used for?**
> `wtmp` is the historical login log — same binary format as utmp, but records ALL events: every login, logout, system boot, shutdown, and runlevel change. Never truncated (unlike utmp which only has current sessions). Read by `last`. Rotated by `logrotate` (typically monthly, keeping 1-2 months by default).

---

## Basic Usage

**Q6. How do you show column headers with `who`?**
> ```bash
> who -H
> # NAME     LINE         TIME             COMMENT
> # alice    pts/0        2024-06-15 10:23 (192.168.1.5)
> ```

---

**Q7. How do you show the last system boot time?**
> ```bash
> who -b
> # system boot  2024-06-10 08:15
>
> # Or:
> uptime -s         # shows boot time
> last reboot | head -1  # from wtmp
> ```

---

**Q8. What does the message status column in `who -T` mean?**
> Shows whether the user's terminal accepts messages from `write` and `wall`:
> - `+` — terminal is writable (`mesg y` — default for console logins)
> - `-` — terminal is not writable (`mesg n` — default for SSH logins)
> - `?` — status cannot be determined
>
> ```bash
> who -T
> # alice    - pts/0   2024-06-15 10:23 (192.168.1.5)
> #            ↑ minus: can't send write messages to alice
>
> # Change:
> mesg y    # allow messages
> mesg n    # block messages
> ```

---

**Q9. How do you count how many users are currently logged in?**
> ```bash
> who | wc -l          # counts sessions (one user may have multiple)
> users | wc -w        # counts sessions as well
> users | tr ' ' '\n' | sort -u | wc -l   # unique users only
> ```

---

**Q10. How do you show only your own session with `who`?**
> ```bash
> who am i       # show current session
> who -m         # same effect
> who mom likes  # same (any two words work — historical joke)
> ```

---

## Output Interpretation

**Q11 🔥 Explain all columns of `w` output.**
> ```
>  10:45:23 up 5 days, 2:30,  3 users,  load average: 0.15, 0.12, 0.10
> USER     TTY      FROM          LOGIN@   IDLE  JCPU   PCPU  WHAT
> alice    pts/0    192.168.1.5   10:23    5:00  0.05s  0.02s vim server.py
> ```
> Header: current time, uptime, user count, 1/5/15-minute load averages.
>
> Columns:
> - `USER` — username
> - `TTY` — terminal device (pts/N = SSH/pseudoterm, ttyN = virtual console)
> - `FROM` — remote host or IP
> - `LOGIN@` — login time (HH:MM or date if > 24h ago)
> - `IDLE` — time since last keypress on this terminal
> - `JCPU` — CPU time used by ALL processes attached to this TTY
> - `PCPU` — CPU time used by the current foreground process (WHAT)
> - `WHAT` — current foreground process and its arguments

---

**Q12. What does a `pts/N` TTY vs `ttyN` TTY mean?**
> - `pts/N` (pseudo-terminal slave) — a virtual terminal created for SSH connections, terminal emulators, tmux/screen windows. `N` increases with each new connection.
> - `ttyN` — a virtual console (Ctrl+Alt+F1 through F6 on a physical machine). Physical keyboard input.
> - `console` or `tty` — the system console itself.
>
> On a server, you'd expect to see `pts/N` only (remote logins). `ttyN` on a server suggests someone is physically at the machine.

---

**Q13 🔥 A user shows `2days` in the IDLE column of `w`. What does this tell you?**
> The user hasn't pressed any key on their terminal for 2 days. This usually means:
> - A forgotten SSH session left open
> - A `screen` or `tmux` session that was reattached but never used
> - An automated process that opened a terminal but never interacted with it
>
> The session itself is still active (they appear in `who`/`w`), but their terminal has been completely idle. Good reason to investigate or clean up:
> ```bash
> w | grep "2days"   # find them
> # Consider: pkill -9 -t pts/N  # kill the idle session
> ```

---

**Q14. What is the difference between JCPU and PCPU in `w`?**
> - `JCPU` — **J**ob CPU: total CPU time used by ALL processes that have the same controlling TTY. Includes background jobs, subshells, etc.
> - `PCPU` — **P**rocess CPU: CPU time used only by the **current foreground process** shown in the WHAT column.
>
> If JCPU >> PCPU, there are background processes consuming CPU on that terminal.

---

## Security & Auditing

**Q15 🔥 Can you always trust `who` and `w` to show all active users?**
> No — `who` and `w` can be spoofed or incomplete:
> 1. Root (or an attacker with root) can modify `/var/run/utmp` to hide sessions — specialized rootkit tools do this automatically.
> 2. `screen`/`tmux` detached sessions don't appear after the original SSH session closes.
> 3. Cron jobs, setuid processes, and nohup processes running as a user don't appear.
> 4. Processes running inside containers are not visible in the host's `who` output.
>
> More reliable (harder to fake): `ps aux`, `/proc/*/loginuid`, `audit` daemon logs.

---

**Q16. How do you find users with suspicious idle times or suspicious source IPs?**
> ```bash
> # Users idle > 2 hours (potential forgotten sessions or compromised accounts)
> w -h | awk '$5 ~ /^[2-9]:[0-9][0-9]$/ || $5 ~ /days/ {print $1, $2, $5}'
>
> # Logins from non-internal IPs
> who | grep -vE "\(192\.168\.|10\.|172\.(1[6-9]|2[0-9]|3[01])\.|localhost\)"
>
> # Root logged in via SSH (should be rare/prohibited)
> who | grep "^root.*pts/"
>
> # Multiple sessions from same IP (potential compromised credentials)
> who | awk '{print $5}' | sort | uniq -d
> ```

---

**Q17 🔥 What is `/var/log/btmp` and how do you use it for security monitoring?**
> `btmp` records all **failed login attempts** in `struct utmp` format. Read by `lastb` (requires root):
> ```bash
> sudo lastb           # all failed attempts
> sudo lastb -n 50     # last 50
> sudo lastb root      # failed root logins (brute force indicator)
>
> # Top IPs attempting brute force:
> sudo lastb -i | awk 'NF>0 && !/^btmp/ {print $3}' \
>   | sort | uniq -c | sort -rn | head -20
> ```
> Key use: detecting brute-force attacks. If you see thousands of failed `root` attempts from one IP, it's being attacked.

---

**Q18. How does `mesg` relate to `who -T`?**
> `mesg` controls whether other users can send messages to your terminal via `write` or `wall`. `who -T` shows the current status:
> ```bash
> mesg y       # allow messages → who -T shows "+"
> mesg n       # block messages → who -T shows "-"
> mesg         # show current status
> ```
> Most SSH sessions default to `mesg n` (no messages). This prevents other users from sending disruptive output to your terminal. Server administrators sometimes use `mesg y` + `wall` to broadcast maintenance notices.

---

## Login History

**Q19 🔥 What is the difference between `last`, `lastb`, and `lastlog`?**
> - `last` — reads `/var/log/wtmp`. Shows login/logout history for ALL users, including when they logged in, from where, and how long they were logged in. Also shows system reboots.
> - `lastb` — reads `/var/log/btmp`. Shows **failed login attempts** only. Requires root.
> - `lastlog` — reads `/var/log/lastlog` (indexed by UID). Shows the **most recent login** for each user account. Shows "Never logged in" for system accounts.
>
> Use `last` for: history and session duration. Use `lastb` for: brute-force detection. Use `lastlog` for: finding stale/unused accounts.

---

**Q20. How do you find how long a user's session lasted using `last`?**
> ```bash
> last alice
> # alice  pts/0  192.168.1.5  Sat Jun 15 10:23 - 14:35  (04:12)
> #                                                          ↑ duration
>
> # "still logged in" means the session is active
> # "gone - no logout" means the session ended abnormally (crash, network loss)
>
> last -F alice    # full timestamps with seconds
> ```

---

**Q21. How do you check when a system was last rebooted?**
> ```bash
> last reboot | head -3       # from wtmp history
> who -b                      # current boot time (from utmp)
> uptime -s                   # boot time in YYYY-MM-DD HH:MM:SS
> uptime                      # time since boot
> cat /proc/uptime            # seconds since boot
> systemctl show --property=KernelTimestamp   # systemd systems
> ```

---

**Q22. How do you view login history beyond what `last` shows (rotated wtmp)?**
> ```bash
> # Older wtmp after rotation:
> last -f /var/log/wtmp.1
> last -f /var/log/wtmp.2.gz   # if compressed
>
> # Search all available wtmp files:
> for f in /var/log/wtmp*; do
>   last -f "$f" | grep "alice"
> done
>
> # Extend retention (edit logrotate config):
> # /etc/logrotate.d/wtmp: rotate 12  (keep 12 months)
> ```

---

## Scenario-Based

**Q23 🔥 A user says "someone is using my account" but they're not logged in right now. How do you investigate?**
> ```bash
> # 1. Check current logins
> who
>
> # 2. Check recent login history
> last alice | head -20
>
> # 3. Check failed attempts (password guessing?)
> sudo lastb alice | head -20
>
> # 4. Check last successful login time and source
> lastlog -u alice
>
> # 5. Check for unusual source IPs in history
> last alice | awk '{print $3}' | sort | uniq -c | sort -rn
>
> # 6. Check SSH auth log for anomalies
> grep "alice" /var/log/auth.log | grep -E "Accepted|Failed" | tail -30
>
> # 7. Check if account has active processes
> ps aux | grep alice
> ```

---

**Q24. How do you detect that a system has been compromised using login-related commands?**
> ```bash
> # 1. Unexplained reboots (attacker installing kernel module?)
> last reboot | head -10
>
> # 2. Root SSH logins (should be disabled)
> last root | grep "pts/"
>
> # 3. Logins at unusual hours
> last | awk '{print $4, $5, $6, $1}' | grep -E "0[0-3]:[0-9]"
>
> # 4. Logins from external IPs
> last -i | grep -vE "^reboot|^shutdown|192\.168\.|10\.|127\.|::1"
>
> # 5. Massive brute force on btmp
> sudo lastb | wc -l   # thousands = being attacked
>
> # 6. Users that never log in showing login activity
> lastlog | grep -v "Never" | awk '$1 ~ /^(www-data|nobody|daemon)/'
>
> # 7. who shows users not in /etc/passwd
> who | awk '{print $1}' | while read u; do
>   id "$u" &>/dev/null || echo "UNKNOWN USER: $u"
> done
> ```

---

**Q25 🔥 `who` shows no users logged in but `ps aux` shows many user processes. How is this possible?**
> Several valid scenarios:
> 1. **Detached screen/tmux sessions** — users disconnected SSH but their sessions live on in screen/tmux. Their processes run but no utmp entry.
> 2. **Cron jobs** — scheduled tasks running as users, no terminal, no utmp entry.
> 3. **systemd user services** — user services started at boot, not login sessions.
> 4. **nohup processes** — user ran a background process and logged out.
> 5. **Compromised system** — rootkit modified utmp to hide attacker's sessions.
>
> Verify:
> ```bash
> ps aux        # see all processes
> screen -list  # detached screen sessions
> tmux ls       # detached tmux sessions
> ls /proc/*/loginuid   # kernel-maintained login audit
> ```

---

**Q26. During a security incident, you need to immediately see all active connections to the server. What commands do you run?**
> ```bash
> # 1. Who's logged in right now
> w
>
> # 2. All network connections
> ss -tnp      # TCP connections + PIDs
> netstat -tnp  # alternative
>
> # 3. Active SSH sessions specifically
> ss -tnp | grep ":22"
>
> # 4. All processes by all users
> ps auxf      # forest format showing parent/child
>
> # 5. Recent logins
> last -n 20
>
> # 6. Failed attempts (ongoing brute force?)
> sudo lastb -n 20
>
> # 7. Check for hidden processes (discrepancy = rootkit indicator)
> ls /proc | wc -l   # number of process directories
> ps aux | wc -l     # should be similar
> ```

---

## Advanced & Internals

**Q27 🔥 How does `w` calculate the IDLE time?**
> `w` calls `stat()` on the TTY device file (e.g., `/dev/pts/0`) and reads its `st_atime` (access time) — the last time the terminal device was read from, which corresponds to the last keypress. IDLE = current time - st_atime of the TTY device.
>
> This is why:
> - Programs that output to the terminal (like `top`) don't reset IDLE (they write, not read)
> - SSH keepalives don't reset IDLE (they don't generate TTY read events)
> - Actual typing resets IDLE to 0

---

**Q28. What syscalls does `who` use to get its information?**
> ```
> open("/var/run/utmp")    ← open the database
> read(fd, &utmp, sizeof)  ← read each record sequentially
> close(fd)
> write(1, output, len)    ← write formatted output to stdout
> ```
> For each record, it checks `ut_type == USER_PROCESS` to filter active sessions. It's essentially just a formatted reader of a binary file — one of the simplest implementations in Unix.

---

**Q29. Why can root hide from `who` but not easily from `/proc`?**
> `/var/run/utmp` is a regular file writable by root and members of the `utmp` group. A process with root privileges can directly modify it to remove their session record.
>
> `/proc/PID/` directories are created by the **kernel** for every running process. They cannot be modified by userspace programs — they exist as long as the process runs and disappear when it exits. To hide from `/proc`, an attacker would need a kernel module (rootkit) that intercepts and filters `/proc` filesystem calls.
>
> Therefore: `who` can be spoofed with root access; `/proc` requires kernel-level compromise.

---

**Q30 🔥 What is `utmpdump` and when would you use it?**
> `utmpdump` converts the binary utmp/wtmp/btmp files to human-readable text (and back). Used for:
> 1. **Forensics** — read raw records without using `who`/`last` (which may themselves be compromised)
> 2. **Debugging** — see all record types including BOOT_TIME, RUN_LVL, DEAD_PROCESS
> 3. **Editing** — modify records and write back (forensic cleanup or testing)
>
> ```bash
> utmpdump /var/run/utmp          # current sessions (all types)
> utmpdump /var/log/wtmp | grep alice  # alice's history in raw format
> utmpdump /var/log/btmp | tail   # recent failed logins in raw format
>
> # Edit and write back:
> utmpdump /var/run/utmp > /tmp/utmp.txt
> # edit /tmp/utmp.txt
> utmpdump -r /tmp/utmp.txt > /var/run/utmp
> ```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
