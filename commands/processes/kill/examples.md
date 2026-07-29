# kill — Practical Examples

> Real-world patterns for terminating, reloading, and checking on processes gracefully.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Escalating from Polite to Forceful](#escalating-from-polite-to-forceful)
- [Checking Whether a Process Is Still Alive](#checking-whether-a-process-is-still-alive)
- [Killing by Name Instead of PID](#killing-by-name-instead-of-pid)
- [Reloading a Daemon's Configuration](#reloading-a-daemons-configuration)
- [Working with Process Groups and Jobs](#working-with-process-groups-and-jobs)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Send the default signal (SIGTERM) — ask a process to terminate cleanly
kill 1234

# Send a specific signal by name
kill -SIGTERM 1234
kill -HUP 1234

# Send a specific signal by number
kill -15 1234

# Kill multiple PIDs at once
kill 1234 5678 9012
```

---

## Escalating from Polite to Forceful

```bash
# Standard graceful-then-forceful shutdown pattern
kill 1234          # ask nicely first (SIGTERM)
sleep 5            # give it a moment to clean up and exit
kill -0 1234 2>/dev/null && kill -9 1234   # still alive? force it

# A small reusable function encapsulating the same idea
graceful_kill() {
  local pid=$1
  kill "$pid"
  for i in {1..5}; do
    kill -0 "$pid" 2>/dev/null || return 0
    sleep 1
  done
  kill -9 "$pid"
}
```

---

## Checking Whether a Process Is Still Alive

```bash
# kill -0 sends NO actual signal — it only tests existence/permission
if kill -0 1234 2>/dev/null; then
  echo "Process 1234 is still running"
else
  echo "Process 1234 is not running (or you lack permission to signal it)"
fi

# Wait in a loop until a process actually exits
while kill -0 1234 2>/dev/null; do
  sleep 1
done
echo "Process 1234 has exited"
```

---

## Killing by Name Instead of PID

```bash
# Find the PID first, then kill it explicitly
pgrep -f myapp
kill "$(pgrep -f myapp)"

# Or use the purpose-built alternatives directly
killall myapp
pkill -f myapp

# Kill every process owned by a specific user (careful!)
pkill -u alice
```

---

## Reloading a Daemon's Configuration

```bash
# Many long-running daemons repurpose SIGHUP as "reload your config"
# rather than its historical "terminal hung up" meaning
kill -HUP "$(cat /var/run/nginx.pid)"

# The systemd-managed equivalent, which typically sends the same
# signal under the hood for services that support it
sudo systemctl reload nginx
```

---

## Working with Process Groups and Jobs

```bash
# Kill an entire process group (a process and all its children) at once
kill -TERM -1234
# Note the leading minus sign — this is what makes it target the GROUP
# rather than a single process with PID 1234

# Launch something in the background, then signal it via job control
long_running_task &
kill %1              # targets job 1 — shell builtin only, not /usr/bin/kill

# List current jobs to find the right job number first
jobs
```

---

## Real-World Recipes

```bash
# --- Gracefully Restart a Long-Running Script ---
OLD_PID=$(cat /var/run/myapp.pid)
kill "$OLD_PID"
while kill -0 "$OLD_PID" 2>/dev/null; do sleep 1; done
./myapp &
echo $! > /var/run/myapp.pid

# --- Kill Every Process Matching a Pattern, After Reviewing First ---
pgrep -fl 'stale_worker'
# review the output above carefully, THEN:
pkill -f 'stale_worker'

# --- Safe Cleanup Trap Inside a Script ---
cleanup() {
  echo "Caught signal, cleaning up..."
  rm -f /tmp/myapp.lock
  exit 1
}
trap cleanup SIGINT SIGTERM
./long_task.sh

# --- Force-Kill Anything Still Alive After a Timeout ---
timeout 10 ./might_hang.sh
if [ $? -eq 124 ]; then
  echo "Timed out — process was killed by SIGTERM automatically"
fi

# --- Check for Zombie/Orphaned Processes Before Killing by Name ---
ps -eo pid,ppid,stat,cmd | grep '[m]yapp'
# confirm you're targeting the right processes, THEN:
pkill -f myapp
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
