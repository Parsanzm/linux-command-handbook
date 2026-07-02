# sudo — The Complete Reference

> **SuperUser DO: run a command as another user, most commonly root**
> Created in 1980 at SUNY Buffalo to give selective root access without sharing the root password.
> The standard mechanism for privilege escalation on virtually every modern Linux distro.

---

## Table of Contents

- [What is sudo?](#what-is-sudo)
- [Where does sudo live?](#where-does-sudo-live)
- [How sudo works internally](#how-sudo-works-internally)
- [Syntax](#syntax)
- [The sudoers File](#the-sudoers-file)
- [All Key Options](#all-key-options)
- [sudo vs su vs doas](#sudo-vs-su-vs-doas)
- [Password Caching / Timestamps](#password-caching--timestamps)
- [Logging & Auditing](#logging--auditing)
- [Related Commands](#related-commands)

---

## What is sudo?

`sudo` stands for **"superuser do"** (though it more generally means "substitute user do," since it can run commands as *any* user, not just root). It lets an authorized user run a command with another user's privileges — typically root — without knowing that user's password, using **their own** password instead (or no password, if configured).

**Why sudo exists instead of just sharing the root password:**
1. **Granular control** — grant specific users specific commands, not full root access
2. **Accountability** — every sudo invocation is logged with who ran what, and when
3. **No shared secrets** — nobody needs to know the actual root password
4. **Revocable** — remove a user's sudo rights instantly without changing the root password for everyone
5. **Session-based** — privilege is temporary, scoped to the command (or a timed window), not a standing login

```bash
sudo apt update              # run one command as root
sudo -u alice whoami         # run as a specific user (not root)
sudo -i                      # get an interactive root shell
```

---

## Where does sudo live?

```
/usr/bin/sudo
```

```bash
which sudo
sudo --version
# Sudo version 1.9.15p5
```

Configuration lives in `/etc/sudoers` and `/etc/sudoers.d/`. The `sudo` binary itself is **setuid root** — this is *why* it's able to grant root privileges in the first place.

```bash
ls -l /usr/bin/sudo
# -rwsr-xr-x 1 root root ... /usr/bin/sudo
#     ^ setuid bit — sudo runs as root internally, then decides whether to
#       let YOUR command proceed based on the sudoers policy
```

---

## How sudo works internally

### The privilege escalation flow

1. You run `sudo <command>`.
2. `sudo` (itself setuid root) checks `/etc/sudoers` (and `/etc/sudoers.d/*`) to see if **your user or group** is permitted to run that command as the target user.
3. If allowed, `sudo` prompts for **your own password** (not root's) — unless `NOPASSWD` is configured, or a valid cached timestamp already exists.
4. On success, `sudo` forks a new process with the effective UID set to the target user (root by default) and executes the command.
5. The action is logged (typically to `/var/log/auth.log` or via `syslog`/`journald`).

### Why sudo needs to be setuid root

Only root can change a process's UID. `sudo` itself starts running with root privileges (because its executable has the setuid bit), evaluates the sudoers policy, and only *then* decides to either drop to the target UID and exec the command, or refuse and exit. Removing sudo's setuid bit breaks it entirely.

### PAM integration

Password verification, timestamp/session handling, and MFA (like `pam_google_authenticator`) go through **PAM** (Pluggable Authentication Modules) — this is why sudo can integrate with fingerprint readers, LDAP, Kerberos, or hardware keys without sudo itself knowing the details.

---

## Syntax

```bash
sudo [OPTIONS] COMMAND [ARGS...]
sudo -u USER COMMAND        # run COMMAND as USER instead of root
sudo -g GROUP COMMAND       # run COMMAND with GROUP as the primary group
sudo -i                     # interactive login shell as root
sudo -s                     # shell as root, keeping current environment
sudo -l                     # list what commands YOU are allowed to run
sudo -v                     # refresh/extend your cached credential timestamp
sudo -k                     # invalidate cached credentials immediately
```

---

## The sudoers File

### Location and safe editing
```
/etc/sudoers            ← main file
/etc/sudoers.d/*        ← drop-in files (preferred for custom rules)
```

**Never edit `/etc/sudoers` with a regular editor.** Always use `visudo` — it locks the file, validates syntax before saving, and prevents you from locking yourself out with a typo.

```bash
sudo visudo                          # edit main file safely
sudo visudo -f /etc/sudoers.d/alice  # edit/create a drop-in file safely
```

### Basic sudoers syntax

```
user   host = (runas_user:runas_group)   command_list
```

```
# alice can run any command as any user on any host
alice   ALL=(ALL:ALL)   ALL

# bob can only restart nginx, and only as root — no password required
bob     ALL=(root)   NOPASSWD: /usr/bin/systemctl restart nginx

# The %wheel group (or %sudo on Debian/Ubuntu) can run everything, with password
%sudo   ALL=(ALL:ALL)   ALL
%wheel  ALL=(ALL)       ALL

# deploy user can run only specific deployment scripts as the "www-data" user
deploy  ALL=(www-data) NOPASSWD: /opt/scripts/deploy.sh, /opt/scripts/rollback.sh
```

### Aliases (reusable groups in sudoers)

```
User_Alias    ADMINS = alice, bob, carol
Host_Alias    WEBSERVERS = web1, web2, web3
Cmnd_Alias    SERVICES = /usr/bin/systemctl start *, /usr/bin/systemctl stop *

ADMINS  WEBSERVERS = SERVICES
```

### Common directives

| Directive | Meaning |
|-----------|---------|
| `NOPASSWD:` | Don't prompt for a password for this rule |
| `PASSWD:` | Explicitly require a password (overrides a broader NOPASSWD) |
| `Defaults` | Set global or per-user sudo behavior |
| `!command` | Explicitly deny a command (negation) |
| `%groupname` | Apply the rule to a Unix group instead of one user |

```
Defaults        timestamp_timeout=15      # cached credential lasts 15 minutes
Defaults        logfile="/var/log/sudo.log"
Defaults        requiretty                # require an actual terminal
Defaults:alice  !authenticate             # alice never needs a password at all
```

---

## All Key Options

| Option | Long | Description |
|--------|------|-------------|
| `-u USER` | `--user=USER` | Run command as USER instead of root |
| `-g GROUP` | `--group=GROUP` | Run with GROUP as the primary group |
| `-i` | `--login` | Simulate an initial login as the target user (loads their shell/environment) |
| `-s` | `--shell` | Run a shell as the target user, keeping most of the current environment |
| `-l` | `--list` | List the commands allowed/denied for the invoking user |
| `-v` | `--validate` | Refresh the cached authentication timestamp without running a command |
| `-k` | `--reset-timestamp` | Invalidate the cached timestamp — next sudo will re-prompt |
| `-K` | `--remove-timestamp` | Completely remove the timestamp file |
| `-b` | `--background` | Run command in the background |
| `-e` | `--edit` | Edit a file with the privileges of the target user (safer than `sudo vim file`) |
| `-n` | `--non-interactive` | Fail instead of prompting for a password (useful in scripts) |
| `-E` | `--preserve-env` | Preserve the caller's environment variables |
| `--preserve-env=VAR1,VAR2` | | Preserve only specific environment variables |
| `-A` | `--askpass` | Use a graphical password prompt helper program |
| `-H` | `--set-home` | Set `$HOME` to the target user's home directory |

```bash
sudo -u www-data whoami
sudo -e /etc/nginx/nginx.conf       # edit as root safely (sudoedit)
sudo -n apt update                  # fail immediately if a password would be needed
sudo -E ./build.sh                  # keep your own environment variables
```

---

## sudo vs su vs doas

| Feature | `sudo` | `su` | `doas` |
|---------|--------|------|--------|
| Password used | Your own | Target user's (root's) | Your own (configurable) |
| Per-command granularity | ✅ Yes (sudoers rules) | ❌ No — full shell as target | ✅ Yes (doas.conf rules) |
| Logging | ✅ Detailed, per-command | Minimal | Basic |
| Config file | `/etc/sudoers` (complex) | None | `/etc/doas.conf` (minimal) |
| Common on | Most Linux distros | Traditional Unix / minimal systems | OpenBSD, some minimal Linux setups |
| Timestamp caching | ✅ Yes | ❌ No (full session) | Optional |

```bash
su -                  # become root, must know ROOT's password, full login shell
sudo -i               # become root, uses YOUR password, logged, revocable

doas ls               # doas equivalent (OpenBSD-style, simpler config)
```

**Rule of thumb:** `sudo` for auditable, granular privilege escalation (the modern default); `su` when you specifically need a full session as another user and already have their password; `doas` as a lightweight sudo alternative on minimal systems.

---

## Password Caching / Timestamps

sudo caches successful authentication for a period (default **15 minutes** on most distros) so you're not re-prompted for every command in a short window.

```bash
sudo whoami          # prompts for password
sudo ls              # no prompt — within timestamp window
sleep 900
sudo ls              # prompts again — timestamp expired

sudo -v              # extend the timestamp without running a command
sudo -k              # invalidate immediately — next sudo call re-prompts
sudo -K              # remove timestamp entirely (also clears cached state)
```

The cache is normally tied to the **terminal session** (tty) — a fresh terminal generally requires a fresh password, even if another terminal has an active sudo timestamp.

---

## Logging & Auditing

Every `sudo` invocation (successful or failed) is logged, typically to:
```
/var/log/auth.log        (Debian/Ubuntu)
/var/log/secure          (RHEL/CentOS/Fedora)
journalctl -u sudo       (systemd-based logging)
```

```bash
# View recent sudo activity
sudo grep sudo /var/log/auth.log | tail -20
sudo journalctl _COMM=sudo

# See exactly what you're currently permitted to run
sudo -l

# See what another user can run (requires sufficient privilege)
sudo -l -U bob
```

For stricter environments, `Defaults logfile=` in sudoers writes a dedicated log, and `sudo -e`/`sudoedit` should be preferred over `sudo vim` so file edits go through a safer, auditable path (avoiding running an entire, feature-rich editor as root).

---

## Related Commands

| Command | Relation |
|---------|---------|
| `su` | Switch user entirely, using the target's own password |
| `visudo` | Safely edit `/etc/sudoers` with syntax validation |
| `passwd` | Change a user's password (may itself require sudo/root) |
| `chmod` / `chown` | Often the actual thing being done *via* sudo |
| `doas` | Lightweight sudo alternative, common on OpenBSD |
| `pkexec` | PolicyKit equivalent, common in desktop/GUI contexts |
| `sudoedit` | Alias for `sudo -e`, safer file editing as another user |
| `id` | Check current effective UID/GID (e.g., after `sudo -u`) |
| `whoami` | Quick check of which user a sudo'd shell is running as |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
