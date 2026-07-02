# sudo — Practical Examples

> Real-world patterns from system administration, scripting, DevOps, and daily use.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Running as a Different User](#running-as-a-different-user)
- [Interactive Shells](#interactive-shells)
- [Editing Files Safely](#editing-files-safely)
- [Environment Variables](#environment-variables)
- [sudoers Configuration Examples](#sudoers-configuration-examples)
- [Scripting with sudo](#scripting-with-sudo)
- [Password Caching Control](#password-caching-control)
- [Piping & Redirection Gotchas](#piping--redirection-gotchas)
- [Auditing & Checking Permissions](#auditing--checking-permissions)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Run a single command as root
sudo apt update
sudo systemctl restart nginx

# Re-run the previous command with sudo (bash history trick)
sudo !!

# Check your current sudo privileges
sudo -l

# Run a command and immediately drop back to normal user
sudo whoami
# root

whoami
# your_normal_user
```

---

## Running as a Different User

```bash
# Run as a specific non-root user
sudo -u www-data whoami
sudo -u postgres psql

# Run as a specific user AND group
sudo -u deploy -g deploy ./run_task.sh

# Run as root explicitly (same as default, but explicit)
sudo -u root systemctl status nginx

# Common pattern: run a service-owned script as the service account
sudo -u www-data php artisan migrate
```

---

## Interactive Shells

```bash
# Full login shell as root (loads root's environment, .bashrc, changes to root's home)
sudo -i

# Shell as root but keeping most of YOUR environment
sudo -s

# Login shell as a specific non-root user
sudo -i -u postgres
sudo -iu postgres     # short form

# Difference in practice:
sudo -i               # pwd → /root, $HOME=/root, root's own .bashrc loaded
sudo -s               # pwd → unchanged, $HOME may still be yours depending on config
```

---

## Editing Files Safely

```bash
# Preferred: sudoedit / sudo -e (edits a temp copy, then moves it back with correct perms)
sudo -e /etc/nginx/nginx.conf
sudoedit /etc/hosts

# Discouraged (works, but runs the FULL editor as root — larger attack surface)
sudo vim /etc/fstab

# Edit the sudoers file itself — ALWAYS use visudo, never a direct editor
sudo visudo
sudo visudo -f /etc/sudoers.d/custom_rules

# Set your preferred editor for sudoedit/visudo
sudo update-alternatives --config editor
EDITOR=nano sudoedit /etc/hosts
```

---

## Environment Variables

```bash
# By default, sudo resets most environment variables for security
sudo env | grep PATH
# Shows a minimal, controlled PATH — not your shell's PATH

# Preserve your entire current environment
sudo -E ./deploy.sh

# Preserve only specific variables
sudo --preserve-env=PATH,HOME ./script.sh

# Common gotcha: a script relying on your custom env vars silently breaks under plain sudo
MY_VAR=hello sudo -E bash -c 'echo $MY_VAR'
# hello

MY_VAR=hello sudo bash -c 'echo $MY_VAR'
# (empty) — variable was stripped
```

---

## sudoers Configuration Examples

```bash
# Grant a user full sudo access (password required)
echo "alice ALL=(ALL:ALL) ALL" | sudo tee /etc/sudoers.d/alice
sudo chmod 440 /etc/sudoers.d/alice   # sudoers.d files MUST be 440

# Grant passwordless access to a SPECIFIC command only (least privilege)
echo "deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp" | \
  sudo tee /etc/sudoers.d/deploy_restart
sudo chmod 440 /etc/sudoers.d/deploy_restart

# Grant a whole group access via visudo
sudo visudo
# Add: %devops  ALL=(ALL)  NOPASSWD: /usr/bin/docker, /usr/bin/docker-compose

# Add a user to the sudo group instead of editing sudoers directly (Debian/Ubuntu)
sudo usermod -aG sudo alice
# RHEL/CentOS equivalent group is usually "wheel":
sudo usermod -aG wheel alice

# Verify syntax without applying (visudo does this automatically on save)
sudo visudo -c
# /etc/sudoers: parsed OK
```

---

## Scripting with sudo

```bash
# Fail immediately instead of hanging on a password prompt (good for automation)
sudo -n true && echo "have cached sudo" || echo "would need a password"

# Run a whole block of commands as root without repeating sudo each time
sudo bash -c '
  apt update
  apt upgrade -y
  systemctl restart nginx
'

# Check sudo access from inside a script before proceeding
if ! sudo -n true 2>/dev/null; then
  echo "This script requires passwordless sudo or an already-cached credential."
  exit 1
fi

# Cron jobs running as a user that needs elevated one-off access
# (better: grant a specific NOPASSWD rule for the exact command needed)
* * * * * deploy sudo -n /usr/bin/systemctl restart myapp
```

---

## Password Caching Control

```bash
# Extend/refresh the cached credential without running a command
sudo -v

# Force the next sudo call to prompt for a password again
sudo -k
sudo whoami        # prompts, even if within the normal timeout window

# Completely wipe cached credentials (all tty sessions)
sudo -K

# Check how long your timestamp is configured to last
sudo grep -r timestamp_timeout /etc/sudoers /etc/sudoers.d/ 2>/dev/null

# Keep sudo "alive" during a long-running script (common pattern)
sudo -v
while true; do sudo -n true; sleep 60; kill -0 "$$" || exit; done 2>/dev/null &
long_running_command_that_needs_root_midway
```

---

## Piping & Redirection Gotchas

```bash
# ❌ This does NOT work as expected — the redirect runs as YOUR user, not root
sudo echo "options" > /etc/some_protected_file
# "echo" runs as root, but the shell (not sudo) performs the ">" redirect
# with YOUR permissions — Permission denied if the file isn't writable by you

# ✅ Correct: wrap the whole pipeline in a root shell
echo "options" | sudo tee /etc/some_protected_file
echo "options" | sudo tee -a /etc/some_protected_file   # append instead of overwrite

# ✅ Or run the entire command line inside sudo bash -c
sudo bash -c 'echo "options" > /etc/some_protected_file'

# Multiple commands with proper root redirection
sudo bash -c 'echo "line1" >> /etc/config && echo "line2" >> /etc/config'
```

---

## Auditing & Checking Permissions

```bash
# List what commands you're allowed to run
sudo -l

# List what another user can run (needs sufficient privilege yourself)
sudo -l -U bob

# Show recent sudo activity from the auth log
sudo grep sudo /var/log/auth.log | tail -20      # Debian/Ubuntu
sudo grep sudo /var/log/secure | tail -20        # RHEL/CentOS

# Follow sudo activity live via journalctl
sudo journalctl -f _COMM=sudo

# Find all custom sudoers drop-in files
ls -la /etc/sudoers.d/

# Validate sudoers syntax before trusting it
sudo visudo -c
```

---

## Real-World Recipes

```bash
# --- Server Setup ---

# Create a new admin user with full sudo access
sudo adduser newadmin
sudo usermod -aG sudo newadmin        # Debian/Ubuntu
sudo usermod -aG wheel newadmin       # RHEL/CentOS

# --- CI/CD & Automation ---

# Grant a deploy user passwordless access to ONLY the restart command it needs
echo "ciuser ALL=(root) NOPASSWD: /usr/bin/systemctl restart app.service" | \
  sudo tee /etc/sudoers.d/ci_restart
sudo chmod 440 /etc/sudoers.d/ci_restart

# --- Database Administration ---

# Run psql as the postgres service account
sudo -u postgres psql
sudo -u postgres createdb myapp_production

# --- Package Management ---

# Standard update/upgrade flow
sudo apt update && sudo apt upgrade -y
sudo dnf upgrade --refresh              # RHEL/Fedora equivalent

# --- Config File Editing Workflow ---

sudo -e /etc/nginx/sites-available/mysite.conf
sudo nginx -t                            # validate before reloading
sudo systemctl reload nginx

# --- Temporary Elevated Debugging Session ---

sudo -i                                  # full root shell for deep debugging
# ... do work ...
exit                                     # return to normal user immediately
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
