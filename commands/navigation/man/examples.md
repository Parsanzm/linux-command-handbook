# man — Practical Examples

> Real-world patterns for looking things up fast, searching by keyword, and navigating efficiently.

---

## Table of Contents

- [Basic Lookups](#basic-lookups)
- [Section-Specific Lookups](#section-specific-lookups)
- [Keyword & Description Search](#keyword--description-search)
- [Navigating Long Pages](#navigating-long-pages)
- [Finding Man Page File Locations](#finding-man-page-file-locations)
- [Searching Man Page Content](#searching-man-page-content)
- [Working with Custom / Local Man Pages](#working-with-custom--local-man-pages)
- [Saving or Printing Man Pages](#saving-or-printing-man-pages)
- [Scripting Around man](#scripting-around-man)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Lookups

```bash
# The most common form
man ls
man grep
man tar

# Quit a man page: press 'q'

# Look up a page, then immediately search for a specific flag once inside
man chmod
# then type: /recursive
# then: n     (to jump to the next match)
```

---

## Section-Specific Lookups

```bash
# The file format of /etc/passwd (section 5), not the passwd COMMAND
man 5 passwd

# The passwd COMMAND itself (section 1)
man 1 passwd

# The C library printf() function (section 3), not the shell command
man 3 printf

# System call documentation, useful when reading C source or strace output
man 2 open
man 2 fork
man 2 socket

# Admin/root-level commands (section 8)
man 8 iptables
man 8 useradd

# See every section that has a page for this name, one after another
man -a printf
```

---

## Keyword & Description Search

```bash
# Don't know the exact command name? Search by what it DOES
apropos "list directory"
man -k "list directory"        # identical to apropos

apropos compress
# gzip (1)             - compress or expand files
# zip (1)              - package and compress (archive) files
# bzip2 (1)             - a block-sorting file compressor

# Quick one-line reminder of what a command you already know does
whatis ls
# ls (1) - list directory contents

whatis grep
# grep (1) - print lines that match patterns

# See EVERY section a name appears in (useful when you're not sure
# if something is a command, syscall, or library function)
whatis printf
# printf (1)  - format and print data
# printf (3)  - formatted output conversion
```

---

## Navigating Long Pages

```bash
man bash
# Once inside the pager (less):

# Jump to a specific section by searching for its heading
/^ARITHMETIC EVALUATION
n                    # repeat search if not the first match

# Jump straight to the very end (often has SEE ALSO / AUTHOR / BUGS)
G

# Jump back to the very beginning
g

# Search for a specific flag's documentation
/^\s*-c\b

# Case-insensitive search inside the page
-i                   # toggles case sensitivity in less, then search normally
/pattern
```

---

## Finding Man Page File Locations

```bash
# Show the actual file path of the man page without displaying it
man -w ls
# /usr/share/man/man1/ls.1.gz

man -w 5 passwd
# /usr/share/man/man5/passwd.5.gz

# Combine with other tools once you have the path
zcat $(man -w ls) | head -50    # peek at the raw groff source

# Check your current search path for man pages
manpath
echo $MANPATH
```

---

## Searching Man Page Content

```bash
# Full-text search across EVERY installed man page (slow but thorough)
man -K "recursive delete"

# man -K opens each matching page one at a time — press 'q' to move
# to the next match, or Ctrl+C to stop searching entirely

# Faster alternative for a quick, targeted grep-style search across
# man page SOURCE files directly (bypasses man's own formatting)
zgrep -l "recursive" /usr/share/man/man1/*.gz | head -10
```

---

## Working with Custom / Local Man Pages

```bash
# View a specific man page FILE directly, without it being "installed"
# anywhere in the standard search path
man -l ./mytool.1
man ./mytool.1.gz     # also works directly on a compressed file in many versions

# Add a custom directory of man pages to your search path permanently
export MANPATH="$HOME/local/man:$(manpath)"
echo 'export MANPATH="$HOME/local/man:$(manpath)"' >> ~/.bashrc

# After installing a tool that ships its own man page in a nonstandard
# location, rebuild the index so apropos/whatis pick it up
sudo mandb

# Render a groff source file manually to preview it before installing
groff -man -Tascii ./mytool.1 | less
groff -man -Tpdf ./mytool.1 > mytool.pdf     # export as PDF for sharing
```

---

## Saving or Printing Man Pages

```bash
# Save a man page as plain text, stripped of terminal formatting codes
man ls | col -bx > ls_manual.txt

# Save with formatting preserved (viewable later with `less -R`)
man ls | col -bx | less -R

# Convert a man page to PDF for offline reading/printing
man -Tpdf ls > ls.pdf

# Convert to HTML (if groff's html driver is available)
man -Thtml ls > ls.html

# Print directly to a physical printer (classic Unix workflow)
man -t ls | lpr
```

---

## Scripting Around man

```bash
# Check if a man page exists before trying to display it in a script
if man -w somecommand > /dev/null 2>&1; then
  echo "man page exists"
else
  echo "no man page found for somecommand"
fi

# Extract just the one-line description for use in another tool/report
whatis ls | head -1

# Generate a quick reference sheet of installed commands and their descriptions
for cmd in ls grep awk sed find; do
  whatis "$cmd" 2>/dev/null
done

# Check which of several commands actually HAS documentation installed
for cmd in mytool othertool thirdtool; do
  man -w "$cmd" >/dev/null 2>&1 && echo "$cmd: documented" || echo "$cmd: NO man page"
done
```

---

## Real-World Recipes

```bash
# --- Learning a New Command From Scratch ---

whatis tar                 # quick one-liner: what does it even do?
man tar                    # full detail
man tar | grep -A 3 "^\s*-z"   # find just the -z flag's explanation quickly

# --- Debugging a Config File Format ---

man 5 sshd_config          # the FILE FORMAT, not the sshd command itself
man 5 crontab               # crontab FILE syntax, distinct from `man 1 crontab` (the command)

# --- Finding the Right Tool for a Job (don't know the command name yet) ---

apropos "disk usage"
# du (1)     - estimate file space usage
# df (1)     - report file system disk space usage

# --- Understanding a C System Call While Reading Source Code ---

man 2 read
man 2 write
man 7 signal                # the signal(7) OVERVIEW page, not a specific syscall

# --- Onboarding a New Team Member to a Server ---

man -k . | wc -l            # rough count of how many man pages are installed system-wide
apropos network | less      # explore everything network-related available on this box

# --- Verifying an Obscure Flag Before Running a Destructive Command ---

man rm
/--no-preserve-root
q
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
