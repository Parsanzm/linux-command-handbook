# alias — The Complete Reference

> **Create shortcuts for commands, right inside your shell**
> A shell builtin, not an external program — present in `sh`, `bash`, `zsh`, and most POSIX shells.
> The simplest form of shell customization: turning long, repeated commands into short, memorable ones.

---

## Table of Contents

- [What is alias?](#what-is-alias)
- [Where does alias live?](#where-does-alias-live)
- [How alias works internally](#how-alias-works-internally)
- [Syntax](#syntax)
- [Creating Aliases](#creating-aliases)
- [Viewing Aliases](#viewing-aliases)
- [Removing Aliases (unalias)](#removing-aliases-unalias)
- [Making Aliases Permanent](#making-aliases-permanent)
- [Alias Expansion Rules](#alias-expansion-rules)
- [alias vs function vs script](#alias-vs-function-vs-script)
- [Shell-Specific Differences](#shell-specific-differences)
- [Related Commands](#related-commands)

---

## What is alias?

`alias` lets you define a **shortcut name** for a longer command or command sequence. When you type the alias name, the shell substitutes it with the full command **before** execution — it's a pure text-substitution mechanism, not a separate process or a real command in its own right.

```bash
alias ll='ls -alF'
ll
# runs exactly as if you had typed: ls -alF
```

**Why aliases exist:** they reduce typing for commands you run constantly, encode "safer defaults" into common commands (like always asking for confirmation before `rm`), and let you build a personalized shorthand vocabulary for your own workflow — all without writing an actual script or function.

---

## Where does alias live?

`alias` is a **shell builtin**, not a file on disk like `/usr/bin/ls`. It's implemented directly inside the shell's source code (bash, zsh, dash, etc.), which is why it only affects the **current shell session** by default and isn't visible via `which`:

```bash
type alias
# alias is a shell builtin

which alias
# (nothing, or "no alias in ...", depending on shell — it's not a file)

command -v alias
# alias
```

Because it's a builtin, its behavior can differ slightly between shells (bash vs zsh vs dash) — see [Shell-Specific Differences](#shell-specific-differences) below.

---

## How alias works internally

### Pure text substitution, before parsing

When you type a command, the shell checks if the **first word** matches a defined alias. If so, it literally substitutes the alias's replacement text in place of that word, then continues parsing the (now-expanded) line as if you'd typed it that way from the start.

```bash
alias ll='ls -alF'
ll /tmp
# The shell expands this to: ls -alF /tmp
# Note: /tmp is appended AFTER the alias substitution — arguments you
# type after the alias name are simply tacked onto the end of the
# expanded text, they are NOT inserted into the middle of it.
```

### Aliases are NOT inherited by subshells or scripts

Because alias expansion happens at the interactive shell parsing stage (and is disabled by default in non-interactive bash), aliases defined in your current session:
- **Do NOT** carry over into shell scripts run with `bash script.sh` (non-interactive shells don't expand aliases by default in bash)
- **Do NOT** carry over into a subshell spawned by `bash -c '...'` unless explicitly enabled
- **DO** exist in `source`d scripts within the same interactive session, since sourcing runs in the current shell, not a new process

```bash
alias greet='echo hello'
greet
# hello

bash -c 'greet'
# bash: greet: command not found
# ⚠️ New non-interactive bash process — aliases from your session don't exist there

echo 'greet' > script.sh
bash script.sh
# bash: greet: command not found
# ⚠️ Scripts run as new, non-interactive shells by default — no aliases
```

This is one of the most important practical facts about aliases: **they're for interactive convenience, not for use inside scripts.** Scripts should use functions or full commands instead.

### Alias expansion is chainable (recursive), but not infinitely

```bash
alias ll='ls -la'
alias lla='ll -h'
lla
# expands ll first -> ls -la, then appends -h -> ls -la -h
# Alias expansion re-checks the FIRST word again after substitution,
# so chains of aliases pointing to other aliases work — but the
# shell prevents true infinite loops (an alias can't expand into
# itself as its own first word indefinitely).
```

---

## Syntax

```bash
alias                          # list all currently defined aliases
alias name                     # show the definition of one specific alias
alias name='command'           # define a new alias
alias name1='cmd1' name2='cmd2'  # define multiple aliases in one call
unalias name                   # remove one alias
unalias -a                     # remove ALL aliases in the current session
```

**Quoting matters:** always wrap the replacement command in quotes, especially if it contains spaces, pipes, or special characters.

```bash
alias ll=ls -la          # ❌ WRONG — shell interprets "-la" as a separate command attempt
alias ll='ls -la'        # ✅ correct
```

---

## Creating Aliases

```bash
# Simple shortcut
alias ll='ls -alF'

# Add safety confirmation to destructive commands
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

# Shorten frequently-typed multi-word commands
alias gs='git status'
alias gc='git commit'
alias gp='git push'

# Combine multiple commands with && or ;
alias update='sudo apt update && sudo apt upgrade -y'

# Add default flags to a command every time you run it
alias grep='grep --color=auto'
alias df='df -h'
alias du='du -h'

# Navigate common directories quickly
alias docs='cd ~/Documents'
alias proj='cd ~/projects/current'

# Correct common typos
alias sl='ls'
alias cd..='cd ..'
```

---

## Viewing Aliases

```bash
# List every alias currently defined in this shell session
alias

# Show the definition of one specific alias
alias ll
# alias ll='ls -alF'

# Check whether a given command name is actually an alias
type ll
# ll is aliased to `ls -alF'

# Check if a command is aliased, a function, a builtin, or an actual binary
type cd ll grep
# cd is a shell builtin
# ll is aliased to `ls -alF'
# grep is aliased to `grep --color=auto'
```

---

## Removing Aliases (unalias)

```bash
# Remove one specific alias
unalias ll

# Remove ALL aliases defined in the current session
unalias -a

# Temporarily bypass an alias for ONE invocation, without removing it —
# prefixing with a backslash skips alias lookup entirely
\rm file.txt          # runs the REAL rm, ignoring any `alias rm='rm -i'`
"rm" file.txt          # quoting also bypasses alias expansion
command rm file.txt    # the "command" builtin also bypasses aliases and functions
```

---

## Making Aliases Permanent

Aliases defined directly on the command line only last for the **current shell session** — they disappear when you close the terminal. To make them permanent, add them to your shell's startup/config file:

```bash
# Bash — typically one of these, depending on distro and login type:
~/.bashrc          # interactive non-login shells (most terminal windows)
~/.bash_profile    # login shells (some systems, especially macOS Terminal.app)
~/.bash_aliases    # dedicated aliases file, sourced FROM .bashrc on many distros

# Zsh:
~/.zshrc

# Add an alias permanently
echo "alias ll='ls -alF'" >> ~/.bashrc

# Apply changes to your CURRENT session without closing the terminal
source ~/.bashrc
# or the shorthand:
. ~/.bashrc
```

### Organizing many aliases in a dedicated file

```bash
# ~/.bashrc typically contains something like this near the top or bottom:
if [ -f ~/.bash_aliases ]; then
    . ~/.bash_aliases
fi

# Then keep all your aliases neatly separated in ~/.bash_aliases
cat ~/.bash_aliases
# alias ll='ls -alF'
# alias gs='git status'
# alias update='sudo apt update && sudo apt upgrade -y'
```

---

## Alias Expansion Rules

### Only the first word of a command is checked
```bash
alias ls='ls --color=auto'
echo "ls is great"
# Does NOT expand — "ls" here is just an argument to echo, not the
# first word of the command being run, so alias substitution never applies.
```

### A trailing space enables the NEXT word to also be alias-expanded
```bash
alias sudo='sudo '     # note the trailing space
alias ll='ls -alF'
sudo ll
# The trailing space after "sudo" tells bash to ALSO check the next
# word for alias expansion — so "sudo ll" correctly expands to
# "sudo ls -alF" instead of failing with "ll: command not found"
# under sudo's own (aliasless) environment.

alias sudo='sudo'      # WITHOUT trailing space
sudo ll
# sudo: ll: command not found
# ⚠️ Without the trailing space, only "sudo" itself is checked for
# expansion (and finds none), so "ll" is passed through literally
# to sudo, which has no idea what "ll" means.
```

### Aliases don't accept "parameters" in the middle
```bash
alias backup='tar -czf'
backup archive.tar.gz myfolder/
# Expands to: tar -czf archive.tar.gz myfolder/
# This works because tar's argument ORDER happens to fit — but there's
# no way to make an alias insert an argument into the MIDDLE of a
# command, or reorder arguments. For that, you need a FUNCTION instead:
backup() { tar -czf "$1.tar.gz" "$2"; }
```

---

## alias vs function vs script

| Feature | `alias` | shell `function` | script file |
|---------|---------|-------------------|-------------|
| Accepts positional arguments (`$1`, `$2`) meaningfully | ❌ No — args just append to the end | ✅ Yes | ✅ Yes |
| Can contain conditional logic (`if`, loops) | ❌ No | ✅ Yes | ✅ Yes |
| Persists across new shells automatically | ❌ No (needs sourcing) | ❌ No (needs sourcing) | ✅ Yes (executable file, run directly) |
| Available inside scripts by default | ❌ No | ✅ Yes, if defined/exported appropriately | ✅ Yes, it IS the script |
| Simplicity for a quick shortcut | ✅ Simplest | ⚠️ Slightly more setup | ⚠️ Most setup (file, permissions, PATH) |
| Can be piped into cleanly like a real command | ⚠️ Usually fine, but with caveats | ✅ Yes | ✅ Yes |

```bash
# alias — fine for a static shortcut
alias ll='ls -alF'

# function — needed when arguments must be used non-trivially
mkcd() { mkdir -p "$1" && cd "$1"; }
mkcd new_project    # creates AND enters the directory — impossible with a plain alias

# script — needed for anything reusable across machines, shareable, or
# run non-interactively (cron, CI, other users)
#!/bin/bash
# ~/bin/mkcd.sh
mkdir -p "$1" && cd "$1"
```

**Rule of thumb:** use `alias` for static shortcuts with no logic; use a `function` the moment you need to reference arguments (`$1`, `$2`) inside the command, use conditionals, or run multiple sequential steps that depend on input; use a full script when it needs to run outside your interactive shell (cron, another user, another machine, CI).

---

## Shell-Specific Differences

| Behavior | bash | zsh | dash/sh (POSIX) |
|----------|------|-----|------------------|
| Aliases expand in non-interactive shells by default | ❌ No (unless `shopt -s expand_aliases`) | ✅ Yes, more permissive | ❌ No, and aliases are often unsupported entirely in scripts |
| Suffix aliases (alias tied to a file extension) | ❌ Not supported | ✅ Supported (`alias -s txt=cat`) | ❌ Not supported |
| Global aliases (expand anywhere in the line, not just first word) | ❌ Not supported | ✅ Supported (`alias -g`) | ❌ Not supported |
| `unalias -a` | ✅ Supported | ✅ Supported | ⚠️ Varies |

```bash
# zsh-specific: suffix alias — running a filename with this extension
# directly invokes the associated command
alias -s txt='cat'
myfile.txt
# in zsh, this actually runs: cat myfile.txt

# zsh-specific: global alias — substituted ANYWHERE on the line, not
# just as the first word
alias -g G='| grep'
ps aux G firefox
# expands to: ps aux | grep firefox
```

`dash` (Debian's default `/bin/sh`) has minimal or no interactive alias support in script contexts at all — another reason aliases should never be relied upon inside portable shell scripts.

---

## Related Commands

| Command | Relation |
|---------|----------|
| `unalias` | Remove a defined alias |
| `type` | Determine whether a name is an alias, function, builtin, or binary |
| `function` (or `name() { }`) | More powerful alternative that supports arguments and logic |
| `command` | Bypass alias/function lookup and run the real underlying binary |
| `.bashrc` / `.zshrc` | Where persistent aliases are typically defined |
| `source` (`.`) | Reload a config file's aliases into the current shell |
| `hash` | Shell builtin for caching resolved command paths (related but distinct concept) |
| `export -f` | Export a function (not an alias) to be usable in subshells |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
