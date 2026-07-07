# pwd — Practical Examples

> Real-world patterns for scripting, symlink resolution, and everyday navigation.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Logical vs Physical Path in Practice](#logical-vs-physical-path-in-practice)
- [Using pwd with $PWD and $OLDPWD](#using-pwd-with-pwd-and-oldpwd)
- [Getting a Script's Own Directory](#getting-a-scripts-own-directory)
- [Combining pwd with Other Commands](#combining-pwd-with-other-commands)
- [Subshells and Directory Isolation](#subshells-and-directory-isolation)
- [Prompt Customization Using pwd](#prompt-customization-using-pwd)
- [Scripting Safety Patterns](#scripting-safety-patterns)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Show the current directory
pwd

# Navigate, then confirm where you ended up
cd /var/log
pwd
# /var/log

# Check your location before running a destructive command, as a habit
pwd && rm -rf ./build

# Combine with ls to see both location and contents at once
pwd; ls -la
```

---

## Logical vs Physical Path in Practice

```bash
# Set up a symlink for demonstration
ln -s /var/log ~/logs_shortcut
cd ~/logs_shortcut

pwd
# /home/alice/logs_shortcut   ← logical: exactly what you navigated to

pwd -P
# /var/log                     ← physical: fully resolved real location

# Force cd itself to resolve symlinks immediately when moving
cd -P ~/logs_shortcut
pwd
# /var/log                     ← now even plain "pwd" shows the physical path,
# because $PWD itself was updated to the resolved location by cd -P

# Compare both explicitly, side by side
echo "Logical:  $(pwd -L)"
echo "Physical: $(pwd -P)"
```

---

## Using pwd with $PWD and $OLDPWD

```bash
# $PWD mirrors pwd's output without spawning a new process
cd /tmp
echo $PWD
# /tmp

# Jump back and forth between two directories using $OLDPWD
cd /var/log
cd /tmp
cd -
# /var/log     (cd - prints the directory, unlike a normal silent cd)
cd -
# /tmp          (toggles back again)

# Use $OLDPWD directly in a script for a "return to previous" step
cd /some/deep/path
# ... do some work ...
cd "$OLDPWD"

# Quick directory bookmark pattern using a plain variable
BOOKMARK=$(pwd)
cd /somewhere/else
# ... later ...
cd "$BOOKMARK"
```

---

## Getting a Script's Own Directory

```bash
#!/bin/bash
# The single most common real-world use of pwd inside scripts:
# finding out where the SCRIPT FILE ITSELF is located, regardless of
# which directory the user happened to be in when they ran it.

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
echo "Script is located in: $SCRIPT_DIR"

# Load a config file that sits NEXT TO the script, no matter where
# the script is invoked FROM
source "$SCRIPT_DIR/config.sh"

# Reference a data file relative to the script's own location
cat "$SCRIPT_DIR/data/settings.json"
```

```bash
# The WRONG way (a very common beginner mistake):
#!/bin/bash
echo "$(pwd)/config.sh"
# This points to config.sh relative to wherever the user RAN the
# script from — completely unreliable if the script is called from
# a different directory, or via a symlink, or through a cron job
# with an unexpected working directory.
```

---

## Combining pwd with Other Commands

```bash
# Store the current directory before a risky series of commands
CURRENT=$(pwd)
cd /tmp/scratch
rm -rf ./*
cd "$CURRENT"

# Include the working directory in a log message
echo "[$(date)] Running from: $(pwd)" >> activity.log

# Use pwd's output as part of a backup filename
tar -czf "backup_$(basename $(pwd))_$(date +%Y%m%d).tar.gz" .

# Confirm you're in the expected directory before a git operation
if [ "$(pwd)" != "/srv/myproject" ]; then
  echo "Warning: not in the expected project directory!"
  exit 1
fi
git pull
```

---

## Subshells and Directory Isolation

```bash
# Run a command in a different directory WITHOUT changing your
# current shell's own working directory at all
(cd /tmp/build && make)
pwd
# unchanged — still wherever you were before, since the cd only
# affected the subshell created by the parentheses

# Compile several projects in sequence, each in its own subshell,
# without needing to manually cd back after each one
for project in proj1 proj2 proj3; do
  (cd "$project" && make clean && make)
done
pwd
# still the original directory the loop started in

# Contrast with NOT using a subshell (this DOES change your shell):
cd /tmp/build
make
cd ..    # must manually return — easy to forget in a longer script
```

---

## Prompt Customization Using pwd

```bash
# Many shell prompts embed the current directory using $PWD directly
# rather than calling pwd as a separate process (faster, no fork needed)
PS1='\u@\h:\w\$ '
# \w in bash's PS1 syntax already expands to something equivalent to pwd's output

# A custom prompt function that shortens long paths using pwd's logic
short_pwd() {
  local dir
  dir=$(pwd)
  echo "${dir/#$HOME/~}"    # replace home directory prefix with ~
}
PS1='[$(short_pwd)]\$ '

# Show only the last two path components in the prompt
PS1='[\W]\$ '     # \W (capital) shows just the final directory name
```

---

## Scripting Safety Patterns

```bash
# Always verify a cd succeeded before proceeding — a failed cd leaves
# you in your ORIGINAL directory, which can be dangerous for scripts
# that assume they're somewhere else
cd /expected/deploy/path || { echo "cd failed, aborting"; exit 1; }
pwd    # confirm exactly where we ended up before continuing
rm -rf ./old_build

# Defensive pattern: capture and restore, even if the script exits early
ORIGINAL_DIR="$(pwd)"
trap 'cd "$ORIGINAL_DIR"' EXIT
cd /tmp/work
# ... any exit path (error, success, signal) still restores the original dir

# Sanity-check that pwd and $PWD agree (they can drift if $PWD was
# manually tampered with somewhere earlier in a long script)
if [ "$(pwd)" != "$PWD" ]; then
  echo "Warning: \$PWD is out of sync with actual working directory!"
fi
```

---

## Real-World Recipes

```bash
# --- Deployment Script Anchoring Paths to Its Own Location ---

#!/bin/bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$SCRIPT_DIR"
docker compose -f docker-compose.yml up -d

# --- Backup Script Naming Archives After the Current Folder ---

tar -czf "$(basename $(pwd))_backup_$(date +%Y%m%d_%H%M).tar.gz" .

# --- Build Script That Restores the Original Directory No Matter What ---

#!/bin/bash
set -e
ORIGINAL_DIR=$(pwd)
trap 'cd "$ORIGINAL_DIR"' EXIT
cd build/
cmake .. && make

# --- Verifying You're NOT in a Dangerous Directory Before rm -rf ---

if [ "$(pwd)" = "/" ] || [ "$(pwd)" = "$HOME" ]; then
  echo "Refusing to run in $(pwd) — looks like the wrong directory!"
  exit 1
fi
rm -rf ./build

# --- Logging Every Command's Context During a Long-Running Script ---

log() { echo "[$(date +%H:%M:%S)] [$(pwd)] $1"; }
log "Starting deployment"
cd /srv/app
log "Now in app directory"
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
