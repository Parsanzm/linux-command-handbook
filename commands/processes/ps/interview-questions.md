# ps — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Option Styles](#option-styles)
- [Process States](#process-states)
- [Internals](#internals)
- [ps vs Other Tools](#ps-vs-other-tools)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does `ps` do?**
> Prints a single, static snapshot of currently running processes on the system — their PID, owning user, resource usage, state, and the command that launched them — then exits, without continuously refreshing.

---

**Q2 🔥 Why is `ps` described as a "snapshot" rather than a live view?**
> It reads the process table once at the moment it's run and prints that result; it does not refresh automatically. By the time the output is displayed (or acted upon), the actual process list may have already changed — some listed processes may have exited, and new ones may have started.

---

**Q3. What's the difference between `ps` with no arguments and `ps aux`?**
> Plain `ps` (no options) shows only processes owned by the current user and attached to the current terminal — typically a very short list. `ps aux` shows every process from every user on the system, including those without a controlling terminal.

---

## Option Styles

**Q4 🔥 Why does `ps` accept options both with and without a leading dash (e.g., `ps aux` vs `ps -ef`)?**
> `ps` supports multiple historically distinct option conventions simultaneously: BSD-style (no leading dash, e.g. `aux`), UNIX/POSIX-style (leading dash, e.g. `-ef`), and GNU long-option style (double dash, e.g. `--pid`). These trace back to different Unix lineages that `ps` implementations later merged support for.

---

**Q5. Are `ps aux` and `ps -ef` interchangeable?**
> No — they show overlapping but genuinely different sets of columns and use different underlying option semantics (BSD vs UNIX style), even though both are commonly used as a "show me everything" command. `ps aux` includes `%CPU`/`%MEM` directly; `ps -ef` includes `PPID` directly.

---

**Q6 🔥 How would you display only PID, user, and command, with nothing else?**
> `ps -eo pid,user,cmd` — the `-o` (or `--format`) option lets you specify an exact, ordered list of columns to display.

---

## Process States

**Q7. What does the `STAT` column's `Z` code mean, and can you `kill` a process in that state?**
> `Z` means zombie — the process has already finished executing and released its resources; only a small table entry remains, holding its exit status until the parent process reaps it via `wait()`. There's nothing left to signal, so `kill`ing a zombie's PID has no effect — the fix is addressing the parent process instead.

---

**Q8 🔥 What does the `D` state mean, and why is it notable compared to other states?**
> Uninterruptible sleep — typically a process blocked on a kernel-level I/O operation (slow disk, an unresponsive NFS mount). It's notable because a process in `D` state cannot be interrupted by any signal, including `SIGKILL` — the kernel won't process signal delivery until the blocking operation itself resolves.

---

**Q9. What's the difference between `R` and `S` states?**
> `R` means running or currently runnable (on the CPU run queue, ready to execute). `S` means interruptible sleep — the process is waiting for some event (input, a timer, a lock) and can be woken by a signal at any time.

---

## Internals

**Q10 🔥 Where does `ps` actually get its data from on Linux?**
> The `/proc` filesystem — specifically per-process directories like `/proc/<pid>/stat`, `/proc/<pid>/status`, and `/proc/<pid>/cmdline`, which the kernel maintains continuously. `ps` re-reads these fresh on every invocation; it has no persistent daemon or cache of its own.

---

**Q11. What does the `%CPU` column actually represent — is it a live, real-time reading?**
> No — it's roughly the ratio of total CPU time the process has consumed to the total time it has existed, making it closer to a running average since the process started than an instantaneous "right now" measurement. A process that spiked heavily earlier but is currently idle can still show a moderate `%CPU` value.

---

**Q12 🔥 What's the practical difference between `VSZ` and `RSS`?**
> `VSZ` (virtual size) is the total address space the process has mapped, including memory that may never actually be touched or resident in physical RAM. `RSS` (resident set size) is the portion of that address space currently held in physical RAM. `RSS` is generally the more meaningful figure for "how much real memory is this process using right now."

---

## ps vs Other Tools

**Q13 🔥 When would you use `ps` instead of `top`, and vice versa?**
> `ps` is better for scripting, one-off lookups, and piping into other tools (`grep`, `awk`, `sort`) because it prints once and exits cleanly. `top` is better for interactively watching resource usage change over time, since it continuously refreshes and offers live interactive sorting — but its default output isn't well-suited for direct scripting.

---

**Q14. What's the advantage of `pgrep` over `ps aux | grep pattern`?**
> `pgrep` is purpose-built for exactly this — it returns matching PIDs directly without the classic problem of `grep` matching its own invocation in the pipeline (since `grep pattern` itself briefly appears in the process list and can match its own search pattern). It's also more script-friendly, since its output is just PIDs (or PID + name with `-l`) rather than full formatted `ps` columns needing further parsing.

---

## Scenario-Based

**Q15 🔥 A colleague runs `ps aux | grep myapp` and is confused to see two matching lines, one of which they don't recognize. What's likely happening?**
> The second line is very likely `grep`'s own process, briefly appearing in the process list and matching its own search term "myapp" (since the full `grep myapp` command line contains that string). The fix is either bracketing one character of the pattern (`grep '[m]yapp'`) so grep's own invocation no longer matches, or using `pgrep -l myapp` instead, which avoids the issue entirely.

---

**Q16. You spot a process stuck in `D` state and `kill -9` doesn't seem to terminate it. What's going on, and what should you actually investigate?**
> `D` (uninterruptible sleep) processes are blocked inside a kernel operation that can't be interrupted by any signal, including `SIGKILL` — so the command appears to succeed but the process remains unaffected until the underlying blocking operation resolves. The real investigation should focus on whatever the process is blocked on — commonly slow or failing disk I/O, an unresponsive NFS/network filesystem mount, or a hardware issue — rather than treating it as a process-management problem solvable via signals.

---

**Q17 🔥 You want to find the total memory used by all instances of a particular application. Why is simply summing the `RSS` column across matching processes potentially misleading?**
> Processes that share common libraries (like `libc`) each report those shared pages as part of their own individual `RSS`. Summing `RSS` across multiple processes therefore double- (or many-times-) counts memory that's physically resident in RAM only once, overstating true memory usage. Tools that account for proportional shared memory (like `smem`), or system-wide views like `free -h`, give a more accurate total.

---

**Q18. A script records a process's PID early on and plans to `kill` it later after some long-running task finishes. What's a subtle risk with this approach, and how would you make it safer?**
> PIDs are recycled by the kernel once they're free — if the originally recorded process has already exited by the time the script acts, that PID number may have since been reassigned to a completely unrelated process, and the script would end up killing the wrong thing. A safer approach re-verifies the process's identity right before acting — for example, checking that `ps -p <pid> -o cmd=` still matches the expected command — rather than trusting a PID captured long earlier without confirmation.

---

**Q19 🔥 Inside a Docker container, `ps aux` shows only a couple of processes, even though the host is running hundreds. Why, and is that always guaranteed?**
> Containers typically run inside their own PID namespace, so a process inside the container can generally only see processes within that same namespace — not the host's full process table or other containers' processes. This isolation is not absolute, though — a container started with `--pid=host` (or certain privileged configurations) intentionally shares the host's PID namespace and would see the full host process list instead.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
