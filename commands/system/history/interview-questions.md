# history — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage & Expansion](#basic-usage--expansion)
- [Configuration Variables](#configuration-variables)
- [Multi-Terminal Behavior](#multi-terminal-behavior)
- [Security](#security)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What is the history command, and is it a builtin or an external program?**
> `history` is a **shell builtin** that displays a numbered list of previously executed commands from the current (and, depending on configuration, prior) sessions. It's not a standalone file on disk — it's implemented directly inside the shell itself.

---

**Q2 🔥 Where is persisted command history typically stored for bash?**
> `~/.bash_history` by default — a plain text file containing previously executed commands, one per line (without timestamps, unless additional configuration like `HISTTIMEFORMAT` is set for display purposes, though the underlying file format itself remains simple).

---

**Q3. Does bash write each command to the history file immediately as it's typed?**
> Not by default — bash typically keeps the session's history in **memory** and only writes it out to the history file when the shell exits **normally**. Commands from a still-open terminal session aren't necessarily reflected in the file yet, and an abrupt crash or `kill -9` can lose that session's entire unsaved history.

---

## Basic Usage & Expansion

**Q4 🔥 What does `!!` do, and give a common real-world use case.**
> `!!` re-executes the most recently run command. A very common pattern is `sudo !!` — used when you forgot to prefix a command with `sudo` and it failed due to insufficient permissions, allowing you to instantly retry it with elevated privileges without retyping the whole command.

---

**Q5. What does `!$` expand to?**
> The **last argument** of the previous command. For example, after `mkdir /tmp/project`, running `cd !$` expands to `cd /tmp/project`.

---

**Q6 🔥 How would you re-run a specific command from history by its number, and how do you find that number?**
> Run `history` (optionally piped through `grep` to narrow it down) to find the command's number, then use `!N` (e.g., `!512`) to re-execute that specific numbered entry.

---

**Q7. What does the quick substitution syntax `^old^new` do?**
> It re-runs the previous command with the first occurrence of "old" replaced by "new" — useful for quickly fixing a typo without retyping the entire command, e.g., `^googl^google` after mistyping `ping googl.com`.

---

## Configuration Variables

**Q8 🔥 What's the difference between HISTSIZE and HISTFILESIZE?**
> `HISTSIZE` controls how many commands are kept in memory during the current session. `HISTFILESIZE` controls how many lines the history file on disk can grow to — these can differ significantly, since the file accumulates history across many sessions over time while `HISTSIZE` only governs a single session's working set.

---

**Q9. What does setting `HISTCONTROL=ignoredups` do, and what about `ignorespace`?**
> `ignoredups` prevents a command from being saved to history if it's identical to the immediately preceding entry (avoiding repetitive clutter from re-running the same command multiple times in a row). `ignorespace` prevents a command from being saved at all if it starts with a literal leading space — commonly used to deliberately keep sensitive commands (e.g., ones containing a password) out of history.

---

**Q10 🔥 How would you exclude common, low-value commands like `ls`, `cd`, and `pwd` from ever being recorded in history?**
> ```bash
> export HISTIGNORE="ls:cd:pwd:clear:history"
> ```
> `HISTIGNORE` takes a colon-separated list of patterns; any command matching one of these patterns is excluded from being saved to history at all.

---

## Multi-Terminal Behavior

**Q11 🔥 If you have two terminal windows open simultaneously, does a command run in one immediately appear in the other's history?**
> Not by default. Each terminal maintains its own in-memory history, and bash only writes a session's accumulated history to the shared file when that specific session exits normally — so a second, still-open terminal has no way to see the first terminal's commands until one of them closes (or explicit synchronization is configured).

---

**Q12. How would you configure bash so that multiple open terminals share history updates in near real time?**
> Add to `~/.bashrc`:
> ```bash
> export PROMPT_COMMAND="history -a; history -c; history -r; $PROMPT_COMMAND"
> ```
> This appends new history to the file (`-a`), clears the session's in-memory copy (`-c`), and re-reads the full file back into memory (`-r`) before every new prompt — effectively syncing history across terminals continuously rather than only at shell exit.

---

## Security

**Q13 🔥 What's a genuine security risk of typing a password directly as a command-line argument (e.g., `mysql -u root -pMyPassword`)?**
> The password gets recorded in **plain text** in the shell's history file (`~/.bash_history`), persisting long after the command finishes — readable by anyone with access to that file (via backups, other processes, or later unauthorized access to the account), unlike the brief, momentary exposure of appearing in `ps aux` output while the command is actually running.

---

**Q14. How would you prevent one specific sensitive command from ever being recorded in history, assuming HISTCONTROL is configured appropriately?**
> Type the command with a literal **leading space** before it: ` mysql -u root -pSecret db` (note the space before "mysql"). With `HISTCONTROL` set to `ignorespace` or `ignoreboth`, a command starting with a space is excluded from history entirely.

---

**Q15 🔥 If a sensitive command was already accidentally recorded in history, how do you remove it, including from the persisted file on disk?**
> ```bash
> history -d N    # delete entry number N from the in-memory session history
> history -w      # write out the current (now-corrected) history, overwriting the file
> ```
> Simply running `history -d` alone only affects the in-memory copy; `-w` is needed afterward to ensure the change is actually persisted to the history file on disk.

---

## Scenario-Based

**Q16 🔥 A user's terminal application crashes unexpectedly after a long troubleshooting session full of useful commands. They check `~/.bash_history` afterward and none of that session's commands are there. What happened, and how could this have been prevented?**
> By default, bash only writes accumulated in-memory history to the history file upon a **normal** shell exit — an abrupt crash or forced kill skips that flush entirely, losing the whole session's history. This can be prevented by configuring history to append immediately after every command rather than waiting for exit: `export PROMPT_COMMAND="history -a; $PROMPT_COMMAND"` in `~/.bashrc`, which flushes each command to disk right after it runs.

---

**Q17. Someone runs a destructive command in the wrong directory, immediately realizes the mistake, and instinctively types `sudo !!` to "fix" a permission error. What's the actual risk here, and what should they do instead?**
> `sudo !!` re-executes the **exact same** previous command, now with root privileges — if that previous command was itself the destructive mistake (rather than a legitimate permission-denied error), this can make the damage significantly worse by removing any permission-based limitation that might have contained the original mistake's scope. Before blindly using `!!` (especially combined with `sudo`), it's safer to preview the expansion first with `history -p '!!'` to confirm exactly what would be re-executed.

---

**Q18 🔥 A script explicitly calls `history` and history expansion (`!!`) to try to reuse a previous command, but this doesn't work as expected inside the script. Why?**
> `history` and history expansion are fundamentally **interactive shell** features; non-interactive script execution doesn't maintain the same kind of command history, and history expansion is typically disabled by default in non-interactive bash for safety (preventing an unrelated literal `!` character in a script's text, such as inside a string, from being unexpectedly interpreted as history expansion syntax).

---

**Q19. A user sets `HISTCONTROL=ignorespace` and carefully types a leading space before a command containing a password, but the password still shows up in history afterward. What are two possible explanations?**
> (1) The leading space wasn't actually the very first character as typed — some terminal integrations, auto-indentation, or copy-pasting from a source that stripped leading whitespace can silently remove the intended space before the shell ever sees it. (2) `HISTCONTROL` might not have actually been set to include `ignorespace` in the CURRENT session (e.g., it was added to `.bashrc` but the shell wasn't restarted/re-sourced, or a competing `HISTCONTROL` value was set elsewhere later in the startup sequence, overriding it). Verifying with `echo $HISTCONTROL` and checking `history | tail -1` immediately after typing the sensitive command confirms whether the protection is actually active.

---

**Q20 🔥 After running `history -d 500` to remove a sensitive entry, the user is surprised to find the entry still present in `~/.bash_history` after closing and reopening their terminal. Why, and what step was missing?**
> `history -d` only removes the entry from the **in-memory** session history — it does not automatically rewrite the persisted history file on disk. Without following it with `history -w` (which overwrites the history file with the current, now-corrected in-memory list), the shell's normal exit-flush behavior can still result in the sensitive entry persisting in the file. The correct sequence is `history -d 500` followed immediately by `history -w`.

---

**Q21. Two terminals are open, and a user runs `history -a` in Terminal A expecting Terminal B to immediately show A's new commands, but Terminal B's `history` output remains unchanged. What's missing?**
> `history -a` only **appends** Terminal A's new entries to the shared history file — it doesn't automatically push that update into Terminal B's own in-memory history. Terminal B also needs to explicitly **read** the updated file back into its own memory, typically via `history -r`, for the newly appended entries to actually appear in Terminal B's `history` output. Full near-real-time sharing between terminals requires both sides cooperating (commonly automated via a `PROMPT_COMMAND` that runs both `-a` and `-r` before every prompt).

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
