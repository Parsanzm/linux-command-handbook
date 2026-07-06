# man — Edge Cases & Gotchas

> man feels simple until section ambiguity, missing pages, stale indexes, or
> locale mismatches leave you staring at the wrong documentation entirely.

---

## Table of Contents

- [Section Ambiguity — Getting the Wrong Page](#section-ambiguity--getting-the-wrong-page)
- [No Manual Entry — Missing Man Pages](#no-manual-entry--missing-man-pages)
- [apropos/whatis Returning Nothing (Stale mandb)](#aproposwhatis-returning-nothing-stale-mandb)
- [Shell Builtins Have No Standalone Man Page](#shell-builtins-have-no-standalone-man-page)
- [man Shows an Outdated Page vs the Installed Binary](#man-shows-an-outdated-page-vs-the-installed-binary)
- [Locale Mismatches Show the Wrong Language (or Broken Formatting)](#locale-mismatches-show-the-wrong-language-or-broken-formatting)
- [Pager Differences Change Navigation Keys](#pager-differences-change-navigation-keys)
- [Piping man Output Loses Formatting Unexpectedly](#piping-man-output-loses-formatting-unexpectedly)
- [man -k / apropos False Positives from Overly Broad Keywords](#man--k--apropos-false-positives-from-overly-broad-keywords)
- ["What Manual Page Do I Even Need?" — Same Name, Different Sections](#what-manual-page-do-i-even-need--same-name-different-sections)
- [Containers and Minimal Images Often Ship With NO man Pages](#containers-and-minimal-images-often-ship-with-no-man-pages)
- [Terminal Width Reflows Content Unexpectedly](#terminal-width-reflows-content-unexpectedly)
- [man Pages Can Be Out of Sync With --help](#man-pages-can-be-out-of-sync-with---help)

---

## Section Ambiguity — Getting the Wrong Page

### `man name` doesn't always show what you expect
```bash
man printf
# Shows section 1 (the SHELL command printf), because man's default
# section search order puts commands before library functions —
# even though you might have actually wanted the C function's exact
# format-string behavior.

man 3 printf
# ✅ explicitly requests the C LIBRARY function instead

# This bites people constantly with names that exist in MULTIPLE
# sections: printf, crontab, passwd, socket, kill, exit...
man crontab
# Shows section 1 (the crontab COMMAND) by default

man 5 crontab
# ✅ shows the crontab FILE FORMAT (syntax of the actual crontab file
# content) — completely different, equally important information

# When in doubt, check what's available first:
whatis crontab
# crontab (1)  - maintain crontab files for individual users
# crontab (5)  - tables for driving cron
```

---

## No Manual Entry — Missing Man Pages

### Not every installed command actually ships a man page
```bash
man mycustomtool
# No manual entry for mycustomtool

# This is common for:
# - shell builtins that don't have their OWN standalone page (see below)
# - locally-written scripts placed directly in $PATH without documentation
# - minimal/embedded systems that strip man pages to save space
# - some newer tools that only ship --help text, relying on it entirely
#   instead of a full man page (increasingly common in the Go/Rust
#   CLI tool ecosystem, e.g., many modern tools ship no man page at all)

mycustomtool --help
# Often the ONLY documentation available for such tools

# Confirm definitively whether a page exists before assuming it's a
# system problem rather than simply absent:
man -w mycustomtool
# man: no manual entry for mycustomtool (confirms it's genuinely missing)
```

---

## apropos/whatis Returning Nothing (Stale mandb)

### The keyword search index needs to be built at least once
```bash
apropos network
# nothing found for network

man network
# (the actual page might exist and display fine!)

# apropos/whatis rely on a SEPARATE pre-built index (managed by mandb),
# NOT a live search of installed pages — if that index was never built
# (common right after a minimal OS install, or after manually copying
# man pages into place outside the package manager), keyword search
# comes back completely empty even though direct lookups work fine.

sudo mandb
# Processing manual pages...
apropos network
# ss (8)      - another utility to investigate sockets
# netstat (8) - Print network connections...
# ✅ now it works
```

---

## Shell Builtins Have No Standalone Man Page

### `man cd` shows something surprising (or nothing useful)
```bash
man cd
# man: No manual entry for cd
# OR, on some systems:
# Shows the bash(1) page directly, or a stub redirecting you there,
# because "cd" is a shell BUILTIN, not a standalone executable file —
# there's no /usr/bin/cd for man to document independently.

# The REAL documentation for shell builtins lives inside the shell's
# OWN big man page, under a "SHELL BUILTIN COMMANDS" section:
man bash
/^\s*cd \[
# search within bash's own man page to find the "cd" builtin's
# specific documentation

# Alternatively, bash provides a dedicated builtin-help command:
help cd
# cd: cd [-L|[-P [-e]] [-@]] [dir]
#     Change the shell working directory.
```

---

## man Shows an Outdated Page vs the Installed Binary

### The man page and the actual binary can drift out of sync
```bash
mytool --version
# mytool version 2.5.0 (installed via a manual build or non-package method)

man mytool
# Shows documentation that might describe version 1.8's behavior,
# if the man page was installed separately (or bundled with an older
# package) and never updated alongside a manually-compiled newer binary.

# This commonly happens when:
# - software was built from source without proper "make install" of docs
# - a newer binary was manually dropped into /usr/local/bin, but the
#   man page still comes from the distro's OLDER packaged version
# - a tool was updated via a language-specific package manager (pip,
#   npm, cargo) that doesn't install/update man pages at all

# Always cross-check with --help or --version output if behavior
# described in `man` doesn't match what you're actually observing:
mytool --help
mytool --version
man -w mytool    # check WHERE the page is actually coming from
```

---

## Locale Mismatches Show the Wrong Language (or Broken Formatting)

### $LANG affects which translated page (if any) man selects
```bash
echo $LANG
# fr_FR.UTF-8

man ls
# May show a FRENCH-translated man page if one is installed, which
# could be less detailed, outdated relative to the English original,
# or occasionally have formatting glitches from imperfect translation
# markup — genuinely confusing if you were expecting English.

# Force English explicitly for a single lookup:
LANG=C man ls
# or:
man -L C ls          # some man-db versions support -L directly

# Persist the fix if this bothers you often:
export LC_MESSAGES=en_US.UTF-8   # affects man's language selection
# without necessarily changing your ENTIRE locale/formatting setup
```

---

## Pager Differences Change Navigation Keys

### $PAGER or $MANPAGER overrides can silently change how man "feels"
```bash
export MANPAGER="more"
man ls
# 'q' to quit still generally works, but search (/), backward scroll (b),
# and other less-specific keybindings may behave DIFFERENTLY or not
# work at all — `more` is a simpler, less feature-rich pager than `less`.

# Check which pager man will actually use:
echo $MANPAGER
echo $PAGER
man --debug ls 2>&1 | grep -i pager

# Reset to the standard, most fully-featured default:
unset MANPAGER
man ls
```

---

## Piping man Output Loses Formatting Unexpectedly

### Bold/underline become garbage control characters without `col`
```bash
man ls > ls_output.txt
cat ls_output.txt
# ls(1)      ls^H^Hls(1)     ← garbled backspace/overstrike sequences
# NAME^H^H^H^HNAME
# These ^H (backspace) sequences are man's classic way of encoding
# BOLD text for terminals (character, backspace, character again) —
# perfectly readable live in a terminal, but ugly and confusing when
# redirected to a plain file and viewed elsewhere.

# Fix: strip the formatting control sequences with `col -bx`
man ls | col -bx > ls_clean.txt
cat ls_clean.txt
# NAME
#        ls - list directory contents    ✅ clean, readable plain text
```

---

## man -k / apropos False Positives from Overly Broad Keywords

### A common English word matches far more pages than expected
```bash
apropos file
# Returns potentially HUNDREDS of matches, since "file" appears in the
# description of an enormous number of commands (nearly everything
# touches "files" in some way) — the results become nearly useless
# without further filtering.

apropos file | wc -l
# 200+   ← too broad to be useful as-is

# Narrow with a more specific multi-word phrase, or pipe into grep:
apropos "compare files"
apropos file | grep -i compress
```

---

## "What Manual Page Do I Even Need?" — Same Name, Different Sections

### whatis reveals the full picture before you commit to `man name`
```bash
whatis socket
# socket (2)  - create an endpoint for communication
# socket (7)  - Linux socket interface

# Both are legitimate, but answer completely different questions:
# section 2 = the actual syscall signature/behavior for C programmers
# section 7 = a broader conceptual OVERVIEW of the socket interface

man 2 socket    # if you're calling socket() from C and need the exact signature
man 7 socket    # if you want to understand sockets conceptually/broadly
```

---

## Containers and Minimal Images Often Ship With NO man Pages

### man appears "broken" simply because it was never installed
```bash
docker run -it ubuntu:24.04 bash
man ls
# bash: man: command not found
# Minimal container base images frequently strip man-db and all man
# page packages entirely to reduce image size — this is expected
# behavior, not a bug, and doesn't indicate anything wrong with `ls`
# itself, which works completely normally despite having no man page.

apt-get update && apt-get install -y man-db manpages
man ls
# ✅ now works, after explicitly installing the documentation packages
```

---

## Terminal Width Reflows Content Unexpectedly

### Resizing your terminal mid-session doesn't reflow an already-open page
```bash
man ls
# opened while your terminal is 80 columns wide — text is wrapped
# and formatted for that width

# ... you resize your terminal window wider while the page is still open ...

# The ALREADY-RENDERED text does NOT reflow live to the new width;
# you'd need to quit ('q') and reopen `man ls` for it to re-render
# using the new terminal dimensions, since man renders once up front
# rather than continuously re-wrapping like some modern text editors.
```

---

## man Pages Can Be Out of Sync With --help

### The two documentation sources aren't guaranteed to agree
```bash
mytool --help
# Shows a NEWLY ADDED --json flag

man mytool
# Doesn't mention --json anywhere — the man page simply wasn't updated
# in the same release/commit that added the flag to the actual --help
# text output, a common maintenance gap in fast-moving projects.

# When they conflict, --help (generated directly from the CURRENT
# running binary's own argument parser, in most well-built tools) is
# usually more likely to reflect the exact installed version's real
# behavior than a man page that may have lagged behind during development.
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
