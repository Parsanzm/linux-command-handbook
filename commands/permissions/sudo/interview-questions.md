# sudo — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [sudoers Configuration](#sudoers-configuration)
- [sudo vs su](#sudo-vs-su)
- [Security](#security)
- [Environment & Scripting](#environment--scripting)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does sudo stand for, and what problem does it solve?**
> Commonly read as "superuser do" (more precisely "substitute user do"). It lets an authorized user run commands as another user — typically root — using their **own** password rather than the target user's. It solves the problem of sharing the root password among administrators: instead, each person gets logged, revocable, and often granular access.

---

**Q2 🔥 Why must the sudo binary itself have the setuid bit set?**
> Only root can change a running process's effective UID. `sudo` needs to start with root privileges in order to have the authority to eventually execute a command as root (or any other target user). It achieves this because its own executable is owned by root with the setuid bit set — so it runs with root's privileges internally, checks the sudoers policy, and then either proceeds or refuses.

---

**Q3. What file(s) control sudo's authorization rules?**
> `/etc/sudoers` is the main file, plus any drop-in files under `/etc/sudoers.d/`. Both should only ever be edited with `visudo` (or `visudo -f` for a specific file), which validates syntax before saving to prevent a broken sudoers file from locking everyone out.

---

**Q4 🔥 What role does PAM play in sudo?**
> PAM (Pluggable Authentication Modules) handles the actual authentication step — verifying your password, or integrating with alternatives like fingerprint readers, LDAP, Kerberos, or hardware security keys. sudo delegates that part to PAM rather than implementing its own password-checking logic, which is why sudo can transparently support many different authentication backends.

---

## Basic Usage

**Q5. What's the difference between `sudo -i` and `sudo -s`?**
> `sudo -i` simulates a full initial login as the target user — it loads their environment, runs their shell's login scripts, and changes to their home directory. `sudo -s` starts a shell as the target user but preserves more of your **current** shell's environment and working directory, without simulating a fresh login.

---

**Q6 🔥 How do you check what commands your current user is permitted to run with sudo?**
> ```bash
> sudo -l
> ```
> This lists the sudoers rules that apply to you, including any `NOPASSWD` entries, without actually executing anything.

---

**Q7. What does `sudo -u USER command` do?**
> Runs `command` as `USER` instead of the default target (root). Useful for tasks that should run as a service account, e.g., `sudo -u postgres psql` or `sudo -u www-data php script.php`.

---

## sudoers Configuration

**Q8 🔥 Explain the syntax of a sudoers rule: `alice ALL=(ALL:ALL) ALL`.**
> - `alice` — the user the rule applies to
> - First `ALL` — the hosts this rule applies to (useful when one sudoers file is shared across machines)
> - `(ALL:ALL)` — the user and group alice is allowed to run commands as
> - Last `ALL` — the commands allowed (in this case, everything)
>
> This grants alice full sudo access to run any command as any user/group, on any host covered by this file.

---

**Q9 🔥 What does NOPASSWD do, and when would you use it?**
> `NOPASSWD:` tells sudo not to prompt for a password for that specific rule. It's typically used for automation (CI/CD pipelines, cron jobs, deploy scripts) where an interactive password prompt isn't possible, scoped to the *minimum specific command* needed — never applied broadly to `ALL` commands for a user, which would defeat the purpose of password protection entirely.

---

**Q10. Why should you always use `visudo` instead of editing `/etc/sudoers` directly?**
> `visudo` locks the file during editing and, critically, **validates the syntax before saving**. A raw editor lets you save a syntactically broken sudoers file, which can leave sudo entirely non-functional for every user on the system — potentially locking everyone (including yourself) out of privilege escalation until it's fixed via another access path (recovery mode, cloud console, etc.).

---

**Q11 🔥 What permissions should files in `/etc/sudoers.d/` have, and why?**
> `440` (read-only for owner and group, root-owned) — sudo actively refuses to load sudoers drop-in files that are group- or world-writable, as a safeguard against a non-root user tampering with sudo's own policy files.

---

## sudo vs su

**Q12 🔥 What's the key difference between sudo and su?**
> `su` switches to another user's full session and requires **that user's own password** (typically root's). `sudo` runs a single command with elevated privileges using **your own password**, is logged per-command, and can be scoped very granularly via sudoers — without ever needing to know or share root's actual password.

---

**Q13. When might you still prefer su over sudo?**
> When you need a genuinely full, standing session as another user for extended work, already have that user's password, and don't need per-command granularity or detailed audit logging — e.g., some minimal/embedded systems that don't ship a configured sudo setup at all.

---

## Security

**Q14 🔥 Why is granting sudo access to an interactive program like vim or less potentially dangerous?**
> Many interactive programs support a "shell escape" (e.g., `:!bash` in vim, or `!bash` inside `less`), which spawns a full interactive shell running with whatever privileges the parent process has. If that program was invoked via a sudo rule intended to restrict access to "just editing/viewing one file," the shell escape effectively grants full, unrestricted root access — far beyond the rule's intent. Tools like GTFOBins catalog which binaries are exploitable this way.

---

**Q15. Why does `sudo command > /etc/protected_file` often fail with "Permission denied" even though `command` ran as root?**
> Because the `>` redirection is performed by the **shell**, not by `sudo` or the command itself — and the shell is still running as the original, unprivileged user. Only the command between `sudo` and the redirect operator actually runs as root. The fix is to pipe into `sudo tee`, or wrap the entire redirect inside `sudo bash -c '...'` so the write itself happens with root privileges.

---

**Q16 🔥 What's a TOCTOU-style risk with a sudoers rule that grants NOPASSWD access to a script by path?**
> If the script (or any directory in its path) is writable by the same user granted sudo access to run it, that user can simply rewrite the script's contents, then run it via the trusted sudo rule — sudo only verifies the *path*, not the *content* at execution time. The fix is ensuring the script and its containing directories are owned by root and not writable by the grantee.

---

**Q17. Why is `alice ALL=(root) NOPASSWD: /usr/bin/systemctl *` more dangerous than it looks?**
> The wildcard matches the entire remainder of the command line, so it doesn't just permit "restart a specific service" — it permits `systemctl stop <anything>`, `systemctl restart <anything>`, and any other systemctl subcommand/target combination. A rule intended to be narrow ends up granting broad control over every systemd-managed unit on the system.

---

**Q18. Can negating a command in sudoers (`!/bin/rm`) reliably prevent a user with `ALL` access from deleting files?**
> No. If the user still has a broad `ALL` grant alongside the negation, they can trivially achieve the same result through an alternate path — `sudo bash -c 'rm file'`, a scripting language's file-removal function, `find -delete`, etc. Negated exceptions in sudoers are not a real security boundary when a broad grant coexists with them; true restriction means listing only the specific allowed commands, not "everything except X."

---

## Environment & Scripting

**Q19 🔥 Does sudo preserve your shell's environment variables by default? Why or why not?**
> No — by default sudo resets most environment variables (`env_reset` in sudoers) as a security measure, preventing a user from manipulating things like `PATH`, `LD_PRELOAD`, or other variables to influence how the elevated command behaves. Use `sudo -E` to preserve the full environment, or `sudo --preserve-env=VAR1,VAR2` to preserve only specific variables.

---

**Q20. How do you make a script fail immediately instead of hanging on a sudo password prompt (for use in automation)?**
> ```bash
> sudo -n true && echo "cached credential available" || echo "would prompt"
> ```
> `-n` (`--non-interactive`) makes sudo fail right away rather than displaying a password prompt, which is essential in non-interactive automation contexts like CI pipelines where there's no terminal to type into.

---

**Q21. How would you keep a sudo credential "alive" during a long-running script that needs root access partway through?**
> Refresh the timestamp proactively in the background:
> ```bash
> sudo -v
> while true; do sudo -n true; sleep 60; kill -0 "$$" || exit; done 2>/dev/null &
> ```
> This periodically re-validates the cached credential so it doesn't expire mid-script, without requiring another interactive password prompt partway through execution.

---

## Scenario-Based

**Q22 🔥 A junior admin ran `sudo nano /etc/sudoers`, saved a typo, and now `sudo` fails system-wide for everyone. How did this happen, and how do you prevent it?**
> Editing `/etc/sudoers` with a plain editor bypasses syntax validation — a saved typo leaves sudo unable to parse its own policy file, breaking sudo for all users immediately. Prevention: always use `visudo` (or `visudo -f` for drop-in files), which validates syntax before committing the change and refuses to save if there's an error. Recovery from an already-broken file typically requires booting into recovery/single-user mode or another root-access path (e.g., cloud provider's console) to fix `/etc/sudoers` directly.

---

**Q23. You add a user to the `sudo` group with `usermod -aG sudo alice`, but she immediately tries `sudo whoami` in her current terminal and gets denied. Why, and what's the fix?**
> Group membership changes are evaluated at login/session start, not applied live to already-open sessions. Alice's current shell still reflects her old group list. Fix: she needs to start a new session (log out and back in, or open a new SSH connection), or run `newgrp sudo` to refresh group membership within that specific shell only.

---

**Q24 🔥 You want to grant a CI/CD user the ability to restart exactly one systemd service without a password, and nothing else. Write the sudoers rule and explain your choices.**
> ```
> ciuser  ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp.service
> ```
> - Absolute path to `systemctl`, not a bare command name, to avoid `PATH`-hijacking risks.
> - The exact command and argument, not a wildcard (`systemctl *`), to avoid granting control over unrelated services.
> - `NOPASSWD` scoped to this one rule only, so the CI pipeline can run non-interactively without granting broad passwordless access elsewhere.
> - The file should be placed in `/etc/sudoers.d/` with `440` permissions and edited via `visudo -f`.

---

**Q25. Why does `sudo -u www-data whoami` sometimes still show unexpected file ownership issues when a web app writes files, even though the command itself ran successfully as www-data?**
> `sudo -u` changes the effective UID for that single command invocation, but it doesn't change the surrounding shell's environment or any subsequently-spawned processes outside that command — for instance, `$HOME`, working directory, or umask may still reflect the original user unless `-i` (full login simulation) or explicit flags like `-H` are used. Mismatched `$HOME` or umask during file creation is a common source of unexpected ownership/permission results even when the correct user ran the command.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
