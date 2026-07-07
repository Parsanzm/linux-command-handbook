# pwd — The Complete Reference

> **Print Working Directory: show exactly where you are in the filesystem**
> One of the oldest and simplest Unix commands, present since the earliest Unix releases (1970s).
> Both a shell builtin AND a standalone binary — a rare case where both exist side by side.

---

## Table of Contents

- [What is pwd?](#what-is-pwd)
- [Where does pwd live?](#where-does-pwd-live)
- [How pwd works internally](#how-pwd-works-internally)
- [Syntax](#syntax)
- [Logical vs Physical Path (-L vs -P)](#logical-vs-physical-path--l-vs--p)
- [All Key Options](#all-key-options)
- [pwd and Environment Variables ($PWD, $OLDPWD)](#pwd-and-environment-variables-pwd-oldpwd)
- [Builtin pwd vs /bin/pwd](#builtin-pwd-vs-binpwd)
- [pwd and Symlinks](#pwd-and-symlinks)
- [pwd in Scripts](#pwd-in-scripts)
- [Related Commands](#related-commands)

---

## What is pwd?

`pwd` stands for **Print Working Directory**. It outputs the absolute path of the directory the current shell (or process) considers itself to be "in" right now — the directory that relative paths (like `./file.txt` or `../folder`) are resolved against.

```bash
cd /var/log
pwd
# /var/log
```

**Why pwd exists as a separate concept at all:** every process on a Unix system has a notion of its **current working directory** (CWD), tracked by the kernel as part of the process's state. `pwd` is simply the tool that reports what that kernel-tracked value currently is, since there's no other obvious way to "see" it without asking.

---

## Where does pwd live?

`pwd` exists in **two forms simultaneously**:

```bash
type pwd
# pwd is a shell builtin

which pwd
# /usr/bin/pwd
# (which searches $PATH and finds the STANDALONE binary, even though
# the shell builtin version is what actually runs by default)

command -v pwd
# pwd    (ambiguous by itself — doesn't tell you builtin vs binary directly)
```

Almost every shell (bash, zsh, dash, ksh) implements `pwd` as a **builtin** for speed (no process fork needed), while the standalone `/bin/pwd` or `/usr/bin/pwd` binary also exists on the filesystem, largely for:
- scripts using `#!/bin/sh` on systems where `sh` might not have a builtin pwd
- explicitly bypassing the shell builtin's behavior (`/bin/pwd` directly)
- POSIX compliance/testing tools that need a real external binary, not a builtin

```bash
which pwd
# /usr/bin/pwd
ls -l /usr/bin/pwd
# -rwxr-xr-x 1 root root ... /usr/bin/pwd     ← the real, standalone binary
```

---

## How pwd works internally

### The kernel tracks CWD per-process

Every process has a current working directory recorded by the kernel, accessible via the `getcwd()` system call. `pwd` (whether builtin or binary) is, at its core, just a thin wrapper that calls `getcwd()` and prints the result.

```c
// Simplified conceptual equivalent of what pwd does internally
char buf[PATH_MAX];
getcwd(buf, sizeof(buf));
printf("%s\n", buf);
```

### Why the shell builtin is usually preferred

Because `cd` (also a shell builtin) changes the **shell process's own** working directory, only a builtin `pwd` running in that **same process** can report the shell's current state without any ambiguity. An external `/bin/pwd` binary, if it purely called `getcwd()` fresh, would actually still work correctly too (it inherits the CWD from its parent shell at fork time) — but the builtin is faster (no fork/exec overhead) and slightly more consistent with how the shell itself is internally tracking the logical path (see [Logical vs Physical](#logical-vs-physical-path--l-vs--p) below).

### $PWD as a shell-maintained shortcut

Most shells also maintain an environment variable, `$PWD`, updated automatically every time `cd` runs, mirroring what `pwd` would print — this lets other programs and scripts read the current directory without spawning `pwd` as a separate process at all.

```bash
cd /tmp
echo $PWD
# /tmp
pwd
# /tmp
# Both report the same thing — $PWD is simply cached/maintained
# by the shell, while `pwd` actively re-queries and confirms it.
```

---

## Syntax

```bash
pwd [OPTIONS]
```

`pwd` takes no arguments — it can't be told to print the working directory of a specific *other* process or path; it always and only reports the calling shell/process's own current directory.

```bash
pwd
# /home/alice/projects

pwd -L      # logical path (respects symlinks as-typed, default in most shells)
pwd -P      # physical path (fully resolved, no symlinks)
```

---

## Logical vs Physical Path (-L vs -P)

This is `pwd`'s single most important nuance, and the source of nearly all its edge cases.

### -L (logical) — the default in most interactive shells
Reports the path **as you navigated to it**, including any symlinks you `cd`'d through, without resolving them to their real target.

### -P (physical) — fully resolved
Reports the path with **every symlink resolved** to its actual target location on disk — the "real," canonical path with no symbolic links remaining anywhere in it.

```bash
ls -l /var/run
# lrwxrwxrwx ... /var/run -> /run    (a classic symlink on many distros)

cd /var/run
pwd
# /var/run          ← logical (default): shows the path you typed/navigated
pwd -P
# /run              ← physical: shows where /var/run ACTUALLY points

cd -P /var/run       # if you cd WITH -P from the start, the shell's
                     # internal $PWD is updated to the physical path too
pwd
# /run              ← now even the default pwd (no flag) shows the physical path,
                     # because -P during cd changed how $PWD itself was recorded
```

**Why this distinction exists at all:** logical paths preserve your **mental model** of how you navigated (useful for readability and matching what you actually typed), while physical paths are essential when you need the **true, canonical location** for something like resolving a real file's exact position on disk, avoiding ambiguity in scripts, or debugging symlink-related confusion.

---

## All Key Options

| Option | Long | Description |
|--------|------|-------------|
| `-L` | `--logical` | Print the logical path (as navigated, symlinks NOT resolved) — default in most shells |
| `-P` | `--physical` | Print the physical path (symlinks fully resolved to their real target) |
| (bash-specific) | `--help` | Show usage help (works on the standalone binary; less consistently on some builtins) |
| (bash-specific) | `--version` | Show version info (only meaningful for the standalone `/bin/pwd` binary) |

```bash
pwd -L
pwd -P
/bin/pwd --version
# pwd (GNU coreutils) 9.4
```

Note: the **builtin** `pwd` in bash supports `-L`/`-P` directly per POSIX, but does **not** support `--help`/`--version` the way the standalone binary does — those flags only work when explicitly invoking `/bin/pwd`.

```bash
pwd --help
# bash: pwd: --help: invalid option
# pwd: usage: pwd [-LP]
# (the BUILTIN doesn't understand --help the same way a typical GNU tool would)

/bin/pwd --help
# Usage: pwd [OPTION]...
# Print the full filename of the current working directory.
# (the STANDALONE BINARY does support the familiar GNU-style --help)
```

---

## pwd and Environment Variables ($PWD, $OLDPWD)

```bash
# $PWD — automatically maintained, mirrors the shell's logical current directory
cd /var/log
echo $PWD
# /var/log

# $OLDPWD — automatically set to the PREVIOUS directory every time cd runs
cd /tmp
echo $OLDPWD
# /var/log     ← where you were before this most recent cd

# The classic "toggle between two directories" trick relies on $OLDPWD
cd -
# /var/log     (cd - is shorthand for "cd $OLDPWD", and also PRINTS the
# directory it switched to, unlike a normal cd which is silent)

# You can manually inspect or even override these variables (rarely useful,
# but shows they're just regular shell variables under the hood)
echo "Current: $PWD, Previous: $OLDPWD"
```

**Important distinction:** `$PWD` is a **shell-maintained cache**, updated only when `cd` (or `pushd`/`popd`) runs through the shell. Running the actual `pwd` **command** instead performs a fresh `getcwd()` system call every time, which is more authoritative if `$PWD` were ever somehow manually corrupted or unset.

```bash
PWD="/completely/fake/path"    # manually corrupt the variable (contrived example)
echo $PWD
# /completely/fake/path        ← the variable now lies
pwd
# /home/alice/actual/real/directory   ← the actual COMMAND still reports the truth,
# because it queries the kernel directly rather than trusting the variable
```

---

## Builtin pwd vs /bin/pwd

| Aspect | Shell builtin `pwd` | Standalone `/bin/pwd` |
|--------|----------------------|--------------------------|
| Speed | Faster (no fork/exec) | Slightly slower (spawns a real process) |
| Default flag behavior | Usually `-L` (logical) unless shell option changes it | Usually `-L` too, but strictly POSIX-defined |
| `--help` / `--version` | Often unsupported or minimal | Full GNU-style support |
| Reflects shell's own $PWD tracking | Yes, directly tied to shell's internal state | Independently re-queries the kernel; may momentarily differ if $PWD was manually altered |
| Guaranteed present on minimal/embedded systems | Depends on the shell in use | More consistently present as a real package/binary |

```bash
type pwd
# pwd is a shell builtin

# Force use of the standalone binary explicitly
/bin/pwd
env pwd            # "env" bypasses shell builtins, forcing external binary
command pwd        # note: `command` still uses the SHELL'S OWN understanding
                   # of builtins first in most shells, so this often still
                   # calls the builtin, not the binary — use the full path
                   # (/bin/pwd) if you specifically need the real binary
```

---

## pwd and Symlinks

### The classic scenario that trips people up
```bash
ln -s /var/log logs_link
cd logs_link
pwd
# /home/alice/logs_link      ← logical: shows the path AS YOU TYPED/navigated
pwd -P
# /var/log                    ← physical: shows the REAL, resolved location

ls -la
# Depends on -L vs -P context for how "." resolves in some tools,
# though ls itself isn't directly affected by pwd's -L/-P setting —
# this mainly affects pwd and cd's own path-tracking behavior.
```

### Nested symlinks
```bash
ln -s /var/log level1
ln -s level1 level2      # a symlink pointing to ANOTHER symlink
cd level2
pwd
# /home/alice/level2       ← logical, shows exactly what you cd'd into
pwd -P
# /var/log                  ← physical, fully resolved through BOTH symlink hops
```

---

## pwd in Scripts

### Getting a script's own directory (a common real-world need)
```bash
#!/bin/bash
# Get the directory the SCRIPT ITSELF lives in, NOT the caller's CWD —
# a classic and important distinction: pwd alone would show wherever
# the user happened to run the script FROM, not where the script IS.

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
echo "This script lives in: $SCRIPT_DIR"

# Contrast with the WRONG approach:
echo "Wrong: $(pwd)"
# Shows wherever the user's shell happened to be when they RAN the
# script, which could be anywhere — completely unrelated to the
# script file's actual location on disk.
```

### Safe directory switching with automatic return
```bash
#!/bin/bash
ORIGINAL_DIR="$(pwd)"
cd /tmp/some/other/place
# ... do work ...
cd "$ORIGINAL_DIR"     # explicitly return, rather than assuming subshell scoping
```

### Using a subshell to avoid affecting the caller's directory at all
```bash
(cd /tmp/build && make)
# The cd here only affects the SUBSHELL created by the parentheses —
# the original/calling shell's own working directory is completely
# untouched once the subshell exits, without needing to manually cd back.
pwd
# still shows whatever directory you were in BEFORE running that line
```

---

## Related Commands

| Command | Relation |
|---------|----------|
| `cd` | Changes the working directory that `pwd` reports |
| `dirname` | Extracts the directory portion of a given path (not the CWD itself) |
| `readlink -f` | Fully resolves a path's symlinks, similar in spirit to `pwd -P` but for any arbitrary path |
| `realpath` | Modern, more flexible alternative to `readlink -f` for canonicalizing any path |
| `basename` | Extracts just the final component of a path |
| `$PWD` / `$OLDPWD` | Shell variables that shadow/cache what pwd would report |
| `pushd` / `popd` | Directory stack navigation, which also updates `$PWD`/`$OLDPWD` as you move |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
