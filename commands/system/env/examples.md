# env — Practical Examples

> Real-world patterns for shebangs, sandboxing, secrets hygiene, and NUL-safe scripting.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Shebang Usage](#shebang-usage)
- [Setting and Unsetting Variables for a Single Command](#setting-and-unsetting-variables-for-a-single-command)
- [Minimal / Sandboxed Environments](#minimal--sandboxed-environments)
- [Inspecting a Running Process's Environment](#inspecting-a-running-processs-environment)
- [NUL-Safe Scripting](#nul-safe-scripting)
- [Combining env with Other Tools](#combining-env-with-other-tools)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Print the full current environment
env

# Filter for one variable
env | grep '^PATH='

# Equivalent, more idiomatic read-only lookup
printenv PATH
```

---

## Shebang Usage

```bash
# The standard PATH-resolving shebang — locates python3 wherever it lives
#!/usr/bin/env python3

# Multi-argument shebangs require -S (GNU coreutils >= 8.30) since the
# kernel's shebang parser only splits into interpreter + ONE argument
#!/usr/bin/env -S python3 -u

# Passing an environment variable override directly via a shebang line
#!/usr/bin/env -S PYTHONDONTWRITEBYTECODE=1 python3
```

```bash
# Verify which interpreter a PATH-based shebang will actually resolve to
# before trusting it in a security-sensitive context
which -a python3
```

---

## Setting and Unsetting Variables for a Single Command

```bash
# Inject a variable for one command only — does NOT persist in your shell
env DEBUG=1 ./run-tests.sh

# Confirm it didn't leak into the current shell
echo "$DEBUG"
# (empty)

# Remove a specific inherited variable for one command, keep everything else
env -u HTTP_PROXY curl -sI https://internal.example.com

# Override PATH just for this invocation, without touching your real PATH
env PATH=/opt/toolchain/bin:"$PATH" make build
```

---

## Minimal / Sandboxed Environments

```bash
# Run with a fully empty environment — nothing inherited survives
env -i whoami
# whoami: cannot find name for user ID ...   (some tools may behave
# oddly without HOME/USER — build the minimal set you actually need)

# Build a deliberately minimal, reproducible environment for CI or testing
env -i \
  PATH=/usr/bin:/bin \
  HOME=/home/ci \
  LANG=C.UTF-8 \
  ./build.sh

# Strip cloud credentials before invoking a third-party or untrusted script
env -u AWS_ACCESS_KEY_ID -u AWS_SECRET_ACCESS_KEY -u AWS_SESSION_TOKEN \
  ./third-party-plugin.sh

# Verbose mode: see exactly which unset/ignore actions env performs
env -v -i PATH=/usr/bin true
```

---

## Inspecting a Running Process's Environment

```bash
# Inspect your own shell's raw exec-time environment (NUL-separated)
cat /proc/self/environ | tr '\0' '\n'

# Inspect another process's environment (requires matching UID, or root)
cat /proc/1234/environ | tr '\0' '\n'

# Audit a process for anything that looks like a leaked secret
tr '\0' '\n' < /proc/1234/environ | grep -iE 'key|secret|token|password'
```

---

## NUL-Safe Scripting

```bash
# Values containing embedded newlines corrupt naive newline-based parsing —
# use -0 for a NUL-delimited, unambiguous machine-readable format
env -0 | tr '\0' '\n' | grep -c '='

# Feed NUL-delimited env output into xargs safely
env -0 | xargs -0 -n1 echo

# Read NUL-delimited pairs into a bash array
mapfile -d '' env_pairs < <(env -0)
echo "${env_pairs[0]}"
```

---

## Combining env with Other Tools

```bash
# Diff two environments to see exactly what changed between two shells/sessions
diff <(env | sort) <(ssh otherhost env | sort)

# Run a command under a clean environment AND inside a restricted namespace
env -i unshare --net --pid --fork bash -c 'ip addr'

# Confirm a shebang-based script resolves its interpreter the way you expect
env -v python3 --version
```

---

## Real-World Recipes

```bash
# --- Reproducible CI Build Step ---
env -i \
  PATH=/usr/local/bin:/usr/bin:/bin \
  HOME="$HOME" \
  CI=true \
  ./scripts/build.sh

# --- Pre-Deploy Secrets Hygiene Check ---
# Fail the deploy if any variable name looks like it holds a live secret
env | awk -F= '{print $1}' | grep -iE 'password|secret|private_key' \
  && { echo "Refusing to deploy: secret-looking var in environment"; exit 1; }

# --- Sandboxed Test Run of an Untrusted Script ---
env -i PATH=/usr/bin:/bin HOME=/tmp/sandbox-home \
  timeout 10 ./untrusted_script.sh

# --- Comparing "what does my shebang actually resolve to" across hosts ---
for host in build1 build2 build3; do
  echo "=== $host ==="
  ssh "$host" 'env which python3'
done

# --- Verifying No Stale Proxy Variables Leak Into a Container Build ---
docker build --build-arg http_proxy= --build-arg https_proxy= \
  -t myapp:latest .
env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY \
  docker build -t myapp:latest .
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
