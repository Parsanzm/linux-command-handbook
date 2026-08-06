# htop — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Reading the Interface](#reading-the-interface)
- [Interactive Controls](#interactive-controls)
- [Internals](#internals)
- [htop vs Other Tools](#htop-vs-other-tools)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What is htop, and how does it differ fundamentally from ps?**
> htop is an interactive, continuously refreshing, full-screen process viewer. `ps` prints a single static snapshot and exits; htop stays open, redraws itself on an interval, and lets you sort, filter, and act on processes (kill, renice) directly from within the interface.

---

**Q2 🔥 Is htop preinstalled on most Linux systems by default?**
> No — unlike `ps`, `top`, and `kill` (part of core system packages present almost everywhere), htop is a separate, third-party package that generally needs to be explicitly installed, and is frequently missing from minimal servers and fresh containers.

---

**Q3. What does htop primarily improve on compared to top?**
> A more approachable, colorized interface: per-core CPU bars with color-coded breakdown, mouse support, a visible tree view of parent/child process relationships, and generally more discoverable keyboard controls — while showing largely the same underlying information as `top`.

---

## Reading the Interface

**Q4 🔥 In htop's per-core CPU bars, what do the different colors typically represent?**
> Different categories of CPU time — commonly green for normal user-space processes, red for kernel/system time, blue for low-priority ("niced") processes, and additional colors (varying by theme/version) for I/O wait or virtualization steal time.

---

**Q5. Why might a single process show a CPU% figure above 100% in htop?**
> Because the percentage is relative to a single CPU core's full capacity, not the system's total capacity — a heavily multithreaded process using multiple cores simultaneously can legitimately show, for example, 340% if it's using roughly 3.4 cores' worth of CPU time at once.

---

**Q6 🔥 What's the difference between htop's "Tasks" count and its "Load average" figure in the header?**
> "Tasks" is a simple current count of processes/threads that exist right now (and how many are actively running this instant). "Load average" reports the same damped, exponentially-decaying 1/5/15-minute figures that `uptime` shows, representing demand over time rather than an instantaneous count.

---

## Interactive Controls

**Q7 🔥 How would you kill a selected process from within htop, and what happens after pressing the kill key?**
> Select the process, then press `F9` — this opens a menu of signals to choose from (typically with `SIGTERM` highlighted by default), and pressing Enter on the chosen signal sends it, the same as running `kill` with that signal against the process's PID from the command line.

---

**Q8. How do you filter the visible process list down to only processes matching a name, versus just searching for one?**
> `F4` filters the list, hiding everything that doesn't match the typed text until the filter is cleared. `F3` (or `/`) instead searches and jumps to the next matching process without hiding any of the others — the list stays complete, it just highlights and scrolls to matches.

---

**Q9 🔥 How would you select and signal multiple processes at once in htop?**
> Press `Space` on each process to tag it, repeating for every process you want to include, then press `F9` and choose a signal — it will be sent to every tagged process simultaneously, rather than requiring one kill action per process.

---

## Internals

**Q10. Where does htop actually get its process and resource data from on Linux?**
> The `/proc` filesystem, same as `ps` and `top` — htop has no special privileged access beyond what any user-space program reading `/proc` files (like `/proc/<pid>/stat` and `/proc/<pid>/status`) can see; it just re-reads this data on a regular interval and redraws the screen.

---

**Q11 🔥 Why does htop's own process appear in its own process list while it's running?**
> Because htop is itself an ordinary running process on the system, and it displays all processes it has visibility into (subject to any active filters) — including its own, the same way `ps` or `top` would also show their own process if checked from within their own output.

---

## htop vs Other Tools

**Q12 🔥 Why would someone choose plain top over htop, given htop is generally considered friendlier?**
> Guaranteed availability — `top` is part of core system packages present on virtually every Linux install by default, while htop requires a separate package installation that isn't always possible or desirable, especially on minimal servers or in restricted environments.

---

**Q13. What's the difference between htop and glances?**
> htop is focused specifically on process-level CPU/memory monitoring in an interactive interface. `glances` covers a broader scope in one dashboard — network activity, disk I/O, sensors, and containers — in addition to process information, at the cost of being an even less commonly preinstalled tool than htop.

---

**Q14 🔥 Can htop's output be piped or redirected to a file the way ps's output can?**
> Not usefully — htop is built around full-screen terminal control codes for its interactive display, so redirecting it produces garbled control-sequence noise rather than readable text. For scripting or file output, `ps` (or `top -b` for a batch-mode single snapshot) is the appropriate tool instead.

---

## Scenario-Based

**Q15 🔥 A colleague is confused because summing up the MEM% column across several related processes in htop gives a total that seems way higher than the system's actual physical RAM usage shown in the header. What's the explanation?**
> Processes sharing common libraries (like `libc`) each report those shared memory pages as part of their own individual `RES`/`MEM%` figure — summing across multiple processes double- or many-times-counts memory that's only physically resident in RAM once. The header's `Mem[ ]` bar correctly accounts for this sharing and gives an accurate system-wide total, unlike a manual sum of the per-process column.

---

**Q16. Inside a Docker container, htop shows what appears to be the ENTIRE host system's processes, not just the container's own. What configuration would explain this?**
> The container was very likely started with `--pid=host` (or an equivalent privileged configuration) that intentionally shares the host's PID namespace rather than isolating the container into its own — this also means signals sent from inside such a container can affect processes on the host machine itself, not just processes belonging to that container.

---

**Q17 🔥 Someone tries to lower a process's nice value (raise its scheduling priority) using F7 in htop as a regular, non-root user, and nothing seems to happen. Why?**
> Ordinary users can only ever make their own processes' priority worse (raise the nice value via F8), never improve it below the default or below its current value, without elevated privileges — running htop with `sudo` is required to actually lower a process's nice value below where it currently sits.

---

**Q18. A user set up htop sorted by CPU%, then toggled tree view on, and now the list no longer appears sorted purely by CPU usage from top to bottom. What happened?**
> Tree view (`F5`) reorganizes the process list by parent/child hierarchy first — the chosen sort criterion (CPU%, in this case) is still applied, but only *within* each level of that hierarchy, not as a single flat, system-wide ranking anymore. Toggling tree view back off restores the purely flat, sort-order-driven view.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
