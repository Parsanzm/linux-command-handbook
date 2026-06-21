# ls — The Complete Reference

> **List directory contents**
> The first command most people ever type in a terminal — and one of the deepest.

---

## Table of Contents

- [What is ls?](#what-is-ls)
- [Where does ls live?](#where-does-ls-live)
- [How ls works internally](#how-ls-works-internally)
- [Syntax](#syntax)
- [All Options](#all-options)
- [Output Formats](#output-formats)
- [Sorting](#sorting)
- [Colors & File Types](#colors--file-types)
- [Alternatives: exa, eza, lsd, tree](#alternatives)
- [Related Commands](#related-commands)

---

## What is ls?

`ls` lists the contents of a directory. Without arguments it shows the current directory, non-hidden files only, sorted alphabetically.

First appeared in Unix Version 1 (1971), written by Ken Thompson and Dennis Ritchie. On Linux today it's part of **GNU coreutils**; on macOS/BSD it's the BSD version with slightly different flags.

**Three core uses:**
1. See what's in a directory
2. Inspect file metadata (permissions, size, timestamps, owner)
3. Distinguish file types (regular, directory, symlink, executable, etc.)

---

## Where does ls live?

```
/bin/ls           ← most Linux systems (Debian, Ubuntu, Arch)
/usr/bin/ls       ← some distros and macOS
```

Verify on your system:

```bash
which ls          # → /bin/ls
type ls           # may reveal alias (ls='ls --color=auto')
type -a ls        # shows ALL matches: alias + binary
command -v ls     # → /bin/ls (bypasses aliases)
ls --version      # GNU coreutils version (Linux only)
```

**Important:** On most systems `ls` is aliased to `ls --color=auto` in your shell config (`~/.bashrc`). `command ls` or `\ls` bypasses the alias.

---

## How ls works internally

```
opendir(path) → readdir(fd) → stat()/lstat() each entry → sort → write(stdout)
```

1. Opens the directory with `opendir()` — gets a directory stream
2. Calls `readdir()` in a loop to get each entry (name + inode number)
3. For each entry, calls `stat()` (or `lstat()` for symlinks) to get metadata:
   - file type, permissions, owner, size, timestamps
4. Sorts the entries (default: alphabetical, locale-aware)
5. Formats and writes to stdout

**Key distinction — `stat` vs `lstat`:**
- `stat()` follows symlinks → shows target file's info
- `lstat()` does NOT follow → shows the symlink itself's info
- `ls -l` uses `lstat()` for entries, but `stat()` when you pass a symlink as argument

**Performance note:** `ls` on a directory with millions of files is slow because it calls `stat()` on every entry. `ls -f` skips sorting and `stat()` calls — much faster for huge directories.

**Exit codes:**
| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Minor problems (e.g. cannot access some files) |
| `2` | Serious trouble |

---

## Syntax

```
ls [OPTION]... [FILE]...
```

- With no argument → lists current directory (`.`)
- Arguments can be files (shows info about them) or directories (lists their contents)
- Multiple directories → lists each in turn with a header

```bash
ls                    # current directory
ls /etc               # specific directory
ls /etc /var /tmp     # multiple directories
ls file.txt           # info about a specific file
ls -d /etc            # directory itself, not its contents
```

---

## All Options

### Display Format

| Option | Description |
|--------|-------------|
| `-l` | Long format: permissions, links, owner, group, size, time, name |
| `-1` | One entry per line (digit one, not letter l) |
| `-C` | Multi-column output (default when stdout is a terminal) |
| `-m` | Comma-separated list |
| `-x` | Multi-column, sorted across instead of down |
| `-o` | Like `-l` but omit group |
| `-g` | Like `-l` but omit owner |
| `--full-time` | Show full ISO timestamp in long format |

### Showing Hidden Files

| Option | Description |
|--------|-------------|
| `-a` | Show all files including `.` and `..` |
| `-A` | Show hidden files but NOT `.` and `..` |

### Sorting

| Option | Description |
|--------|-------------|
| (none) | Alphabetical (locale-aware) |
| `-t` | Sort by modification time, newest first |
| `-u` | Sort by access time (use with `-l` to show it) |
| `-c` | Sort by change time (metadata change) |
| `-S` | Sort by file size, largest first |
| `-X` | Sort by file extension |
| `-v` | Natural sort of version numbers (`file2` before `file10`) |
| `-r` | Reverse the sort order |
| `-f` | Do not sort (raw directory order) — also enables `-a` |
| `--sort=WORD` | `none`, `size`, `time`, `version`, `extension` |

### File Sizes

| Option | Description |
|--------|-------------|
| `-s` | Print allocated blocks for each file |
| `-h` | Human-readable sizes (1K, 2M, 3G) — use with `-l` or `-s` |
| `-H` | Like `-h` but uses powers of 1000 (not 1024) |
| `--block-size=SIZE` | Set block size (e.g., `--block-size=MB`) |
| `-k` | Use 1024-byte blocks |

### File Type Indicators

| Option | Description |
|--------|-------------|
| `-F` | Append type indicator: `/` dir, `*` exec, `@` symlink, `=` socket, `|` pipe |
| `-p` | Append `/` to directories only |
| `--file-type` | Like `-F` but without `*` for executables |

### Symlinks

| Option | Description |
|--------|-------------|
| `-l` | Shows symlink → target |
| `-L` | Follow symlinks (show target's info, not symlink's) |
| `-H` | Follow symlinks on command line only |

### Recursion

| Option | Description |
|--------|-------------|
| `-R` | List subdirectories recursively |
| `-d` | List directories themselves, not their contents |

### Colors & Style (GNU only)

| Option | Description |
|--------|-------------|
| `--color=auto` | Color when output is a terminal |
| `--color=always` | Always colorize (even in pipes) |
| `--color=never` | No color |
| `-G` (BSD) | Enable color on macOS/BSD |

### Timestamps

| Option | Description |
|--------|-------------|
| `-l` | Shows modification time by default |
| `-lu` | Show last access time |
| `-lc` | Show last status change time |
| `--time-style=STYLE` | `full-iso`, `long-iso`, `iso`, `locale`, `+FORMAT` |

### Other

| Option | Description |
|--------|-------------|
| `-i` | Print inode number |
| `-n` | Like `-l` but show numeric UID/GID instead of names |
| `-q` | Replace non-printable chars in names with `?` |
| `-Q` | Quote names with double quotes |
| `--quoting-style=STYLE` | `literal`, `shell`, `shell-always`, `c`, `escape` |
| `--group-directories-first` | List directories before files |
| `-Z` | Show SELinux security context (Linux) |
| `--hide=PATTERN` | Hide files matching shell pattern |
| `--ignore=PATTERN` | Like `--hide` but affects `-a` too |

---

## Output Formats

### Default (short)
```
Desktop  Documents  Downloads  file.txt  script.sh
```

### Long format (`-l`)
```
-rw-r--r--  1  alice  staff   4096  Jun 15 10:23  file.txt
drwxr-xr-x  3  alice  staff    128  Jun 14 09:00  Documents/
lrwxrwxrwx  1  alice  staff     11  Jun 13 08:00  link -> target.txt
```

Column breakdown:

```
-rw-r--r--   1      alice    staff    4096   Jun 15 10:23   file.txt
│            │      │        │        │      │              │
│            │      │        │        │      │              └─ name
│            │      │        │        │      └─ modification time
│            │      │        │        └─ size in bytes
│            │      │        └─ group
│            │      └─ owner
│            └─ hard link count
└─ type + permissions (10 chars)
```

**First character (file type):**

| Char | Type |
|------|------|
| `-` | Regular file |
| `d` | Directory |
| `l` | Symbolic link |
| `b` | Block device |
| `c` | Character device |
| `p` | Named pipe (FIFO) |
| `s` | Socket |
| `D` | Door (Solaris) |

**Permissions (9 chars after type):**
```
rw-r--r--
│││││││││
│││││││└┘─ other: read only
│││││└┘─── group: read only
│││└┘───── owner: read + write
```

### Human-readable sizes (`-lh`)
```
-rw-r--r--  1  alice  staff   4.0K  Jun 15  file.txt
-rw-r--r--  1  alice  staff   1.5M  Jun 14  archive.tar
-rw-r--r--  1  alice  staff   2.3G  Jun 10  backup.img
```

---

## Sorting

```bash
ls -lt          # by modification time, newest first
ls -ltr         # by modification time, oldest first
ls -lS          # by size, largest first
ls -lSr         # by size, smallest first
ls -lX          # by extension
ls -lv          # natural version sort (file1 file2 file10, not file1 file10 file2)
ls -f           # unsorted (raw filesystem order — fastest)
ls --group-directories-first -l   # dirs on top
```

---

## Colors & File Types

Default color scheme (GNU `ls` with `--color`):

| Color | Meaning |
|-------|---------|
| Blue | Directory |
| Green | Executable |
| Cyan | Symbolic link |
| Red | Archive (tar, zip...) |
| Magenta | Image file |
| Yellow | Device file |
| Red (bold) | Broken symlink |
| White/Gray | Regular file |

Colors are controlled by `$LS_COLORS` environment variable.

```bash
echo $LS_COLORS             # see current settings
dircolors --print-database  # print full default database
eval $(dircolors)           # reload defaults
```

Customize in `~/.bashrc`:
```bash
export LS_COLORS="di=1;34:ln=36:ex=32:*.tar=31"
```

---

## Alternatives

Modern rewrites of `ls` with better defaults and features:

| Command | Language | Highlights |
|---------|----------|------------|
| `eza` | Rust | Git status, icons, tree view, better defaults (successor to `exa`) |
| `exa` | Rust | Original modern ls (unmaintained since 2023) |
| `lsd` | Rust | Icons, tree, git info, colorful |
| `tree` | C | Recursive tree view only |
| `broot` | Rust | Interactive directory browser |

```bash
# eza examples
eza -l                    # long format
eza -la                   # with hidden files
eza --tree                # tree view
eza -l --git              # show git status per file
eza -l --icons            # file type icons (needs Nerd Font)

# tree
tree /etc                 # recursive tree
tree -L 2 /etc            # max depth 2
tree -a                   # include hidden files
tree -h                   # human-readable sizes
```

---

## Related Commands

| Command | Relation |
|---------|---------|
| `dir` | GNU version: like `ls -C -b` |
| `vdir` | GNU version: like `ls -l -b` |
| `find` | Search files with complex criteria |
| `stat` | Detailed metadata of a single file |
| `file` | Detect file type by content |
| `tree` | Recursive directory tree |
| `du` | Disk usage per file/directory |
| `lsof` | List open files by process |
| `namei` | Follow a pathname to its endpoint |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
