# alias — Edge Cases & Gotchas

> Aliases feel simple, but their quirks around scripts, quoting, sudo, and
> shell startup files trip up beginners and experienced users alike.

---

## Table of Contents

- [Aliases Don't Work in Scripts by Default](#aliases-dont-work-in-scripts-by-default)
- [Aliasing a Command with sudo Needs a Trailing Space](#aliasing-a-command-with-sudo-needs-a-trailing-space)
- [Aliases Can't Take Positional Arguments Meaningfully](#aliases-cant-take-positional-arguments-meaningfully)
- [Recursive/Self-Referencing Aliases](#recursiveself-referencing-aliases)
- [Overriding a Built-in or Common Command Silently](#overriding-a-built-in-or-common-command-silently)
- [Quoting Mistakes When Defining Aliases](#quoting-mistakes-when-defining-aliases)
- [Aliases Only Match the First Word of a Command](#aliases-only-match-the-first-word-of-a-command)
- [.bashrc vs .bash_profile Confusion](#bashrc-vs-bash_profile-confusion)
- [Aliases Silently Shadowing Functions (or Vice Versa)](#aliases-silently-shadowing-functions-or-vice-versa)
- [Forgetting to source After Editing](#forgetting-to-source-after-editing)
- [Aliases Are Not Exported to Child Processes](#aliases-are-not-exported-to-child-processes)
- [unalias -a Removing Aliases You Forgot You Depended On](#unalias--a-removing-aliases-you-forgot-you-depended-on)
- [Alphabetical/Load-Order Conflicts Across Multiple Config Files](#alphabeticalload-order-conflicts-across-multiple-config-files)

---

## Aliases Don't Work in Scripts by Default

### A script silently fails to recognize an alias defined in your interactive shell
```bash
# In your interactive terminal:
alias greet='echo "Hello there"'
greet
# Hello there

# In a script file, run non-interactively:
cat > test.sh << 'EOF'
#!/bin/bash
greet
EOF
bash test.sh
# test.sh: line 2: greet: command not found
# ⚠️ Non-interactive bash shells do NOT expand aliases by default,
# even if the SAME alias is active in the shell that launched the script.

# Fix #1: explicitly enable alias expansion in non-interactive bash
# (must appear BEFORE the alias definition in the script)
cat > test.sh << 'EOF'
#!/bin/bash
shopt -s expand_aliases
alias greet='echo "Hello there"'
greet
EOF
bash test.sh
# Hello there

# Fix #2 (much more common/idiomatic): just use a FUNCTION instead,
# which works reliably inside scripts without any special flags:
greet() { echo "Hello there"; }
greet
```

---

## Aliasing a Command with sudo Needs a Trailing Space

### Without the trailing space, aliases after sudo silently fail to expand
```bash
alias ll='ls -alF'
alias sudo='sudo'          # ⚠️ no trailing space
sudo ll
# sudo: ll: command not found
# sudo runs "ll" in its OWN context, which has no idea "ll" is an alias
# in YOUR shell — and without the trailing space, bash doesn't even
# attempt to alias-expand the word immediately following "sudo".

alias sudo='sudo '         # ✅ WITH a trailing space
sudo ll
# correctly expands to: sudo ls -alF

# This one-character difference (a trailing space) is one of the most
# commonly misunderstood alias quirks — it specifically tells bash to
# ALSO check the next word for alias expansion, not just "sudo" itself.
```

---

## Aliases Can't Take Positional Arguments Meaningfully

### Arguments only ever get appended to the END, never inserted or reordered
```bash
alias greet='echo "Hello,"'
greet World
# expands to: echo "Hello," World
# → Hello, World
# This "works" only because appending happens to read naturally here.

# But this does NOT work as many beginners expect:
alias backup='tar -czf $1.tar.gz'
backup myfolder
# $1 is NOT interpreted as "myfolder" — inside an alias definition,
# $1 either expands to whatever was in $1 when the ALIAS was DEFINED
# (usually empty/unset), not a placeholder for arguments given later.
echo $?
# often results in a nonsensical filename, or an outright error

# Fix: use a function, where $1 genuinely refers to the first argument
# passed WHEN THE FUNCTION IS CALLED:
backup() { tar -czf "$1.tar.gz" "$2"; }
backup myfolder myfolder/
```

---

## Recursive/Self-Referencing Aliases

### Naming an alias the same as the command it wraps requires care
```bash
alias ls='ls --color=auto'
ls
# Works fine — bash detects that further expanding "ls" (the alias's
# OWN replacement text) as an alias again would be infinite, and stops
# after one level, running the REAL /bin/ls with --color=auto appended.

# But chaining through OTHER aliases can create confusing (though not
# infinite) expansion chains that are hard to trace:
alias grep='grep --color=auto'
alias search='grep -r'
search "pattern" .
# expands to: grep -r "pattern" .   (search expands to grep -r, but
# note that grep itself is ALSO aliased — however, only the FIRST
# word of a freshly-expanded line gets re-checked, so whether the
# "grep" inside search's expansion get RE-aliased again depends on
# shell/version behavior — don't rely on deep alias chains for
# anything where the exact resulting command matters.)
```

---

## Overriding a Built-in or Common Command Silently

### Aliasing a core command can hide real errors from other tools/scripts
```bash
alias rm='rm -i'
# Feels safer interactively, but:

find /tmp -name "*.tmp" -exec rm {} \;
# find calls the REAL /bin/rm directly (find doesn't go through your
# shell's alias table at all — it execs the binary), so the -i
# confirmation NEVER appears here regardless of your alias.
# This can create a false sense of safety: you assume "rm always asks
# me first" because that's true when YOU type rm interactively, but
# it's NOT true for anything invoked by another program or script.

# The alias only protects INTERACTIVE, direct typing of "rm" —
# it provides zero protection against automated/scripted rm calls.
```

---

## Quoting Mistakes When Defining Aliases

### Unquoted or improperly quoted definitions behave unexpectedly
```bash
alias greet=echo hello
# bash: alias: hello: not found
# ⚠️ Without quotes, only "echo" is treated as the alias's value;
# "hello" is interpreted as a SEPARATE command to the alias builtin itself

alias greet='echo hello'
# ✅ correct — the entire replacement text is one quoted string

# Nested quotes need care:
alias say='echo "It's a nice day"'
# ⚠️ The apostrophe in "It's" PREMATURELY closes the outer single-quote,
# breaking the alias definition in a confusing way

alias say="echo \"It's a nice day\""
# ✅ using double quotes outside, with escaped inner double quotes, works

alias say='echo "It'"'"'s a nice day"'
# ✅ also works, but is famously hard to read — the escaped-double-quote
# version above is usually clearer for cases involving apostrophes
```

---

## Aliases Only Match the First Word of a Command

### Putting an alias name anywhere else in a line does nothing
```bash
alias ll='ls -alF'
echo "run ll to see files"
# Does NOT expand "ll" here — it's just text inside an echo argument,
# not the first word of the command being executed.

# Piped commands: EACH command in the pipeline gets its own first-word check
ll | grep txt
# "ll" expands (first word of first command); "grep" would ALSO be
# checked for alias expansion since it's the first word of the SECOND
# command in the pipeline — every command position after a pipe,
# semicolon, &&, or || is independently eligible for alias expansion.
```

---

## .bashrc vs .bash_profile Confusion

### An alias added to the "wrong" file doesn't load where you expect
```bash
# Added an alias here:
echo "alias ll='ls -alF'" >> ~/.bash_profile

# Then opened a NEW terminal window (typically an interactive
# NON-LOGIN shell on many Linux desktop environments):
ll
# bash: ll: command not found
# ⚠️ .bash_profile is usually only read for LOGIN shells (e.g., a
# fresh SSH session, or the very first shell after logging into a
# text console) — most graphical terminal emulators open
# non-login interactive shells, which read ~/.bashrc instead.

# Fix: put aliases in ~/.bashrc (or a file it sources, like
# ~/.bash_aliases), OR have .bash_profile explicitly source .bashrc:
echo '[ -f ~/.bashrc ] && . ~/.bashrc' >> ~/.bash_profile

# macOS Terminal.app is a notable exception — it traditionally opens
# LOGIN shells by default for every new window, the opposite of
# most Linux terminal emulators, which is a frequent source of
# cross-platform dotfile confusion.
```

---

## Aliases Silently Shadowing Functions (or Vice Versa)

### Defining both an alias and a function with the same name creates ambiguity
```bash
ll() { ls -la "$@"; }
alias ll='ls -alF'
ll
# Bash generally gives ALIASES priority over functions when both exist
# with the same name in the same interactive session — meaning the
# alias silently "wins," and your function is never actually called,
# with no error or warning that this happened.

type ll
# ll is aliased to `ls -alF'
# (the function is still defined internally, just never reached)

# Always check with `type` if a command isn't behaving as your function
# defines it — an unnoticed alias with the same name is a common cause.
```

---

## Forgetting to source After Editing

### Editing ~/.bashrc doesn't affect the currently running shell automatically
```bash
echo "alias newcmd='echo test'" >> ~/.bashrc
newcmd
# bash: newcmd: command not found
# ⚠️ The file was edited, but THIS shell process already finished
# reading .bashrc when it first started — it has no way of knowing
# the file changed afterward.

source ~/.bashrc
# or: . ~/.bashrc
newcmd
# test    ✅ now it works, because .bashrc was explicitly re-read

# A brand new terminal window opened AFTER the edit would also work
# correctly without needing an explicit `source`, since it reads the
# updated file fresh when it starts.
```

---

## Aliases Are Not Exported to Child Processes

### Even "export"-like syntax doesn't make aliases available elsewhere
```bash
alias greet='echo hello'
export -f greet
# bash: export: `-f': option not valid for aliases (functions only)
# There is NO equivalent of "export" for aliases — they are fundamentally
# tied to the interactive shell session that defined them and cannot be
# passed down to child processes, subshells, or other terminals at all.

# Functions CAN be exported (bash-specific):
greet() { echo hello; }
export -f greet
bash -c 'greet'
# hello   ✅ works, because functions (unlike aliases) support export -f
```

---

## unalias -a Removing Aliases You Forgot You Depended On

### A blanket reset can break workflow muscle memory unexpectedly
```bash
# After years of using ll, gs, gc as second nature...
unalias -a
# ALL aliases in the current session are gone immediately, with no
# confirmation prompt and no easy "undo" short of re-sourcing your
# config file.

ll
# bash: ll: command not found  ← muscle memory betrays you mid-task

# Recovery is simple IF your aliases are defined in a config file:
source ~/.bashrc
# But if any aliases were defined ad-hoc directly in THIS session
# (never saved to a file), they are permanently gone until manually
# retyped.
```

---

## Alphabetical/Load-Order Conflicts Across Multiple Config Files

### The LAST definition read wins, silently overriding earlier ones
```bash
# ~/.bashrc:
alias ll='ls -alF'

# ~/.bash_aliases (sourced AFTER .bashrc's own definitions, later in the file):
alias ll='ls -la --group-directories-first'

# The second definition silently overrides the first — whichever file
# is sourced LAST determines the final behavior, with no warning that
# a name was already taken by an earlier file.

alias ll
# alias ll='ls -la --group-directories-first'
# (the .bashrc version is completely gone, without any error message)

# When debugging "why doesn't my alias do what I defined," always check
# EVERY config file that might define the same name, in the order
# they're actually sourced:
grep -rn "alias ll=" ~/.bashrc ~/.bash_aliases ~/.zshrc 2>/dev/null
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
