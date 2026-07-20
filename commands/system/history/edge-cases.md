# history — Edge Cases & Gotchas

> history feels simple, but multi-terminal sync, sensitive data exposure, and
> shell-specific quirks routinely surprise even experienced command-line users.

---

## Table of Contents

- [Multiple Terminals Don't Share History in Real Time by Default](#multiple-terminals-dont-share-history-in-real-time-by-default)
- [Sensitive Commands (Passwords, Tokens) Get Recorded in Plain Text](#sensitive-commands-passwords-tokens-get-recorded-in-plain-text)
- [A Crashed or Killed Terminal Loses Its Session's History](#a-crashed-or-killed-terminal-loses-its-sessions-history)
- [!! Can Silently Repeat a Dangerous Command](#-can-silently-repeat-a-dangerous-command)
- [History Numbers Shift as Entries Are Deleted or Age Out](#history-numbers-shift-as-entries-are-deleted-or-age-out)
- [HISTCONTROL=ignorespace Requires an ACTUAL Leading Space](#histcontrolignorespace-requires-an-actual-leading-space)
- [history -c Doesn't Touch the History FILE by Default](#history--c-doesnt-touch-the-history-file-by-default)
- [Deleting an Entry Doesn't Automatically Persist to Disk](#deleting-an-entry-doesnt-automatically-persist-to-disk)
- [Non-Interactive Shells (Scripts) Don't Use History At All](#non-interactive-shells-scripts-dont-use-history-at-all)
- [HISTSIZE vs HISTFILESIZE Confusion](#histsize-vs-histfilesize-confusion)
- [Different Shells' History Files Aren't Interchangeable](#different-shells-history-files-arent-interchangeable)
- [!$ and !* Referring to the WRONG Previous Command](#-and--referring-to-the-wrong-previous-command)

---

## Multiple Terminals Don't Share History in Real Time by Default

### A command run in one terminal is invisible to another, open at the same time
```bash
# Terminal A:
cd /var/log
ls -la

# Terminal B (opened earlier, still active):
history | tail -3
# (does NOT show "cd /var/log" or "ls -la" from Terminal A at all)
# ⚠️ By DEFAULT, bash only writes a session's history to the shared
# FILE when that session EXITS — two simultaneously open terminals
# each maintain their OWN separate in-memory history, unaware of each
# other's activity until one of them closes.

# Fix: configure near-real-time sharing (see README.md/examples.md
# for the full PROMPT_COMMAND-based setup):
export PROMPT_COMMAND="history -a; history -c; history -r; $PROMPT_COMMAND"
```

---

## Sensitive Commands (Passwords, Tokens) Get Recorded in Plain Text

### A very common, genuinely risky mistake
```bash
mysql -u admin -pMySecretPassword123 production_db
history | tail -1
#   612  mysql -u admin -pMySecretPassword123 production_db
# ⚠️ This password is now sitting in PLAIN TEXT in ~/.bash_history,
# potentially readable by anyone with access to this account's home
# directory (backups, other processes running as this user, or
# anyone who later gains access to this machine) — command-line
# arguments are NOT a safe way to pass secrets in the first place,
# and history recording makes the exposure PERSISTENT, not just
# momentarily visible in `ps aux` while running.

# Prevention: use a LEADING SPACE (with HISTCONTROL=ignorespace/ignoreboth
# configured) to keep a specific sensitive command out of history entirely:
 mysql -u admin -pMySecretPassword123 production_db
# (note the space before "mysql" — this line is never recorded, IF
# HISTCONTROL is configured appropriately)

# Better yet: avoid passing secrets as command-line arguments AT ALL —
# use environment variables, config files with restricted
# permissions, or interactive password prompts instead:
mysql -u admin -p production_db
# Enter password: (typed, never appears in history OR ps aux)
```

---

## A Crashed or Killed Terminal Loses Its Session's History

### kill -9 or a crash means that session's commands are simply gone
```bash
# A long troubleshooting session, dozens of useful commands typed...
kill -9 $$
# (or the terminal application crashes, or the SSH connection drops
# abruptly without a clean logout)

# Because bash only writes accumulated history to the FILE on a
# NORMAL exit, an abrupt kill/crash means that ENTIRE session's
# history is LOST — none of it ever made it to ~/.bash_history.

# Mitigation: configure history to be appended IMMEDIATELY after
# every command, not just at clean shell exit:
export PROMPT_COMMAND="history -a; $PROMPT_COMMAND"
# With this in place, each command is flushed to disk right after
# it's typed, so even an abrupt crash only loses, at most, the very
# last command typed (rather than the entire session).
```

---

## !! Can Silently Repeat a Dangerous Command

### History expansion's power is also its risk
```bash
rm -rf /important/directory/*
# (accidentally run in the WRONG directory)

# Realizing the mistake, but muscle memory triggers:
sudo !!
# ⚠️ This RE-RUNS the EXACT same "rm -rf /important/directory/*"
# command, now WITH ROOT privileges — potentially making an already
# bad mistake catastrophically worse, since sudo removes any
# permission-based safety net that might have limited the original
# command's damage.

# Always be CONSCIOUS of exactly what !! (or any history expansion)
# will actually re-run before using it, especially combined with
# sudo — when in doubt, preview it first:
history -p '!!'
# shows what !! WOULD expand to, without executing it
```

---

## History Numbers Shift as Entries Are Deleted or Age Out

### A number you saw a moment ago might not mean the same thing now
```bash
history | grep docker
#   845  docker run -it ubuntu bash

history -d 800
# (deleted an EARLIER, unrelated entry)

history | grep docker
#   844  docker run -it ubuntu bash
# ⚠️ The docker command's number SHIFTED from 845 to 844, simply
# because an earlier entry was removed — history numbers are NOT
# stable, permanent IDs; they reflect CURRENT POSITION in the list,
# which changes whenever entries are added, removed, or aged out
# once HISTSIZE's limit is reached.

# Always re-check the CURRENT number immediately before using `!N`,
# rather than relying on a number you saw even a few commands ago.
```

---

## HISTCONTROL=ignorespace Requires an ACTUAL Leading Space

### An easy-to-miss formatting requirement
```bash
export HISTCONTROL=ignorespace

echo "test"
history | tail -1
#   700  echo "test"        ← recorded normally, no leading space

 echo "test2"
history | tail -1
#   700  echo "test"        ← "test2" NOT recorded, due to the
# literal leading space character before "echo"

# The space must be the VERY FIRST character of the command as
# actually typed — auto-indentation in some terminal/editor
# integrations, or copy-pasting a command that LOOKS like it has
# leading whitespace but doesn't (or vice versa), can cause this
# protection to silently fail to work as expected. Always verify:
history | tail -1
# immediately after typing a sensitive command, to CONFIRM it was
# actually excluded, rather than assuming the leading space "worked."
```

---

## history -c Doesn't Touch the History FILE by Default

### Clearing history feels complete, but the file on disk may still have everything
```bash
history -c
# Clears the CURRENT SESSION's in-memory history ONLY

cat ~/.bash_history
# (still shows ALL previous commands from prior sessions — history -c
# alone does NOT touch the file on disk at all)

# If a session with sensitive commands EXITS NORMALLY after just
# `history -c`, those commands could still get WRITTEN to the file
# during the normal exit-flush process, since -c only cleared MEMORY,
# not any pending write-on-exit behavior.

# To ALSO clear the persisted file:
history -c
history -w
# -w overwrites the history FILE with the (now-empty) current
# in-memory history, genuinely wiping persisted history too.
```

---

## Deleting an Entry Doesn't Automatically Persist to Disk

### history -d only affects memory, unless followed by -w
```bash
history -d 612
# Removes entry 612 from the CURRENT session's in-memory list...

# ...but if the shell later exits NORMALLY without an explicit
# history -w first, bash's default exit-flush behavior might still
# write out a history that, depending on internal bookkeeping,
# doesn't necessarily guarantee the deleted entry stays gone from the
# FILE — the safest, most explicit approach is ALWAYS following a
# deletion with an explicit write:
history -d 612
history -w
cat ~/.bash_history | grep -c "MySecretPassword"
# 0    ✅ confirm the sensitive entry is GENUINELY gone from the file
```

---

## Non-Interactive Shells (Scripts) Don't Use History At All

### Trying to use history-related commands inside a script silently fails or does nothing useful
```bash
#!/bin/bash
echo "test1"
echo "test2"
history
# (typically empty, or an error, or simply not meaningful) — history
# is fundamentally an INTERACTIVE shell feature; non-interactive
# script execution doesn't build up or maintain a command history in
# the same way an interactive session does, since there's no
# "previous command a human typed" concept in a script's straight-
# line execution.

# History expansion (!!, !$, etc.) is ALSO typically disabled by
# default in non-interactive bash scripts for safety reasons (a
# literal "!" in a script, e.g., inside a string or regex, could
# otherwise be misinterpreted as history expansion syntax):
echo "Wow!!"
# In an interactive shell, this could trigger unwanted history
# expansion; in a script, it's simply treated as literal text,
# avoiding this exact hazard.
```

---

## HISTSIZE vs HISTFILESIZE Confusion

### Two similarly-named variables controlling two DIFFERENT things
```bash
export HISTSIZE=100
export HISTFILESIZE=10000

# HISTSIZE limits how many commands are kept in MEMORY during the
# CURRENT session — running `history` mid-session shows AT MOST 100 entries

# HISTFILESIZE limits how many LINES the history FILE on disk can
# grow to — this can be MUCH LARGER than HISTSIZE, since the file
# accumulates history across MANY sessions over time, while HISTSIZE
# only governs a single session's in-memory working set

# Setting HISTSIZE small but HISTFILESIZE large is actually a common,
# intentional pattern: keep the ACTIVE session's memory footprint
# modest, while still retaining a much larger LONG-TERM history on disk
history | wc -l
# 100    (capped by HISTSIZE)
wc -l ~/.bash_history
# 8500   (much larger, governed by HISTFILESIZE instead)
```

---

## Different Shells' History Files Aren't Interchangeable

### Switching from bash to zsh (or vice versa) doesn't carry history over automatically
```bash
# bash's history file:
cat ~/.bash_history
# plain list of commands, one per line, NO timestamps by default

# zsh's history file (DIFFERENT format, especially with EXTENDED_HISTORY):
cat ~/.zsh_history
# : 1705320245:0;ls -la
# : 1705320251:0;cd /var/log
# (a completely different format, including timestamps and a
# specific `: timestamp:duration;command` structure)

# Switching your default shell from bash to zsh does NOT automatically
# merge or convert previous bash history into zsh's history file —
# they're separate files with incompatible formats, and zsh won't
# natively understand ~/.bash_history's plain format (or vice versa)
# without manual conversion.
```

---

## !$ and !* Referring to the WRONG Previous Command

### These expand based on the LITERAL previous command, which might not be what you expect
```bash
ls /var/log
history  # (just checking history, doesn't count as a "real" previous command for THIS purpose in some contexts, but generally DOES update the "previous command" pointer)
mkdir !$
# ⚠️ Depending on exactly what ran most recently (including commands
# you might consider "just checking things," like `history` itself),
# !$ expands to the LAST ARGUMENT of whatever ACTUALLY executed right
# before this line — which might be "history" itself (with no useful
# argument) rather than the "/var/log" you were mentally expecting
# from the ls command two steps back.

# Always double check EXACTLY what the immediately preceding command
# was before relying on !$ or !* — or use `history -p '!$'` to PREVIEW
# the expansion before trusting it blindly:
history -p '!$'
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
