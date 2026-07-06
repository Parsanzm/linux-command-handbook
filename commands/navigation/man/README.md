# man — The Complete Reference

> **The manual: offline, structured documentation for nearly every Unix command, syscall, and config file**
> Introduced in the very first Unix release (1971, Bell Labs) as "the Unix Programmer's Manual"
> Still the fastest, most reliable way to get authoritative documentation without leaving the terminal.

---

## Table of Contents

- [What is man?](#what-is-man)
- [Where does man live?](#where-does-man-live)
- [How man works internally](#how-man-works-internally)
- [Syntax](#syntax)
- [The Manual Sections](#the-manual-sections)
- [Navigating a Man Page](#navigating-a-man-page)
- [Anatomy of a Man Page](#anatomy-of-a-man-page)
- [Searching Man Pages](#searching-man-pages)
- [All Key Options](#all-key-options)
- [The man Database (mandb) and Caching](#the-man-database-mandb-and-caching)
- [man vs --help vs info vs tldr](#man-vs---help-vs-info-vs-tldr)
- [Writing / Viewing Man Pages in groff/troff](#writing--viewing-man-pages-in-groff-troff)
- [Related Commands](#related-commands)

---

## What is man?

`man` displays the **manual page** for a command, system call, library function, file format, or configuration file. Every well-behaved Unix command, and most system-level facilities, ship with a man page — meaning `man` is often the single most reliable, always-available source of documentation on any Unix-like system, needing no internet connection at all.

```bash
man ls           # manual page for the ls command
man 5 passwd     # manual page for the passwd FILE FORMAT (section 5), not the command
man printf       # may show either the command or the C library function, depending on section priority
```

**Why man still matters in the AI/search-engine era:** man pages are **authoritative** (written by or reviewed alongside the actual software), **versioned to match your exact installed system** (not some other distro's config or version), and work completely **offline** — critical during remote server administration, air-gapped systems, or whenever the specific behavior of the exact binary on the exact machine in front of you matters more than a generic answer from the web.

---

## Where does man live?

```
/usr/bin/man
```

```bash
which man
man --version
# man-db 2.12.0
```

Man pages themselves are **not** part of the `man` program — they're separate files, typically compressed, stored under standard system directories:

```bash
/usr/share/man/man1/    # section 1: user commands
/usr/share/man/man5/    # section 5: file formats
/usr/share/man/man8/    # section 8: admin commands
# ... etc, one directory per section

ls /usr/share/man/man1/ | head
# ls.1.gz  cat.1.gz  grep.1.gz  ...
```

The `.gz` extension shows they're typically **gzip-compressed** on disk — `man` decompresses them on the fly when displaying, saving disk space across potentially thousands of pages.

---

## How man works internally

### The rendering pipeline

Man pages are written in a markup language called **groff** (GNU troff), a typesetting system originally designed for producing printed documentation. When you run `man ls`:

1. `man` locates the correct compressed source file (e.g., `/usr/share/man/man1/ls.1.gz`)
2. Decompresses it on the fly
3. Pipes the groff-formatted source through `groff` (or the older `nroff`) to render it into plain, formatted text suitable for a terminal
4. Pipes that rendered output into a **pager** (by default, `less`) for you to scroll through interactively

```bash
# You can see this pipeline manually by checking your pager and formatter:
echo $MANPAGER
echo $PAGER
man --debug ls 2>&1 | head -20    # shows some of man's internal resolution steps
```

### Search path resolution (MANPATH)

`man` searches a configured list of directories (similar in concept to `$PATH` for executables) to find the right page for a given name:

```bash
manpath
# /usr/share/man:/usr/local/share/man:/usr/local/man

echo $MANPATH
# usually empty unless explicitly customized — man derives its search
# path from /etc/manpath.config and $PATH by default

# Adding a custom directory of man pages (e.g., for a locally installed tool)
export MANPATH="$HOME/myapp/man:$(manpath)"
man mytool
```

### Locale-aware page selection

If localized man pages are installed (e.g., `/usr/share/man/fr/man1/ls.1.gz` for French), `man` selects the appropriate translation based on your active locale (`$LANG`, `$LC_ALL`) automatically, falling back to the default (usually English) if no translation exists for a given page.

---

## Syntax

```bash
man [SECTION] PAGE_NAME
man [OPTIONS] PAGE_NAME
```

```bash
man ls                 # the most common form — look up "ls"
man 5 passwd           # explicitly request section 5 for "passwd"
man -a printf          # show ALL man pages named "printf" across every section, one after another
man -f ls              # equivalent to whatis — one-line description
man -k network          # equivalent to apropos — search page names/descriptions for a keyword
```

---

## The Manual Sections

The manual is divided into numbered **sections**, a convention going back to the earliest Unix documentation, because the same name can refer to different things depending on context (a command vs. a system call vs. a config file format, for example):

| Section | Contents | Example |
|---------|----------|---------|
| **1** | User commands (executable programs or shell commands) | `man 1 ls` |
| **2** | System calls (kernel functions invoked from C) | `man 2 open` |
| **3** | Library calls (C standard library functions) | `man 3 printf` |
| **4** | Special files (usually devices, in `/dev`) | `man 4 tty` |
| **5** | File formats and conventions | `man 5 passwd` |
| **6** | Games | `man 6 fortune` |
| **7** | Miscellaneous (conventions, protocols, standards) | `man 7 signal` |
| **8** | System administration commands (usually root-only) | `man 8 iptables` |
| **9** | (Linux-specific, rare) Kernel routines | `man 9 kmalloc` |

### Why section numbers matter

```bash
man printf
# Without a section, man shows the FIRST match it finds, searching
# sections in a predefined priority order (commonly 1, then 8, 2, 3, 4,
# 5, 6, 7, 9 by default, though this order IS configurable) — so
# "man printf" typically shows the SHELL/COMMAND version (section 1),
# NOT the C library function (section 3), even though both exist.

man 3 printf
# Explicitly requests the C library function's documentation instead —
# essential when you're writing C code and need the exact function
# signature, return value, and behavior, not the shell command's.

man 5 passwd
# The FILE FORMAT of /etc/passwd — field meanings, syntax, structure

man 1 passwd
# The COMMAND used to change a user's password — completely different content
```

---

## Navigating a Man Page

Since `man` pipes output through `less` by default, all of `less`'s navigation keys apply:

| Key | Action |
|-----|--------|
| `Space` / `f` | Next page |
| `b` | Previous page |
| `↓` / `j` | Scroll down one line |
| `↑` / `k` | Scroll up one line |
| `/pattern` | Search forward for a pattern |
| `?pattern` | Search backward for a pattern |
| `n` | Repeat the last search, same direction |
| `N` | Repeat the last search, opposite direction |
| `g` | Jump to the very beginning |
| `G` | Jump to the very end |
| `q` | Quit and return to the shell |
| `h` | Show less's own help screen |

```bash
man grep
# Once inside the pager:
/PATTERN         # search for "PATTERN" within the page
n                # jump to next occurrence
q                # exit back to the shell
```

---

## Anatomy of a Man Page

Every man page follows a broadly consistent structural convention:

```
NAME
       ls - list directory contents

SYNOPSIS
       ls [OPTION]... [FILE]...

DESCRIPTION
       List information about the FILEs (the current directory by default).

       -a, --all
              do not ignore entries starting with .
       ...

EXAMPLES
       (not always present)

FILES
       Related configuration or data files, if any

ENVIRONMENT
       Environment variables that affect behavior (e.g., $LS_COLORS for ls)

EXIT STATUS
       Meaning of different return codes, if non-trivial

SEE ALSO
       Related commands/man pages worth checking

BUGS
       Known limitations or issues, if any

AUTHOR
       Who wrote the original program (historical interest, mostly)
```

Not every section appears in every page — a simple utility might only have NAME, SYNOPSIS, and DESCRIPTION, while something complex like `bash` or `printf` has dozens of subsections.

### Reading SYNOPSIS notation

```
ls [OPTION]... [FILE]...
```
- `[brackets]` — optional
- `...` — can repeat
- **bold/plain text** (not shown here, but present in rendered output) — literal text to type exactly
- *italics/underline* (also lost in plain-text rendering, but present in `less`) — a placeholder you replace with your own value

```
chmod [OPTION]... MODE[,MODE]... FILE...
```
Reads as: `chmod`, optional flags, one or more comma-separated MODE values, then one or more FILE arguments.

---

## Searching Man Pages

### apropos / man -k — search by keyword across all page descriptions

```bash
apropos network
# equivalent to: man -k network
# Searches every man page's NAME/description line for the keyword,
# even if you don't know the exact command name you're looking for.

apropos "copy files"
# cp (1)               - copy files and directories
# install (1)          - copy files and set attributes
# scp (1)              - secure copy (remote file copy program)
```

### whatis — one-line summary for an exact command name

```bash
whatis ls
# ls (1) - list directory contents

whatis printf
# printf (1)  - format and print data
# printf (3)  - formatted output conversion
# (shows EVERY section that has a page for this exact name)
```

### Searching WITHIN a single already-open page

```bash
man grep
/color            # while inside the pager, search for "color"
n                 # jump to the next match
```

---

## All Key Options

| Option | Description |
|--------|-------------|
| `-a` | Show ALL matching pages across sections, one after another |
| `-f` | Equivalent to `whatis` — one-line description for exact name |
| `-k` | Equivalent to `apropos` — keyword search across descriptions |
| `-K` | Search the full TEXT of every man page for a string (slow, thorough) |
| `-w` | Show the file path of the man page instead of displaying it |
| `-l` | Interpret the argument as a literal local file path, not a page name |
| `-P PAGER` | Use a specific pager instead of the default |
| `-M PATH` | Use a specific manpath instead of the default search path |
| `-L LOCALE` | Force a specific locale/language for the page |
| `-c` | Force reformatting, ignoring any cached rendered version |
| `-7` / `--ascii` | Force ASCII output, avoiding special typographic characters |

```bash
man -a printf                    # see every section's "printf" page in sequence
man -w ls                        # /usr/share/man/man1/ls.1.gz
man -K "recursive"               # search the CONTENT of every man page for "recursive"
man -l ./my_local_page.1         # view a specific file directly, bypassing the search path
```

---

## The man Database (mandb) and Caching

`apropos`/`whatis`/`man -k` rely on a pre-built **index database** (managed by `mandb`) rather than scanning every man page's content live each time — this is why they're fast even across thousands of installed pages.

```bash
# Rebuild the man page index database (needed after manually installing
# new man pages outside your package manager, or if searches seem stale)
sudo mandb

# Check when the database was last updated
mandb --version
ls -l /var/cache/man/  # (path varies by distro)

# If apropos/whatis return NOTHING even for pages you know exist:
apropos ls
# nothing appears — likely the mandb index was never built
sudo mandb
apropos ls
# ls (1)  - list directory contents   ✅ now it works
```

---

## man vs --help vs info vs tldr

| Tool | Depth | Best for |
|------|-------|----------|
| `command --help` | Shortest — usually just a flag list | Quick reminder of exact flag syntax when you already know the command |
| `man command` | Full, authoritative, detailed | Complete reference: every flag, edge case, related file, exit code |
| `info command` | Sometimes even MORE detailed than man, hyperlinked/navigable (GNU-specific) | Deep-dive into GNU tools that have extensive info documentation (e.g., `info coreutils`) |
| `tldr command` (third-party, needs install) | Extremely short, example-driven, community-maintained | Fast "just show me a working example" lookup, especially for less-common tools |

```bash
ls --help          # short flag summary, printed instantly, no pager
man ls              # full manual page, in a pager, with every detail
info ls             # GNU's alternative documentation system, if installed
tldr ls             # community-maintained quick examples (separate tool, not built-in)
```

**Rule of thumb:** reach for `--help` when you just need to jump your memory on a flag; reach for `man` when you need the authoritative, complete picture (exit codes, edge cases, related files, environment variables); reach for `info` specifically for GNU tools whose `info` pages are richer than their `man` pages (classic examples: `coreutils`, `tar`, `sed`); reach for `tldr` (if installed) when you just want a working example without reading prose.

---

## Writing / Viewing Man Pages in groff/troff

Man pages are authored in **groff's `man` macro package**, a specific set of formatting macros built on top of the general-purpose `troff` typesetting language.

```groff
.TH LS 1 "January 2024" "GNU coreutils 9.4" "User Commands"
.SH NAME
ls \- list directory contents
.SH SYNOPSIS
.B ls
[\fIOPTION\fR]... [\fIFILE\fR]...
.SH DESCRIPTION
List information about the FILEs.
```

- `.TH` — title heading (command name, section, date, source, manual name)
- `.SH` — section header (NAME, SYNOPSIS, DESCRIPTION, etc.)
- `.B` — bold text (typically literal commands/flags)
- `\fI...\fR` — italic text (typically placeholders/arguments)
- `\-` — a literal hyphen (escaped so groff doesn't treat it as a formatting minus)

```bash
# View the RAW groff source of an installed man page (before rendering)
zcat /usr/share/man/man1/ls.1.gz | head -30

# Render a groff source file manually, without going through `man` at all
groff -man -Tascii ./mypage.1 | less
```

---

## Related Commands

| Command | Relation |
|---------|----------|
| `apropos` | Keyword search across all man page descriptions (same as `man -k`) |
| `whatis` | One-line description for an exact name (same as `man -f`) |
| `mandb` | Rebuild the search index database used by apropos/whatis |
| `info` | GNU's alternative, often more detailed documentation system |
| `--help` | Quick built-in flag summary most commands provide directly |
| `less` | The pager `man` uses by default to display content |
| `groff` / `nroff` | The typesetting engines that render man page source into readable text |
| `manpath` | Show the current search path man uses to locate pages |
| `col` | Historically used to strip formatting control characters from man output |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
