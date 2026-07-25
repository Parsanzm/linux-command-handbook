# uname — Practical Examples

> Real-world patterns for scripting, compatibility checks, and system identification.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Getting Just One Piece of Information](#getting-just-one-piece-of-information)
- [Architecture-Aware Scripting](#architecture-aware-scripting)
- [Kernel Version Checks](#kernel-version-checks)
- [Combining uname with Other System Info](#combining-uname-with-other-system-info)
- [Checking Multiple Servers at Once](#checking-multiple-servers-at-once)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Full system identification, one line
uname -a
# Linux myhost 6.8.0-31-generic #31-Ubuntu SMP PREEMPT_DYNAMIC x86_64 x86_64 x86_64 GNU/Linux

# Just the kernel name (default, no options)
uname
# Linux
```

---

## Getting Just One Piece of Information

```bash
# Kernel release — commonly needed for matching kernel module directories
uname -r
# 6.8.0-31-generic

# Machine architecture — commonly needed for picking the right binary/package
uname -m
# x86_64

# Hostname
uname -n
# myhost

# Operating system label
uname -o
# GNU/Linux
```

---

## Architecture-Aware Scripting

```bash
# Download the correct binary release for the current architecture
ARCH=$(uname -m)
case "$ARCH" in
  x86_64)  URL="https://example.com/releases/app-linux-amd64.tar.gz" ;;
  aarch64) URL="https://example.com/releases/app-linux-arm64.tar.gz" ;;
  armv7l)  URL="https://example.com/releases/app-linux-armv7.tar.gz" ;;
  *)       echo "Unsupported architecture: $ARCH"; exit 1 ;;
esac
curl -LO "$URL"

# Fail early in an install script if the architecture isn't supported
if [ "$(uname -m)" != "x86_64" ]; then
  echo "This package only supports x86_64 systems."
  exit 1
fi

# Branch behavior between Linux and macOS in a cross-platform script
case "$(uname -s)" in
  Linux)  echo "Running Linux-specific setup" ;;
  Darwin) echo "Running macOS-specific setup" ;;
  *)      echo "Unsupported OS: $(uname -s)"; exit 1 ;;
esac
```

---

## Kernel Version Checks

```bash
# Print just the kernel release, useful when reporting a bug
echo "Kernel: $(uname -r)"

# Locate the matching kernel headers directory for building a module
ls "/lib/modules/$(uname -r)/build"

# Quick numeric comparison — is the kernel at least version 5.10?
KVER=$(uname -r | cut -d. -f1,2)
if awk -v v="$KVER" 'BEGIN { exit !(v >= 5.10) }'; then
  echo "Kernel is 5.10 or newer"
else
  echo "Kernel is older than 5.10 — some features may be unavailable"
fi

# Check whether the kernel version string mentions a specific distro build tag
uname -v | grep -i ubuntu
```

---

## Combining uname with Other System Info

```bash
# A quick "what am I working with" snapshot
echo "Kernel:       $(uname -s) $(uname -r)"
echo "Architecture: $(uname -m)"
echo "Hostname:     $(uname -n)"
echo "Distro:       $(. /etc/os-release; echo "$PRETTY_NAME")"

# One-shot friendlier summary, if systemd is present
hostnamectl

# Full picture: kernel info plus CPU details
uname -a
lscpu | grep -E "Model name|Architecture|CPU\(s\)"
```

---

## Checking Multiple Servers at Once

```bash
# Quick kernel/architecture audit across a fleet
for server in web1 web2 web3 db1 db2; do
  echo "=== $server ==="
  ssh "$server" uname -a
done

# Confirm all servers are on a consistent kernel release before a rollout
for server in $(cat server_list.txt); do
  echo "$server: $(ssh "$server" uname -r)"
done | sort -t: -k2

# Flag any server whose architecture doesn't match the expected value
EXPECTED_ARCH="x86_64"
for server in $(cat server_list.txt); do
  ACTUAL=$(ssh "$server" uname -m)
  if [ "$ACTUAL" != "$EXPECTED_ARCH" ]; then
    echo "MISMATCH on $server: expected $EXPECTED_ARCH, got $ACTUAL"
  fi
done
```

---

## Real-World Recipes

```bash
# --- Bug Report Template Header ---
echo "System info for bug report:"
uname -a
cat /etc/os-release 2>/dev/null | grep PRETTY_NAME

# --- Pre-Install Compatibility Check ---
if [ "$(uname -s)" != "Linux" ]; then
  echo "This installer only supports Linux."
  exit 1
fi
if [ "$(uname -m)" != "x86_64" ] && [ "$(uname -m)" != "aarch64" ]; then
  echo "Unsupported architecture: $(uname -m)"
  exit 1
fi
echo "Compatibility check passed."

# --- Build a Download Filename Dynamically ---
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)
FILENAME="myapp-${OS}-${ARCH}.tar.gz"
echo "Fetching: $FILENAME"

# --- Simple Environment Banner on Login ---
printf "%-12s %s\n" "Kernel:" "$(uname -r)"
printf "%-12s %s\n" "Arch:" "$(uname -m)"
printf "%-12s %s\n" "Hostname:" "$(uname -n)"
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
