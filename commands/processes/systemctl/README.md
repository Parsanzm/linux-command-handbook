# systemctl — The Complete Reference

> **Control and inspect the systemd system and service manager**
> The primary interface to systemd — starting, stopping, enabling, and
> diagnosing everything from background services to mount points.

---

## Table of Contents

- [What is systemctl?](#what-is-systemctl)
- [Where does systemctl live?](#where-does-systemctl-live)
- [How systemctl works internally](#how-systemctl-works-internally)
- [Syntax](#syntax)
- [Understanding Unit States](#understanding-unit-states)
- [Unit Types, Not Just Services](#unit-types-not-just-services)
- [Enabled vs Active — Two Independent Concepts](#enabled-vs-active--two-independent-concepts)
- [All Key Subcommands](#all-key-subcommands)
- [systemctl and Unit Files](#systemctl-and-unit-files)
- [systemctl vs service vs journalctl](#systemctl-vs-service-vs-journalctl)
- [Related Commands](#related-commands)

---

## What is systemctl?

`systemctl` is the command-line tool for controlling **systemd**, the init system and service manager used by most modern Linux distributions (Ubuntu, Debian, Fedora, RHEL/CentOS, Arch, and others). It starts and stops services, enables or disables them at boot, reports their current status, and reloads or manages the systemd manager itself.

```bash
systemctl status nginx
# ● nginx.service - A high performance web server
#      Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled)
#      Active: active (running) since Mon 2026-07-27 09:00:00 UTC; 2h ago
#    Main PID: 1234 (nginx)
```

**Why systemctl matters so much day-to-day:** it's the single tool for nearly every service-lifecycle question — is this running, will it start on boot, why did it fail, what does it actually depend on — replacing the older, more fragmented `service`/`chkconfig`/init-script world.

---

## Where does systemctl live?

```
/usr/bin/systemctl  (or /bin/systemctl on some systems)
```

```bash
which systemctl
systemctl --version
# systemd 255 (255.4-1ubuntu8)
```

Part of the **systemd** project. Present on the vast majority of modern Linux distributions; **not** present on systems still using other init systems (older Debian with sysvinit, Alpine's default OpenRC, Devuan, etc.) — `systemctl status` simply won't exist there.

---

## How systemctl works internally

`systemctl` is a client that talks to **PID 1** — the systemd process itself — over D-Bus. It doesn't manage processes directly; it sends requests to the systemd manager, which does the actual work (forking processes, tracking cgroups, applying dependency ordering) and reports results back.

```bash
# systemctl is essentially a D-Bus client for the org.freedesktop.systemd1 service
busctl status org.freedesktop.systemd1
```

Systemd organizes everything it manages into **units** — services, mount points, timers, sockets, devices, and more — each described by a plain-text **unit file**. `systemctl` reads and writes references to these files, and asks the running systemd manager to act on the units they describe.

```bash
# The manager's own view of a unit is queryable directly, bypassing
# the human-friendly `status` formatting:
systemctl show nginx.service --property=ActiveState,SubState,MainPID
```

---

## Syntax

```bash
systemctl [OPTIONS] COMMAND [UNIT...]
```

Most day-to-day usage takes the form `systemctl <verb> <unit-name>`:

```bash
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl status nginx
```

---

## Understanding Unit States

`systemctl status` and `systemctl list-units` report state through a few related fields:

| Field | Meaning |
|---|---|
| `Loaded` | Whether the unit file was found and parsed successfully (`loaded`, `not-found`, `masked`) |
| `Active` | The unit's current runtime state (`active`, `inactive`, `failed`, `activating`, `deactivating`) |
| `Sub` | A more granular sub-state within the active state (e.g., `running`, `exited`, `dead`, `waiting`) |

```bash
systemctl is-active nginx     # active / inactive / failed / unknown
systemctl is-enabled nginx    # enabled / disabled / static / masked
systemctl is-failed nginx     # exit status 0 if failed, useful in scripts
```

A unit can be `active (running)` for a long-lived daemon, `active (exited)` for a one-shot task that ran once and finished successfully (this is normal, not an error), or `failed` when the process exited with an error and no automatic restart resolved it.

---

## Unit Types, Not Just Services

`systemctl` manages far more than just background services — "unit" is the general term, distinguished by file extension:

| Unit type | Extension | Purpose |
|---|---|---|
| Service | `.service` | A managed background process (the most common type) |
| Socket | `.socket` | A network or IPC socket, often used for on-demand service activation |
| Timer | `.timer` | Scheduled activation of another unit — systemd's modern alternative to cron |
| Mount | `.mount` | A filesystem mount point |
| Target | `.target` | A grouping/synchronization point for other units (e.g., `multi-user.target`) |
| Path | `.path` | Activates another unit when a watched file/directory changes |
| Device | `.device` | Represents a kernel-exposed device |

```bash
systemctl list-units --type=timer     # see all active timers (cron alternative)
systemctl list-units --type=service   # see only services
```

---

## Enabled vs Active — Two Independent Concepts

These two questions are answered completely separately, and conflating them is the single most common `systemctl` misunderstanding:

```bash
systemctl is-enabled nginx
# enabled     ← WILL this start automatically at next boot?

systemctl is-active nginx
# active      ← IS this running right now, at this exact moment?
```

A service can be any combination of the two:

| Enabled | Active | Meaning |
|---|---|---|
| enabled | active | Normal running service that also starts on boot |
| enabled | inactive | Will start on next boot, but isn't running right now (maybe stopped manually) |
| disabled | active | Running right now, but won't survive a reboot unless started again manually |
| disabled | inactive | Fully off, and won't come back on its own |

```bash
systemctl enable nginx     # ONLY changes boot behavior — does NOT start it now
systemctl start nginx      # ONLY starts it now — does NOT affect boot behavior
systemctl enable --now nginx   # does BOTH in one command
```

---

## All Key Subcommands

| Subcommand | Description |
|---|---|
| `start UNIT` | Start a unit now |
| `stop UNIT` | Stop a unit now |
| `restart UNIT` | Stop then start a unit |
| `reload UNIT` | Ask a running service to reload its configuration without a full restart (if supported) |
| `status UNIT` | Show current state, recent log lines, and key metadata |
| `enable UNIT` | Configure a unit to start automatically at boot (creates a symlink) |
| `disable UNIT` | Remove a unit's automatic boot startup |
| `enable --now UNIT` | Enable AND start in one command |
| `is-active UNIT` | Print/return whether a unit is currently active |
| `is-enabled UNIT` | Print/return whether a unit is enabled for boot |
| `is-failed UNIT` | Print/return whether a unit is in a failed state |
| `mask UNIT` | Make a unit completely unstartable, even manually, by linking it to `/dev/null` |
| `unmask UNIT` | Reverse a mask |
| `daemon-reload` | Re-read all unit files after editing/adding one — required after manual changes |
| `list-units` | List currently loaded units and their states |
| `list-unit-files` | List all installed unit files and their enabled/disabled state |
| `cat UNIT` | Print the full contents of a unit file |
| `edit UNIT` | Open an override file for a unit in `$EDITOR` |
| `show UNIT` | Print all low-level properties systemd tracks for a unit |

---

## systemctl and Unit Files

```bash
# View a unit's actual configuration file
systemctl cat nginx
# [Unit]
# Description=A high performance web server
# After=network.target
#
# [Service]
# ExecStart=/usr/sbin/nginx -g 'daemon on; master_process on;'
# ExecReload=/usr/sbin/nginx -s reload
# Restart=on-failure
#
# [Install]
# WantedBy=multi-user.target
```

Unit files typically live in `/usr/lib/systemd/system/` (distro-provided) or `/etc/systemd/system/` (local overrides/custom units, which take precedence). After manually creating or editing a unit file directly on disk, systemd's in-memory view is **not** automatically updated:

```bash
sudo systemctl daemon-reload
# Required any time a unit file is added/changed by hand, so systemd
# re-parses it before you try to start/enable it.
```

---

## systemctl vs service vs journalctl

| Tool | Best for | Key difference from systemctl |
|---|---|---|
| `systemctl` | Full unit lifecycle control (start/stop/enable/status) on systemd systems | The comprehensive, modern tool |
| `service NAME start` | Legacy compatibility wrapper | Still works on many systemd distros as a thin shim, but exposes far less information and control than systemctl directly |
| `journalctl` | Reading a service's logs in detail | `systemctl status` shows only the last few log lines as a preview; `journalctl -u NAME` shows the full log history |

```bash
systemctl status nginx      # quick state + a few recent log lines
journalctl -u nginx -f      # full, followable log stream for that unit
journalctl -u nginx --since "1 hour ago"
```

---

## Related Commands

| Command | Relation |
|---|---|
| `journalctl` | View the full logs for any systemd-managed unit |
| `systemd-analyze` | Analyze boot time and unit dependency ordering |
| `loginctl` | Manage user sessions/logins (systemd's session manager) |
| `timedatectl` | Manage system time/date/timezone (systemd's time tool) |
| `hostnamectl` | Manage the system hostname (systemd's hostname tool) |
| `crontab` | Older scheduling mechanism now largely supplemented by `.timer` units |
| `service` | Legacy wrapper command, still present for compatibility on many distros |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
