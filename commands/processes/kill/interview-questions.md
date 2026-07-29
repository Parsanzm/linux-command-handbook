# kill — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Signals Fundamentals](#signals-fundamentals)
- [SIGTERM vs SIGKILL](#sigterm-vs-sigkill)
- [Internals](#internals)
- [kill vs Other Tools](#kill-vs-other-tools)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does the kill command actually do?**
> Sends a signal to one or more processes, identified by PID. Despite the name, it's a general signal-delivery mechanism — termination is only the most common use, not the only one.

---

**Q2 🔥 What signal does kill send if you don't specify one?**
> `SIGTERM` (signal 15) — a request for the process to terminate, which it can catch and handle gracefully before actually exiting.

---

**Q3. Can a process ignore a signal sent by kill?**
> Yes, for most signals — a process can register a custom handler or explicitly ignore many signals. `SIGKILL` (9) and `SIGSTOP` (19) are the two exceptions: their default action is enforced unconditionally by the kernel and cannot be caught, blocked, or ignored.

---

## Signals Fundamentals

**Q4 🔥 What are three equivalent ways to send SIGTERM to PID 1234?**
> `kill 1234` (default signal), `kill -TERM 1234` (by name), and `kill -15 1234` (by number) — all three send the identical signal.

---

**Q5. What does kill -l do?**
> Lists all available signal names (optionally converting a given number to its corresponding name), useful for looking up exactly which signal a given number corresponds to.

---

**Q6 🔥 What signal does pressing Ctrl+C send to a foreground process?**
> `SIGINT` (signal 2) — an interrupt request, distinct from the `SIGTERM` that plain `kill` sends by default.

---

## SIGTERM vs SIGKILL

**Q7 🔥 What's the practical difference between SIGTERM and SIGKILL?**
> `SIGTERM` is a request the target process can catch and handle — commonly used to run cleanup logic (closing files, flushing buffers, saving state) before exiting on its own terms. `SIGKILL` is enforced immediately and unconditionally by the kernel, with no opportunity for the process to run any cleanup code at all.

---

**Q8. Why is SIGKILL generally recommended only as a last resort?**
> Because the target process gets no chance to clean up — open files, in-progress writes, or held locks can be left in an inconsistent state. The generally recommended pattern is sending `SIGTERM` first, waiting briefly, and only escalating to `SIGKILL` if the process is still running after that grace period.

---

**Q9 🔥 What does kill -0 PID actually do?**
> Sends no real signal at all — it's used purely to test whether a process with that PID currently exists and whether the caller has permission to signal it, based solely on the command's exit status.

---

## Internals

**Q10. What system call does kill ultimately rely on?**
> `kill(2)` — the kernel asks the target process to receive a pending signal, and delivers it at the next opportunity the scheduler provides. What happens after delivery depends entirely on the receiving process's own signal handling.

---

**Q11 🔥 Is there a difference between the kill shell builtin and the standalone /usr/bin/kill binary?**
> Yes — most shells provide their own builtin `kill`, which is what typically runs by default and which additionally understands shell-specific job-control syntax (like `%1`). The standalone `/usr/bin/kill` binary only understands literal PIDs and process groups, not job specs, since job numbers are a shell-level concept the standalone binary has no knowledge of.

---

**Q12. What does a negative PID mean when passed to kill?**
> It targets the entire **process group** with that ID, rather than a single process — e.g., `kill -TERM -1234` signals every process in process group 1234, not just a single process numbered 1234.

---

## kill vs Other Tools

**Q13 🔥 What's the difference between kill and killall?**
> `kill` requires an exact PID (or process group ID/job spec) to target a process. `killall` targets processes by **command name** instead, signaling every process that exactly matches the given name at once.

---

**Q14. When would you use pkill instead of kill?**
> When you want to target processes by a flexible pattern (command name substring, owning user, or other filters) rather than by a PID you already know — `pkill` combines the matching logic of `pgrep` with the signal-sending behavior of `kill`.

---

## Scenario-Based

**Q15 🔥 A process doesn't respond to kill 1234 at all — it's still running afterward. What are the most likely explanations?**
> Either the process has a signal handler that's deliberately catching `SIGTERM` and choosing not to exit (or is still in the middle of a slow cleanup routine), or — less commonly — it's in an uninterruptible kernel sleep state (`D` in `ps`) and hasn't even had the chance to process the signal yet. Escalating to `kill -9` addresses the first case; a persistent `D`-state issue points to an underlying I/O or driver problem instead.

---

**Q16. You run kill -9 on a PID you recorded from a script an hour ago, and it seems to succeed, but the intended process is still running under a different, new PID. What likely happened?**
> PID reuse — the kernel recycles PID numbers once they're freed. If the originally targeted process had already exited by the time the script ran `kill -9`, that PID number may have since been reassigned to a completely unrelated process, which is what actually received the signal. Re-verifying the process's identity (e.g., checking its command line) before signaling a PID recorded some time earlier avoids this.

---

**Q17 🔥 You want to kill a background job and every child process it spawned, not just the parent process itself. How would you approach this with kill?**
> Target the process group rather than the single parent PID, using a negative PID: `kill -TERM -- "-$PARENT_PID"`. Killing only the parent PID directly does not automatically signal any of its already-spawned children — they'd simply become orphaned and continue running independently unless the whole group is signaled together.

---

**Q18. A teammate accidentally runs kill -9 -1 while logged in as root. What actually happens, and why is this dangerous?**
> `-1` as a PID argument is a special-cased value meaning "every process the caller has permission to signal" — for root, that effectively means every process on the entire system (except `kill`'s own process and PID 1), making this an extremely destructive, system-wide command rather than an ordinary negative process-group reference. This is why scripts should always validate that a PID variable is a genuine, expected positive value before passing it to `kill`.

---

**Q19 🔥 A long-running daemon is sent SIGHUP, and instead of terminating, it re-reads its configuration file and keeps running. Is this expected behavior?**
> Yes — while `SIGHUP`'s historical default action is termination (originally signaling a disconnected controlling terminal), many long-running daemons deliberately install a custom handler that reinterprets `SIGHUP` as "reload configuration" instead, by long-standing convention rather than any kernel-enforced meaning. Whether a given daemon treats `SIGHUP` as reload or terminate depends entirely on that specific program's own documented behavior.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
