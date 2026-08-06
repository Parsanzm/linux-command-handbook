# top — Edge Cases & Gotchas

> `top` looks like a straightforward live dashboard, but its very first
> reading, its %CPU math, and its scripting limitations routinely
> mislead people who trust it too literally.

---

## Table of Contents

- [The First Screen's %CPU Values Can Be Inaccurate](#the-first-screens-cpu-values-can-be-inaccurate)
- [Per-Process %CPU Can Exceed 100% on Multi-Core Systems](#per-process-cpu-can-exceed-100-on-multi-core-systems)
- [%MEM Reflects RSS — Summing It Across Processes Overstates Real Usage](#mem-reflects-rss--summing-it-across-processes-overstates-real-usage)
- [top's Own Process Appears in Its Own List](#tops-own-process-appears-in-its-own-list)
- [Piping or Redirecting top's Default Output Produces Garbage](#piping-or-redirecting-tops-default-output-produces-garbage)
- [A High "wa" Doesn't Mean the CPU Is Overloaded](#a-high-wa-doesnt-mean-the-cpu-is-overloaded)
- [Killing from Inside top Defaults to SIGTERM, Not Always What You Expect](#killing-from-inside-top-defaults-to-sigterm-not-always-what-you-expect)
- [top Inside a Container Can Show the HOST's Full Resource Picture](#top-inside-a-container-can-show-the-hosts-full-resource-picture)
- [Saved Settings (W) Persist Silently Across Future Sessions](#saved-settings-w-persist-silently-across-future-sessions)
- [macOS/BSD top Has a Meaningfully Different Option Set and Output Format](#macosbsd-top-has-a-meaningfully-different-option-set-and-output-format)
- [A Brief Spike Can Be Completely Missed Between Refresh Cycles](#a-brief-spike-can-be-completely-missed-between-refresh-cycles)

---

## The First Screen's %CPU Values Can Be Inaccurate

### CPU percentage is a rate, and a rate needs two data points
```bash
top
# The very FIRST screen shown immediately after launch can display
# unusually high or otherwise off %CPU figures.
# ⚠️ %CPU is calculated by comparing CPU time used between the
# CURRENT reading and the PREVIOUS one — on the first refresh, top
# has no true "previous" reading yet and effectively compares against
# process start, which can produce a misleadingly large or otherwise
# unrepresentative initial number.

# The numbers settle into accurate, comparable values starting from
# the SECOND refresh cycle onward — don't judge %CPU accuracy from
# the very first screen alone; wait for at least one full refresh
# interval to pass.
```

---

## Per-Process %CPU Can Exceed 100% on Multi-Core Systems

### A number well above 100% doesn't indicate anything broken
```bash
top
#  PID  ...  %CPU   COMMAND
# 5678  ...  340.2  some_multithreaded_app
# ⚠️ The percentage is relative to a SINGLE core's full capacity, not
# the system's total capacity — a process using multiple threads
# across several cores simultaneously can legitimately show a
# combined figure well above 100%. 340% roughly means "using 3.4
# cores' worth of CPU time," which is entirely normal for heavily
# multithreaded software.

# To sanity-check against total system capacity, compare against the
# number of cores shown in the header (or via nproc), rather than
# reading a single process's %CPU figure as if it were capped at 100%.
```

---

## %MEM Reflects RSS — Summing It Across Processes Overstates Real Usage

### The same shared-memory accounting issue that affects ps also affects top
```bash
# Multiple processes sharing common libraries (e.g., libc) each report
# those shared pages as part of THEIR OWN individual %MEM figure.
# ⚠️ Manually adding up %MEM across several related processes
# therefore double- or many-times-counts memory that's only
# physically resident in RAM ONCE — "total %MEM across processes X, Y,
# Z" is not a valid way to estimate real memory pressure.

# The header's Mem line already accounts for sharing correctly —
# trust that summary figure over a manual sum of individual rows:
free -h
```

---

## top's Own Process Appears in Its Own List

### A minor, sometimes confusing self-referential quirk
```bash
top
# While running, top itself shows up as an entry in its OWN process
# table — typically using very little CPU/memory, but ⚠️ this can
# briefly confuse someone specifically hunting for "what is this
# process called top doing here," forgetting that the monitoring tool
# necessarily appears in the very data it's displaying, same as ps
# would if checked from within its own output.
```

---

## Piping or Redirecting top's Default Output Produces Garbage

### top's interactive mode is not designed for text redirection
```bash
top > output.txt
# ⚠️ This does NOT produce a useful, readable text snapshot the way
# `ps aux > output.txt` would — top's default mode is built around
# full-screen terminal control codes for continuous redrawing, so
# redirecting it captures garbled escape-sequence noise instead of
# clean process listings.

# The correct tool for scripting/piping/redirecting is BATCH MODE:
top -bn1 > output.txt
# -b (batch) switches to plain, sequential text output;
# -n1 exits after exactly one iteration instead of running forever
```

---

## A High "wa" Doesn't Mean the CPU Is Overloaded

### I/O wait is counted as part of the CPU summary line, but it isn't CPU demand
```bash
%Cpu(s):  4.2 us,  1.5 sy,  0.0 ni, 39.5 id, 54.6 wa,  0.0 hi,  0.2 si,  0.0 st
# ⚠️ 54.6% "wa" (I/O wait) does NOT mean the CPU is 54.6% busy — it
# means the CPU was IDLE, specifically while at least one process was
# blocked waiting on disk/network I/O to complete. A high "wa"
# figure points toward a storage or network bottleneck, NOT a need
# for more CPU cores or CPU-bound code optimization.

# Confirm and investigate an I/O-bound bottleneck specifically:
iostat -x 1 5
```

---

## Killing from Inside top Defaults to SIGTERM, Not Always What You Expect

### Pressing 'k' and then Enter too quickly can send an unintended signal
```bash
# Inside top:
# k → prompts "PID to signal:" → type PID, Enter
# → prompts "Signal [15]:" → pressing Enter here WITHOUT typing a
#   number sends the DEFAULT, SIGTERM (15) — ⚠️ if a stronger signal
#   like SIGKILL (9) was actually intended, forgetting to explicitly
#   type "9" at this second prompt sends the wrong signal instead,
#   silently defaulting to the polite request rather than the forceful one.

# Always read both prompts carefully — the PID prompt AND the signal
# prompt — rather than pressing Enter twice on autopilot.
```

---

## top Inside a Container Can Show the HOST's Full Resource Picture

### Depends entirely on the container's namespace/cgroup configuration
```bash
docker run --rm -it --pid=host debian bash -c "apt install -y procps && top"
# ⚠️ With --pid=host specifically, top inside the container sees and
# can signal the ENTIRE HOST's process list, not just processes
# belonging to that container — a deliberately shared PID namespace,
# not a bug. Separately, even WITHOUT --pid=host, top's memory/CPU
# SUMMARY figures can still reflect the host's totals rather than a
# container-specific cgroup limit on older kernels/configurations,
# depending on cgroup version and container runtime.

# Cross-check against the container's own cgroup-reported limits for
# an accurate, container-scoped resource picture:
cat /sys/fs/cgroup/cpu.max 2>/dev/null
cat /sys/fs/cgroup/memory.max 2>/dev/null
```

---

## Saved Settings (W) Persist Silently Across Future Sessions

### A previous session's customization can make top look "different" unexpectedly
```bash
# Pressing W inside top saves the CURRENT sort order, displayed
# columns, and other display settings to ~/.toprc, making them the
# new PERSISTENT default for all future top sessions on that account.
# ⚠️ Someone who customized top once, weeks ago, and forgot about it
# may be confused when top's default startup view (sort column,
# visible fields) doesn't match what a colleague sees on a different
# machine, or what tutorials/documentation assume as the default.

cat ~/.toprc
# Check whether a saved custom configuration exists and is silently
# altering the default startup view from what's normally expected.
```

---

## macOS/BSD top Has a Meaningfully Different Option Set and Output Format

### A script or muscle-memory habit built on Linux's top may not transfer directly
```bash
# Linux (procps-ng):
top -bn1

# macOS's built-in top:
top -l 1
# ⚠️ -b/-n (batch mode, iteration count) are Linux/procps-ng specific
# flags — macOS's native top uses ENTIRELY different flags (-l for a
# fixed sample count, no true "batch mode" concept in the same sense)
# and formats its summary/process lines noticeably differently.
# Scripts intended to run on both need OS-specific branching rather
# than assuming a shared, portable top invocation.
```

---

## A Brief Spike Can Be Completely Missed Between Refresh Cycles

### top only samples periodically — it isn't a continuous recording
```bash
# A process spikes to 100% CPU for half a second, then returns to idle
top -d 3
# ⚠️ With a 3-second refresh interval, a spike shorter than that
# window can be ENTIRELY invisible in top's display if it happens to
# fall between two consecutive samples — top shows discrete snapshots
# at each refresh, not a continuous recording of everything that
# happened in between.

# For catching very brief, transient spikes, a finer-grained sampling
# tool is more appropriate:
top -d 0.5           # sample more frequently (sub-second, if supported)
sar -u 1 60           # or a dedicated higher-resolution sampling tool
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
