# who / w — The Complete Reference

> **Show who is logged in and what they are doing**
> Two of the oldest monitoring commands in Unix — both read from the same
> kernel-maintained database, but `w` tells you far more.

---

## Table of Contents

- [What are who and w?](#what-are-who-and-w)
- [Where do they live?](#where-do-they-live)
- [How they work internally](#how-they-work-internally)
- [The utmp/wtmp/btmp Database](#the-utmpwtmpbtmp-database)
- [Syntax](#syntax)
- [who: All Options](#who-all-options)
- [w: All Options](#w-all-options)
- [Output Format Deep Dive](#output-format-deep-dive)
- [Related Commands: last, lastb, lastlog, users, finger](#related-commands-family)
- [who vs w vs users vs finger](#who-vs-w-vs-users-vs-finger)
- [Related Commands](#related-commands)

---

## What are who and w?

**`who`** — shows which users are currently logged in: username, terminal, login time, and optionally where they logged in from. Clean and simple.

**`w`** — shows everything `who` shows, plus: system uptime, load average, and what each user is currently doing (their active process). More informative, same data source.

Both read from `/var/run/utmp` — a binary file maintained by the kernel and login subsystem that records all current login sessions.

**Origin:**
- `who` — Unix V1 (1971), one of the original Unix commands
- `w` — appeared in BSD Unix (~1979), written to extend `who` with process information

---

## Where do they live?

```
/usr/bin/who        ← most Linux systems
/usr/bin/w          ← most Linux systems
/bin/who            ← some older distros (symlink)
/bin/w              ← some older distros
```

```bash
which who
which w
who --version       # GNU coreutils version
w --version         # procps-ng version
```

- `who` is part of **GNU coreutils**
- `w` is part of **procps-ng** (same package as `ps`, `top`, `uptime`, `free`)

```bash
# Debian/Ubuntu:
dpkg -S $(which who)    # coreutils: /usr/bin/who
dpkg -S $(which w)      # procps: /usr/bin/w
```

No special privileges required — both are readable by any user.

---

## How they work internally

### who
```
open("/var/run/utmp") → read utmp records → filter (type=USER_PROCESS)
→ format output → write to stdout
```

1. Opens `/var/run/utmp` (binary file, fixed-size records)
2. Reads each `struct utmp` record sequentially
3. Filters for records with `ut_type == USER_PROCESS` (active logins)
4. Formats: username, tty, login time, remote host
5. Writes to stdout

### w
```
open("/var/run/utmp") → read utmp records → for each user:
  stat("/dev/TTY") → get idle time (atime of tty device)
  read("/proc/PID/stat") → get current process
  read("/proc/loadavg") → get load average
  read("/proc/uptime") → get system uptime
→ format and write
```

`w` does significantly more work per user:
- Reads `/proc/PID/stat` to find what the user is running
- Stats the TTY device to calculate idle time (last keypress)
- Reads `/proc/loadavg` for the header line
- Reads `/proc/uptime` for uptime display

**The `struct utmp` record** (defined in `<utmp.h>`):
```c
struct utmp {
    short   ut_type;          // type of record
    pid_t   ut_pid;           // PID of login process
    char    ut_line[UT_LINESIZE];  // device name (tty)
    char    ut_id[4];         // terminal name suffix
    char    ut_user[UT_NAMESIZE];  // username
    char    ut_host[UT_HOSTSIZE];  // remote hostname
    struct exit_status ut_exit;    // exit status
    long    ut_session;       // session ID
    struct timeval ut_tv;     // time entry was made
    int32_t ut_addr_v6[4];   // IP address of remote host
};
```

**Record types (`ut_type`):**
| Value | Name | Meaning |
|-------|------|---------|
| 0 | `EMPTY` | Empty record |
| 1 | `RUN_LVL` | System runlevel change |
| 2 | `BOOT_TIME` | System boot time |
| 5 | `INIT_PROCESS` | init spawned process |
| 6 | `LOGIN_PROCESS` | login process |
| 7 | `USER_PROCESS` | active user login ← what who/w show |
| 8 | `DEAD_PROCESS` | logged-out session |

---

## The utmp/wtmp/btmp Database

Three related binary files, all using `struct utmp` format:

### /var/run/utmp — Current sessions
```bash
ls -l /var/run/utmp
# -rw-rw-r-- 1 root utmp 1920 /var/run/utmp

# Current logins only (USER_PROCESS records)
# Managed by: login, sshd, su, screen, tmux, gdm
# Read by: who, w, users, finger
```

### /var/log/wtmp — Historical login log
```bash
ls -l /var/log/wtmp
# -rw-rw-r-- 1 root utmp 45056 /var/log/wtmp

# All logins AND logouts, boot/shutdown records
# Read by: last, lastb (for btmp)
# Rotated by logrotate

# Human-readable dump:
last                    # reads wtmp
last -F                 # full timestamps
last reboot             # system reboots only
last alice              # alice's login history
```

### /var/log/btmp — Failed login attempts
```bash
ls -l /var/log/btmp
# -rw------- 1 root utmp 1536 /var/log/btmp

# Records failed login attempts (wrong password)
# Read by: lastb (requires root)
# Important for security monitoring

sudo lastb              # list failed logins
sudo lastb -n 20        # last 20 failed attempts
```

### utmpdump — Inspect raw records
```bash
# Dump utmp/wtmp in human-readable format:
utmpdump /var/run/utmp
utmpdump /var/log/wtmp | head -50
utmpdump /var/log/btmp | head -20

# Output format:
# [type] [pid] [id] [user] [line] [host] [exit] [session] [time] [addr]
```

---

## Syntax

```
who [OPTION]... [FILE]
who [OPTION]... am i
```

```
w [OPTIONS] [USER]
```

- `who` with no args → reads `/var/run/utmp`, shows all logged-in users
- `who am i` → shows only the current terminal's session
- `w` with no args → shows all users with process info
- `w USERNAME` → show only that user's sessions

---

## who: All Options

| Option | Long | Description |
|--------|------|-------------|
| `-a` | `--all` | Same as `-b -d --login -p -r -t -T -u` |
| `-b` | `--boot` | Show time of last system boot |
| `-d` | `--dead` | Show dead processes |
| `-H` | `--heading` | Print column headers |
| `-l` | `--login` | Show system login processes |
| `-m` | | Only show entry for current terminal (like `who am i`) |
| `-p` | `--process` | Show active processes spawned by init |
| `-q` | `--count` | List only usernames and count |
| `-r` | `--runlevel` | Show current runlevel |
| `-s` | `--short` | Short format: name, line, time (default) |
| `-t` | `--time` | Show last system clock change |
| `-T` | `--mesg` | Add message status: `+` (writable), `-` (not writable), `?` (unknown) |
| `-u` | `--users` | Show idle time and PID |
| `-w` | `--mesg` | Same as `-T` |
|  | `--lookup` | Lookup hostnames via DNS |
| `-f` | `--from` | Show the remote hostname (default on most systems) |

```bash
who                     # basic: user, tty, time, from
who -H                  # with column headers
who -a                  # all info
who -b                  # last boot time
who -q                  # just usernames + count
who -T                  # with message status (+ - ?)
who -u                  # with idle time and PID
who am i                # current session only
who mom likes           # same as "who am i" (any two words work!)
```

---

## w: All Options

| Option | Description |
|--------|-------------|
| `-h` | No header line |
| `-u` | Ignore username when figuring out current process |
| `-s` | Short format (omit login time and CPU columns) |
| `-f` | Toggle showing remote hostname (from field) |
| `-i` | Display IP address instead of hostname |
| `-o` | Old format (show blank for idle < 1 minute) |
| `-V` | Display version |

```bash
w                       # full output
w -h                    # no header (useful in scripts)
w -s                    # short: omit login time and CPU times
w -f                    # toggle hostname display
w -i                    # show IP instead of hostname
w alice                 # only show alice's sessions
w | grep -v "^USER\|load"   # just the user lines
```

---

## Output Format Deep Dive

### who output

```
alice    pts/0    2024-06-15 10:23 (192.168.1.5)
│        │        │                │
│        │        │                └─ remote host or display
│        │        └─ login time
│        └─ TTY (terminal device)
└─ username
```

**TTY field values:**
| Value | Meaning |
|-------|---------|
| `tty1`-`tty6` | Virtual console (physical keyboard) |
| `pts/0`-`pts/N` | Pseudo-terminal (SSH, tmux, terminal emulator) |
| `:0`, `:1` | X11/Wayland display (graphical session) |
| `console` | System console |

**Message status (with `-T`):**
| Symbol | Meaning |
|--------|---------|
| `+` | Terminal accepts `write` messages (mesg y) |
| `-` | Terminal rejects messages (mesg n) |
| `?` | Terminal status unknown |

### w output

```
 10:45:23 up 5 days, 2:30,  3 users,  load average: 0.15, 0.12, 0.10
USER     TTY      FROM             LOGIN@   IDLE JCPU   PCPU WHAT
alice    pts/0    192.168.1.5      09:12    1:23  0.05s  0.02s vim project.py
bob      pts/1    10.0.0.2         08:30   15:00  0.12s  0.00s bash
carol    tty1     -                07:00    2days 0.01s  0.01s -bash
```

**Header line:**
```
10:45:23 up 5 days, 2:30, 3 users, load average: 0.15, 0.12, 0.10
│        │                │         │
│        │                │         └─ 1min, 5min, 15min load averages
│        │                └─ number of logged-in users
│        └─ system uptime
└─ current time
```

**Column breakdown:**
| Column | Description |
|--------|-------------|
| `USER` | Username |
| `TTY` | Terminal device |
| `FROM` | Remote host or IP |
| `LOGIN@` | Login time (or date if > 24h ago) |
| `IDLE` | Time since last keypress on terminal |
| `JCPU` | CPU time used by all processes on this TTY |
| `PCPU` | CPU time used by current process (WHAT) |
| `WHAT` | Current foreground process and arguments |

**IDLE field format:**
| Value | Meaning |
|-------|---------|
| `0s` | Active right now |
| `1:23` | 1 minute 23 seconds idle |
| `15:00` | 15 minutes idle |
| `2days` | Idle for 2 days (forgotten session!) |
| `-` | Not calculable |

---

## Related Commands: Family

### last — Login history from wtmp
```bash
last                        # all login/logout history
last alice                  # alice's history only
last -n 20                  # last 20 events
last -F                     # full timestamps
last reboot                 # system reboots
last shutdown               # shutdowns
last -s "2024-01-01" -t "2024-02-01"   # date range
last -i                     # show IPs instead of hostnames
```

### lastb — Failed login attempts from btmp
```bash
sudo lastb                  # all failed attempts
sudo lastb -n 50            # last 50 failures
sudo lastb alice            # failed attempts for alice
sudo lastb -i               # show IPs
```

### lastlog — Most recent login per user
```bash
lastlog                     # all users' last login
lastlog -u alice            # specific user
lastlog -t 7                # users who logged in last 7 days
lastlog -b 30               # users who haven't logged in for 30+ days

# Output:
# Username  Port    From    Latest
# alice     pts/0   server  Sat Jun 15 10:23:00 +0000 2024
# nobody                    **Never logged in**
```

### users — Just the usernames
```bash
users                       # space-separated list of logged-in users
# alice alice bob carol     ← alice is logged in twice

users | tr ' ' '\n' | sort -u   # unique users only
```

### finger — Detailed user info
```bash
finger                      # all logged-in users with full info
finger alice                # specific user (local or remote)
finger alice@remotehost     # remote user

# Output includes: login name, real name, home dir, shell,
# last login, mail status, .plan and .project files
```

---

## who vs w vs users vs finger

| Feature | `who` | `w` | `users` | `finger` |
|---------|-------|-----|---------|---------|
| Usernames | ✅ | ✅ | ✅ | ✅ |
| TTY | ✅ | ✅ | ❌ | ✅ |
| Login time | ✅ | ✅ | ❌ | ✅ |
| Remote host | ✅ | ✅ | ❌ | ✅ |
| Idle time | `-u` | ✅ | ❌ | ✅ |
| Current process | ❌ | ✅ | ❌ | ❌ |
| CPU usage | ❌ | ✅ | ❌ | ❌ |
| Load average | ❌ | ✅ | ❌ | ❌ |
| Uptime | ❌ | ✅ | ❌ | ❌ |
| GECOS/full name | ❌ | ❌ | ❌ | ✅ |
| Home/shell info | ❌ | ❌ | ❌ | ✅ |
| .plan/.project | ❌ | ❌ | ❌ | ✅ |
| Remote users | ❌ | ❌ | ❌ | ✅ |
| Data source | utmp | utmp+/proc | utmp | utmp+passwd |

---

## Related Commands

| Command | Relation |
|---------|---------|
| `last` | Login/logout history (reads `/var/log/wtmp`) |
| `lastb` | Failed login attempts (reads `/var/log/btmp`) |
| `lastlog` | Most recent login per user |
| `users` | Just list logged-in usernames |
| `finger` | Detailed user info including .plan file |
| `id` | Show UID/GID/groups of a user |
| `uptime` | System uptime and load (subset of `w` header) |
| `ps` | Process list (more detail than `w`'s WHAT column) |
| `pinky` | Lightweight finger (GNU coreutils) |
| `mesg` | Control whether others can write to your terminal |
| `write` | Send message to another logged-in user |
| `wall` | Broadcast message to all logged-in users |
| `utmpdump` | Dump utmp/wtmp binary files in text format |
| `ac` | Print statistics about user connection time |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
