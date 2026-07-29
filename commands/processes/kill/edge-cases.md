# kill — Edge Cases & Gotchas

> `kill` looks like a blunt "make it stop" tool, but signal-catching,
> zombie processes, and PID reuse routinely make its behavior less
> predictable than it first appears.

---

## Table of Contents

- [kill Doesn't Actually Kill Anything — It Only Requests It](#kill-doesnt-actually-kill-anything--it-only-requests-it)
- [SIGKILL Can't Be Caught, Blocked, or Ignored — But It Can Still "Not Work"](#sigkill-cant-be-caught-blocked-or-ignored--but-it-can-still-not-work)
- [A Negative PID Targets a Process Group, Not a Process](#a-negative-pid-targets-a-process-group-not-a-process)
- [kill -9 on a Zombie Process Does Nothing — There's Nothing Left to Signal](#kill--9-on-a-zombie-process-does-nothing--theres-nothing-left-to-signal)
- [kill 0 and kill -1 Are Special-Cased, Not Ordinary PIDs](#kill-0-and-kill--1-are-special-cased-not-ordinary-pids)
- [PID Reuse Can Make You Signal the Wrong Process Entirely](#pid-reuse-can-make-you-signal-the-wrong-process-entirely)
- [The Shell Builtin and /usr/bin/kill Aren't Always Identical](#the-shell-builtin-and-usrbinkill-arent-always-identical)
- [SIGHUP No Longer Means "Terminal Hung Up" for Most Daemons](#sighup-no-longer-means-terminal-hung-up-for-most-daemons)
- [Killing a Parent Doesn't Automatically Kill Its Children](#killing-a-parent-doesnt-automatically-kill-its-children)
- [kill Exit Status Tells You Whether the Signal Was DELIVERED, Not Whether the Process Actually Died](#kill-exit-status-tells-you-whether-the-signal-was-delivered-not-whether-the-process-actually-died)
- [Permissions — You Can Only Signal Processes You Own (Usually)](#permissions--you-can-only-signal-processes-you-own-usually)

---

## kill Doesn't Actually Kill Anything — It Only Requests It

### A common misconception, most visible with SIGTERM
```bash
kill 1234
# ⚠️ This sends SIGTERM — a REQUEST for the process to terminate,
# which it can catch with a signal handler and use to run its own
# cleanup logic (closing files, flushing buffers, saving state)
# before exiting on its own terms, or even ignore entirely for many
# signals. `kill`'s job ends at signal DELIVERY — what happens next
# is entirely up to the receiving process.

# A process that ignores SIGTERM can appear completely unaffected:
ps -p 1234
# still shows the process running, seemingly unaffected by the "kill"

# This is expected, working-as-designed behavior, not a bug in kill —
# escalate to SIGKILL (see below) if a stronger guarantee is needed.
```

---

## SIGKILL Can't Be Caught, Blocked, or Ignored — But It Can Still "Not Work"

### The one truly unconditional signal, with real exceptions in practice
```bash
kill -9 1234
# ⚠️ SIGKILL genuinely CANNOT be intercepted by the target process —
# the kernel enforces immediate termination unconditionally. However,
# this can still APPEAR not to work in a couple of specific situations:

# 1. The process is in D state (uninterruptible sleep, usually blocked
#    on slow/failing I/O) — the kernel won't even deliver SIGKILL
#    until that blocking kernel operation itself resolves:
ps -o pid,stat -p 1234
# STAT  D    ← stuck here, SIGKILL will queue but not act until the
#               underlying I/O wait resolves on its own

# 2. The PID you're targeting has already changed identity (see PID
#    reuse below) — kill -9 "succeeds" against a completely different,
#    unrelated process that happens to have reused that number now.
```

---

## A Negative PID Targets a Process Group, Not a Process

### A single missing/extra minus sign completely changes the blast radius
```bash
kill -TERM 1234
# Targets ONLY process 1234 itself.

kill -TERM -1234
# ⚠️ Targets the ENTIRE PROCESS GROUP whose ID is 1234 — every process
# in that group, which may include child processes spawned by 1234,
# receives the signal, not just the one process. This is sometimes
# exactly what's wanted (killing a whole pipeline/job tree at once)
# and sometimes an accidental, much wider blast radius than intended
# from a simple typo of an extra dash.

# Double-check which one you actually mean before running it against
# a PID you got from a variable, since a negative sign is easy to lose
# or add accidentally in scripts:
echo "About to signal group: $PGID"
kill -TERM -- "-$PGID"
```

---

## kill -9 on a Zombie Process Does Nothing — There's Nothing Left to Signal

### The same limitation that affects ps also affects kill
```bash
ps aux | awk '$8 ~ /^Z/'
# alice  9999  ...  Z  ...  [myapp] <defunct>

kill -9 9999
# ⚠️ Appears to succeed (no error), but does nothing meaningful — a
# zombie process has ALREADY finished executing; only a small kernel
# table entry (holding its exit status) remains, waiting for its
# PARENT process to reap it via wait(). There's no running process
# left to actually receive or act on a signal.

# The fix targets the PARENT, not the zombie:
ps -o ppid= -p 9999
# Either the parent needs to properly reap its children (a bug in that
# program if it habitually doesn't), or restarting the parent process
# will clean up the zombie entry along with it.
```

---

## kill 0 and kill -1 Are Special-Cased, Not Ordinary PIDs

### Two PID values with entirely different meanings than a normal process ID
```bash
kill -TERM 0
# ⚠️ PID 0 means "every process in the CALLER'S OWN process group" —
# NOT a literal process with PID 0. Running this in an interactive
# shell can signal your entire current job/session's processes at
# once, which is rarely the intended target.

kill -TERM -1
# ⚠️ PID -1 (as a magic special case, distinct from an ordinary
# negative process-group ID) means "every process the caller has
# permission to signal" — for a non-root user, this can mean EVERY
# process they own system-wide; for root, this can mean literally
# every process on the entire system except kill's own process and
# PID 1 — an extremely broad and rarely-intended blast radius.

# Always double, triple-check a PID variable isn't accidentally 0,
# -1, or empty before passing it to kill in a script.
if [ -z "$PID" ] || [ "$PID" -le 0 ]; then
  echo "Refusing to signal an invalid/dangerous PID: '$PID'"
  exit 1
fi
kill "$PID"
```

---

## PID Reuse Can Make You Signal the Wrong Process Entirely

### A stored PID doesn't necessarily still refer to the process you think it does
```bash
# A script records PID 4321 for a process it's monitoring...
echo 4321 > /var/run/myapp.pid
# ... time passes; that process crashes/exits; the system starts many
# new processes; PID 4321 gets reassigned to something else entirely.

kill "$(cat /var/run/myapp.pid)"
# ⚠️ This can end up signaling a COMPLETELY UNRELATED process that
# simply happened to be assigned PID 4321 afterward — the kernel
# recycles PID numbers once they're free, with no built-in guarantee
# tying a stored PID number to the specific process it originally
# identified.

# Safer pattern: verify the process's identity before signaling a
# PID that was recorded some time ago, rather than trusting it blindly
if ps -p "$(cat /var/run/myapp.pid)" -o cmd= | grep -q myapp; then
  kill "$(cat /var/run/myapp.pid)"
else
  echo "Stored PID no longer refers to myapp — refusing to signal it"
fi
```

---

## The Shell Builtin and /usr/bin/kill Aren't Always Identical

### Job-control syntax only works through one of the two
```bash
long_task &
kill %1
# ⚠️ Works via the shell's OWN builtin kill, since %1 (job-control
# notation) is a concept the SHELL tracks, not something the kernel
# or a standalone binary knows anything about.

/usr/bin/kill %1
# kill: %1: arguments must be process or job IDs
# ⚠️ Fails — the standalone binary has no concept of shell jobs at
# all; it only understands literal numeric PIDs (and process groups),
# not shell-specific job specs.

# Confirm which kill is actually running before assuming job-control
# syntax will work as expected in a given context:
type kill
```

---

## SIGHUP No Longer Means "Terminal Hung Up" for Most Daemons

### A historical signal meaning that's been widely repurposed by convention
```bash
kill -HUP 1234
# ⚠️ Historically, SIGHUP signaled that a process's controlling
# terminal had disconnected, and its DEFAULT action is still
# termination if the receiving process doesn't handle it. In practice
# though, a great many long-running daemons (nginx, many others)
# deliberately install a custom handler that reinterprets SIGHUP as
# "reload your configuration file" instead of "terminate" — a
# widespread convention, not a kernel-enforced meaning.

# Don't assume SIGHUP always means "reload" OR always means
# "terminate" without checking the specific daemon's documented
# behavior — the actual effect depends entirely on whether (and how)
# that particular program has chosen to handle it.
```

---

## Killing a Parent Doesn't Automatically Kill Its Children

### Orphaned child processes can keep running independently after their parent is gone
```bash
some_parent_script.sh &
PARENT_PID=$!
kill "$PARENT_PID"
# ⚠️ Any child processes some_parent_script.sh had spawned do NOT
# automatically receive any signal just because their parent was
# killed — they become ORPHANED (reparented to init/PID 1, or a
# subreaper) and continue running completely independently unless the
# parent script had explicitly arranged to signal its own children on exit.

# To ensure children are cleaned up too, target the whole process
# GROUP instead of just the parent PID (see the negative-PID section above):
kill -TERM -- "-$PARENT_PID"
```

---

## kill Exit Status Tells You Whether the Signal Was DELIVERED, Not Whether the Process Actually Died

### A successful kill command doesn't guarantee termination happened
```bash
kill 1234
echo $?
# 0    ← this means the signal was SUCCESSFULLY DELIVERED to the
# kernel for that PID — it says NOTHING about whether the process
# actually reacted to it, cleaned up, or has exited yet (or ever will,
# if it's ignoring the signal or stuck in an uninterruptible state).

# To confirm actual termination, check separately, ideally after a
# short pause to allow the process a chance to react:
sleep 1
if kill -0 1234 2>/dev/null; then
  echo "Still alive despite kill's success"
fi
```

---

## Permissions — You Can Only Signal Processes You Own (Usually)

### A quiet, common failure mode when scripts run as the wrong user
```bash
kill 1234
# kill: (1234): Operation not permitted
# ⚠️ You can generally only send signals to processes owned by your
# own user (with SIGCONT to a stopped process in the same session
# being a notable, narrower exception) — root can signal any process
# regardless of owner. A script running as an unprivileged user that
# expects to kill a root-owned service's process will fail exactly
# like this, often without an obviously clear reason at first glance.

# Confirm ownership before assuming a permission error means the PID
# is wrong:
ps -o user= -p 1234
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
