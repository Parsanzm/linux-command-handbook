# htop — Edge Cases & Gotchas

> htop's friendliness hides a few sharp edges — CPU% math, %MEM
> assumptions, and the fact it isn't installed everywhere you might
> expect it to be.

---

## Table of Contents

- [htop Isn't Preinstalled — Don't Assume It's Always Available](#htop-isnt-preinstalled--dont-assume-its-always-available)
- [Per-Process CPU% Can Exceed 100% on Multi-Core Systems](#per-process-cpu-can-exceed-100-on-multi-core-systems)
- [MEM% Reflects RSS, Which Overstates Real Usage With Shared Libraries](#mem-reflects-rss-which-overstates-real-usage-with-shared-libraries)
- [htop Inside a Container May Show the HOST's Full Process List](#htop-inside-a-container-may-show-the-hosts-full-process-list)
- [F9 (Kill) Defaults to a Specific Signal — Easy to Send the Wrong One by Rushing](#f9-kill-defaults-to-a-specific-signal--easy-to-send-the-wrong-one-by-rushing)
- [Renicing to a Lower (Higher-Priority) Value Requires Elevated Privileges](#renicing-to-a-lower-higher-priority-value-requires-elevated-privileges)
- [htop's Own Process Shows Up in Its Own List](#htops-own-process-shows-up-in-its-own-list)
- [Tree View Can Hide the Sort Order You Just Set](#tree-view-can-hide-the-sort-order-you-just-set)
- [A Filtered View (F4) Can Make It Look Like Processes "Disappeared"](#a-filtered-view-f4-can-make-it-look-like-processes-disappeared)
- [htop's Output Isn't Meant for Scripting or Piping](#htops-output-isnt-meant-for-scripting-or-piping)

---

## htop Isn't Preinstalled — Don't Assume It's Always Available

### A very common surprise on minimal servers and fresh containers
```bash
htop
# bash: htop: command not found
# ⚠️ Unlike ps, top, and kill (part of core system packages present on
# virtually every Linux install), htop is a SEPARATE, third-party
# package that must be explicitly installed — minimal server images,
# fresh containers, and stripped-down distributions frequently don't
# include it by default.

sudo apt install htop      # Debian/Ubuntu
sudo dnf install htop      # Fedora/RHEL
sudo pacman -S htop        # Arch

# Always have a fallback in mind for environments where installing
# new packages isn't possible or desirable:
top          # virtually always present, if htop genuinely isn't available
```

---

## Per-Process CPU% Can Exceed 100% on Multi-Core Systems

### A number well above 100% doesn't mean anything is broken
```bash
# In htop's process table:
# PID  ...  CPU%   Command
# 1234 ...  340.2  some_multithreaded_app
# ⚠️ On a multi-core system, a single process using MULTIPLE threads
# simultaneously across several cores can legitimately show a CPU%
# well above 100% — the percentage is relative to ONE core's full
# capacity, not the system's total capacity. A process using 3.4 cores
# worth of CPU time shows roughly 340%, which is entirely normal for
# heavily multithreaded software, not a display bug.

# To sanity-check against total system capacity, compare against the
# core count shown in the header, or check the aggregate load average
# alongside it rather than reading a single process's % in isolation.
```

---

## MEM% Reflects RSS, Which Overstates Real Usage With Shared Libraries

### The same shared-memory accounting issue that affects ps also affects htop
```bash
# Multiple processes using the same shared library (e.g., libc) each
# report those shared pages as part of their OWN individual RES/MEM%
# figure — ⚠️ summing MEM% across many similar processes therefore
# overstates actual total memory pressure, since shared pages are
# only physically resident in RAM once, not once per process.

# For an accurate picture of TOTAL system memory usage, trust the
# header's Mem[ ] bar (which correctly accounts for sharing) rather
# than mentally adding up individual processes' MEM% column values.
```

---

## htop Inside a Container May Show the HOST's Full Process List

### Depends entirely on how the container's PID namespace was configured
```bash
docker run --rm -it --pid=host debian bash -c "apt install -y htop && htop"
# ⚠️ With --pid=host specifically, htop inside the container sees and
# can signal the ENTIRE HOST's process list, not just processes
# belonging to that container — a deliberately shared PID namespace,
# not a bug. Sending a kill signal from inside such a container can
# affect processes on the HOST machine itself, well outside what the
# container's own filesystem/isolation would otherwise suggest is possible.

# A properly isolated container (the default, without --pid=host) only
# shows that container's own processes — verify which situation you're
# actually in before assuming isolation:
docker inspect --format '{{.HostConfig.PidMode}}' <container_id>
```

---

## F9 (Kill) Defaults to a Specific Signal — Easy to Send the Wrong One by Rushing

### The signal menu has a default highlighted entry that isn't always what you want
```bash
# Pressing F9 opens a scrollable list of signals (SIGTERM highlighted
# by default in most themes/versions) — ⚠️ pressing Enter too quickly
# without actually reading which signal is currently highlighted can
# send something other than the intended SIGTERM (or intended SIGKILL),
# especially if arrow keys were pressed earlier in the session and the
# selection moved without it being obvious at a glance.

# Take a moment to confirm the highlighted signal name before pressing
# Enter, particularly when selecting SIGKILL deliberately — it's easy
# to have the cursor land one row off from where you expect.
```

---

## Renicing to a Lower (Higher-Priority) Value Requires Elevated Privileges

### F7 can silently fail to do anything for a regular user
```bash
# As a non-root user, selecting a process you own and pressing F7
# (increase priority / lower the nice value) below 0:
# ⚠️ This FAILS silently or shows a permission error — ordinary users
# can only ever make their OWN processes' priority WORSE (raise the
# nice value, F8), never better than the default (0) or better than
# their process's current value, without root privileges.

# Confirm you're running htop as root (sudo htop) if genuinely
# increasing a process's scheduling priority below its current value
# is the goal.
```

---

## htop's Own Process Shows Up in Its Own List

### A minor but sometimes confusing self-referential quirk
```bash
# While htop is running, it appears as an entry in its own process
# table — typically using very little CPU/memory, but ⚠️ this can
# briefly confuse someone hunting for "what's this mystery process
# called htop doing here" if they didn't realize the monitoring tool
# itself necessarily shows up in what it's monitoring, same as `ps`
# or `top` would when checked from within their own output.
```

---

## Tree View Can Hide the Sort Order You Just Set

### F5 (tree view) reorganizes the list by hierarchy, overriding a flat sort
```bash
# You set sort-by-CPU% (F6), then toggle tree view (F5):
# ⚠️ In tree view, processes are grouped by PARENT/CHILD relationship
# first, with the chosen sort criterion only applied WITHIN each
# level of that hierarchy — the overall list is no longer simply
# "highest CPU% at the top" across the whole system, which can be
# confusing if you expected the flat sort order to persist unchanged.

# Toggle tree view off (F5 again) to return to a purely flat,
# sort-order-driven view if that's specifically what's needed.
```

---

## A Filtered View (F4) Can Make It Look Like Processes "Disappeared"

### Forgetting an active filter is a common source of "where did my process go" confusion
```bash
# F4, type "firefox", see only firefox-related rows...
# ... later, forgetting the filter is still active, wonder why a
# completely different process (e.g., a newly started backup job)
# isn't showing up in the list at all.
# ⚠️ An active F4 filter persists until explicitly cleared — it's easy
# to forget one is still applied, especially after switching contexts
# or stepping away from the terminal briefly.

# Press Escape (or clear the filter text entirely) to confirm no
# filter is silently still narrowing the visible process list.
```

---

## htop's Output Isn't Meant for Scripting or Piping

### Redirecting or piping htop doesn't behave like a normal text-output command
```bash
htop > output.txt
# ⚠️ This does NOT produce a useful text snapshot the way `ps aux >
# output.txt` would — htop is built around full-screen terminal
# control codes for its interactive display, not plain sequential
# text output, so redirecting it produces garbled control-sequence
# noise rather than a readable process listing.

# For scripting/piping/redirecting use cases, ps (or htop's own
# batch-friendly cousin, `top -b`) is the correct tool instead:
ps aux --sort=-%cpu > output.txt
top -bn1 > output.txt
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
