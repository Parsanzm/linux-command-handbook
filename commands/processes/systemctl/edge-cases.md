# systemctl — Edge Cases & Gotchas

> `systemctl` looks like a simple start/stop switch, but enabled-vs-active
> confusion, masking, drop-in overrides, and dependency ordering routinely
> catch people off guard.

---

## Table of Contents

- [enable Does NOT Start a Service, and start Does NOT Enable It](#enable-does-not-start-a-service-and-start-does-not-enable-it)
- [Editing a Unit File Directly Requires daemon-reload Afterward](#editing-a-unit-file-directly-requires-daemon-reload-afterward)
- [Masked Units Can't Be Started Even Manually — disable Isn't Enough](#masked-units-cant-be-started-even-manually--disable-isnt-enough)
- [Active (exited) Doesn't Mean the Service Crashed](#active-exited-doesnt-mean-the-service-crashed)
- [restart Can Fail Silently on a Unit That Was Never Running](#restart-can-fail-silently-on-a-unit-that-was-never-running)
- [Editing /usr/lib/systemd/system/ Files Gets Overwritten on Package Updates](#editing-usrlibsystemdsystem-files-gets-overwritten-on-package-updates)
- [A Service Can Be enabled But Still Not Start — Check Its Dependencies](#a-service-can-be-enabled-but-still-not-start--check-its-dependencies)
- [status Only Shows a Few Recent Log Lines — Not the Full History](#status-only-shows-a-few-recent-log-lines--not-the-full-history)
- [Restart=on-failure Can Mask a Recurring Crash Loop](#restarton-failure-can-mask-a-recurring-crash-loop)
- [Not Every Distro Uses systemd — systemctl Simply Won't Exist There](#not-every-distro-uses-systemd--systemctl-simply-wont-exist-there)
- [Static Units Can't Be Enabled or Disabled — That's Expected](#static-units-cant-be-enabled-or-disabled--thats-expected)

---

## enable Does NOT Start a Service, and start Does NOT Enable It

### The single most common systemctl misunderstanding
```bash
sudo systemctl enable nginx
systemctl is-active nginx
# inactive     ← ⚠️ enable ONLY set up the boot-time symlink; it did
# NOT start the service right now. A very common mistake is running
# `enable` and assuming the service is now running, when it's only
# configured to run at the NEXT boot.

sudo systemctl start nginx
sudo reboot
# after reboot:
systemctl is-active nginx
# inactive     ← ⚠️ start ONLY ran it right now; it did nothing to
# configure boot behavior, so it did NOT come back after the reboot.

# The command that actually does both at once:
sudo systemctl enable --now nginx
```

---

## Editing a Unit File Directly Requires daemon-reload Afterward

### Changes on disk don't automatically reach the running systemd manager
```bash
sudo nano /etc/systemd/system/myapp.service
# ... make changes, save ...

sudo systemctl restart myapp
# ⚠️ This may appear to work, or may silently continue using the OLD,
# already-loaded unit definition — systemd caches parsed unit files in
# memory and does not automatically detect changes made directly to
# files on disk.

# The required step after ANY manual unit file edit or addition:
sudo systemctl daemon-reload
sudo systemctl restart myapp
```

---

## Masked Units Can't Be Started Even Manually — disable Isn't Enough

### mask is a much stronger action than disable, and people often reach for the wrong one
```bash
sudo systemctl disable some-unwanted.service
sudo systemctl start some-unwanted.service
# ⚠️ This SUCCEEDS — disable only removes automatic boot activation;
# the unit can still be started manually at any time afterward.

sudo systemctl mask some-unwanted.service
sudo systemctl start some-unwanted.service
# Failed to start some-unwanted.service: Unit some-unwanted.service is masked.
# mask actually replaces the unit file reference with a symlink to
# /dev/null, making it impossible to start EVEN manually until
# explicitly unmasked — the correct tool when you need to guarantee a
# unit can never run, not just that it won't auto-start.

sudo systemctl unmask some-unwanted.service   # reverses it
```

---

## Active (exited) Doesn't Mean the Service Crashed

### A "Type=oneshot" service finishing successfully looks alarming if you don't know to expect it
```bash
systemctl status some-setup-task
# ● some-setup-task.service - One-time setup task
#      Loaded: loaded
#      Active: active (exited) since Mon 2026-07-27 09:00:00 UTC; 2h ago
# ⚠️ "active (exited)" is the NORMAL, expected final state for a
# oneshot-type unit that runs once, completes its task, and exits
# successfully — it is NOT an error state, even though "exited" can
# sound alarming out of context. Check the actual exit code and the
# "Loaded"/"Active" line together before assuming something went wrong:
systemctl show some-setup-task --property=ExecMainStatus
# ExecMainStatus=0     ← 0 means it exited cleanly
```

---

## restart Can Fail Silently on a Unit That Was Never Running

### Depending on the unit type, "restart" on a never-started unit isn't always a no-op you'd expect
```bash
sudo systemctl restart some-unit-that-never-ran
systemctl status some-unit-that-never-ran
# ⚠️ Depending on the specific service's ExecStart/ExecStop behavior
# and any conditional directives (ConditionPathExists=, etc.), a
# restart on a unit that was never active can behave differently than
# expected — some units start cleanly, others may fail a Condition
# check silently and simply not run, without a loud error.

# When in doubt about whether a unit actually did anything, check its
# state and log explicitly rather than assuming restart always "just works":
systemctl is-active some-unit-that-never-ran
journalctl -u some-unit-that-never-ran -n 20
```

---

## Editing /usr/lib/systemd/system/ Files Gets Overwritten on Package Updates

### The distro-provided unit file location isn't meant for local customization
```bash
sudo nano /usr/lib/systemd/system/nginx.service
# ⚠️ This file is OWNED by the nginx package — the next time the
# package is updated (apt upgrade, dnf update, etc.), this file will
# likely be OVERWRITTEN, silently discarding your changes.

# The correct approach for customization: a drop-in override, which
# systemd merges ON TOP of the original file, surviving package updates
sudo systemctl edit nginx.service
# creates/edits /etc/systemd/system/nginx.service.d/override.conf
# Only the specific directives you add here override the original —
# everything else from the package-provided file still applies.
```

---

## A Service Can Be enabled But Still Not Start — Check Its Dependencies

### Enabled just means "wanted at boot" — it doesn't guarantee its prerequisites are met
```bash
systemctl is-enabled myapp
# enabled

systemctl status myapp
# ● myapp.service
#      Active: inactive (dead)
# ⚠️ A service can be fully enabled and still fail to come up at boot
# if a dependency it requires (a database it connects to, a network
# target, a mount point specified in `After=`/`Requires=`) wasn't
# ready in time or failed itself. "Enabled" only describes systemd's
# INTENT to start it — actual success also depends on everything the
# unit declares as a dependency.

# Inspect what a unit actually depends on:
systemctl list-dependencies myapp
```

---

## status Only Shows a Few Recent Log Lines — Not the Full History

### Relying on `systemctl status` alone during a real investigation is often insufficient
```bash
systemctl status myapp
# ... only the last ~10 log lines appear at the bottom of this output ...
# ⚠️ For anything beyond a quick glance, this preview is easy to
# mistake for "the whole story" — a root-cause error message from
# hours ago has almost certainly already scrolled past this small window.

# For a genuine investigation, go to the full log directly:
journalctl -u myapp --since "2 hours ago"
journalctl -u myapp -b        # since the current boot
journalctl -u myapp -f        # live-follow going forward
```

---

## Restart=on-failure Can Mask a Recurring Crash Loop

### A service that keeps crashing and auto-restarting can look "fine" at a glance
```bash
systemctl status myapp
# ● myapp.service
#      Active: active (running) since 3s ago
# ⚠️ "active (running)" looks healthy in a single snapshot — but if
# this service has Restart=on-failure and has actually been crash-
# looping every few seconds for the last hour, a single status check
# taken at a random moment will simply catch it mid-restart, looking
# perfectly normal, and hide the underlying instability entirely.

# Check the actual restart count / start pattern before trusting a
# single "active (running)" snapshot at face value:
systemctl show myapp --property=NRestarts
journalctl -u myapp --since "1 hour ago" | grep -c "Started myapp"
```

---

## Not Every Distro Uses systemd — systemctl Simply Won't Exist There

### A frequent source of confusion for anyone assuming systemd is universal
```bash
systemctl status ssh
# bash: systemctl: command not found
# ⚠️ Alpine Linux (default OpenRC), Devuan, older Debian releases with
# sysvinit, and some minimal/embedded distributions intentionally don't
# ship systemd at all — systemctl simply doesn't exist there, and this
# isn't a misconfiguration to "fix," just a genuinely different init
# system requiring its own separate tooling (`rc-service`, `service`,
# direct init scripts, depending on the system).
```

---

## Static Units Can't Be Enabled or Disabled — That's Expected

### is-enabled reporting "static" isn't an error condition
```bash
systemctl is-enabled some-dependency.service
# static
# ⚠️ "static" means this unit has no [Install] section at all — it
# was never designed to be independently enabled/disabled by an admin;
# it's typically pulled in automatically as a dependency of some other
# unit whenever needed. Attempting `systemctl enable` on a static unit
# will report that enabling has no effect, which is the CORRECT,
# expected behavior, not a bug or a sign of a broken unit file.
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
