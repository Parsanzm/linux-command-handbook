# cd — The Complete Reference

> **Change the current working directory**
> The most frequently typed command in a terminal — and the only one
> that cannot exist as a regular program. `cd` is a shell builtin, always.

---

## Table of Contents

- [What is cd?](#what-is-cd)
- [Why cd Must Be a Builtin](#why-cd-must-be-a-builtin)
- [How cd works internally](#how-cd-works-internally)
- [Syntax](#syntax)
- [Special Arguments](#special-arguments)
- [All Options](#all-options)
- [Environment Variables: HOME, CDPATH, OLDPWD](#environment-variables)
- [Logical vs Physical Directory](#logical-vs-physical-directory)
- [cd in Different Shells](#cd-in-different-shells)
- [Modern Alternatives](#modern-alternatives)
- [Related Commands](#related-commands)

---

## What is cd?

`cd` changes the **current working directory** of the shell process. Every process on Linux has a working directory — the directory used as the base when resolving relative paths.

`cd` is specified by **POSIX** and is present in every Unix shell: bash, zsh, dash, ksh, fish, and others. The behavior is mostly identical across shells, with small additions in bash and zsh.

---

## Why cd Must Be a Builtin

This is the most fundamental fact about `cd` — and a classic interview question.

Every process on Linux has its own working directory, stored in the kernel. When a shell runs an external command, it **forks a child process**. The child can change its own working directory, but that change does **not** affect the parent (the shell).

```
shell (pid=1000, cwd=/home/alice)
  └── fork → child (pid=1001, cwd=/home/alice)
                └── chdir("/tmp")    ← only changes child's cwd
                └── exit
shell (pid=1000, cwd=/home/alice)   ← unchanged!
```

If `cd` were an external program, running it would:
1. Fork a child process
2. Child changes its own cwd to `/tmp`
3. Child exits
4. Shell's cwd is still `/home/alice` — nothing changed

**Solution:** `cd` must run inside the shell process itself, using `chdir()` directly. That's what a **builtin** is — a command implemented inside the shell, not a separate executable.

```bash
type cd
# cd is a shell builtin

which cd        # may return nothing, or a stub
/usr/bin/cd     # this exists but is useless as a standalone program
```

---

## How cd works internally

When you type `cd /tmp`:

1. Shell parses the command, recognizes `cd` as a builtin
2. Shell calls `chdir("/tmp")` syscall directly (no fork)
3. On success: shell updates `$PWD` and `$OLDPWD` environment variables
4. On failure: shell prints an error, `$PWD` unchanged

```
chdir(path)  →  update $OLDPWD = $PWD  →  update $PWD = new path
```

**The `chdir()` syscall:**
- Takes an absolute or relative path
- Updates the process's working directory in the kernel
- Returns 0 on success, -1 on error (ENOENT, ENOTDIR, EACCES, etc.)

**`$PWD` is maintained by the shell, not the kernel:**
- The kernel tracks the real (physical) path
- The shell tracks the logical path (preserving symlink names)
- `pwd` (builtin) shows `$PWD` (logical); `pwd -P` shows the kernel's real path

---

## Syntax

```
cd [-L|-P] [-e] [-@] [directory]
```

- With no argument → goes to `$HOME`
- With `-` → goes to previous directory (`$OLDPWD`)
- `directory` can be absolute, relative, or a `$CDPATH` shortcut

---

## Special Arguments

### `cd` with no argument → home directory
```bash
cd          # same as: cd $HOME  or  cd ~
cd ~        # explicit home
cd ~/docs   # subdirectory of home
```

### `cd -` → previous directory
```bash
cd /var/log
cd /etc
cd -        # back to /var/log
cd -        # back to /etc
# Toggles between two directories
# Also prints the directory it switched to
```

### `cd ~username` → another user's home
```bash
cd ~root        # /root
cd ~alice       # /home/alice (if you have permission)
cd ~www-data    # web server user's home
```

### `cd ..` → parent directory
```bash
cd ..           # one level up
cd ../..        # two levels up
cd ../../etc    # up two, then into etc
```

### `cd /` → root
```bash
cd /            # filesystem root
```

### `cd .` → current directory (no-op)
```bash
cd .            # does nothing (stays in same dir)
```

---

## All Options

### `-L` — Logical (default)
Follow the logical path. Symlink names are preserved in `$PWD`.

```bash
ln -s /var/log /tmp/logs
cd /tmp/logs       # $PWD = /tmp/logs  (logical)
cd -L /tmp/logs    # same
```

### `-P` — Physical
Resolve all symlinks. `$PWD` shows the real path.

```bash
ln -s /var/log /tmp/logs
cd -P /tmp/logs    # $PWD = /var/log  (physical/real)
```

### `-e` — Exit with error if `-P` fails (bash 4+)
```bash
cd -Pe /tmp/logs   # if physical path can't be determined, return error
```

### `-@` — macOS/zsh: open extended attributes
```bash
cd -@ file.txt     # treats file's extended attribute namespace as a directory
# macOS/zsh specific — rarely used
```

---

## Environment Variables

### `$HOME`
The default destination for bare `cd`. Set at login by PAM/the system.

```bash
echo $HOME      # /home/alice
cd              # goes to $HOME
cd ~            # same

# Temporarily change home:
HOME=/tmp cd    # doesn't work — builtin doesn't take env prefix
export HOME=/tmp && cd   # changes home for session
```

### `$PWD`
Current working directory — maintained by the shell (not the kernel).

```bash
echo $PWD       # /home/alice/projects
pwd             # same (reads $PWD)
pwd -P          # physical path (resolves symlinks, asks kernel)
```

### `$OLDPWD`
The previous working directory — set by `cd` each time you change directories.

```bash
cd /var/log
cd /etc
echo $OLDPWD    # /var/log
cd -            # uses $OLDPWD to go back
```

### `$CDPATH`
A colon-separated list of directories to search when `cd` is given a relative path that doesn't exist locally.

```bash
export CDPATH=".:$HOME:$HOME/projects:/var"

cd log          # tries ./log, then ~/log, then ~/projects/log, then /var/log
                # /var/log exists → goes there!
                # Also prints the full path it resolved to
```

`$CDPATH` is powerful but can cause surprising behavior — you type `cd src` expecting `./src` but end up in `~/projects/src`.

---

## Logical vs Physical Directory

This distinction matters when symlinks are involved.

```bash
mkdir -p /real/path
ln -s /real/path /tmp/link

# Logical (default: -L)
cd /tmp/link
pwd             # /tmp/link       ← shell's $PWD (preserves symlink name)
pwd -P          # /real/path      ← kernel's real path

# Physical (-P)
cd -P /tmp/link
pwd             # /real/path      ← $PWD is updated to real path

# Traversal difference:
cd /tmp/link
cd ..
pwd             # /tmp            ← went up from the logical path

cd -P /tmp/link
cd ..
pwd             # /real           ← went up from the real path
```

---

## cd in Different Shells

### bash
- All standard options: `-L`, `-P`, `-e`
- Supports `$CDPATH`
- `cd -` prints destination

### zsh
- All bash options plus `-@` (extended attributes on macOS)
- `$CDPATH` supported
- `AUTO_CD` option: type a directory name without `cd` to navigate
  ```zsh
  setopt AUTO_CD
  /etc          # same as: cd /etc
  ..            # same as: cd ..
  ```
- `PUSHD_SILENT`, `AUTO_PUSHD`: automatically push to directory stack

### fish
- No `-L`/`-P` flags (uses `builtin cd`)
- `cdh` shows recent directory history interactively
- Abbreviation system can make `..` work as `cd ..`

### dash / sh
- Only `-L` and `-P` (POSIX minimum)
- No `$CDPATH` required by POSIX (but usually supported)

### ksh
- Same as bash options
- `cd old new` — replaces `old` with `new` in current path:
  ```ksh
  pwd           # /home/alice/project/src
  cd src lib    # goes to /home/alice/project/lib
  ```

---

## Modern Alternatives

Tools that enhance or replace `cd` with smarter navigation:

### `z` / `zoxide` — frecency-based jumping
```bash
# After visiting /home/alice/projects/myapp a few times:
z myapp         # jumps to /home/alice/projects/myapp
z proj          # fuzzy matches most frequent dir with "proj"

# zoxide (Rust, faster z):
eval "$(zoxide init bash)"   # add to ~/.bashrc
z myapp
zi myapp        # interactive selection with fzf
```

### `pushd` / `popd` — directory stack
```bash
pushd /var/log    # go to /var/log AND push current dir to stack
pushd /etc        # go to /etc AND push /var/log
popd              # return to /var/log (pop from stack)
popd              # return to original dir
dirs              # show directory stack
dirs -v           # show with index numbers
cd ~2             # go to index 2 in stack (zsh)
```

### `fzf` — fuzzy directory finder
```bash
# Add to ~/.bashrc:
bind '"\C-f": "cd $(find . -type d | fzf)\n"'
# Ctrl+F → fuzzy search all subdirectories
```

### `autojump` — similar to z
```bash
j myapp         # jump to most used dir matching "myapp"
jc myapp        # jump to child directory matching "myapp"
jo myapp        # open directory in file manager
```

### `broot` — interactive directory browser
```bash
br              # open interactive tree, navigate and cd
```

---

## Related Commands

| Command | Relation |
|---------|---------|
| `pwd` | Print current working directory (`$PWD` or physical) |
| `pushd` | Like `cd` but pushes to directory stack |
| `popd` | Return to previous directory from stack |
| `dirs` | Show directory stack |
| `ls` | List contents of a directory |
| `mkdir` | Create a directory to `cd` into |
| `realpath` | Resolve a path to its absolute physical form |
| `readlink -f` | Resolve symlinks in a path |
| `z` / `zoxide` | Smart frecency-based directory jumping |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
