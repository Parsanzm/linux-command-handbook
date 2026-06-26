# less / more — The Complete Reference

> **Terminal pagers: view file content one screen at a time**
> `more` came first — `less` was written to fix everything `more` got wrong.
> Today, `less` is the standard. But `more` is everywhere, even in containers.

---

## Table of Contents

- [What are less and more?](#what-are-less-and-more)
- [Where do they live?](#where-do-they-live)
- [How they work internally](#how-they-work-internally)
- [Syntax](#syntax)
- [less: All Options](#less-all-options)
- [more: All Options](#more-all-options)
- [less: Navigation Keys](#less-navigation-keys)
- [more: Navigation Keys](#more-navigation-keys)
- [less: Search](#less-search)
- [less: Marks & Multiple Files](#less-marks--multiple-files)
- [less vs more — Full Comparison](#less-vs-more--full-comparison)
- [LESSOPEN: Preprocessor Support](#lessopen-preprocessor-support)
- [Environment Variables](#environment-variables)
- [Alternatives: bat, most, pg](#alternatives)
- [Related Commands](#related-commands)

---

## What are less and more?

Both are **terminal pagers** — programs that display text one screenful at a time, letting you scroll through content without it flying past.

**`more`** — 1978, written by Daniel Halbert at UC Berkeley. One of the first interactive Unix programs. Can only scroll **forward**. Simple, minimal, universal.

**`less`** — 1983, written by Mark Nudelman. Name is a joke: *"less is more"* (as in, it does more than `more`). Can scroll **forward and backward**, has search, marks, multi-file support, and much more.

The manpage for `less` famously begins:
> *"Less is a program similar to more, but it has many more features."*

**Three core uses:**
1. View large files without loading them all into memory
2. Browse command output (`git log | less`)
3. Search through content interactively

---

## Where do they live?

```
/usr/bin/less       ← less (most Linux distros)
/bin/less           ← some distros (symlink)
/usr/bin/more       ← more (most Linux distros)
/bin/more           ← some distros
```

```bash
which less
which more
less --version      # GNU less version
more --version      # version info

# less is part of the 'less' package
dpkg -S $(which less)    # Debian: less: /usr/bin/less
rpm -qf $(which less)    # RHEL: less-...

# more is part of util-linux
dpkg -S $(which more)    # Debian: util-linux: /usr/bin/more
```

**Important:** On most systems, `man` uses `less` as its pager. Many tools (git, systemctl, apt) pipe output through `less` automatically.

```bash
echo $PAGER          # what's set as system pager
echo $MANPAGER       # pager for man pages
```

---

## How they work internally

### more
```
open file → get terminal size → read one screenful → display → wait for keypress
→ on space: read next screen → on q: exit
```
- Reads forward only
- Exits after last line (no waiting at EOF)
- Very simple state machine

### less
```
open file → build index → display current window → event loop:
  keypress → move window / search / mark / exec → redisplay
```
- Maintains a **position index** into the file — allows backward movement
- Uses **termios** to put terminal in raw mode (reads keystrokes without Enter)
- Uses **terminfo/curses** for screen control (cursor positioning, clearing)
- Reads file **lazily** — doesn't load entire file into memory
- For pipes: buffers input in a temp file to allow backward scrolling
- Handles **SIGWINCH** (terminal resize) — redraws on window size change

**Exit codes:**
| Code | Meaning |
|------|---------|
| `0` | Normal exit |
| `1` | Error |
| `2` | Usage error (less only) |

---

## Syntax

```
less [OPTIONS] [FILE...]
more [OPTIONS] [FILE...]
```

- Multiple files: navigate between them with `:n` (next) and `:p` (prev) in less
- No file: reads from stdin
- `-` explicitly means stdin

```bash
less file.txt
less file1.txt file2.txt file3.txt
cat file.txt | less
less -                  # read from stdin explicitly
command | less
```

---

## less: All Options

### Display

| Option | Description |
|--------|-------------|
| `-N` | Show line numbers |
| `-S` | Chop long lines (don't wrap) — side-scroll with arrow keys |
| `-F` | Quit if content fits on one screen |
| `-X` | Don't clear screen on exit |
| `-e` | Quit at end of file (second time reaching EOF) |
| `-E` | Quit at end of file (first time reaching EOF) |
| `-i` | Case-insensitive search (if pattern is all lowercase) |
| `-I` | Case-insensitive search (always) |
| `-M` | Show more info in prompt (filename, line, percent) |
| `-m` | Show percent in prompt |
| `-q` | Quiet — no bell on errors |
| `-Q` | Very quiet — no bell ever |
| `-r` | Display raw control characters |
| `-R` | Display ANSI color codes (render colors) |
| `-s` | Squeeze multiple blank lines into one |
| `-x n` | Set tab stop width (default: 8) |

### Search

| Option | Description |
|--------|-------------|
| `-p pattern` | Start at first occurrence of pattern |
| `-P prompt` | Set custom prompt string |
| `--pattern=pattern` | Same as `-p` |

### Following

| Option | Description |
|--------|-------------|
| `-f` | Force open non-regular files (devices, FIFOs) |
| `+F` | Start in follow mode (like `tail -f`) |

### Multiple Files

| Option | Description |
|--------|-------------|
| `+n` | Start at line n |
| `+/pattern` | Start at first match of pattern |

### Performance

| Option | Description |
|--------|-------------|
| `-b n` | Buffer size in KB |
| `-B` | Don't automatically allocate buffers |

---

## more: All Options

| Option | Description |
|--------|-------------|
| `-d` | Display help message at prompt (Press space... etc.) |
| `-f` | Count logical lines (not screen lines — matters for long lines) |
| `-l` | Don't pause at form feed (^L) characters |
| `-p` | Clear screen before displaying each page |
| `-c` | Display from top of screen (like `-p` but different refresh) |
| `-s` | Squeeze multiple blank lines |
| `-u` | Suppress underline |
| `-n N` | Set screen size to N lines |
| `+N` | Start at line N |
| `+/pattern` | Start at first occurrence of pattern |

---

## less: Navigation Keys

### Moving

| Key | Action |
|-----|--------|
| `Space` / `f` / `PageDown` | Forward one screen |
| `b` / `PageUp` | Backward one screen |
| `d` | Forward half screen |
| `u` | Backward half screen |
| `j` / `↓` / `Enter` | Forward one line |
| `k` / `↑` | Backward one line |
| `g` / `<` | Go to first line |
| `G` / `>` | Go to last line |
| `Ng` | Go to line N |
| `Np` | Go to N percent through file |
| `{` | Jump to matching `}` (for `{}` pairs) |
| `}` | Jump to matching `{` |

### Files

| Key | Action |
|-----|--------|
| `:n` | Next file |
| `:p` | Previous file |
| `:d` | Remove current file from list |
| `:e file` | Open a new file |
| `:f` | Print current filename and line |

### Other

| Key | Action |
|-----|--------|
| `q` / `Q` | Quit |
| `h` | Help (full key reference) |
| `=` | Show file info (name, lines, bytes, position) |
| `v` | Open current file in `$VISUAL` or `$EDITOR` |
| `!cmd` | Execute shell command |
| `\|cmd` | Pipe marked section to shell command |
| `F` | Follow mode (like `tail -f`) — Ctrl+C to stop |
| `Ctrl+L` | Redraw screen |
| `R` | Repaint screen, discard buffered input |
| `m X` | Set mark X (any letter) |
| `' X` | Go to mark X |
| `''` | Go to position before last search/jump |

---

## less: Search

| Key | Action |
|-----|--------|
| `/pattern` | Search forward for pattern |
| `?pattern` | Search backward for pattern |
| `n` | Next match (same direction) |
| `N` | Previous match (reverse direction) |
| `&pattern` | Show only lines matching pattern (filter mode) |
| `&` | Clear filter (show all lines) |
| `Esc-u` | Toggle search highlight |

**Search patterns are regular expressions** (ERE by default).

```bash
# Start less with search highlighted
less +/error logfile.txt

# Case-insensitive search
less -i logfile.txt    # then /error matches Error, ERROR, error

# Filter: show only lines with "error"
# (inside less, type:)
# &error
# Press & again to clear
```

---

## less: Marks & Multiple Files

### Marks
```
m a     ← set mark 'a' at current position
' a     ← jump to mark 'a'
''      ← jump to position before last big jump
```

### Multiple files
```bash
less file1.txt file2.txt file3.txt

# Inside less:
# :n    → next file
# :p    → previous file
# :f    → show current filename
# :e newfile.txt  → open another file without leaving less
```

### Open file from within less
```
v       ← open current file in $EDITOR
:e /etc/hosts   ← switch to /etc/hosts
```

---

## more: Navigation Keys

| Key | Action |
|-----|--------|
| `Space` | Forward one screen |
| `Enter` | Forward one line |
| `b` | Backward one screen (if supported) |
| `/pattern` | Search forward |
| `n` | Next search match |
| `q` | Quit |
| `h` | Help |
| `=` | Show current line number |
| `:f` | Show filename and line |
| `!cmd` | Execute shell command |

**`more` exits automatically at end of file** — no waiting at EOF.

---

## less vs more — Full Comparison

| Feature | `less` | `more` |
|---------|--------|--------|
| Scroll backward | ✅ | ❌ (or limited) |
| Scroll forward | ✅ | ✅ |
| Search forward | ✅ | ✅ |
| Search backward | ✅ | ❌ |
| Regex search | ✅ | Limited |
| Filter lines (`&`) | ✅ | ❌ |
| Line numbers (`-N`) | ✅ | ❌ |
| Follow mode (`F`) | ✅ | ❌ |
| Multiple files | ✅ | Limited |
| Open new file (`:e`) | ✅ | ❌ |
| Marks | ✅ | ❌ |
| Pipe to command (`\|`) | ✅ | ❌ |
| ANSI colors (`-R`) | ✅ | Limited |
| Edit in `$EDITOR` (`v`) | ✅ | ❌ |
| Stays at EOF | ✅ | ❌ (exits) |
| Memory usage | Low (lazy) | Low |
| Available in minimal containers | Sometimes | Almost always |
| POSIX standard | No | Yes |
| Part of util-linux | No | ✅ |

**Rule of thumb:** Use `less` always, except when it's not available (minimal Docker containers, some embedded systems) — then use `more`.

---

## LESSOPEN: Preprocessor Support

`less` can preprocess files before display using `LESSOPEN`:

```bash
# View compressed files directly in less
export LESSOPEN="| lesspipe %s"
# or:
eval "$(lesspipe)"   # add to ~/.bashrc

# Now:
less file.gz        # automatically decompresses
less file.tar.gz    # shows contents
less image.png      # shows file info
less data.csv       # formatted view (if configured)

# lesspipe typically handles:
# .gz .bz2 .xz .Z .lz — compressed files
# .tar .tar.gz .tgz — archives (shows file list)
# .pdf — text extraction
# .deb .rpm — package contents
# .jpg .png — image info (dimensions, format)
```

---

## Environment Variables

### less

| Variable | Purpose |
|----------|---------|
| `$LESS` | Default options for less (e.g., `LESS="-RNi"`) |
| `$LESSOPEN` | Input preprocessor command |
| `$LESSCLOSE` | Cleanup command after LESSOPEN |
| `$LESSHISTFILE` | File for search history (default: `~/.lesshst`) |
| `$LESSSECURE` | If set to 1, disables shell commands (`!`, `v`) |
| `$PAGER` | System-wide pager — many tools use this |
| `$MANPAGER` | Pager used by `man` (overrides `$PAGER`) |
| `$LESSBINFMT` | Format for displaying binary chars |
| `$LESSCHARSET` | Character set for display |

```bash
# Useful defaults in ~/.bashrc:
export LESS="-RFXi"
# -R: render ANSI colors
# -F: quit if fits one screen
# -X: don't clear screen on exit
# -i: case-insensitive search

# Make man use less with color support:
export MANPAGER="less -R"
# or for colored man pages:
export MANPAGER="sh -c 'col -bx | bat -l man -p'"
```

### more

| Variable | Purpose |
|----------|---------|
| `$MORE` | Default options |
| `$TERM` | Terminal type (affects display) |

---

## Alternatives

### bat — cat with syntax highlighting and paging
```bash
bat file.py          # syntax highlighted, line numbers, git diff
bat -p file.py       # plain output (no decorations)
bat -l json file     # force language
bat *.py             # multiple files with headers

# Use bat as MANPAGER:
export MANPAGER="sh -c 'col -bx | bat -l man -p'"
```

### most — similar to less, more color support
```bash
most file.txt        # can open multiple windows
most -s file.txt     # squeeze blanks
```

### pg — older pager (now rare)
```bash
pg file.txt          # mostly historical interest
```

---

## Related Commands

| Command | Relation |
|---------|---------|
| `cat` | Display whole file at once (no paging) |
| `head` | Show first N lines |
| `tail` | Show last N lines (+ follow with `-f`) |
| `bat` | Modern pager with syntax highlighting |
| `grep` | Search without paging |
| `man` | Uses less as pager internally |
| `git log` | Pipes through less by default |
| `lesspipe` | Preprocessor for viewing compressed/binary files |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
