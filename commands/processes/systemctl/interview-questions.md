# systemctl — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Enabled vs Active](#enabled-vs-active)
- [Unit Files and Internals](#unit-files-and-internals)
- [Unit Types](#unit-types)
- [systemctl vs Other Tools](#systemctl-vs-other-tools)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What is systemctl, and what does it actually control?**
> The command-line interface to systemd, the init system and service manager used by most modern Linux distributions. It controls "units" — services, sockets, mounts, timers, and more — by sending requests to the running systemd manager process (PID 1).

---

**Q2 🔥 Does systemctl manage processes directly itself?**
> No — it's a client that communicates with the systemd manager (PID 1) over D-Bus. The actual work (forking processes, applying dependency ordering, tracking resource usage via cgroups) is done by systemd itself; systemctl just sends commands and displays results.

---

**Q3. What is a "unit" in systemd terminology?**
> The general term for anything systemd manages — not just background services. Distinguished by file extension: `.service`, `.socket`, `.timer`, `.mount`, `.target`, `.path`, and `.device` are all different unit types, each configured by its own unit file.

---

## Enabled vs Active

**Q4 🔥 What's the difference between `systemctl enable` and `systemctl start`?**
> `enable` only configures whether a unit will start automatically at the *next boot* — it does not start it right now. `start` only starts it *right now* — it has no effect on boot behavior. They are independent operations addressing two separate questions.

---

**Q5. If a service shows `enabled` but `inactive`, what does that mean?**
> It's configured to start automatically at the next boot, but isn't currently running — likely because it was manually stopped at some point after boot, or hasn't been started yet since being enabled.

---

**Q6 🔥 How do you both enable and start a service in a single command?**
> `systemctl enable --now UNIT`

---

## Unit Files and Internals

**Q7. What's required after manually creating or editing a unit file directly on disk?**
> `systemctl daemon-reload` — systemd caches parsed unit files in memory and doesn't automatically detect changes made directly to files on disk; without a reload, it may continue operating on the old, already-loaded definition.

---

**Q8 🔥 What's the difference between editing a unit file in `/usr/lib/systemd/system/` versus using `systemctl edit`?**
> Files under `/usr/lib/systemd/system/` are typically owned by the installed package and get overwritten on the next package update, silently discarding any manual changes. `systemctl edit` creates a drop-in override file under `/etc/systemd/system/<unit>.d/`, which systemd merges on top of the original — surviving future package updates since it's a separate file entirely.

---

**Q9. What does `systemctl mask` do, and how is it different from `disable`?**
> `disable` removes a unit's automatic boot activation but still allows it to be started manually afterward. `mask` goes further — it replaces the unit's file reference with a symlink to `/dev/null`, making the unit impossible to start even manually until it's explicitly unmasked.

---

## Unit Types

**Q10 🔥 What's the systemd equivalent of a cron job, and how does it differ conceptually from a `.service` unit?**
> A `.timer` unit, which is paired with a corresponding `.service` unit it triggers on a schedule. Unlike a long-running `.service`, the timer itself doesn't do the work — it just triggers activation of its associated service at scheduled times, and `systemctl list-timers` shows all currently scheduled timers and their next run.

---

**Q11. What does it mean if `systemctl is-enabled` reports `static` for a unit?**
> The unit has no `[Install]` section, meaning it was never designed to be independently enabled or disabled by an administrator — it's typically pulled in automatically as a dependency of some other unit whenever needed. This is expected, normal behavior, not an error.

---

## systemctl vs Other Tools

**Q12 🔥 What's the difference between `systemctl status myapp` and `journalctl -u myapp`?**
> `systemctl status` shows a quick summary — current state, key metadata, and only a small preview of the most recent log lines. `journalctl -u myapp` shows the full, unabridged log history for that specific unit, and can be filtered by time range or followed live with `-f`.

---

**Q13. Is the older `service` command still usable on a systemd-based distro, and what's the trade-off?**
> Yes, on many systemd distros `service NAME start` still works as a compatibility shim that forwards to systemd, but it exposes far less information and control than using `systemctl` directly — status detail, dependency inspection, and enable/disable management are systemctl-specific capabilities the `service` wrapper doesn't fully replicate.

---

## Scenario-Based

**Q14 🔥 A teammate enables a service, then reboots the server expecting it to be running — but it isn't. What's the most likely explanation?**
> `enable` alone doesn't start anything immediately; it only configures the unit for future boots. If it was never separately `start`ed (or `enable --now` wasn't used) before the reboot happened, the very *next* boot after enabling should still bring it up correctly — but if it's still not running after that, the more likely explanation is a dependency (network, mount, another service) not being ready, or the service failing on startup. `systemctl status` and `journalctl -u` on that unit would clarify which.

---

**Q15. You edit `/etc/systemd/system/myapp.service` to add an environment variable, then restart the service — but the new variable doesn't seem to take effect. What's the most likely missing step?**
> `systemctl daemon-reload` was likely skipped. Without it, systemd may still be operating on its previously cached in-memory definition of the unit rather than the updated file on disk, so the restart doesn't pick up the change until a reload happens first.

---

**Q16 🔥 A service shows `active (running)` when you check it, but a teammate insists it's been crashing repeatedly all night. How do you reconcile this?**
> If the unit has `Restart=on-failure` (or similar), a crash-loop can look perfectly healthy in any single snapshot, since a status check taken at a random moment will simply catch it mid-restart, already running again. Checking the actual restart count (`systemctl show myapp --property=NRestarts`) or scanning the log history for repeated "Started"/"Stopped" entries over the relevant time window reveals the instability that a single status check hides.

---

**Q17. You want to guarantee a specific service can never run on a server — not just that it won't start automatically at boot. What command should you use, and why not just `disable`?**
> `systemctl mask`. `disable` only removes automatic boot activation but still permits a manual `systemctl start` afterward — insufficient if the goal is to make the unit completely unstartable under any circumstance until deliberately reversed with `unmask`.

---

**Q18 🔥 A one-shot setup service shows `Active: active (exited)`. Is this something to be concerned about?**
> Not by itself — "active (exited)" is the normal, expected final state for a `Type=oneshot` unit that ran once, completed its task, and exited. Whether it's actually healthy depends on its exit status, which can be checked directly with `systemctl show <unit> --property=ExecMainStatus` (0 meaning it exited cleanly) rather than assuming "exited" alone implies a problem.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
