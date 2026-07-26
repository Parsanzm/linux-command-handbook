# ps — Edge Cases & Gotchas

> `ps` looks like a simple listing tool, but option-style mixing, snapshot
> timing, and process-state misreadings trip up beginners and veterans alike.

---

## Table of Contents

- [ps aux and ps -ef Are Not the Same Command](#ps-aux-and-ps--ef-are-not-the-same-command)
- [ps Is a Snapshot — the Process List Can Already Be Stale](#ps-is-a-snapshot--the-process-list-can-already-be-stale)
- [%CPU Is an Average Since Process Start, Not "Right Now"](#cpu-is-an-average-since-process-start-not-right-now)
- [A Zombie Process Can't Be Killed — It's Already Dead](#a-zombie-process-cant-be-killed--its-already-dead)
- [D State (Uninterruptible Sleep) Processes Ignore kill -9](#d-state-uninterruptible-sleep-processes-ignore-kill--9)
- [grep Finding Itself in ps aux | grep Pipelines](#grep-finding-itself-in-ps-aux--grep-pipelines)
- [RSS Overstates "Real" Memory Usage With Shared Libraries](#rss-overstates-real-memory-usage-with-shared-libraries)
- [Command Line Truncation Hides the Full Invocation](#command-line-truncation-hides-the-full-invocation)
- [PID Reuse — an Old PID Number Can Belong to a Completely Different Process Later](#pid-reuse--an-old-pid-number-can-belong-to-a-completely-different-process-later)
- [ps Inside a Container Only Shows That Container's Processes (Usually)](#ps-inside-a-container-only-shows-that-containers-processes-usually)
- [A Process Renaming Its Own Displayed Command Name](#a-process-renaming-its-own-displayed-command-name)

---

## ps aux and ps -ef Are Not the Same Command

### Two historically distinct option conventions, both extremely common
```bash
ps aux
# USER PID %CPU %MEM  VSZ  RSS TTY STAT START TIME COMMAND

ps -ef
# UID  PID PPID  C STIME TTY  TIME CMD
# ⚠️ These produce OVERLAPPING but genuinely DIFFERENT column sets —
# ps aux includes %CPU/%MEM directly, ps -ef includes PPID directly.
# Mixing up which style a tutorial/script uses, or assuming the column
# positions are interchangeable between the two, is a very common
# source of confusion. They are not aliases for each other, even
# though both are "show me everything" style invocations.
```

---

## ps Is a Snapshot — the Process List Can Already Be Stale

### By the time output reaches your screen, reality may have moved on
```bash
ps aux | grep myapp
# alice  5678  ...  myapp
# ⚠️ This reflects the process table at the EXACT MOMENT ps read
# /proc — a fast-spawning short-lived process can be completely
# missing from the output if it started and exited between two ps
# invocations, and a process shown as running may have already exited
# by the time you act on this output (e.g., attempting to kill a PID
# that's already gone).

# For continuously updated monitoring instead of repeated manual polling:
top
# or, for scripted repeated snapshots at a fixed interval:
watch -n 1 'ps aux --sort=-%cpu | head'
```

---

## %CPU Is an Average Since Process Start, Not "Right Now"

### A common misreading when hunting for what's causing a CURRENT slowdown
```bash
ps aux --sort=-%cpu | head -5
# A process that used 100% CPU for 2 minutes right after starting,
# then dropped to near-idle, can still show a MODERATE %CPU value
# hours later — because ps's %CPU is roughly (total CPU time consumed)
# / (total time the process has existed), NOT a live, instantaneous
# reading of current CPU consumption.

# ⚠️ Don't rely on ps's %CPU column alone to identify what's causing
# a slowdown happening RIGHT NOW — a long-lived process with a high
# average can look worse than a brand-new process actually spiking
# hard at this exact moment. Use `top` (which recalculates over a
# short refresh interval) for genuinely current CPU activity instead.
```

---

## A Zombie Process Can't Be Killed — It's Already Dead

### `kill`ing a Z-state process does nothing, because there's nothing left to signal
```bash
ps aux | awk '$8 ~ /^Z/'
# alice  9999  0.0  0.0     0     0 ?    Z    10:00   0:00 [myapp] <defunct>
# ⚠️ A zombie is a process that has ALREADY finished executing and
# released its resources — only a small table entry (holding its exit
# status) remains, waiting for its PARENT to call wait() and "reap" it.
# Sending kill -9 to a zombie's PID does nothing, because there's no
# actual running process left to receive or act on the signal.

# The correct fix targets the PARENT, not the zombie itself:
ps -o ppid= -p 9999
# Either the parent process needs to properly reap its children (a
# bug in that program if it doesn't), or terminating/restarting the
# PARENT process will cause the zombie's entry to be cleaned up too.
```

---

## D State (Uninterruptible Sleep) Processes Ignore kill -9

### Even SIGKILL can't touch a process stuck deep in an uninterruptible kernel wait
```bash
ps aux | awk '$8 ~ /^D/'
# alice  4321  0.0  0.1  ...  D    09:00   0:02 some_process
# ⚠️ A process in D state is blocked inside a kernel-level operation
# (typically slow/failing disk I/O, an unresponsive NFS mount, or
# similar) that cannot be interrupted by ANY signal, including
# SIGKILL — the kernel simply won't deliver signal handling until that
# specific blocking operation completes or times out on its own.

# kill -9 4321 will appear to succeed (no error) but the process will
# remain in D state, seemingly unaffected, until whatever it's
# blocked on resolves. Persistent D-state processes are usually a sign
# of an underlying storage/NFS/hardware problem worth investigating
# directly, rather than a process-management issue solvable via kill.
```

---

## grep Finding Itself in `ps aux | grep` Pipelines

### The classic beginner "why does grep show up in its own search results" moment
```bash
ps aux | grep firefox
# alice  5678  ...  firefox
# alice  6789  ...  grep --color=auto firefox    ← grep matching its
#                                                   OWN command line,
#                                                   since "firefox"
#                                                   appears there too

# The traditional workaround — bracket one character so the pattern
# no longer matches grep's own invocation:
ps aux | grep '[f]irefox'

# The cleaner modern alternative — purpose-built and avoids the issue entirely:
pgrep -la firefox
```

---

## RSS Overstates "Real" Memory Usage With Shared Libraries

### Adding up every process's RSS drastically overcounts actual system memory usage
```bash
ps -eo pid,rss,cmd --sort=-rss | head
# Multiple processes using the same shared library (libc, for example)
# each report that library's pages as part of THEIR OWN RSS —
# ⚠️ summing RSS across many processes therefore double-, triple-, or
# many-times-counts memory that's actually shared and only resident in
# physical RAM ONCE. "Total RSS across all processes" is NOT a valid
# way to compute actual system memory usage.

# For an accurate picture of real memory pressure, use tools that
# account for sharing correctly instead:
free -h
smem -t          # (if installed) accounts for proportional shared memory
```

---

## Command Line Truncation Hides the Full Invocation

### Long commands get cut off in default-width terminals
```bash
ps aux | grep myscript
# alice 1234 ... /usr/bin/python3 /opt/myapp/very/long/path/to/scr+
# ⚠️ The trailing "+" (or simple truncation without one, depending on
# terminal width) signals the COMMAND column was cut off — you may be
# missing critical arguments (a config file path, a flag that changes
# behavior entirely) without any obvious indication of exactly how
# much was hidden.

# Force wide, untruncated output:
ps auxww
# or query the full command line directly for a known PID:
cat /proc/1234/cmdline | tr '\0' ' '; echo
```

---

## PID Reuse — an Old PID Number Can Belong to a Completely Different Process Later

### PIDs are recycled, not permanently assigned to whatever first used them
```bash
# You note down PID 4321 for a process you're monitoring...
# (time passes; that process exits; the system starts many new processes)
ps -p 4321
# alice  4321  ...  some_totally_unrelated_process
# ⚠️ The kernel reuses PID numbers once they're free — there's no
# guarantee that a PID you recorded minutes or hours ago still refers
# to the same process, or to any process at all. Scripts that capture
# a PID and act on it much later (without re-verifying identity, e.g.
# via the process's own recorded start time or command line) risk
# accidentally targeting a completely unrelated process that happened
# to reuse that number.

# Safer pattern: re-verify identity before acting on a stored PID
if ps -p 4321 -o cmd= | grep -q myapp; then
  kill 4321
fi
```

---

## ps Inside a Container Only Shows That Container's Processes (Usually)

### Namespace isolation changes what ps can even see — but not always completely
```bash
docker run --rm ubuntu:24.04 ps aux
# USER  PID  ... COMMAND
# root    1  ... ps aux
# ⚠️ Thanks to PID namespace isolation, a process inside a properly
# isolated container generally only sees processes within its OWN
# namespace — not the host's full process table, and not other
# containers' processes. However, this depends entirely on the
# container being launched with proper namespace isolation; a
# container run with `--pid=host` (or certain privileged
# configurations) WILL see the full host process list, which can be
# surprising if you expected isolation by default.
```

---

## A Process Renaming Its Own Displayed Command Name

### What ps shows in COMMAND isn't necessarily the original binary name
```bash
ps aux | grep myserver
# alice 5678 ... myserver: worker process (idle)
# ⚠️ Many programs (notably things like PostgreSQL, and anything using
# libraries that call setproctitle() / modify argv[0]) deliberately
# CHANGE what appears in ps's COMMAND column at runtime — to show a
# custom status string instead of the literal binary path/name that
# was originally exec'd. This is intentional and useful for at-a-
# glance status, but it means grepping ps output for the ORIGINAL
# binary name may fail to match processes that have since renamed
# their own displayed identity.

# The original exec path is still recoverable regardless of any
# self-renaming, since it doesn't affect this symlink:
readlink -f /proc/5678/exe
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
