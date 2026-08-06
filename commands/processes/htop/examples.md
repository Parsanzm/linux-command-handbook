# htop — Practical Examples

> Real-world patterns for launching, filtering, and acting on processes
> from within htop's interactive interface.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Starting Pre-Filtered or Pre-Sorted](#starting-pre-filtered-or-pre-sorted)
- [Finding a Specific Process Quickly](#finding-a-specific-process-quickly)
- [Sorting by Resource Usage](#sorting-by-resource-usage)
- [Killing and Renicing from Inside htop](#killing-and-renicing-from-inside-htop)
- [Customizing the Display](#customizing-the-display)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Launch the full interactive view
htop

# Quit at any time
# press F10, or 'q'

# Exit automatically after a fixed number of screen updates
# (useful when scripting a bounded capture rather than leaving it open)
htop -n 1
```

---

## Starting Pre-Filtered or Pre-Sorted

```bash
# Start already sorted by CPU usage, descending
htop -s PERCENT_CPU

# Start already sorted by memory usage
htop -s PERCENT_MEM

# Show only one user's processes
htop -u alice

# Show only specific known PIDs
htop -p 1234,5678,9012

# Start already in tree view
htop -t

# Slow down the refresh rate (in tenths of a second — 50 = 5 seconds)
htop -d 50
```

---

## Finding a Specific Process Quickly

```bash
# Inside htop:
# Press F3 (or /) and start typing a process name
# — htop jumps to and highlights the first match, without hiding
#   anything else in the list

# Press F4 and start typing instead
# — htop FILTERS the visible list down to only matching rows,
#   hiding everything else until the filter is cleared (Esc)
```

---

## Sorting by Resource Usage

```bash
# Inside htop:
# Press F6, then choose PERCENT_CPU or PERCENT_MEM from the menu
# Or simply click the CPU%/MEM% column header directly with the mouse
# Or press > / < to cycle the current sort column without opening a menu

# To reverse ascending/descending order on the current sort column:
# press I (capital i)
```

---

## Killing and Renicing from Inside htop

```bash
# Select a process with the arrow keys (or click it with the mouse), then:
# F9 — opens a signal-selection menu (SIGTERM is the default highlighted choice)
# choose a signal, press Enter to send it

# Selecting MULTIPLE processes to signal at once:
# Space to tag each one you want, then F9 to signal all tagged processes together
# U untags everything if you change your mind

# Adjusting priority without leaving the screen:
# F7 — increase priority (lower the nice value, needs privileges to go below 0)
# F8 — decrease priority (raise the nice value)
```

---

## Customizing the Display

```bash
# Inside htop, press F2 (Setup) to:
#   - add/remove/reorder meters shown in the header (CPU, memory, swap,
#     network if a plugin is available, battery, clock, etc.)
#   - choose between one combined CPU bar or one bar PER core
#   - add/remove/reorder columns in the process table itself
#   - switch color schemes

# Persist your customized layout for future sessions —
# htop saves configuration automatically to:
cat ~/.config/htop/htoprc
```

---

## Real-World Recipes

```bash
# --- Quick "what's hammering the CPU right now" check ---
htop -s PERCENT_CPU
# glance at the top few rows, no scripting needed

# --- Investigate a Specific User's Runaway Processes ---
htop -u alice -s PERCENT_MEM

# --- Watch a Known Set of PIDs During a Deployment ---
htop -p "$(pgrep -d, -f myapp)"

# --- Confirm a Restarted Service Actually Came Back Up ---
htop -u myservice-user
# visually confirm the expected process appears and is using a
# reasonable amount of CPU/memory shortly after restart

# --- Kill a Frozen GUI Application's Backing Process ---
htop
# F3, type the app's name, Enter to jump to it
# F9, choose SIGTERM (or SIGKILL if it doesn't respond), Enter

# --- Tree View to Understand a Complex Process's Children ---
htop -t
# visually trace which worker/child processes a parent (e.g. a
# multi-process web server or browser) has actually spawned
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
