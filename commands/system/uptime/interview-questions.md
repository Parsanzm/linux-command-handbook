# uptime — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Load Average Fundamentals](#load-average-fundamentals)
- [Interpreting Load Average](#interpreting-load-average)
- [Internals](#internals)
- [uptime vs Other Tools](#uptime-vs-other-tools)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What three pieces of information does uptime report in its default output?**
> How long the system has been running since boot, how many users are currently logged in, and the system's load average over the trailing 1, 5, and 15-minute windows.

---

**Q2 🔥 Where does uptime get its data from on Linux?**
> Boot time/elapsed duration comes from `/proc/uptime`, and load average figures come from `/proc/loadavg` — both continuously maintained by the kernel. uptime is essentially a human-friendly formatting layer over these two files' raw values, not something that computes anything independently.

---

## Load Average Fundamentals

**Q3 🔥 What does a load average of "1.00" mean on a single-core system?**
> The CPU was, on average, exactly fully occupied over that time window, with no idle time and no processes queued waiting for CPU — full utilization, but not yet overloaded.

---

**Q4. What do the three numbers in "load average: 0.15, 0.22, 0.18" represent?**
> The load average calculated over the trailing 1-minute, 5-minute, and 15-minute windows, respectively (in that order).

---

**Q5 🔥 Is load average a simple arithmetic mean over a fixed time window?**
> No — it's an exponentially-damped, continuously-decaying weighted average, not a literal average over a strict, fixed sliding window. This means a brief, intense spike can continue showing an elevated "1-minute" figure for a short while even after the actual spike-causing process has already finished.

---

## Interpreting Load Average

**Q6 🔥 Why is a raw load average number meaningless without also knowing the number of CPU cores?**
> Load average represents demand relative to available CPU capacity, and that capacity is defined by the core count — a load of 8.00 is comfortably low on a 16-core system (about 50% utilization) but severely overloaded on a 2-core system (400% of available capacity). Checking `nproc` alongside any load average reading is essential for meaningful interpretation.

---

**Q7. What does it mean if a system shows high load average but `top` reveals very low actual CPU usage, with a high "wa" (I/O wait) percentage instead?**
> Load average counts processes in **uninterruptible sleep** (typically waiting on disk or network I/O) in addition to processes actively competing for CPU — a high load driven mostly by I/O wait suggests investigating storage or network performance bottlenecks (slow disks, an overwhelmed remote filesystem, database I/O contention), not necessarily a CPU capacity problem.

---

**Q8 🔥 What does it mean when the three load average numbers show a clearly rising trend (1-minute much higher than 15-minute)?**
> It indicates load has spiked **recently** — something started consuming significant resources within roughly the last minute or so, and the longer-window averages haven't yet caught up to reflect it. This pattern is worth investigating immediately, since it suggests an active, ongoing (or very recent) change in system behavior.

---

## Internals

**Q9. What does the fourth field in `/proc/loadavg` (e.g., "2/458") represent?**
> The number of currently runnable processes over the total number of processes on the system — in this example, 2 processes are currently runnable (competing for CPU) out of 458 total processes that exist.

---

**Q10 🔥 Why might load average inside a Docker container not accurately reflect that specific container's own resource pressure?**
> `/proc/loadavg`, which uptime reads from, is traditionally a **host-wide** kernel statistic — many container runtimes don't provide a fully isolated, per-container view of load average, so a number read inside a container may actually reflect the entire host machine's load (including other unrelated containers), not something scoped specifically to that one container's resource allocation.

---

**Q11. Does uptime's elapsed-duration calculation get thrown off by a large NTP clock correction?**
> Generally no — uptime's duration is based on a monotonic kernel counter that isn't affected by wall-clock adjustments, so a significant NTP correction typically doesn't corrupt the reported uptime duration itself, even though it can affect the "current time" portion of the output and complicate manual correlation with other wall-clock-based timestamps.

---

## uptime vs Other Tools

**Q12 🔥 What's the difference between `uptime` and `w`?**
> `w` shows the same load average and uptime summary information as `uptime`, but additionally lists each currently logged-in user's session details — including their login time, idle time, and the command they're currently running. `uptime` alone provides just the compact one-line summary without per-user detail.

---

**Q13. If uptime shows an elevated load average, what would be the next diagnostic tool to check, and why?**
> `top` (or `htop`), to identify **which specific process(es)** are actually responsible for the elevated load — uptime itself only reports an aggregate number and has no way to attribute that load to specific processes; `top` provides the necessary per-process breakdown to investigate further.

---

## Scenario-Based

**Q14 🔥 A colleague says "the server has a load average of 6, that's really high!" without additional context. What's your first question, and why does it matter?**
> "How many CPU cores does the server have?" — a load average of 6 is comfortably low on a 16-core machine (roughly 37% average utilization) but severely overloaded on a 2-core machine (300% of available capacity). Without core count context, the raw load average number alone provides no meaningful indication of whether the system is actually under stress.

---

**Q15. A monitoring alert fires because a server's 1-minute load average spiked to 20 on an 8-core system, but by the time you SSH in and run `uptime` again a minute later, the number has already dropped back to 3. What's the likely explanation, and how would you investigate what caused the original spike?**
> Load average is a damped, decaying average, not an instantaneous snapshot — a short-lived process or burst of activity can cause a temporary spike that decays out of the reported figure fairly quickly once it ends, meaning the underlying cause may have already finished by the time you check manually. To investigate after the fact, review logs from around the alert's timestamp (application logs, `journalctl`, cron job logs) or check a monitoring system with finer time-resolution than a single manual `uptime` check, since the live system state no longer reflects what caused the original spike.

---

**Q16 🔥 A cloud VM shows a comfortable-looking load average (around 2.0 on a 4-core instance) but users report the application feels sluggish and unresponsive. What additional metric would you check, and why?**
> Check the CPU **steal** percentage via `top` (the "st" column) — on virtualized/cloud instances, a significant portion of the VM's allocated CPU time can be "stolen" by the hypervisor to serve other co-located tenants on an oversubscribed physical host, causing real, user-perceptible sluggishness that doesn't cleanly show up as elevated load average from the VM's own perspective, since load average only reflects processes competing for the CPU time the VM actually believes it has available.

---

**Q17. After investigating a "server feels slow" complaint, `uptime` shows load average around 0.05 across all three windows — essentially idle. The user insists something was clearly wrong just moments before you checked. How do you reconcile this?**
> A very brief, sub-second or few-second spike can occur and fully resolve between the kernel's periodic load-average sampling points, contributing negligibly to the longer-window damped averages that `uptime` reports — load average is fundamentally designed to reflect sustained trends over minutes, not instantaneous or extremely short-lived activity. Reconciling this requires either a monitoring tool with much finer time-resolution (e.g., `sar` sampling every second, or an APM/monitoring agent) or correlating the complaint's timing against application-level logs rather than relying on a single manual `uptime` snapshot taken after the fact.

---

**Q18 🔥 A server has been running for what `uptime` reports as "15 days," but you suspect it may have actually crashed and silently restarted at some point during that window due to an OOM (out-of-memory) event. How would you verify this, given that uptime itself won't tell you?**
> `uptime` only reports elapsed time since the current boot — it has no concept of "reboot cause" and can't distinguish a clean 15-day uptime from a scenario where the reported 15 days actually started with a crash-triggered reboot. To verify, cross-reference `uptime -s` (the exact boot timestamp) against system logs from that period — `journalctl -b -1` (the previous boot's log) or `dmesg`/`/var/log/syslog` around that timestamp would reveal OOM-killer messages, kernel panics, or other reboot-triggering events, if the reported "15 days" actually began with an unplanned crash rather than a clean start.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
