# uptime — The Complete Reference

> **See how long the system has been running, and how busy it's been lately**
> Present since BSD Unix in the early 1980s, later adopted universally across Unix/Linux.
> The fastest single command to answer "is this server okay right now?"

---

## Table of Contents

- [What is uptime?](#what-is-uptime)
- [Where does uptime live?](#where-does-uptime-live)
- [How uptime works internally](#how-uptime-works-internally)
- [Syntax](#syntax)
- [Understanding the Output](#understanding-the-output)
- [Load Average — What the Three Numbers Mean](#load-average--what-the-three-numbers-mean)
- [Interpreting Load Average Correctly](#interpreting-load-average-correctly)
- [All Key Options](#all-key-options)
- [uptime and /proc/loadavg](#uptime-and-procloadavg)
- [uptime vs top vs w vs vmstat](#uptime-vs-top-vs-w-vs-vmstat)
- [Related Commands](#related-commands)

---

## What is uptime?

`uptime` reports how long the system has been running since its last boot, how many users are currently logged in, and the system's **load average** over the last 1, 5, and 15 minutes — a compact summary of both system availability and recent activity level, all in a single line.

```bash
uptime
#  14:32:07 up 45 days,  3:21,  2 users,  load average: 0.15, 0.22, 0.18
```

**Why uptime is often the very first command run when checking a server:** it's fast, requires no arguments, and immediately answers three important questions at once — has this machine been stable (long uptime, no unexpected reboots), is anyone else currently working on it (users logged in), and is it currently under heavy load (the load average figures).

---

## Where does uptime live?

```
/usr/bin/uptime
```

```bash
which uptime
uptime --version
# uptime from procps-ng 4.0.4
```

Part of **procps** (or **procps-ng**) on Linux, the package providing process- and system-monitoring utilities (`ps`, `top`, `free`, `uptime`, `w`, `vmstat`, and others). Present in some form on virtually every Unix-like system, including macOS/BSD.

---

## How uptime works internally

### Reading kernel-tracked boot time and load statistics

`uptime`'s numbers come from two separate sources the kernel maintains continuously:

1. **Boot time**, used to calculate elapsed uptime, derived from `/proc/uptime` on Linux (a simple file containing the number of seconds since boot, plus idle time)
2. **Load average**, read directly from `/proc/loadavg`, which the kernel updates continuously based on the number of processes in a runnable or uninterruptible-sleep state

```bash
cat /proc/uptime
# 3888472.35 3812048.91
# First number: total seconds since boot
# Second number: total seconds the system has spent idle (can exceed
# the first number on multi-core systems, since idle time is summed
# across ALL cores)

cat /proc/loadavg
# 0.15 0.22 0.18 2/458 12345
# First three: 1, 5, 15-minute load averages
# Fourth: currently-runnable processes / total processes
# Fifth: PID of the most recently created process
```

`uptime` is essentially a thin, human-friendly formatting wrapper around these two files' raw values — it doesn't compute anything itself beyond converting seconds into a readable "X days, Y:ZZ" format and reading the pre-computed load averages the kernel already maintains.

### How the kernel computes load average

The kernel samples the number of processes in the **run queue** (ready to run, waiting for CPU) or in **uninterruptible sleep** (typically waiting on I/O) at regular intervals, and feeds this into an exponentially-damped moving average calculation — this is why load average is described as "average," but isn't a simple arithmetic mean of a fixed window; it's a continuously-decaying weighted average that gives more weight to recent samples.

---

## Syntax

```bash
uptime [OPTIONS]
```

```bash
uptime               # standard, full output
uptime -p             # "pretty" format: just the duration, human-phrased
uptime -s             # "since" format: the exact boot date/time
```

---

## Understanding the Output

```bash
uptime
#  14:32:07 up 45 days,  3:21,  2 users,  load average: 0.15, 0.22, 0.18
```

| Segment | Meaning |
|---------|---------|
| `14:32:07` | Current system time |
| `up 45 days, 3:21` | Time elapsed since the last boot |
| `2 users` | Number of currently logged-in user sessions |
| `load average: 0.15, 0.22, 0.18` | Load average over the last 1, 5, and 15 minutes, respectively |

```bash
# On a freshly booted system, the uptime portion might read:
uptime
#  09:15:02 up 3 min,  1 user,  load average: 0.52, 0.31, 0.13
```

---

## Load Average — What the Three Numbers Mean

The three load average figures represent the average number of processes that were either **actively running** or **waiting to run** (or in uninterruptible I/O wait) over the trailing 1, 5, and 15-minute windows.

```bash
load average: 0.15, 0.22, 0.18
#              ↑     ↑     ↑
#              1min  5min  15min
```

- **A load of 1.00** on a single-core system means the CPU was, on average, exactly fully occupied with no idle time and no queued waiting processes.
- **A load of 0.50** means the CPU was busy about half the time on average, with capacity to spare.
- **A load of 2.00** on a single-core system means, on average, twice as many processes wanted to run as there was CPU capacity available — one is running, one is waiting.

### Load average must be interpreted relative to CPU core count

```bash
nproc
# 4    ← this system has 4 CPU cores

uptime
#  load average: 3.80, 3.50, 3.20
# On a 4-core system, a load average around 3.80 means the system is
# using roughly 95% of its total available CPU capacity (3.80 / 4)
# — busy, but NOT necessarily overloaded, since it's still below the
# theoretical maximum of 4.00 (fully saturating all 4 cores).

# The SAME raw number (3.80) on a SINGLE-core system would indicate
# SEVERE overload — nearly 4 times more demand than that one core
# could possibly satisfy.
```

---

## Interpreting Load Average Correctly

```bash
# Rule of thumb: compare load average to nproc's output
nproc
# 8

uptime
#  load average: 4.00, 3.80, 3.50
# 4.00 / 8 cores = 50% average utilization — healthy, comfortable headroom

#  load average: 8.00, 7.50, 7.00
# 8.00 / 8 cores = 100% average utilization — fully saturated, no
# spare capacity, though not necessarily "broken" yet

#  load average: 16.00, 15.00, 14.00
# 16.00 / 8 cores = 200% — the system has, on average, TWICE as many
# processes wanting to run as it has cores to run them; expect
# noticeable slowdowns and increased latency for anything running on this machine
```

### The trend across the three numbers matters as much as the absolute value

```bash
#  load average: 8.50, 2.10, 1.05
# Rising trend (1min >> 5min >> 15min): load spiked VERY recently —
# something just started consuming a lot of resources; worth
# investigating what changed in the last minute or two

#  load average: 1.05, 2.10, 8.50
# Falling trend (1min << 5min << 15min): the system WAS under heavy
# load recently, but has been recovering/calming down over the past
# several minutes — the spike is already resolving itself
```

---

## All Key Options

| Option | Long | Description |
|--------|------|--------------|
| `-p` | `--pretty` | Human-phrased duration only, e.g., "up 3 days, 2 hours, 15 minutes" |
| `-s` | `--since` | Show the exact boot date/time instead of elapsed duration |
| `-V` | `--version` | Show version information |
| `-h` | `--help` | Show usage help |

```bash
uptime -p
# up 3 days, 2 hours, 15 minutes

uptime -s
# 2024-01-12 12:16:52
```

---

## uptime and /proc/loadavg

```bash
# uptime is essentially a human-friendly wrapper around this file:
cat /proc/loadavg
# 0.15 0.22 0.18 2/458 12345

# Scripts needing JUST the raw load average numbers (without uptime's
# extra formatting/boot-time text) often read this file DIRECTLY,
# rather than parsing uptime's more verbose output:
awk '{print $1, $2, $3}' /proc/loadavg
# 0.15 0.22 0.18

# Extract just the 1-minute load average for a monitoring script
LOAD_1MIN=$(awk '{print $1}' /proc/loadavg)
echo "Current 1-minute load: $LOAD_1MIN"
```

---

## uptime vs top vs w vs vmstat

| Tool | Best for | Key difference from uptime |
|------|----------|--------------------------------|
| `uptime` | Quick, one-line health summary | Load average + uptime + user count, nothing more |
| `top` | Live, continuously refreshing, PER-PROCESS resource view | Shows WHICH processes are consuming CPU/memory, not just aggregate load |
| `w` | Who's logged in and what they're doing | Shows load average too, PLUS each user's current command and login time |
| `vmstat` | Detailed system-wide resource statistics over time (CPU, memory, I/O, swap) | Much more granular, sampled at intervals, ideal for diagnosing WHY load is high |
| `htop` | Interactive, visual alternative to top | Friendlier UI, same underlying per-process data as top |

```bash
uptime          # "is the system generally okay right now?"
w                # "uptime info, PLUS who's logged in and what they're running"
top              # "WHICH specific processes are responsible for the current load?"
vmstat 1 5       # "what's the CPU/memory/IO breakdown, sampled every second, 5 times?"
```

**Typical diagnostic workflow:** `uptime` first for a quick health check → if load looks high, `top` (or `htop`) next to identify which specific process(es) are responsible → `vmstat` or other specialized tools for deeper analysis of WHY (CPU-bound vs I/O-bound vs memory pressure).

---

## Related Commands

| Command | Relation |
|---------|----------|
| `w` | Shows the SAME load average info, PLUS logged-in users' current activity |
| `top` / `htop` | Live per-process resource usage, for finding WHAT is causing high load |
| `vmstat` | Detailed system resource statistics sampled over time |
| `free -h` | Memory usage specifically, often checked alongside load average |
| `nproc` | Shows CPU core count, ESSENTIAL context for interpreting load average correctly |
| `dmesg` | Kernel log, useful for correlating a load spike with a specific system event |
| `sar` | Historical system activity reporting (part of sysstat), for reviewing PAST load trends |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
