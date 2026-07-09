# tee — Practical Examples

> Real-world patterns for logging, debugging pipelines, and writing to protected files.

---

## Table of Contents

- [Basic Save-and-View](#basic-save-and-view)
- [Appending to Logs](#appending-to-logs)
- [Writing to Protected Files with sudo](#writing-to-protected-files-with-sudo)
- [Multiple File Outputs](#multiple-file-outputs)
- [Debugging Pipelines with tee](#debugging-pipelines-with-tee)
- [Combining tee with Process Substitution](#combining-tee-with-process-substitution)
- [Long-Running Command Monitoring](#long-running-command-monitoring)
- [tee in Build & Deployment Scripts](#tee-in-build--deployment-scripts)
- [Checking Exit Codes Through tee](#checking-exit-codes-through-tee)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Save-and-View

```bash
# See a command's output AND save it, in one step
ls -la | tee directory_listing.txt

# Save a long-running command's output while watching it live
ping -c 10 google.com | tee ping_results.txt

# Capture a build's output for later review while still watching it happen
make | tee build_output.log

# Quick way to duplicate stdin to a file (useful in manual testing)
cat | tee saved_input.txt
# (type some text, press Ctrl+D when done — it's echoed AND saved)
```

---

## Appending to Logs

```bash
# Add a timestamped entry to a log file WITHOUT overwriting previous entries
echo "$(date): Deployment started" | tee -a deploy.log

# Append the output of a whole command block to a running log
{
  echo "=== Run started at $(date) ==="
  ./run_tests.sh
  echo "=== Run finished at $(date) ==="
} | tee -a test_history.log

# Combine appending with viewing live during a monitoring loop
while true; do
  date | tee -a heartbeat.log
  sleep 60
done
```

---

## Writing to Protected Files with sudo

```bash
# The classic fix for "sudo echo > file" not working
echo "127.0.0.1 myhost.local" | sudo tee -a /etc/hosts

# Overwrite a config file that requires root permissions
echo "PermitRootLogin no" | sudo tee /etc/ssh/sshd_config.d/99-custom.conf

# Write MULTIPLE lines at once using a heredoc piped into sudo tee
sudo tee /etc/sysctl.d/99-custom.conf > /dev/null << 'EOF'
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 1
EOF
# The ">/dev/null" here suppresses the file's content also being
# printed to your terminal — useful when you don't need to SEE the
# echoed output, just confirm the write succeeded (check $? afterward)

# Verify the write actually happened
sudo cat /etc/sysctl.d/99-custom.conf
```

---

## Multiple File Outputs

```bash
# Save the identical output to two files in one pass
command | tee primary_copy.txt backup_copy.txt

# Save a snapshot to a dated archive AND update a "latest" file simultaneously
report_command | tee "reports/report_$(date +%Y%m%d).txt" reports/latest.txt

# Write the same configuration to several services' config directories at once
echo "shared_setting=true" | sudo tee /etc/service1/config /etc/service2/config /etc/service3/config
```

---

## Debugging Pipelines with tee

```bash
# Insert tee at any stage to see EXACTLY what's flowing through at that point,
# without altering what the pipeline ultimately produces
cat data.csv \
  | tee /tmp/stage1_raw.csv \
  | awk -F',' '{print $2}' \
  | tee /tmp/stage2_extracted.txt \
  | sort \
  | tee /tmp/stage3_sorted.txt \
  | uniq -c

# Quickly check what a complex command is ACTUALLY producing before
# it gets consumed by the next step (common when debugging "why is
# my final output wrong")
complex_generator | tee /tmp/debug.txt | grep "unexpected_pattern"
cat /tmp/debug.txt   # inspect the FULL intermediate output afterward
```

---

## Combining tee with Process Substitution

```bash
# Send a stream to a file AND compress a copy simultaneously
long_running_report | tee report.txt >(gzip > report.txt.gz)

# Send output to a file, AND simultaneously feed a live monitoring tool,
# while still allowing the ORIGINAL pipeline to continue normally
tail -f app.log | tee -a full_history.log | grep --line-buffered "CRITICAL"

# Fan out a stream to THREE different destinations at once
data_stream | tee >(process_a > out_a.txt) >(process_b > out_b.txt) > out_main.txt
```

---

## Long-Running Command Monitoring

```bash
# Watch a long build/test process live, while saving the ENTIRE output
# for later review even after the terminal session ends
./run_long_tests.sh 2>&1 | tee test_run_$(date +%Y%m%d_%H%M%S).log

# Keep tee running even if you accidentally hit Ctrl+C (rare, situational)
long_command | tee -i output.txt

# Force unbuffered/line-buffered output so you see progress in real
# time rather than in delayed chunks
some_command | stdbuf -oL tee -a progress.log
```

---

## tee in Build & Deployment Scripts

```bash
#!/bin/bash
# Capture full build output for CI artifacts, while still showing it
# live in the CI console for real-time monitoring
npm run build 2>&1 | tee build.log
BUILD_EXIT="${PIPESTATUS[0]}"

if [ "$BUILD_EXIT" -ne 0 ]; then
  echo "Build failed — see build.log for details"
  exit 1
fi

# Deployment step, logging progress while it runs
{
  echo "Deployment started at $(date)"
  ./deploy.sh
  echo "Deployment finished at $(date)"
} | tee -a deployment_history.log
```

---

## Checking Exit Codes Through tee

```bash
# Naive check — WRONG, checks tee's exit status, not the real command's
risky_command | tee output.txt
if [ $? -ne 0 ]; then
  echo "This almost never triggers, because tee usually succeeds"
fi

# Correct check using PIPESTATUS (bash-specific)
risky_command | tee output.txt
if [ "${PIPESTATUS[0]}" -ne 0 ]; then
  echo "risky_command actually failed — see output.txt for details"
  exit 1
fi

# Same pattern, explicitly naming both stages for clarity
some_command | tee some_command.log
CMD_STATUS="${PIPESTATUS[0]}"
TEE_STATUS="${PIPESTATUS[1]}"
echo "Command exit: $CMD_STATUS, tee exit: $TEE_STATUS"
```

---

## Real-World Recipes

```bash
# --- Server Configuration Change with Audit Trail ---

echo "$(date): Updating nginx config" | sudo tee -a /var/log/config_changes.log
sudo tee /etc/nginx/sites-available/mysite.conf > /dev/null << 'EOF'
server {
    listen 80;
    server_name example.com;
}
EOF
sudo nginx -t && sudo systemctl reload nginx

# --- CI Pipeline Capturing Full Logs While Streaming to Console ---

./run_ci_suite.sh 2>&1 | tee "ci_logs/run_${BUILD_NUMBER}.log"
test "${PIPESTATUS[0]}" -eq 0 || exit 1

# --- Splitting a Data Stream for Both Processing and Archival ---

curl -s https://api.example.com/data \
  | tee "archive/raw_$(date +%Y%m%d).json" \
  | jq '.results[] | select(.active == true)'

# --- Quick One-Off Debug of a Pipe Without Rerunning the Whole Command ---

expensive_command | tee /tmp/last_run_output.txt | head -20
# Later, without rerunning expensive_command again:
grep "specific_thing" /tmp/last_run_output.txt

# --- Broadcasting a Setting Change to Multiple Config Files at Once ---

echo "max_connections = 200" | sudo tee /etc/service/*/limits.conf
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
