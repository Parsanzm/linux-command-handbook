# sudo — Edge Cases & Gotchas

> sudo mistakes range from "annoying" to "you just took down the whole server."
> A single misplaced rule in sudoers can grant far more access than intended.

---

## Table of Contents

- [Redirection Runs Outside sudo](#redirection-runs-outside-sudo)
- [Environment Variables Are Stripped by Default](#environment-variables-are-stripped-by-default)
- [A Broken sudoers File Can Lock You Out](#a-broken-sudoers-file-can-lock-you-out)
- [NOPASSWD Wildcard Traps](#nopasswd-wildcard-traps)
- [sudo Doesn't Protect Against Shell Escapes](#sudo-doesnt-protect-against-shell-escapes)
- ["sudo su" Isn't the Same as "sudo -i"](#sudo-su-isnt-the-same-as-sudo--i)
- [Rule Order Matters — Last Match Wins](#rule-order-matters--last-match-wins)
- [Path Manipulation with Relative Commands](#path-manipulation-with-relative-commands)
- [Timestamp Caching Is Per-TTY, Not Global](#timestamp-caching-is-per-tty-not-global)
- [Editing Scripts Between Check and Use (TOCTOU)](#editing-scripts-between-check-and-use-toctou)
- [sudo Inside Docker / Containers](#sudo-inside-docker--containers)
- [Group Membership Requires a New Login](#group-membership-requires-a-new-login)
- ["!" Negation in sudoers Isn't a Real Security Boundary](#-negation-in-sudoers-isnt-a-real-security-boundary)

---

## Redirection Runs Outside sudo

### `sudo command > file` writes as YOU, not as root
```bash
sudo echo "127.0.0.1 myhost" > /etc/hosts
# ❌ bash: /etc/hosts: Permission denied
# "echo" ran as root, but the SHELL performs the ">" redirection,
# and the shell itself is still running as your normal user.

# ✅ Fix: pipe into a root-owned process that does the writing
echo "127.0.0.1 myhost" | sudo tee -a /etc/hosts

# ✅ Or wrap everything inside a root shell
sudo bash -c 'echo "127.0.0.1 myhost" > /etc/hosts'
```

---

## Environment Variables Are Stripped by Default

### Scripts relying on your shell's env vars silently break
```bash
export API_KEY=secret123
sudo ./deploy.sh
# deploy.sh reads $API_KEY expecting "secret123" — gets nothing!
# sudo resets the environment by default (env_reset in sudoers) for security.

# Fix: explicitly preserve what's needed
sudo -E ./deploy.sh
sudo --preserve-env=API_KEY ./deploy.sh

# Verify what actually gets through:
sudo env | grep API_KEY     # empty by default
sudo -E env | grep API_KEY  # shows it, if -E was used
```

---

## A Broken sudoers File Can Lock You Out

### Editing /etc/sudoers directly (without visudo) is dangerous
```bash
sudo nano /etc/sudoers    # ⚠️ risky — no syntax check before saving
# Typo introduced, file saved
sudo whoami
# sudo: /etc/sudoers: syntax error near line 28
# sudo: no valid sudoers sources found, quitting
# ⚠️ NOW NOBODY, including you, can use sudo — potentially locking you out entirely

# visudo prevents this by validating syntax BEFORE the file is actually saved:
sudo visudo
# If there's a syntax error, visudo refuses to save and asks you to fix it
# or discard changes — the live sudoers file is never left in a broken state.

# If you're already locked out: boot into single-user/recovery mode,
# or use a root shell via another path (cloud provider console, live USB)
# to fix /etc/sudoers directly.
```

---

## NOPASSWD Wildcard Traps

### A "restrictive-looking" rule can grant far more than intended
```
# Intended: only allow restarting the "myapp" service
deploy  ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp

# Looks safe, but consider this variant:
deploy  ALL=(root) NOPASSWD: /usr/bin/systemctl *
# ⚠️ The wildcard also matches:
#   systemctl restart sshd
#   systemctl stop firewalld
#   systemctl restart anything-at-all
# sudoers wildcards match on the COMMAND LINE STRING, not "just the service name"

# Even worse — some commands accept arguments that spawn a shell:
deploy  ALL=(root) NOPASSWD: /usr/bin/less /var/log/app.log
# Inside `less`, typing "!bash" spawns an interactive root shell:
#   less escapes to shell → full root access, far beyond "read a log file"

# Safer: use --restrict-like options where available, or wrap in a
# dedicated script that takes no arguments and does exactly one thing:
deploy  ALL=(root) NOPASSWD: /opt/scripts/restart_myapp_only.sh
```

---

## sudo Doesn't Protect Against Shell Escapes

### Any sudo-granted program with a shell escape = full root
```bash
# Classic example: granting sudo access to an editor
alice  ALL=(root) NOPASSWD: /usr/bin/vim /etc/app/config.conf

# Inside vim, alice can run:
:!bash
# → spawns a ROOT shell, completely bypassing the "just edit this one file" intent

# Similarly dangerous programs to grant unrestricted sudo on:
# vim, vi, nano (with certain plugins), less, more, man, find (-exec), awk, python, perl

# GTFOBins (gtfobins.github.io) catalogs which common binaries can be
# abused this way when granted sudo access — worth checking before
# writing any sudoers rule for a general-purpose tool.

# Safer pattern: use sudoedit instead of granting sudo on the editor directly
alice  ALL=(root) NOPASSWD: sudoedit /etc/app/config.conf
```

---

## "sudo su" Isn't the Same as "sudo -i"

```bash
sudo su -
# Works, but stacks two privilege mechanisms:
# sudo elevates you to root, THEN su (running as root) switches "again"
# Slightly redundant and can behave oddly with some PAM/session configs.

sudo -i
# The idiomatic way to get a full root login shell — sudo alone handles it,
# no need to invoke su at all.

sudo -s
# Root shell, but keeps more of your current environment/cwd than -i does
# (doesn't simulate a fresh login)
```

---

## Rule Order Matters — Last Match Wins

### A broad ALLOW followed by a narrow DENY doesn't behave as some expect
```
alice  ALL=(ALL) ALL
alice  ALL=(ALL) !/usr/bin/passwd root
```
This actually works as intended — the later, more specific `!` rule wins for `passwd root`. But sudoers evaluates rules **in order, last match wins**, so accidentally reordering these:
```
alice  ALL=(ALL) !/usr/bin/passwd root
alice  ALL=(ALL) ALL
```
...silently **re-grants** the "denied" command, because the broad `ALL` rule now comes last and overrides the earlier restriction. Negation-based restrictions are fragile — an easy mistake to introduce during a later edit.

---

## Path Manipulation with Relative Commands

### sudoers rules referencing a bare command name (not absolute path) can be hijacked
```
# Bad sudoers entry (rare, but seen in older configs):
alice  ALL=(root) NOPASSWD: python

# If alice's PATH has a directory she can write to BEFORE the real python:
export PATH=/tmp/evil:$PATH
# /tmp/evil/python is a malicious script alice created
sudo python     # ⚠️ may run the ATTACKER's script as root, not /usr/bin/python

# This is why sudoers entries should always use full, absolute paths:
alice  ALL=(root) NOPASSWD: /usr/bin/python3
# sudo also resets PATH via secure_path by default specifically to reduce this risk,
# but always specifying the absolute path in the rule removes the ambiguity entirely.
```

---

## Timestamp Caching Is Per-TTY, Not Global

```bash
# Terminal 1:
sudo whoami     # prompts for password, then cached

# Terminal 2 (same user, same machine, opened moments later):
sudo whoami     # ⚠️ prompts AGAIN — the cached timestamp is typically tied
                # to the specific terminal/session, not the user globally

# This is intentional (limits blast radius if one terminal is compromised),
# but frequently surprises people expecting one password entry to cover
# every open terminal.
```

---

## Editing Scripts Between Check and Use (TOCTOU)

### A NOPASSWD script rule is only as safe as the script's own permissions
```
deploy  ALL=(root) NOPASSWD: /opt/scripts/deploy.sh
```
```bash
# If /opt/scripts/deploy.sh is writable by "deploy" (or any non-root user):
ls -l /opt/scripts/deploy.sh
# -rwxrwxr-x ... deploy deploy   ⚠️ group-writable by the same user granted sudo!

# The "deploy" user can simply REWRITE deploy.sh to do anything, then:
sudo /opt/scripts/deploy.sh
# ...runs the modified script AS ROOT — the sudoers rule trusted the path,
# not the content, and the content was never actually protected.

# Fix: the script (and every directory in its path) must be owned by root
# and NOT writable by the user granted sudo access to it.
sudo chown root:root /opt/scripts/deploy.sh
sudo chmod 755 /opt/scripts/deploy.sh   # deploy user can execute, not modify
```

---

## sudo Inside Docker / Containers

### sudo often "does nothing useful" as PID 1 in a minimal container
```dockerfile
# Many minimal container images don't even include sudo, or run everything as root already
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y sudo   # extra weight, often unnecessary
```
```bash
# Inside a container already running as root:
whoami
# root
sudo whoami
# sudo: /var/lib/sudo: No such file or directory (or similar)
# sudo may misbehave without a proper environment (no PAM stack, no valid /etc/passwd setup)

# In containers, prefer USER directives / gosu / su-exec to drop privileges
# instead of installing sudo just to escalate them (you often already ARE root).
```

---

## Group Membership Requires a New Login

### Adding a user to a sudo-capable group doesn't take effect immediately
```bash
sudo usermod -aG sudo alice
# alice, still in her CURRENT shell session:
sudo whoami
# sudo: alice is not in the sudoers file. This incident will be reported.
# ⚠️ Group membership is evaluated at LOGIN time, not live

# Fix: alice must start a new session (log out/in, new SSH connection),
# or temporarily refresh group membership in the current shell:
newgrp sudo
# (newgrp only affects that specific shell, not other open sessions)
```

---

## "!" Negation in sudoers Isn't a Real Security Boundary

### Denying a specific command doesn't stop equivalent alternate paths
```
alice  ALL=(ALL) ALL, !/bin/rm

# alice literally cannot run:
sudo rm file.txt          # ❌ denied

# But nothing stops equivalent actions through the broad ALL grant:
sudo bash -c 'rm file.txt'     # ✅ works — rm invoked via bash, not directly
sudo python3 -c "import os; os.remove('file.txt')"   # ✅ also works
sudo find . -delete            # ✅ also works

# Negated rules in sudoers are easily bypassed when a broad ALL grant
# still exists alongside them. Real restriction requires listing ONLY
# the specific commands allowed — never "ALL minus a few exceptions."
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
