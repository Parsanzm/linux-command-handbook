# kill — The Complete Reference

> **Send a signal to a process**
> Despite the name, `kill` doesn't only terminate processes — it's a
> general-purpose tool for sending any signal, of which termination is
> just one common use.

---

## Table of Contents

- [What is kill?](#what-is-kill)
- [Where does kill live?](#where-does-kill-live)
- [How kill works internally](#how-kill-works-internally)
- [Syntax](#syntax)
- [Understanding Signals](#understanding-signals)
- [The Most Important Signals](#the-most-important-signals)
- [SIGTERM vs SIGKILL — Asking Nicely vs Not Asking at All](#sigterm-vs-sigkill--asking-nicely-vs-not-asking-at-all)
- [All Key Options](#all-key-options)
- [Targeting Process Groups and Special PIDs](#targeting-process-groups-and-special-pids)
- [kill vs killall vs pkill vs xkill](#kill-vs-killall-vs-pkill-vs-xkill)
- [Related Commands](#related-commands)

---

## What is kill?

`kill` sends a **signal** to one or more processes, identified by PID. The name is a historical holdover — by far the most common signals sent happen to request termination, but `kill` is really a general signal-delivery mechanism, and plenty of signals have nothing to do with ending a process at all.

```bash
kill 1234
# Sends SIGTERM (the default signal) to PID 1234 — a polite request
# to terminate, which the process can catch and react to before exiting.
```

**Why "signal delivery," not "process destruction," is the right mental model:** a process receiving a signal can choose to ignore it, handle it with custom cleanup logic, or let the default action occur — `kill` only requests an action; it doesn't forcibly reach into a process's memory and stop it (with one key exception, `SIGKILL`, covered below).

---

## Where does kill live?

`kill` exists in two forms that are easy to conflate:

```bash
type kill
# kill is a shell builtin
```

Most shells (bash, zsh) provide their **own builtin** `kill`, which is what runs by default. A separate, standalone binary also typically exists:

```bash
which kill
# /usr/bin/kill

/usr/bin/kill --version
# kill from procps-ng 4.0.4
```

The standalone binary (from **procps**/**procps-ng**) is used when explicitly invoked by full path, or from a context without shell builtins (some scripting scenarios, `exec`'d programs). The two generally behave the same for common usage, but only the shell builtin can target job-control job specs like `%1` (see [Targeting Process Groups and Special PIDs](#targeting-process-groups-and-special-pids)).

---

## How kill works internally

`kill` is a thin wrapper around the `kill(2)` system call, which asks the kernel to deliver a specified signal to a specified process (or process group). The kernel records the pending signal against the target process and delivers it at the next opportunity the scheduler gives that process to handle signals.

```c
int kill(pid_t pid, int sig);
```

What happens next depends entirely on the **receiving process**, not on `kill` itself:

- It may have a registered **signal handler** for that signal, letting it run custom cleanup code.
- It may **ignore** the signal entirely (if permitted for that particular signal).
- It may take the **default action** the kernel defines for that signal if no handler is registered (terminate, ignore, stop, dump core, depending on the signal).

`SIGKILL` (9) and `SIGSTOP` (19) are the two signals a process can **never** intercept, ignore, or override — the kernel enforces their default action unconditionally.

---

## Syntax

```bash
kill [-SIGNAL] PID...
kill -s SIGNAL PID...
kill -l              # list all signal names
```

The signal defaults to `SIGTERM` (15) if none is specified.

---

## Understanding Signals

A signal is identified by both a **name** and a **number**, and either form works interchangeably:

```bash
kill -SIGTERM 1234
kill -TERM 1234
kill -15 1234
# All three are exactly equivalent
```

```bash
kill -l
#  1) SIGHUP    2) SIGINT    3) SIGQUIT   4) SIGILL    5) SIGTRAP
#  6) SIGABRT   7) SIGBUS    8) SIGFPE    9) SIGKILL  10) SIGUSR1
# 11) SIGSEGV  12) SIGUSR2  13) SIGPIPE  14) SIGALRM  15) SIGTERM
# ...
```

Signal numbers are largely consistent across Linux systems, but not perfectly universal across every Unix variant — using signal **names** rather than raw numbers is generally the more portable, self-documenting habit.

---

## The Most Important Signals

| Signal | Number | Default action | Common use |
|---|---|---|---|
| `SIGHUP` | 1 | Terminate | Historically "terminal hung up"; commonly repurposed by daemons today to mean "reload your configuration" |
| `SIGINT` | 2 | Terminate | What `Ctrl+C` sends — an interrupt request |
| `SIGQUIT` | 3 | Terminate + core dump | What `Ctrl+\` sends |
| `SIGKILL` | 9 | Terminate (unblockable) | Forceful, unconditional termination — cannot be caught, blocked, or ignored |
| `SIGSEGV` | 11 | Terminate + core dump | Sent by the kernel on an invalid memory access |
| `SIGUSR1` / `SIGUSR2` | 10 / 12 | Terminate (default) | Reserved for application-defined custom behavior |
| `SIGPIPE` | 13 | Terminate | Sent when writing to a pipe/socket with no reader left |
| `SIGTERM` | 15 | Terminate | The default, "polite" request to terminate — catchable for graceful cleanup |
| `SIGSTOP` | 19 | Stop (unblockable) | Pause a process's execution unconditionally — used by job control |
| `SIGCONT` | 18 | Continue | Resume a previously stopped process |

---

## SIGTERM vs SIGKILL — Asking Nicely vs Not Asking at All

```bash
kill 1234
# Sends SIGTERM (15) — a request the process CAN catch, and typically
# uses to close open files, flush buffers, save state, and exit cleanly.

kill -9 1234
# Sends SIGKILL (9) — the kernel terminates the process IMMEDIATELY,
# with no opportunity for the process to run any cleanup code at all.
# This can leave temp files, database transactions, or locks in an
# inconsistent state, since nothing gets a chance to run first.
```

The generally recommended order of escalation:

```bash
kill 1234          # 1. Try SIGTERM first — give it a chance to exit cleanly
sleep 5
kill -0 1234 && kill -9 1234   # 2. If it's still running after a pause, escalate to SIGKILL
```

`kill -0 PID` sends no actual signal at all — it's a standard idiom purely for checking whether a PID still exists (see below).

---

## All Key Options

| Option | Description |
|---|---|
| `-SIGNAL` | Send the specified signal instead of the default `SIGTERM` (by name or number) |
| `-s SIGNAL` | Same as above, alternate syntax (`kill -s TERM 1234`) |
| `-l` | List all available signal names (or convert a number to a name) |
| `-0` | Send no signal at all — used purely to test whether a PID exists and is signalable |
| `-p` | Print the PID(s) rather than sending a signal (shell builtin only) |

---

## Targeting Process Groups and Special PIDs

```bash
kill 1234        # target a single PID
kill 1234 5678    # target multiple PIDs in one call

kill -TERM -1234
# A NEGATIVE PID targets the entire PROCESS GROUP with that ID instead
# of a single process — useful for killing a process and all its
# children together.

kill %1
# Targets shell job number 1 (job-control syntax) — only works via
# the SHELL BUILTIN kill, not the standalone /usr/bin/kill binary,
# since job numbers are a shell-level concept, not a kernel one.

kill -0 1234
# Sends no signal; exit status alone tells you whether PID 1234
# currently exists and you have permission to signal it — a common
# idiom for checking "is this process still alive" in scripts.
```

---

## kill vs killall vs pkill vs xkill

| Tool | Targets by | Key difference from kill |
|---|---|---|
| `kill` | PID (or process group / job spec) | Requires already knowing the exact PID |
| `killall` | Process NAME (exact match by default) | Signals every process matching a given command name at once |
| `pkill` | Pattern (name, user, and other filters) | More flexible matching than `killall`, script-friendly, similar to `pgrep`'s selection logic |
| `xkill` | Interactive click target (X11 only) | GUI tool — click a window to kill the process owning it, no PID/name needed at all |

```bash
kill 1234              # kill exactly this one known PID
killall firefox         # kill every process literally named "firefox"
pkill -u alice node     # kill every "node" process owned by user alice
xkill                   # click a frozen GUI window to kill it (X11 desktops)
```

---

## Related Commands

| Command | Relation |
|---|---|
| `killall` | Kill by exact process name instead of PID |
| `pkill` | Kill by flexible pattern/user match instead of PID |
| `pgrep` | Find PIDs matching a pattern (the read-only counterpart to `pkill`) |
| `ps` | Find the PID(s) you need before calling `kill` |
| `trap` | Shell builtin for making a script react to a received signal itself |
| `nohup` | Make a process ignore `SIGHUP` specifically, so it survives terminal closure |
| `top` / `htop` | Interactive process views that also let you send signals directly |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
