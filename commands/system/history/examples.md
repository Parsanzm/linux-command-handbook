# history — Practical Examples

> Real-world patterns for re-running commands, searching history, and configuring it well.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [History Expansion Shortcuts](#history-expansion-shortcuts)
- [Searching History](#searching-history)
- [Fixing Mistakes Quickly](#fixing-mistakes-quickly)
- [Configuring History Behavior](#configuring-history-behavior)
- [Sharing History Across Terminals](#sharing-history-across-terminals)
- [Cleaning Up History](#cleaning-up-history)
- [Scripting Around History](#scripting-around-history)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Show all history
history

# Show just the last 20 commands
history 20

# Show history with line numbers, piped into less for scrolling
history | less

# Count how many commands are in history right now
history | wc -l
```

---

## History Expansion Shortcuts

```bash
# Re-run the last command
!!

# Classic pattern: forgot sudo, re-run the same command WITH it
apt update
sudo !!

# Re-run a specific numbered command
history | grep docker
!512

# Re-run the most recent command starting with a given string
!ssh
!git

# Use the last argument of the previous command in a new one
mkdir /tmp/new_project
cd !$
# cd /tmp/new_project

# Use ALL arguments of the previous command
touch file1.txt file2.txt file3.txt
chmod 644 !*
# chmod 644 file1.txt file2.txt file3.txt

# Quick substitution: rerun previous command with one word replaced
ping googl.com
^googl^google
# ping google.com
```

---

## Searching History

```bash
# Basic grep-based search
history | grep ssh

# Find the most recent match
history | grep docker | tail -1

# Interactive reverse search (press Ctrl+R, then type)
# (reverse-i-search)`ssh': ssh alice@server.example.com
# Press Ctrl+R again to cycle to the next older match
# Press Enter to run it, or Right arrow to edit first

# Search and immediately extract just the command text (strip the
# leading history number)
history | grep rsync | awk '{$1=""; print $0}'

# Find commands run around a specific time (if timestamps are enabled)
HISTTIMEFORMAT="%F %T " history | grep "2024-01-15"
```

---

## Fixing Mistakes Quickly

```bash
# Typo in the previous command — fix and rerun without retyping everything
grpe error logfile.txt
^grpe^grep
# grep error logfile.txt

# Forgot to specify a required flag
tar -czf backup.tar.gz /data
# realize you forgot -v for verbose output
!!:s/-czf/-czvf/
# reruns with the substitution applied

# Repeat the second-to-last command
!-2

# Preview what a history expansion WOULD do before running it
history -p '!!'
# shows the expanded command without executing it — useful to
# sanity-check before blindly trusting !! in a risky context
```

---

## Configuring History Behavior

```bash
# ~/.bashrc additions for a much more useful history setup

# Keep a large amount of history
export HISTSIZE=10000
export HISTFILESIZE=20000

# Avoid saving duplicate consecutive commands, and commands starting
# with a space
export HISTCONTROL=ignoreboth

# Exclude common, low-value commands from being recorded at all
export HISTIGNORE="ls:ll:cd:pwd:clear:history"

# Add a timestamp to each history entry
export HISTTIMEFORMAT="%F %T "
history | tail -5
#   9995  2024-01-15 14:22:03  git status
#   9996  2024-01-15 14:22:10  git add .
#   9997  2024-01-15 14:22:15  git commit -m "fix bug"

# Append to history file immediately after each command, instead of
# only when the shell exits
export PROMPT_COMMAND="history -a; $PROMPT_COMMAND"
```

---

## Sharing History Across Terminals

```bash
# Add to ~/.bashrc for near-real-time history sharing between
# multiple simultaneously open terminal windows
export PROMPT_COMMAND="history -a; history -c; history -r; $PROMPT_COMMAND"

# Manually force a sync in one terminal
history -a    # push this session's new commands to the file

# In ANOTHER terminal, pull in what was just pushed
history -r    # re-read the file into this session's memory
history | tail -5   # confirm the other terminal's recent commands now appear
```

---

## Cleaning Up History

```bash
# Clear the CURRENT session's in-memory history only
history -c

# Clear AND persist the clear to the history file too
history -c
history -w

# Delete one specific entry by number
history
#   500  curl -H "Authorization: Bearer secret123" https://api.example.com
history -d 500

# Delete a RANGE of entries (bash 5+, using repeated -d or a loop)
for i in $(seq 495 500); do history -d 495; done
# (deleting repeatedly at the SAME offset, since numbers shift down
# after each deletion)

# Completely wipe the history file from disk (nuclear option)
history -c
rm -f ~/.bash_history
```

---

## Scripting Around History

```bash
# Get just the command text of the last entry, without its number
history 1 | sed 's/^[ ]*[0-9]*[ ]*//'

# Build a "most frequently used commands" report
history | awk '{$1="";print $0}' | sort | uniq -c | sort -rn | head -20

# Find the most commonly used FIRST WORD (i.e., which base commands
# you use most often)
history | awk '{print $2}' | sort | uniq -c | sort -rn | head -10

# Extract all git commands you've run recently
history | grep '\bgit\b' | awk '{$1="";print $0}'
```

---

## Real-World Recipes

```bash
# --- Quickly Recover a Complex Command You Ran Earlier ---

history | grep "docker run"
# find the specific numbered entry with the complex flags you need
!845

# --- Removing an Accidentally Recorded Password ---

history | grep -n "password" 2>/dev/null
history
#   612  mysql -u admin -pMyRealPassword123 mydb
history -d 612
history -w

# --- Building a Personal "Cheat Sheet" of Useful Commands ---

history | grep -E "rsync|tar|ffmpeg" | awk '{$1="";print $0}' | sort -u > ~/my_cheatsheet.txt

# --- Auditing What You Did During an Incident ---

HISTTIMEFORMAT="%F %T " history | grep "2024-01-15 14:"
# review every command run during a specific hour of an incident

# --- Setting Up a New Machine with Good History Defaults ---

cat >> ~/.bashrc << 'EOF'
export HISTSIZE=10000
export HISTFILESIZE=20000
export HISTCONTROL=ignoreboth
export HISTTIMEFORMAT="%F %T "
export HISTIGNORE="ls:cd:pwd:clear:history"
export PROMPT_COMMAND="history -a; $PROMPT_COMMAND"
EOF
source ~/.bashrc
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
