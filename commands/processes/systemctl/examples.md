# systemctl — Practical Examples

> Real-world patterns for managing services, diagnosing failures, and
> working with timers, overrides, and boot behavior.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Checking Status Quickly](#checking-status-quickly)
- [Enabling and Disabling at Boot](#enabling-and-disabling-at-boot)
- [Diagnosing a Failed Service](#diagnosing-a-failed-service)
- [Working with Timers Instead of Cron](#working-with-timers-instead-of-cron)
- [Creating and Editing Unit Files](#creating-and-editing-unit-files)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Start, stop, and restart a service
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx

# Reload configuration without a full restart (if the service supports it)
sudo systemctl reload nginx

# Reload if supported, otherwise fall back to a full restart
sudo systemctl reload-or-restart nginx
```

---

## Checking Status Quickly

```bash
# Full status with recent log lines
systemctl status nginx

# Just a yes/no answer, script-friendly
systemctl is-active nginx
systemctl is-enabled nginx
systemctl is-failed nginx

# All currently running services
systemctl list-units --type=service --state=running

# All services in a failed state
systemctl --failed

# Every installed unit and whether it's enabled
systemctl list-unit-files --type=service
```

---

## Enabling and Disabling at Boot

```bash
# Enable so it starts on future boots, without starting it right now
sudo systemctl enable nginx

# Start it now too, in a single command
sudo systemctl enable --now nginx

# Disable so it no longer starts on boot, without stopping it right now
sudo systemctl disable nginx

# Fully prevent a unit from ever being started, even manually
sudo systemctl mask some-unwanted.service

# Reverse a mask
sudo systemctl unmask some-unwanted.service
```

---

## Diagnosing a Failed Service

```bash
# Step 1: confirm it's actually failed
systemctl status myapp

# Step 2: read the fuller log history for that unit
journalctl -u myapp --since "10 minutes ago"

# Step 3: follow logs live while attempting a restart
journalctl -u myapp -f &
sudo systemctl restart myapp

# Step 4: check exactly what command/config the unit uses
systemctl cat myapp

# See ALL failed units on the system at a glance
systemctl --failed
```

---

## Working with Timers Instead of Cron

```bash
# List all active timers and their next scheduled run
systemctl list-timers

# Check a specific timer's status
systemctl status backup.timer

# Manually trigger the service a timer would normally run, right now
sudo systemctl start backup.service

# Enable a timer so it starts scheduling on boot
sudo systemctl enable --now backup.timer
```

---

## Creating and Editing Unit Files

```bash
# View an existing unit file in full
systemctl cat myapp.service

# Create a drop-in override without editing the original file directly
sudo systemctl edit myapp.service
# opens $EDITOR with a blank override file at
# /etc/systemd/system/myapp.service.d/override.conf

# After creating/editing any unit file by hand, reload systemd's view of it
sudo systemctl daemon-reload
sudo systemctl restart myapp

# See the final, merged configuration after overrides are applied
systemctl cat myapp.service
```

---

## Real-World Recipes

```bash
# --- Post-Deployment Sanity Check ---
sudo systemctl restart myapp
sleep 2
systemctl is-active --quiet myapp && echo "myapp is running" || echo "myapp FAILED to start"

# --- Quick Health Check Across Several Services ---
for svc in nginx postgresql redis myapp; do
  printf "%-15s %s\n" "$svc" "$(systemctl is-active "$svc")"
done

# --- Verify a Service Will Survive a Reboot ---
systemctl is-enabled myapp || echo "WARNING: myapp will NOT start after reboot"

# --- Find Every Unit That's Currently Failed, With Its Last Log Line ---
for unit in $(systemctl --failed --plain --no-legend | awk '{print $1}'); do
  echo "=== $unit ==="
  journalctl -u "$unit" -n 3 --no-pager
done

# --- Safely Reload a Web Server After a Config Change ---
sudo nginx -t && sudo systemctl reload nginx || echo "Config test failed — NOT reloading"

# --- Confirm What a Custom Service Actually Runs Before Trusting It ---
systemctl cat myapp.service | grep -E "ExecStart|User|Restart"
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
