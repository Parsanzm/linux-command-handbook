# cd — Practical Examples

> Every pattern you'll actually use, from daily navigation to scripting tricks.

---

## Table of Contents

- [Basic Navigation](#basic-navigation)
- [Special Shortcuts](#special-shortcuts)
- [CDPATH: Smart Shortcuts](#cdpath-smart-shortcuts)
- [Logical vs Physical Paths](#logical-vs-physical-paths)
- [Directory Stack: pushd & popd](#directory-stack-pushd--popd)
- [Navigation in Scripts](#navigation-in-scripts)
- [Shell-Specific Features](#shell-specific-features)
- [Combining with Other Commands](#combining-with-other-commands)
- [Productivity Patterns](#productivity-patterns)

---

## Basic Navigation

```bash
# Absolute path
cd /var/log
cd /etc/nginx/conf.d
cd /home/alice/projects

# Relative path
cd documents
cd ../sibling_dir
cd ../../parent/other

# Home
cd
cd ~
cd $HOME

# Root
cd /

# Previous directory (toggle)
cd /var/log
cd /etc
cd -          # → /var/log (also prints it)
cd -          # → /etc
```

---

## Special Shortcuts

```bash
# ~ expands to $HOME
cd ~
cd ~/Downloads
cd ~/projects/myapp/src

# ~username: another user's home
cd ~root                # /root
cd ~www-data            # web server home
cd ~$(whoami)           # same as ~

# Parent traversal
cd ..                   # one level up
cd ../..                # two levels up
cd ../sibling           # up then into sibling

# Combining
cd /var
cd log/nginx            # now in /var/log/nginx
cd ../../tmp            # → /tmp
```

---

## CDPATH: Smart Shortcuts

`CDPATH` lets you `cd` to a short name by searching a list of base directories.

```bash
# Setup in ~/.bashrc
export CDPATH=".:$HOME:$HOME/projects:/var:/etc"

# From anywhere:
cd log          # finds /var/log
cd nginx        # finds /etc/nginx
cd myapp        # finds ~/projects/myapp
# cd prints the resolved path when CDPATH is used

echo $CDPATH    # verify
unset CDPATH    # remove

# Practical setup for developers:
export CDPATH=".:$HOME:$HOME/projects:$HOME/work:/opt"
# cd frontend   → ~/projects/frontend
# cd api        → ~/projects/api
```

---

## Logical vs Physical Paths

```bash
# Setup
mkdir -p /real/deeply/nested/path
ln -s /real/deeply/nested/path /tmp/shortcut

# Logical (default -L)
cd /tmp/shortcut
pwd             # /tmp/shortcut   ← symlink name preserved
cd ..
pwd             # /tmp            ← up from logical path

# Physical (-P)
cd -P /tmp/shortcut
pwd             # /real/deeply/nested/path
cd ..
pwd             # /real/deeply/nested   ← real parent

# Check both anytime
pwd             # logical ($PWD)
pwd -P          # physical (kernel cwd)

# Real example: /bin is a symlink on modern Ubuntu
cd /bin
pwd             # /bin       (logical)
pwd -P          # /usr/bin   (physical)

cd -P /bin
pwd             # /usr/bin
cd ..
pwd             # /usr  (not /)
```

---

## Directory Stack: pushd & popd

```bash
# pushd: go to dir AND save current to stack
pushd /var/log        # cwd → /var/log, stack grows
pushd /etc/nginx      # cwd → /etc/nginx
pushd /tmp

# View stack
dirs              # /tmp /etc/nginx /var/log ~
dirs -v           # with index:
                  # 0  /tmp
                  # 1  /etc/nginx
                  # 2  /var/log
                  # 3  ~

# popd: return to top of stack
popd              # → /etc/nginx
popd              # → /var/log
popd              # → ~

# pushd with no args: swap top two (toggle like cd -)
pushd /var/log
pushd /etc
pushd             # back to /var/log
pushd             # back to /etc

# Jump to index
pushd +2          # bring index 2 to top

# Clear stack
dirs -c

# Practical: work across 3 dirs
pushd ~/projects/frontend
pushd ~/projects/backend
pushd /var/log

pushd +1          # go to backend
pushd +1          # go to frontend
```

---

## Navigation in Scripts

```bash
# Subshell (safest — parent unaffected)
(cd /tmp && ls)
echo $PWD         # still original dir

# Save and restore
original_dir=$(pwd)
cd /tmp
# ... work ...
cd "$original_dir"

# pushd/popd (clean)
pushd /tmp > /dev/null
# ... work ...
popd > /dev/null

# Trap for safety (restores on error too)
original_dir=$(pwd)
trap "cd '$original_dir'" EXIT
cd /tmp
# trap auto-restores when script exits or errors

# Script's own directory
#!/bin/bash
script_dir="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$script_dir"
# Now relative paths work regardless of where script was called from

# Error handling
cd /nonexistent || { echo "Failed to cd"; exit 1; }

if [ -d "/path/to/dir" ]; then
  cd /path/to/dir
else
  echo "Directory not found"; exit 1
fi

# Loop over directories
for dir in ~/projects/*/; do
  (
    cd "$dir" || continue
    echo "=== $(basename "$dir") ==="
    git status --short
  )
done
```

---

## Shell-Specific Features

```bash
# bash: cd - prints the destination
cd /tmp && cd /etc && cd -
# prints: /tmp
```

```zsh
# zsh: AUTO_CD — just type directory name
setopt AUTO_CD
/etc              # same as: cd /etc
..                # same as: cd ..

# zsh: AUTO_PUSHD — every cd is also a pushd
setopt AUTO_PUSHD
setopt PUSHD_SILENT
setopt PUSHD_IGNORE_DUPS
cd /var/log       # automatically pushed to stack
dirs -v           # see full history
cd ~3             # jump to 3rd entry

# zsh: named directories
hash -d proj=~/projects
hash -d logs=/var/log
cd ~proj          # → ~/projects
cd ~logs          # → /var/log
```

```fish
# fish: cdh — interactive recent directory picker
cdh
```

```ksh
# ksh: cd old new — string replacement in path
pwd               # /home/alice/project/src
cd src lib        # → /home/alice/project/lib
```

---

## Combining with Other Commands

```bash
# cd then run — use &&
cd /var/log && tail -f nginx/access.log

# Run in another dir without changing shell's cwd
(cd /var/log && grep "error" syslog)

# cd to command output
cd "$(git rev-parse --show-toplevel)"    # git repo root
cd "$(dirname "$(which python3)")"       # python3's directory
cd "$(mktemp -d)"                        # new temp directory

# find a directory and cd into it
cd "$(find . -type d -name "src" | head -1)"
```

---

## Productivity Patterns

```bash
# Jump to git root
alias cdg='cd "$(git rev-parse --show-toplevel)"'

# Go up N levels
up() {
  local n="${1:-1}" path=""
  for ((i=0; i<n; i++)); do path="../$path"; done
  cd "$path"
}
up 3    # same as cd ../../..

# mkdir + cd together
mkcd() { mkdir -p "$1" && cd "$1"; }
mkcd new/project/src

# cd + ls
cl() { cd "$1" && ls -la; }
cl /etc

# Bookmarks
export BM="$HOME/.bookmarks"
mkdir -p "$BM"
bm()  { ln -sfn "$PWD" "$BM/$1"; echo "Saved: $1"; }
go()  { cd -P "$BM/$1"; }
bms() { ls -la "$BM/"; }

bm proj       # bookmark current dir as "proj"
go proj       # jump to "proj"
bms           # list all bookmarks

# fzf: fuzzy directory jump
fcd() {
  local dir
  dir=$(find "${1:-.}" -type d 2>/dev/null | fzf) && cd "$dir"
}
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
