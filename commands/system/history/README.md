# history — The Complete Reference

> **See, search, and reuse your previously typed shell commands**
> A shell builtin present in bash, zsh, ksh, and most interactive shells since the early 1980s.
> The tool behind muscle-memory shortcuts like `!!` and Ctrl+R that every terminal user relies on daily.

---

## Table of Contents

- [What is history?](#what-is-history)
- [Where does history live?](#where-does-history-live)
- [How history works internally](#how-history-works-internally)
- [Syntax](#syntax)
- [Basic Usage](#basic-usage)
- [History Expansion (!!, !n, !string)](#history-expansion---n-string)
- [Searching History Interactively (Ctrl+R)](#searching-history-interactively-ctrlr)
- [All Key Options and Related Variables](#all-key-options-and-related-variables)
- [HISTSIZE, HISTFILESIZE, and the History File](#histsize-histfilesize-and-the-history-file)
- [HISTCONTROL and Ignoring Duplicates/Space-Prefixed Commands](#histcontrol-and-ignoring-duplicatesspace-prefixed-commands)
- [Multiple Terminals and History Sharing](#multiple-terminals-and-history-sharing)
- [Removing Sensitive Commands from History](#removing-sensitive-commands-from-history)
- [history vs Shell-Specific Alternatives](#history-vs-shell-specific-alternatives)
- [Related Commands](#related-commands)

---

## What is history?

`history` is a shell builtin that displays a numbered list of previously executed commands from the current session (and, depending on configuration, from prior sessions too). It's the foundation for a huge amount of everyday terminal productivity — re-running a previous command, correcting a typo without retyping everything, or searching back through what you did five minutes (or five days) ago.

```bash
history
#   501  cd /var/log
#   502  ls -la
#   503  grep ERROR app.log
#   504  history

!503
# re-runs: grep ERROR app.log

!!
# re-runs the MOST RECENT command
```

**Why history matters beyond convenience:** in a long troubleshooting or admin session, being able to quickly recall, search, and re-execute (or slightly modify) a command you ran minutes or hours ago is one of the biggest everyday productivity boosts a shell provides — most experienced command-line users rely on history-based shortcuts (`!!`, `Ctrl+R`, `!$`) constantly, often without consciously thinking about it.

---

## Where does history live?

`history` is a **shell builtin**, not an external program:

```bash
type history
# history is a shell builtin

which history
# (nothing — it's not a file on disk)
```

The **persisted** history (commands remembered across sessions) is stored in a plain text file, typically:

```
~/.bash_history      (bash)
~/.zsh_history       (zsh)
~/.history           (some other shells/configurations)
```

```bash
tail -20 ~/.bash_history
# shows the last 20 commands saved from PREVIOUS sessions
```

---

## How history works internally

### In-memory list during the session, flushed to disk on exit (bash default)

While a shell session is active, bash keeps the command history in **memory**. By default, this in-memory list is only written out to the history file (`~/.bash_history`) when the shell session **exits normally** — meaning commands from a still-open terminal aren't necessarily visible in the history file yet, and a crashed or forcibly-killed terminal may lose that session's history entirely.

```bash
# Terminal A (still open, several commands typed):
ls
pwd
cd /tmp

# Terminal B (opened AFTER terminal A, checking the history FILE):
tail -5 ~/.bash_history
# (does NOT show terminal A's "ls", "pwd", "cd /tmp" yet, because
# terminal A hasn't exited and flushed its in-memory history to disk)
```

### Forcing an immediate flush without closing the shell

```bash
history -a
# Appends the CURRENT session's new history entries to the history
# file immediately, without needing to close the shell — useful when
# you want another terminal to be able to see commands from THIS
# session right away.

history -r
# Re-READS the history file into the current session's memory —
# combined with `history -a` in another terminal, this is how some
# people set up NEAR-real-time history sharing between multiple open terminals.
```

### Numbering is session-relative, and shifts as history grows

```bash
history
#   498  ls
#   499  cd /tmp
#   500  vim file.txt
# The numbers shown are simply the POSITION in the accumulated
# history list, not a permanent, stable ID — as more commands run
# (and older ones potentially age out once HISTSIZE's limit is
# reached), these numbers shift accordingly.
```

---

## Syntax

```bash
history [OPTIONS] [N]
```

```bash
history          # show ALL history (up to HISTSIZE limit)
history 20        # show just the last 20 entries
history -c         # clear the CURRENT session's in-memory history
history -d N       # delete a SPECIFIC entry by its number
history -a         # append current session's new entries to the history file
history -w         # write the ENTIRE current history to the file (overwriting)
```

---

## Basic Usage

```bash
# Show full history
history

# Show just the last 10 commands
history 10

# Search history using grep (a very common combo)
history | grep ssh

# Find the most recent command matching a pattern
history | grep docker | tail -1

# Clear the CURRENT session's history (in memory only, by default)
history -c
```

---

## History Expansion (!!, !n, !string)

Bash (and most shells) support special **history expansion** syntax using `!`, letting you reference and re-run previous commands without retyping them:

| Expansion | Meaning |
|-----------|---------|
| `!!` | Re-run the MOST RECENT command |
| `!n` | Re-run command number `n` from history |
| `!-n` | Re-run the command `n` positions BACK from the current one |
| `!string` | Re-run the MOST RECENT command that STARTS WITH `string` |
| `!?string?` | Re-run the MOST RECENT command CONTAINING `string` anywhere |
| `!$` | Just the LAST ARGUMENT of the previous command |
| `!*` | ALL arguments of the previous command |
| `^old^new` | Quick substitution: re-run the previous command, replacing "old" with "new" |

```bash
ls /var/log
!!
# re-runs: ls /var/log

sudo !!
# a VERY common pattern: forgot to prefix a command with sudo, so run
# `sudo` followed by `!!` to re-execute the JUST-FAILED command with
# root privileges this time

mkdir /tmp/newproject
cd !$
# !$ expands to just the last argument of the PREVIOUS command
# ("/tmp/newproject"), so this becomes: cd /tmp/newproject

history | grep docker
!502
# re-run whichever specific history NUMBER matched what you were
# looking for

grep ERROR /var/log/app.log
!grep
# re-runs the MOST RECENT command starting with "grep"

echo hello world
^hello^goodbye
# quickly re-runs the previous command with "hello" replaced by
# "goodbye": echo goodbye world
```

---

## Searching History Interactively (Ctrl+R)

```bash
# Press Ctrl+R in an interactive shell to start a REVERSE INCREMENTAL
# search through history:
# (reverse-i-search)`ssh': ssh user@server.example.com
# Type characters to narrow the search; press Ctrl+R again to cycle
# to the NEXT older match; press Enter to execute the found command,
# or Right/Left arrow to edit it first without running it immediately.

# Ctrl+S can search FORWARD (toward more recent matches) in some
# configurations, though it's often intercepted by terminal flow
# control (XON/XOFF) unless explicitly disabled:
stty -ixon
# (disables terminal flow control, freeing up Ctrl+S for forward
# history search in bash)
```

---

## All Key Options and Related Variables

| Option | Description |
|--------|-------------|
| `-c` | Clear the current session's history (in memory) |
| `-d OFFSET` | Delete the history entry at position OFFSET |
| `-a` | Append new history lines from this session to the history file |
| `-w` | Write the current history out to the history file (overwrite) |
| `-r` | Read the history file, appending its contents to the current session's history |
| `-n` | Read history lines not already read from the history file |
| `-p ARG` | Perform history expansion on ARG(s) and display the result, without executing |
| `-s ARG` | Add ARG to the history list as a new entry, WITHOUT executing it |

```bash
history -c                       # clear current session's history
history -d 500                    # delete entry number 500 specifically
history -w                        # force-write current history to disk NOW
history -p '!!'                   # preview what !! would expand to, without running it
history -s "echo test"            # manually inject a fake entry into history
```

---

## HISTSIZE, HISTFILESIZE, and the History File

```bash
# HISTSIZE: max number of commands kept in MEMORY for the current session
echo $HISTSIZE
# 1000

# HISTFILESIZE: max number of lines kept in the HISTORY FILE on disk
echo $HISTFILESIZE
# 2000

# Increase these limits (commonly set in ~/.bashrc for a persistent change)
export HISTSIZE=5000
export HISTFILESIZE=10000

# Effectively UNLIMITED history (use with some caution — a truly
# enormous history file can slow down shell startup slightly)
export HISTSIZE=-1
export HISTFILESIZE=-1

# Change WHERE the history file is stored
export HISTFILE=~/.custom_history_location
```

---

## HISTCONTROL and Ignoring Duplicates/Space-Prefixed Commands

```bash
# HISTCONTROL determines what gets EXCLUDED from history at all

# ignoredups: don't save a command if it's IDENTICAL to the
# immediately previous one
export HISTCONTROL=ignoredups

# ignorespace: don't save a command if it starts with a LEADING SPACE
# — a classic trick for keeping a specific command (often one
# containing a password or sensitive token) OUT of history entirely
export HISTCONTROL=ignorespace
 mysql -u root -pSuperSecretPassword    # note the LEADING SPACE
history | tail -1
# (this command does NOT appear, precisely because of that leading space)

# ignoreboth: combines BOTH ignoredups AND ignorespace together
export HISTCONTROL=ignoreboth

# Also commonly paired with HISTIGNORE, to exclude SPECIFIC commonly-
# repeated, low-value commands from history entirely
export HISTIGNORE="ls:cd:pwd:history:clear"
```

---

## Multiple Terminals and History Sharing

```bash
# By DEFAULT, each terminal's history is only written to the shared
# FILE when that specific terminal session EXITS — meaning multiple
# simultaneously open terminals do NOT automatically see each other's
# commands in real time.

# A common approach to make history shared/synced more IMMEDIATELY
# across multiple open terminals (add to ~/.bashrc):
export PROMPT_COMMAND="history -a; history -c; history -r; $PROMPT_COMMAND"
# Explanation:
#   history -a : append this session's new commands to the file
#   history -c : clear this session's in-memory list
#   history -r : re-read the FULL file (including other terminals'
#                newly appended commands) back into memory
# Run before EVERY prompt, this creates near-real-time history
# sharing across multiple simultaneously open terminal windows.
```

---

## Removing Sensitive Commands from History

```bash
# Delete a SPECIFIC entry by its history number
history
#   455  curl -H "Authorization: Bearer abc123secret" https://api.example.com
history -d 455
history -w    # persist the deletion to the history FILE too, not just memory

# Prevent a sensitive command from EVER being recorded in the first
# place, using a leading space (requires HISTCONTROL to include
# "ignorespace" or "ignoreboth" — see above)
 export API_KEY="supersecret123"
# (leading space means this line is never saved to history at all)

# Temporarily DISABLE history recording entirely for a sensitive block
set +o history
# ... run sensitive commands here ...
set -o history
# (re-enables history recording afterward)
```

---

## history vs Shell-Specific Alternatives

| Shell | History file | Notable difference |
|-------|----------------|--------------------------|
| bash | `~/.bash_history` | In-memory during session, flushed on exit by default |
| zsh | `~/.zsh_history` | Includes TIMESTAMPS by default (with `EXTENDED_HISTORY` option); often shares history across terminals more readily via `SHARE_HISTORY` |
| fish | `~/.local/share/fish/fish_history` | Stores history in a structured YAML-like format; provides rich autosuggestions based on history natively |

```bash
# zsh's extended history format includes a timestamp per entry
setopt EXTENDED_HISTORY
cat ~/.zsh_history
# : 1705320245:0;ls -la
# (the number is a Unix timestamp, unlike bash's plain command-only format)

# zsh's SHARE_HISTORY option makes multiple terminals share history
# updates much more seamlessly than bash's default behavior
setopt SHARE_HISTORY
```

---

## Related Commands

| Command | Relation |
|---------|----------|
| `fc` | "Fix command" — a more powerful, POSIX-standard way to edit and re-run history entries in an actual editor |
| `!!` / `!n` / `!$` | History expansion syntax, closely tied to the history builtin's numbering |
| `Ctrl+R` | Interactive reverse search through history (readline feature, not the `history` command itself) |
| `HISTCONTROL` / `HISTIGNORE` | Environment variables controlling WHAT gets recorded into history |
| `HISTSIZE` / `HISTFILESIZE` | Environment variables controlling HOW MUCH history is retained |
| `.bash_history` | The default persisted history file bash writes to/reads from |
| `set -o history` / `set +o history` | Toggle history recording on/off entirely for the current session |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
