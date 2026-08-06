# htop — The Complete Reference

> **An interactive, colorized, real-time process viewer**
> A friendlier, more visual alternative to `top` — same underlying idea
> (live, continuously refreshing process monitoring), much easier to use.

---

## Table of Contents

- [What is htop?](#what-is-htop)
- [Where does htop live?](#where-does-htop-live)
- [How htop works internally](#how-htop-works-internally)
- [Syntax](#syntax)
- [Understanding the Screen](#understanding-the-screen)
- [The Header Meters — CPU, Memory, Swap, Load](#the-header-meters--cpu-memory-swap-load)
- [Keyboard Controls](#keyboard-controls)
- [Sorting, Filtering, and Searching](#sorting-filtering-and-searching)
- [All Key Command-Line Options](#all-key-command-line-options)
- [htop vs top vs ps vs glances](#htop-vs-top-vs-ps-vs-glances)
- [Related Commands](#related-commands)

---

## What is htop?

`htop` is an interactive, full-screen process viewer that continuously refreshes to show what's running on the system right now — CPU and memory usage per core, per process, with color-coded bars, mouse support, and the ability to sort, filter, search, and send signals directly from the interface, without ever leaving it.

```bash
htop
# Launches the full-screen interactive interface — no useful output
# when piped or redirected, since it's designed for live interaction
```

**Why people reach for htop over `top`:** `top`'s interface is functional but famously unfriendly (obscure single-letter commands, plain text, no mouse support by default). `htop` presents the same fundamental information — process list, resource usage, load — with color-coded per-core CPU bars, a visible tree view of parent/child relationships, and mouse-clickable columns/processes, at the cost of being a separate package that isn't always preinstalled.

---

## Where does htop live?

```
/usr/bin/htop
```

```bash
which htop
htop --version
# htop 3.3.0
```

Unlike `ps`, `top`, and `kill` (all part of core packages present on virtually every Linux system), **htop is a separate, third-party package** and is not preinstalled on most distributions by default:

```bash
sudo apt install htop      # Debian/Ubuntu
sudo dnf install htop      # Fedora/RHEL
sudo pacman -S htop        # Arch
brew install htop          # macOS (via Homebrew)
```

---

## How htop works internally

Like `ps` and `top`, `htop` reads process and system information from the **`/proc` filesystem** on Linux — it doesn't have privileged kernel access beyond what any user-space program reading `/proc` can see. It re-reads this data on a fixed interval (configurable, roughly once per second by default) and redraws the entire screen with the refreshed values.

```bash
ls /proc/1234/
# stat  status  cmdline  cwd  fd  maps  ...
# htop draws its process rows, CPU%, and memory columns from exactly
# these same kernel-exposed files that ps and top also read.
```

Because htop is a **separate process itself**, watching htop naturally includes htop's own (typically very small) resource footprint in the very list it's displaying — a minor but sometimes noticed self-referential quirk.

---

## Syntax

```bash
htop [OPTIONS]
```

Nearly all of htop's real functionality is **interactive**, driven by keyboard/mouse input once it's running, rather than through command-line flags — the flags mostly control startup behavior (which user's processes to show, sort order, delay).

---

## Understanding the Screen

```
  1  [||||||||||||        45.2%]   Tasks: 87, 312 thr; 3 running
  2  [||||               12.1%]   Load average: 1.25 0.98 0.87
  Mem[|||||||||||       4.2G/16G]  Uptime: 2 days, 04:12:33
  Swp[                   0K/2G]

  PID USER      PRI  NI  VIRT   RES   SHR S CPU% MEM%   TIME+  Command
 1234 alice      20   0  812M  145M  42M  R 23.4  0.9  1:32.10 firefox
 5678 alice      20   0   22M   9M   6M  S  0.3  0.1  0:00.42 bash
```

| Section | Meaning |
|---|---|
| Per-core CPU bars | One bar per logical CPU core, color-coded by usage type (see below) |
| `Mem` bar | Physical RAM used vs. total |
| `Swp` bar | Swap space used vs. total |
| `Tasks` | Total process/thread count, and how many are actively running right now |
| `Load average` | Same 1/5/15-minute figures reported by `uptime` |
| Process table | Sortable, scrollable list of individual processes — same core fields as `ps`, refreshed continuously |

---

## The Header Meters — CPU, Memory, Swap, Load

The per-core CPU bars use color segments to break down **what kind** of activity is consuming CPU, not just how much:

| Color (default theme) | Meaning |
|---|---|
| Green | Normal user-space processes |
| Red | Kernel (system) time |
| Blue | Low-priority ("niced") processes |
| Orange/Yellow | I/O wait (varies by htop version/theme) |
| Cyan/Magenta | Virtualization steal time or IRQ time, depending on version |

This breakdown is one of htop's genuine advantages over plain `top`'s aggregate percentage — at a glance, a fully "red" bar signals heavy kernel-side activity, quite different from a fully "green" one, without needing to separately run something like `vmstat`.

---

## Keyboard Controls

| Key | Action |
|---|---|
| `F1` / `h` | Help screen |
| `F2` / `S` | Setup — customize meters, columns, colors, display options |
| `F3` / `/` | Search/incremental filter for a process by name |
| `F4` | Filter the list to only matching processes |
| `F5` / `t` | Toggle tree view (parent/child process hierarchy) |
| `F6` | Change the sort column |
| `F9` / `k` | Kill the selected process (choose a signal from a menu) |
| `F7` / `]` | Increase selected process's priority (renice, lower number) |
| `F8` / `[` | Decrease selected process's priority (renice, higher number) |
| `Space` | Tag/select a process (for acting on multiple at once) |
| `U` | Untag all tagged processes |
| `F5` | Toggle tree view |
| `F10` / `q` | Quit |

Mouse support is enabled by default in most terminal emulators — clicking a column header sorts by it, and clicking a process selects it.

---

## Sorting, Filtering, and Searching

```bash
# Interactively, inside htop:
# F6 (or click a column header) — choose which column to sort by
# > or < — cycle sort column without opening the menu
# F4 — type a substring to FILTER the visible list down to matches
# F3 or / — SEARCH and jump to the next matching process without hiding others

# From the command line, at startup:
htop -s PERCENT_CPU     # start already sorted by CPU usage
htop -u alice           # show only processes owned by user alice
htop -p 1234,5678       # show only these specific PIDs
```

---

## All Key Command-Line Options

| Option | Long form | Description |
|---|---|---|
| `-d SECONDS` | `--delay=SECONDS` | Set the refresh interval (in tenths of a second) |
| `-u USER` | `--user=USER` | Show only processes owned by the given user |
| `-p PID,...` | `--pid=PID,...` | Show only the specified PID(s) |
| `-s COLUMN` | `--sort-key=COLUMN` | Start already sorted by the given column |
| `-t` | `--tree` | Start in tree view |
| `-C` | `--no-color` | Disable color output |
| `-n NUMBER` | `--iterations=NUMBER` | Exit automatically after N updates (useful for scripted, bounded runs) |
| — | `--version` | Print version |
| — | `--help` | Print usage help |

---

## htop vs top vs ps vs glances

| Tool | Best for | Key difference from htop |
|---|---|---|
| `top` | Live view, guaranteed present on virtually every Linux system | Plain text, less discoverable controls, no mouse support by default, no built-in tree view toggle in older versions |
| `ps` | One-off, scriptable snapshots | Not live/interactive at all — prints once and exits, ideal for piping into other commands |
| `htop` | Live, human-friendly interactive monitoring | The most approachable of the interactive viewers, at the cost of not being preinstalled everywhere |
| `glances` | Broader system overview (network, disk I/O, sensors, containers) alongside processes | Covers more subsystems in one dashboard than htop's process-centric focus |

```bash
ps aux --sort=-%cpu | head    # scripted, one-shot, pipeable
top                            # live view, always available, plain
htop                           # live view, friendlier, needs installing
glances                        # live view, broader system-wide dashboard
```

---

## Related Commands

| Command | Relation |
|---|---|
| `top` | The original live process viewer htop is modeled after and improves on |
| `ps` | Non-interactive, scriptable snapshot equivalent |
| `glances` | Broader live system-monitoring dashboard covering more than just processes |
| `iotop` | htop-like live view specifically for per-process disk I/O |
| `nmon` / `dstat` | Other live system-resource monitoring alternatives |
| `kill` / `renice` | What htop's `F9`/`F7`/`F8` keys invoke under the hood |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
