# cd — Edge Cases & Gotchas

> cd looks trivial but has surprises — especially in scripts and with symlinks.

---

## Table of Contents

- [cd in Scripts vs Interactive Shell](#cd-in-scripts-vs-interactive-shell)
- [Symlink Confusion](#symlink-confusion)
- [CDPATH Surprises](#cdpath-surprises)
- [Spaces & Special Characters in Paths](#spaces--special-characters-in-paths)
- [$PWD vs Real Path](#pwd-vs-real-path)
- [Deleted Directories](#deleted-directories)
- [Permission Edge Cases](#permission-edge-cases)
- [pushd / popd Gotchas](#pushd--popd-gotchas)
- [Subshell Trap](#subshell-trap)
- [Shell Differences](#shell-differences)

---

## cd in Scripts vs Interactive Shell

### Failed cd doesn't stop a script (without set -e)
```bash
#!/bin/bash
cd /nonexistent      # fails silently!
rm -rf *             # runs in ORIGINAL directory — dangerous!

# Fix 1: check exit code
cd /nonexistent || exit 1

# Fix 2: use set -e (exit on any error)
set -e
cd /nonexistent      # script exits here

# Fix 3: explicit error handling
cd /target || { echo "Error: cannot cd to /target" >&2; exit 1; }
```

### cd exit code is often ignored
```bash
# People write:
cd /some/dir
do_something       # runs wherever we ended up — maybe not /some/dir!

# Safe pattern — always check:
cd /some/dir && do_something
# or:
cd /some/dir || exit 1
do_something
```

### set -e interacts with cd in subshells
```bash
set -e
result=$(cd /nonexistent && echo "ok")   # subshell fails
# With set -e: the outer script may or may not exit depending on context
# Safer: check explicitly
if ! cd /some/dir 2>/dev/null; then
  echo "Cannot cd" >&2
  exit 1
fi
```

---

## Symlink Confusion

### `cd ..` after entering a symlinked directory goes to logical parent
```bash
mkdir -p /real/dir
ln -s /real/dir /tmp/link

cd /tmp/link
cd ..
pwd             # /tmp   ← NOT /real (the real parent of /real/dir)

# This surprises people expecting to "back out" of the real path
# Use -P to avoid:
cd -P /tmp/link
cd ..
pwd             # /real  ← correct physical parent
```

### $PWD can be out of sync with reality
```bash
cd /tmp/link        # $PWD = /tmp/link
# Someone deletes /tmp/link
rm /tmp/link
pwd                 # still shows /tmp/link (shell hasn't noticed)
cd .                # may fail or recompute
ls                  # may work (kernel still has the real path open)
```

### Symlink chains
```bash
ln -s /real/path /tmp/a
ln -s /tmp/a /tmp/b
ln -s /tmp/b /tmp/c

cd /tmp/c
pwd             # /tmp/c   (logical: only first symlink name preserved)
pwd -P          # /real/path  (all resolved)

cd ..
pwd             # /tmp     (logical parent of /tmp/c)
```

### -L and -P affect different things
```bash
# -L/-P on cd affects how $PWD is set going forward
cd -L /tmp/link    # $PWD = /tmp/link
cd -P /tmp/link    # $PWD = /real/dir

# pwd -L and pwd -P are independent of how you cd'd
pwd -L             # reads $PWD (whatever cd set it to)
pwd -P             # asks kernel for real path (always physical)
```

---

## CDPATH Surprises

### CDPATH can override expected local behavior
```bash
export CDPATH=".:$HOME/projects"

mkdir src
cd src              # ✅ goes to ./src  (. is first in CDPATH)

# But if . is NOT first:
export CDPATH="$HOME/projects:."

cd src              # ⚠️ might go to ~/projects/src instead of ./src!
# CDPATH searches in order — put . first to prefer local dirs
```

### CDPATH prints the resolved path
```bash
export CDPATH=".:$HOME/projects"
cd myapp
# Prints: /home/alice/projects/myapp
# This output can break scripts that capture cd output:
result=$(cd myapp)   # result = "/home/alice/projects/myapp" (unexpected)
```

### CDPATH in scripts can cause bugs
```bash
#!/bin/bash
# If the user has CDPATH set, this script may go somewhere unexpected:
cd config           # might not be ./config!

# Fix: always use explicit paths in scripts
cd ./config
cd "$(dirname "$0")/config"  # relative to script

# Or unset CDPATH in scripts:
unset CDPATH
cd config           # now always ./config
```

### Completion behavior with CDPATH
```bash
# bash tab completion respects CDPATH
# cd my<TAB> might complete to ~/projects/myapp even if no local "my*" dir
# Can be surprising if you're not expecting it
```

---

## Spaces & Special Characters in Paths

### Spaces in directory names
```bash
cd my documents      # ❌ tries to cd to "my" then "documents" (two args)

cd "my documents"    # ✅
cd my\ documents     # ✅
cd 'my documents'    # ✅

# In variables — always quote!
dir="my documents"
cd $dir             # ❌ word splits on space
cd "$dir"           # ✅
```

### Paths with newlines (rare but valid)
```bash
# Directory with newline in name:
mkdir $'dir\nwith\nnewline'
cd $'dir\nwith\nnewline'   # works in bash with $'...' quoting

# Storing in variable:
dir=$'dir\nwith\nnewline'
cd "$dir"           # ✅ with quotes
```

### Paths starting with `-`
```bash
mkdir -baddir
cd -baddir          # ❌ interpreted as option (- means previous dir or flag)

cd -- -baddir       # ✅ -- ends option parsing
cd ./-baddir        # ✅ explicit relative path
```

### Unicode in paths
```bash
cd "документы"       # works in UTF-8 locale
cd "日本語"           # works if locale supports it
cd ~/données         # works in UTF-8 locale
```

---

## $PWD vs Real Path

### $PWD is set by the shell, not the kernel
```bash
# The shell maintains $PWD independently
# You can (accidentally) corrupt it:
PWD=/fake/path      # directly overwrite $PWD
pwd                 # shows /fake/path (lie!)
pwd -P              # shows real path (asks kernel)
ls                  # works fine (uses kernel's real cwd)

# cd fixes it:
cd .                # recomputes $PWD from kernel
pwd                 # now correct again
```

### After mounting/unmounting
```bash
cd /mnt/usb
# Someone unmounts /mnt/usb
pwd                 # still shows /mnt/usb (stale $PWD)
ls                  # "Stale file handle" error
cd                  # go home — safest recovery
```

### NFS and network directories
```bash
cd /nfs/share
# Network goes down
pwd                 # still shows /nfs/share
ls                  # hangs or errors
# cd - or cd ~ to escape
```

---

## Deleted Directories

### You can be "in" a deleted directory
```bash
mkdir /tmp/testdir
cd /tmp/testdir
rm -rf /tmp/testdir    # delete the directory you're in!

pwd                    # /tmp/testdir  (shows stale $PWD)
ls                     # may work (inode still open) or fail
cd ..                  # ⚠️ may fail or go to unexpected place

# Recovery:
cd                     # go home (always works)
cd /tmp                # go to known good location
```

### Script deletes its own working directory
```bash
#!/bin/bash
cd /tmp/workdir
# ... processing ...
rm -rf /tmp/workdir    # cleanup
cd                     # should go home, but some shells behave oddly here
# Safer: cd to another place BEFORE deleting
cd /tmp
rm -rf /tmp/workdir
```

---

## Permission Edge Cases

### Need execute (x) permission, not read (r)
```bash
chmod 600 /restricted    # rw for owner, nothing for others
cd /restricted           # ❌ Permission denied — need x bit to enter

chmod 100 /restricted    # x only (no read)
cd /restricted           # ✅ can enter (but can't ls contents)
ls /restricted           # ❌ Permission denied (no r bit)

# Execute on directory = permission to traverse/enter
# Read on directory = permission to list contents
# These are independent
```

### Root can cd anywhere
```bash
sudo su -
cd /root        # ✅
chmod 000 /tmp/secret
cd /tmp/secret  # ✅ root ignores permission bits for traversal
```

### Sticky bit doesn't affect cd
```bash
# /tmp has sticky bit (drwxrwxrwt)
cd /tmp         # ✅ sticky only affects deletion, not traversal
```

---

## pushd / popd Gotchas

### pushd output goes to stdout
```bash
pushd /var/log     # prints: /var/log ~
                   # this output can break scripts

# Suppress:
pushd /var/log > /dev/null
pushd /var/log 2>&1 > /dev/null
```

### Empty stack causes popd to fail
```bash
dirs -c            # clear stack
popd               # ❌ bash: popd: directory stack empty
# Check before popping:
[ "$(dirs -p | wc -l)" -gt 1 ] && popd > /dev/null
```

### pushd +N behavior differs by shell
```bash
# bash: pushd +N rotates the stack bringing index N to top
# zsh: same
# The counting starts at 0 from the top
dirs -v           # 0 = current, 1 = previous, etc.
pushd +2          # rotates so index 2 becomes current
```

---

## Subshell Trap

### cd in a subshell doesn't affect the parent
```bash
# Classic mistake:
$(cd /tmp)          # cd runs in subshell, parent unchanged
`cd /tmp`           # same
(cd /tmp)           # same

# These are ALL no-ops for the parent shell's cwd
echo $PWD           # unchanged

# Correct ways to cd in current shell:
cd /tmp             # direct builtin call
source script.sh    # sourced scripts share the shell's state
. script.sh         # same as source
```

### Functions DO share the shell's cwd
```bash
# Unlike subshells, functions run in the current shell:
go_to_tmp() {
  cd /tmp
}
go_to_tmp
pwd    # /tmp  ✅ — function cd affects caller's shell

# This is why shell functions can wrap cd:
alias cdl='cd_and_list(){ cd "$1" && ls; }; cd_and_list'
```

---

## Shell Differences

### `cd -` output
```bash
# bash: prints destination to stdout
cd /tmp && cd /etc && cd -
# Output: /tmp

# zsh: same
# dash: same
# fish: no cd - (use prevd instead)
```

### `CDPATH` behavior
```bash
# bash, zsh, ksh: supported
# dash: supported (POSIX)
# fish: uses $CDPATH but may behave differently
```

### Missing `-e` flag
```bash
cd -Pe /path    # bash 4+: error if physical path can't be determined
cd -P /path     # dash/sh: -e not available, silently ignored on some
```

### `cd` return value in different shells
```bash
# All POSIX shells: cd returns 0 on success, non-zero on failure
# bash: also updates $OLDPWD on success
# zsh: same as bash + maintains directory stack if AUTO_PUSHD set
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
