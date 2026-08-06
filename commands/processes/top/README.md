# top — The Complete Reference

> **Live, continuously refreshing view of system processes and resource usage**
> Present on virtually every Unix-like system by default, and usually the very
> first thing run when a machine "feels slow."

---

## Table of Contents

- [What is top?](#what-is-top)
- [Where does top live?](#where-does-top-live)
- [How top works internally](#how-top-works-internally)
- [Syntax](#syntax)
- [Understanding the Screen](#understanding-the-screen)
- [The Summary Area — Load, Tasks, CPU, Memory](#the-summary-area--load-tasks-cpu-memory)
- [Interactive Keyboard Commands](#interactive-keyboard-commands)
- [Sorting and Filtering](#sorting-and-filtering)
- [All Key Command-Line Options](#all-key-command-line-options)
- [Batch Mode — Using top in Scripts](#batch-mode--using-top-in-scripts)
- [top vs htop vs ps vs vmstat](#top-vs-htop-vs-ps-vs-vmstat)
- [Related Commands](#related-commands)

---

## What is top?

`top` shows a continuously refreshing, full-screen view of running processes ranked by resource usage, along with a summary of overall system load, CPU, and memory. Unlike `ps`, which prints once and exits, `top` stays open and redraws itself on a fixed interval — the standard tool for watching resource usage change *live*, in real time.

```bash
top
# Launches the interactive live view; press 'q' to quit
```

**Why `top` is often the very first command run when something feels wrong:** it's installed everywhere by default, requires no arguments, and immediately answers "what's actually consuming CPU/memory right now, and is it getting better or worse while I watch."

---

## Where does top live?

```
/usr/bin/top
```

```bash
which top
top -v
# procps-ng 4.0.4
```

Part of **procps** (or **procps-ng**) on Linux — the same package providing `ps`, `free`, `uptime`, `w`, and `kill`'s standalone binary. Present by default on essentially every Linux distribution and most other Unix-like systems (macOS and BSD ship their own separate `top` implementations with a different option set and slightly different default layout).

---

## How top works internally

Like `ps`, `top` reads process and system data from the **`/proc` filesystem** — it holds no special kernel access beyond what any userspace program reading `/proc` can see. On each refresh cycle (3 seconds by default), it re-reads the relevant `/proc` files, recalculates CPU percentages by comparing the current reading against the previous cycle's snapshot, and redraws the full screen.

```bash
cat /proc/stat        # aggregate, system-wide CPU time counters
cat /proc/meminfo      # memory/swap totals and breakdown
cat /proc/1234/stat    # per-process CPU/memory/state data
```

Because CPU percentages are inherently a **rate** (usage over an interval), `top` cannot compute a meaningful `%CPU` from a single snapshot alone — it always needs at least two consecutive readings to calculate the difference, which is why the very first screen after launching sometimes shows numbers that settle into more accurate values a moment later.

---

## Syntax

```bash
top [OPTIONS]
```

Once running, virtually all further control happens through **single-key interactive commands** rather than command-line flags — flags mostly control startup behavior (delay, initial sort, which users/PIDs to show).

---

## Understanding the Screen

```
top - 14:32:07 up 45 days,  3:21,  2 users,  load average: 1.24, 0.98, 0.87
Tasks: 210 total,   2 running, 207 sleeping,   0 stopped,   1 zombie
%Cpu(s):  8.2 us,  2.1 sy,  0.0 ni, 88.9 id,  0.6 wa,  0.0 hi,  0.2 si,  0.0 st
MiB Mem :  16034.0 total,   3120.5 free,   6210.2 used,   6703.3 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   8420.1 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 5678 alice     20   0  812340  145200  42100 R  23.4   0.9   1:32.10 firefox
 1234 alice     20   0   22344    9876   7200 S   0.1   0.1   0:00.42 bash
```

| Section | Meaning |
|---|---|
| Header line | Same info `uptime` reports — current time, uptime, users, load average |
| `Tasks` line | Total process count, broken down by state |
| `%Cpu(s)` line | System-wide CPU time breakdown by category (see below) |
| `Mem`/`Swap` lines | Physical RAM and swap usage summary |
| Process table | Individual processes, sortable, refreshed every cycle |

---

## The Summary Area — Load, Tasks, CPU, Memory

```
%Cpu(s):  8.2 us,  2.1 sy,  0.0 ni, 88.9 id,  0.6 wa,  0.0 hi,  0.2 si,  0.0 st
```

| Field | Meaning |
|---|---|
| `us` | User space — normal, non-niced processes |
| `sy` | System — time spent in the kernel |
| `ni` | Niced — user processes running at a lowered priority |
| `id` | Idle — CPU doing nothing |
| `wa` | I/O wait — CPU idle while waiting on disk/network I/O to complete |
| `hi` | Hardware interrupts |
| `si` | Software interrupts |
| `st` | Steal — CPU time taken by the hypervisor on a virtualized/cloud instance |

A high `wa` figure points toward a storage/I/O bottleneck rather than raw CPU demand; a high `st` figure on a cloud VM points toward hypervisor-level contention with other tenants, not anything happening inside the VM itself.

---

## Interactive Keyboard Commands

| Key | Action |
|---|---|
| `h` | Help |
| `q` | Quit |
| `k` | Kill a process (prompts for PID, then signal) |
| `r` | Renice a process (prompts for PID, then new nice value) |
| `f` | Choose which fields/columns to display |
| `o` | Change sort field |
| `P` | Sort by `%CPU` (shortcut) |
| `M` | Sort by `%MEM` (shortcut) |
| `T` | Sort by cumulative time (shortcut) |
| `R` | Reverse the current sort order |
| `u` | Filter to a specific user |
| `1` | Toggle showing individual per-core CPU lines vs. one combined summary |
| `c` | Toggle showing full command line vs. just the program name |
| `H` | Toggle showing individual threads as separate rows |
| `z` | Toggle color |
| `W` | Save current settings as the new default (writes `~/.toprc`) |
| `Space` | Force an immediate refresh |

---

## Sorting and Filtering

```bash
# Inside top, interactively:
# P — sort by %CPU (most common)
# M — sort by %MEM
# T — sort by TIME+ (cumulative CPU time)
# R — reverse current sort direction
# o — open a menu to sort by any available field
# u — filter down to one specific user's processes
```

```bash
# From the command line, at startup:
top -u alice          # only alice's processes
top -p 1234,5678       # only these specific PIDs
top -o %MEM             # start already sorted by memory
```

---

## All Key Command-Line Options

| Option | Description |
|---|---|
| `-d SECONDS` | Set the refresh delay (default 3 seconds) |
| `-n NUMBER` | Exit automatically after this many refresh iterations |
| `-b` | Batch mode — non-interactive, plain-text output suitable for piping/redirecting |
| `-u USER` | Show only the given user's processes |
| `-p PID,...` | Show only the given PID(s) |
| `-o FIELD` | Start already sorted by the given field |
| `-H` | Show individual threads as separate rows from the start |
| `-i` | Don't show idle/zombie processes |
| `-c` | Show the full command line instead of just the program name |
| `-v` | Print version |

---

## Batch Mode — Using top in Scripts

`top`'s normal interactive mode is unsuitable for scripting (same issue as `htop` — it's built around full-screen redraw control codes). **Batch mode** (`-b`) fixes this by producing plain, sequential text output instead:

```bash
# One single snapshot, then exit
top -bn1

# One snapshot, sorted by memory, top 10 lines
top -bn1 -o %MEM | head -17    # header lines + top 10 process rows

# Log a snapshot every 60 seconds
while true; do
  echo "=== $(date) ==="
  top -bn1 | head -12
  sleep 60
done >> /var/log/top_snapshots.log
```

---

## top vs htop vs ps vs vmstat

| Tool | Best for | Key difference from top |
|---|---|---|
| `top` | Live view, guaranteed present on virtually every system | The universal baseline; plain text, terser controls |
| `htop` | Friendlier, colorized, mouse-driven live view | Same live-monitoring idea, considerably easier interface, not always preinstalled |
| `ps` | One-off, scriptable snapshots | Not live — prints once and exits, better suited to piping |
| `vmstat` | Sampled, tabular system-wide resource stats over time | Broader resource categories (memory, swap, I/O, CPU) in compact rows, less process-level detail |

```bash
top                     # live view, always available
htop                     # live view, friendlier, needs installing
ps aux --sort=-%cpu     # one-shot, scriptable
vmstat 1 5               # sampled system-wide stats, 5 times, 1 second apart
```

---

## Related Commands

| Command | Relation |
|---|---|
| `htop` | Friendlier, colorized alternative with the same live-monitoring purpose |
| `ps` | Non-interactive, scriptable snapshot equivalent |
| `vmstat` | Broader sampled system resource statistics |
| `free -h` | Dedicated memory usage summary, similar data to top's Mem/Swap lines |
| `uptime` | Same load-average figures shown in top's header line |
| `kill` / `renice` | What top's `k`/`r` interactive commands invoke under the hood |
| `iotop` | top-like live view specifically for per-process disk I/O |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
