# ps — The Complete Reference

> **Snapshot of the processes currently running on the system**
> Present since early Unix, standardized in POSIX, and the command
> nearly everyone reaches for the moment something feels "stuck."

---

## Table of Contents

- [What is ps?](#what-is-ps)
- [Where does ps live?](#where-does-ps-live)
- [How ps works internally](#how-ps-works-internally)
- [Syntax](#syntax)
- [Understanding the Output](#understanding-the-output)
- [The Two Option Styles — BSD vs UNIX vs GNU](#the-two-option-styles--bsd-vs-unix-vs-gnu)
- [Process States Explained](#process-states-explained)
- [Customizing Output with -o](#customizing-output-with--o)
- [All Key Options](#all-key-options)
- [ps and /proc](#ps-and-proc)
- [ps vs top vs htop vs pgrep](#ps-vs-top-vs-htop-vs-pgrep)
- [Related Commands](#related-commands)

---

## What is ps?

`ps` prints a **snapshot** — a single point-in-time listing — of currently running processes, along with details like their process ID, the terminal they're attached to, how much CPU time they've used, and the command that launched them. Unlike `top`, it doesn't refresh continuously; it prints once and exits.

```bash
ps aux
# USER  PID  %CPU %MEM    VSZ   RSS TTY  STAT START   TIME COMMAND
# alice 1234  0.1  0.3  22344  9876 pts/0 Ss   09:15   0:00 bash
# alice 5678  2.3  1.1 812340 45200 pts/0 R+   10:02   0:15 firefox
```

**Why "snapshot" matters:** by the time `ps`'s output reaches your screen, the actual process table may have already changed — processes shown as running may have already exited, and new ones may have started. For continuously updating views, use `top` or `htop` instead (see [ps vs top vs htop vs pgrep](#ps-vs-top-vs-htop-vs-pgrep)).

---

## Where does ps live?

```
/usr/bin/ps  (or /bin/ps on some systems)
```

```bash
which ps
ps --version
# ps from procps-ng 4.0.4
```

Part of **procps** (or **procps-ng**) on Linux, the same package providing `top`, `free`, `uptime`, `w`, and `vmstat`. Present in some form on virtually every Unix-like system, though the exact option set and default output differ meaningfully between Linux, macOS, and BSD.

---

## How ps works internally

`ps` reads process information directly from the kernel's `/proc` filesystem — specifically, one subdirectory per running process, `/proc/<pid>/`, each containing files describing that process's state, memory, command line, and more. `ps` doesn't maintain its own process database; every invocation re-reads `/proc` fresh.

```bash
ls /proc/1234/
# cmdline  cwd  environ  exe  fd  maps  mem  root  stat  status  ...

cat /proc/1234/status | head -5
# Name:   bash
# State:  S (sleeping)
# Tgid:   1234
# Pid:    1234
# PPid:   1000
```

Key files `ps` draws from:

| File | What it provides |
|---|---|
| `/proc/<pid>/stat` | Numeric process stats — state, PPID, priority, CPU times, memory |
| `/proc/<pid>/status` | Human-readable version of much of `stat`, plus more detail |
| `/proc/<pid>/cmdline` | The exact command line the process was launched with |
| `/proc/<pid>/environ` | The process's environment variables |

Because `ps` reads `/proc` fresh every time it runs, there's no daemon, no caching, and no persistent state — just a full re-scan on each invocation.

---

## Syntax

```bash
ps [OPTIONS]
```

`ps` is unusual in that it accepts **three different, historically distinct option styles simultaneously** (see [next section](#the-two-option-styles--bsd-vs-unix-vs-gnu)) — which is also the single biggest source of `ps` confusion for newcomers.

---

## Understanding the Output

```bash
ps aux
# USER   PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
# alice  1234  0.1  0.3  22344  9876 pts/0    Ss   09:15   0:00 bash
```

| Column | Meaning |
|---|---|
| `USER` | Owner of the process |
| `PID` | Process ID — the unique identifier used to target it with `kill`, `renice`, etc. |
| `%CPU` | Percentage of CPU time used, calculated as an average since the process started |
| `%MEM` | Percentage of physical RAM used (based on resident set size) |
| `VSZ` | Virtual memory size in KB — total address space, including memory not actually resident |
| `RSS` | Resident Set Size in KB — the portion of memory actually held in physical RAM right now |
| `TTY` | Controlling terminal (`?` means none — typical for daemons/background services) |
| `STAT` | Process state code (see [Process States Explained](#process-states-explained)) |
| `START` | Time or date the process started |
| `TIME` | Cumulative CPU time actually consumed (not wall-clock time running) |
| `COMMAND` | The command and arguments that launched the process |

---

## The Two Option Styles — BSD vs UNIX vs GNU

`ps` supports three overlapping, historically separate option conventions, and mixing them up is the most common source of confusion:

```bash
ps aux      # BSD style — no leading dash, options combined: a, u, x
ps -ef      # UNIX/POSIX style — leading dash: -e, -f
ps --pid 1234   # GNU long-option style — double dash
```

| Style | Example | Notes |
|---|---|---|
| BSD | `ps aux` | No leading dash. `a` = processes for all users with a tty, `u` = user-oriented format, `x` = include processes without a controlling terminal |
| UNIX/POSIX | `ps -ef` | Leading dash. `-e` = every process, `-f` = full-format listing |
| GNU long | `ps --pid 1234` | Double-dash, more explicit/self-documenting |

`ps aux` and `ps -ef` are **not identical** — they show largely overlapping information but with different column sets and formatting conventions inherited from their respective traditions. Both are extremely common; which one you'll see depends heavily on which Unix background the person writing a given script or tutorial came from.

---

## Process States Explained

The `STAT`/`S` column uses single-letter codes, often followed by modifiers:

| Code | Meaning |
|---|---|
| `R` | Running or runnable (on the run queue) |
| `S` | Interruptible sleep — waiting for an event (very common, e.g., waiting for input) |
| `D` | Uninterruptible sleep — usually waiting on I/O; can't be killed or interrupted while in this state |
| `T` | Stopped — by a job-control signal (e.g., `Ctrl+Z`) or being traced |
| `Z` | Zombie — process has exited but its parent hasn't yet reaped its exit status |
| `I` | Idle (kernel thread) |

Common modifiers appended after the base letter:

| Modifier | Meaning |
|---|---|
| `<` | High priority (not nice to others) |
| `N` | Low priority (nice) |
| `L` | Has pages locked into memory |
| `s` | Is a session leader |
| `+` | In the foreground process group of its terminal |

```bash
ps aux | awk '$8 ~ /^Z/'
# Find zombie processes specifically
```

---

## Customizing Output with -o

`-o` (or `--format`) lets you specify exactly which columns to show, in whatever order you want:

```bash
ps -eo pid,ppid,user,%cpu,%mem,cmd
#     PID    PPID USER      %CPU %MEM CMD
#     1234    1000 alice      0.1  0.3 bash

# Sort by memory usage, descending
ps -eo pid,%mem,cmd --sort=-%mem | head

# Sort by CPU usage, descending
ps -eo pid,%cpu,cmd --sort=-%cpu | head
```

Useful field names for `-o`: `pid`, `ppid`, `user`, `%cpu`, `%mem`, `vsz`, `rss`, `stat`, `start`, `time`, `etime` (elapsed real time since start), `cmd`, `args`, `nice`, `pri`, `tty`.

---

## All Key Options

| Option | Style | Description |
|---|---|---|
| `a` | BSD | Show processes for all users attached to a terminal |
| `u` | BSD | User-oriented output format (adds `%CPU`, `%MEM`, etc.) |
| `x` | BSD | Include processes without a controlling terminal (daemons/services) |
| `-e` | UNIX | Show every process on the system |
| `-f` | UNIX | Full-format listing (adds `PPID`, `C`, `STIME`) |
| `-o FORMAT` | UNIX/GNU | Custom column selection and order |
| `-p PID` | UNIX | Show only the specified PID(s) |
| `-u USER` | UNIX | Show only processes owned by the specified user |
| `--sort=FIELD` | GNU | Sort output by a given field (prefix `-` for descending) |
| `-C NAME` | UNIX | Show only processes matching a given command name |
| `--forest` | GNU | Draw ASCII-art lines showing the parent/child process tree |
| `-T` | UNIX | Show threads for the selected processes too |
| `-L` | UNIX | Show LWP (thread) columns |
| `-w` | BSD | Wide output — don't truncate long lines |

---

## ps and /proc

```bash
# Everything ps aux shows for a process is ultimately derived from
# files like these:
cat /proc/1234/cmdline | tr '\0' ' '; echo
cat /proc/1234/status
cat /proc/1234/stat

# Because ps just reads /proc, you can inspect a single process
# manually without ps at all, if you already know its PID
```

---

## ps vs top vs htop vs pgrep

| Tool | Best for | Key difference from ps |
|---|---|---|
| `ps` | A single, static snapshot — great for scripting and piping | No live updates; prints once and exits |
| `top` | Continuously refreshing, interactive live view | Auto-refreshes; sortable interactively; not script-friendly by default |
| `htop` | Friendlier, colorized, interactive alternative to top | Same live-view idea as `top`, with mouse support, tree view, and easier filtering |
| `pgrep` | Finding PIDs matching a name/pattern, for use in scripts | Purpose-built for scripting (`pgrep firefox`), rather than general listing |
| `pidof` | Finding the PID(s) of a specific running program by exact name | Narrower and simpler than `pgrep`; typically used interchangeably for simple cases |

```bash
ps aux | grep firefox     # one-off, scriptable lookup
top                         # live, continuously updating view
htop                        # live view, friendlier UI
pgrep -l firefox            # scripting-oriented PID lookup
```

---

## Related Commands

| Command | Relation |
|---|---|
| `top` / `htop` | Live, continuously refreshing process views |
| `pgrep` / `pkill` | Find/signal processes by name pattern, script-friendly |
| `kill` / `killall` | Send signals to processes, typically found first via `ps` |
| `nice` / `renice` | Set/adjust a process's scheduling priority |
| `lsof` | List files (including network sockets) opened by a process |
| `strace` | Trace system calls made by a running or newly launched process |
| `/proc/<pid>/` | The kernel interface `ps` itself reads from |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
