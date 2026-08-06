# top — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Reading the Screen](#reading-the-screen)
- [Interactive Controls](#interactive-controls)
- [Internals](#internals)
- [top vs Other Tools](#top-vs-other-tools)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does top do, and how does it differ from ps?**
> `top` shows a continuously refreshing, live view of running processes and overall system resource usage. `ps` prints a single static snapshot and exits; `top` stays open, redraws on a fixed interval, and lets you sort/filter/signal processes interactively while it runs.

---

**Q2 🔥 Is top preinstalled on Linux systems by default?**
> Yes — unlike `htop`, `top` is part of the core `procps`/`procps-ng` package present on virtually every Linux distribution by default, requiring no separate installation.

---

**Q3. What are the four main sections of top's default screen?**
> A header line (uptime/load-average style summary), a Tasks line (process count by state), a `%Cpu(s)` line (CPU time breakdown by category), Mem/Swap summary lines, and the process table itself.

---

## Reading the Screen

**Q4 🔥 In the `%Cpu(s)` line, what does a high "wa" percentage indicate?**
> High I/O wait — the CPU was idle specifically while waiting for at least one process's disk or network I/O to complete. It does not mean the CPU is overloaded; it points toward investigating storage or network performance instead.

---

**Q5. What's the difference between "us" and "sy" in the CPU breakdown?**
> "us" (user) is time spent running normal, non-kernel user-space process code. "sy" (system) is time spent inside the kernel on behalf of processes — system calls, context switches, and similar kernel-side work.

---

**Q6 🔥 Why might a single process show a %CPU value above 100%?**
> Because the percentage is relative to a single CPU core's full capacity, not the system's total capacity — a multithreaded process using several cores at once can legitimately show a combined figure well above 100% (e.g., 340% means roughly 3.4 cores' worth of usage).

---

## Interactive Controls

**Q7 🔥 How would you kill a process from within top without leaving the interface?**
> Press `k`, enter the target PID at the prompt, then enter a signal number (or press Enter to accept the default, SIGTERM/15).

---

**Q8. How do you change the sort order while top is running?**
> Press `P` to sort by `%CPU`, `M` to sort by `%MEM`, `T` to sort by cumulative `TIME+`, or `o` to open a menu and choose any available field explicitly. `R` reverses the current sort direction.

---

**Q9 🔥 What does pressing `W` do inside top?**
> Saves the current display settings (sort order, visible columns, and other options) to `~/.toprc`, making them the persistent default for future top sessions on that account.

---

## Internals

**Q10. Where does top get its data from on Linux?**
> The `/proc` filesystem — the same source `ps` reads from (`/proc/<pid>/stat`, `/proc/meminfo`, `/proc/stat`, etc.). top has no special kernel access beyond what any userspace program reading `/proc` can see.

---

**Q11 🔥 Why can top's very first displayed screen show inaccurate %CPU figures?**
> %CPU is a rate, calculated by comparing CPU time used between the current reading and the previous one. On the very first refresh, there is no true prior reading to compare against, so the initial numbers can be misleading — they settle into accurate values starting from the second refresh cycle onward.

---

**Q12. Why does top's own process appear in its own displayed list?**
> Because top is itself an ordinary running process, and it displays all processes it has visibility into — including itself — the same way `ps` would show its own process if checked from within its own output.

---

## top vs Other Tools

**Q13 🔥 Why might someone choose top over htop despite htop being generally friendlier?**
> Guaranteed availability — top is part of the core system packages present by default on virtually every Linux install, while htop is a separate package that must be explicitly installed and isn't always available, especially on minimal servers or containers.

---

**Q14. Can top's regular output be piped or redirected to a file usefully?**
> Not in its default interactive mode — that's built around full-screen terminal control codes and produces garbled output when redirected. Batch mode (`top -b`, typically combined with `-n` to limit iterations) produces clean, sequential plain-text output suitable for piping, redirecting, or logging.

---

## Scenario-Based

**Q15 🔥 A colleague sums the %MEM column across several related processes in top and gets a total higher than the system's actual used memory shown in the header. What's the explanation?**
> Processes sharing common libraries each report those shared memory pages as part of their own individual %MEM figure — summing across multiple processes double- or many-times-counts memory that's physically resident in RAM only once. The header's Mem summary line correctly accounts for sharing and gives an accurate total, unlike a manual sum of the per-process column.

---

**Q16. You want to log a system resource snapshot to a file every 5 minutes via a cron job. Why won't a plain `top` command work for this, and what's the fix?**
> Plain `top` runs in interactive full-screen mode indefinitely and produces garbled control-sequence output if piped or redirected — it also never exits on its own, which would hang a cron job. The fix is batch mode with a bounded iteration count: `top -bn1 >> logfile`, which produces one clean, plain-text snapshot and then exits immediately.

---

**Q17 🔥 During an incident, top shows a high "wa" percentage but relatively low "us"/"sy" figures. What should the next diagnostic step be, and why?**
> Investigate storage/network I/O specifically (e.g., `iostat -x 1 5`), since high I/O wait means the CPU was idle while waiting on slow disk or network operations — not that the CPU itself is under heavy compute demand. Adding CPU capacity or optimizing CPU-bound code wouldn't address a bottleneck that's actually rooted in I/O.

---

**Q18. A very brief CPU spike (lasting under a second) doesn't appear anywhere in top's history when reviewed a few minutes later. Why might that be, given top was running the whole time?**
> top only captures discrete snapshots at each refresh interval (3 seconds by default) — it isn't a continuous recording. A spike shorter than the refresh interval can fall entirely between two consecutive samples and simply never appear in what top displayed, even though top was actively running throughout. A finer-grained sampling tool (a shorter `-d` delay, or a dedicated tool like `sar`) is needed to reliably catch sub-interval spikes.

---

**Q19 🔥 Inside a container, top's memory/CPU summary appears to reflect figures far larger than what the container is actually limited to. What's the likely cause?**
> Depending on the container runtime, cgroup version, and namespace configuration, top's summary lines can reflect the underlying HOST's totals rather than the container's own resource limits — particularly on older configurations or when PID/resource namespaces aren't fully isolated. Cross-checking the container's actual cgroup-reported limits (e.g., `/sys/fs/cgroup/memory.max`) gives the accurate, container-scoped picture instead.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
