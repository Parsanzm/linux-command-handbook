# sed — The Complete Reference

> **Stream EDitor: transform text line-by-line, without opening an editor at all**
> Written by Lee E. McMahon at Bell Labs in 1974, derived from the earlier `qed`/`ed` line editors.
> The workhorse of Unix text transformation — powers countless scripts, build systems, and one-liners.

---

## Table of Contents

- [What is sed?](#what-is-sed)
- [Where does sed live?](#where-does-sed-live)
- [How sed works internally — the pattern space model](#how-sed-works-internally--the-pattern-space-model)
- [Syntax](#syntax)
- [The Substitute Command (s///)](#the-substitute-command-s)
- [Addresses — Selecting Which Lines to Act On](#addresses--selecting-which-lines-to-act-on)
- [Common sed Commands](#common-sed-commands)
- [Regular Expressions in sed (BRE vs ERE)](#regular-expressions-in-sed-bre-vs-ere)
- [The Hold Space](#the-hold-space)
- [In-Place Editing (-i)](#in-place-editing--i)
- [All Key Options](#all-key-options)
- [sed Scripts (Multi-Command Files)](#sed-scripts-multi-command-files)
- [GNU sed vs BSD/macOS sed](#gnu-sed-vs-bsdmacos-sed)
- [sed vs awk vs perl](#sed-vs-awk-vs-perl)
- [Related Commands](#related-commands)

---

## What is sed?

`sed` (**Stream EDitor**) reads input **line by line**, applies a set of editing commands to each line, and writes the result to standard output — all without ever loading the entire file into an interactive editor or requiring user interaction. It's designed for **non-interactive, scriptable, repeatable** text transformation: the same sed command run twice on the same input always produces the same output.

```bash
echo "hello world" | sed 's/world/there/'
# hello there

sed 's/foo/bar/' file.txt          # print file.txt with foo→bar substitution, to stdout
sed -i 's/foo/bar/' file.txt       # same, but edit the FILE in place
```

**Why sed exists (and still matters):** it's built into essentially every Unix-like system, requires no extra installation, handles arbitrarily large files efficiently (processes one line at a time, not the whole file in memory), and is the standard tool baked into countless build scripts, CI pipelines, `Makefile`s, and shell one-liners for tasks like find-and-replace, deleting lines matching a pattern, or extracting specific line ranges.

---

## Where does sed live?

```
/bin/sed
/usr/bin/sed
```

```bash
which sed
sed --version
# sed (GNU sed) 4.9
```

Part of **GNU coreutils/sed** package on Linux. **macOS ships a different, BSD-derived sed** with meaningfully different syntax for several common operations — see [GNU sed vs BSD/macOS sed](#gnu-sed-vs-bsdmacos-sed) below, a frequent source of "works on my machine" bugs.

---

## How sed works internally — the pattern space model

### The core execution loop

sed's entire model can be summarized as a loop:

```
for each line of input:
    1. Load the line into the "pattern space" (a temporary buffer)
    2. Run every command in the sed script against the pattern space, in order
    3. Unless suppressed, print the pattern space to output
    4. Move to the next line
```

```bash
# Conceptually, this is what happens for EACH line when you run:
sed 's/cat/dog/' file.txt
# Line loaded into pattern space -> s/cat/dog/ applied -> printed -> repeat
```

### Pattern space vs hold space

sed maintains **two** buffers:
- **Pattern space** — holds the current line being processed (the "working" buffer)
- **Hold space** — a separate, persistent scratch buffer you can copy data into and out of, which survives across lines (used for multi-line operations — see [The Hold Space](#the-hold-space))

### Line-oriented, not file-oriented

Because sed processes one line at a time by default, it's memory-efficient even on enormous files — it never needs to load an entire multi-gigabyte file into memory at once, unlike some naive scripting approaches. This is also *why* multi-line pattern matching (matching something that spans across a newline) requires special handling (the hold space, or the `N` command) rather than "just working" the way a whole-file regex tool might.

---

## Syntax

```bash
sed [OPTIONS] 'SCRIPT' [FILE...]
sed [OPTIONS] -e 'SCRIPT1' -e 'SCRIPT2' [FILE...]
sed [OPTIONS] -f SCRIPT_FILE [FILE...]
```

```bash
# Basic substitution, reading from a file
sed 's/old/new/' input.txt

# Reading from stdin (no file argument, or explicit -)
cat input.txt | sed 's/old/new/'
echo "text" | sed 's/old/new/'

# Multiple separate commands with -e
sed -e 's/foo/bar/' -e 's/baz/qux/' file.txt

# Load commands from a script FILE instead of the command line
sed -f myscript.sed file.txt
```

---

## The Substitute Command (s///)

`s///` is by far the most commonly used sed command — find and replace.

```bash
sed 's/PATTERN/REPLACEMENT/FLAGS'
```

```bash
# Replace the FIRST occurrence on each line (default behavior)
echo "cat cat cat" | sed 's/cat/dog/'
# dog cat cat

# Replace ALL occurrences on each line — the "g" (global) flag
echo "cat cat cat" | sed 's/cat/dog/g'
# dog dog dog

# Replace only the Nth occurrence on each line
echo "cat cat cat" | sed 's/cat/dog/2'
# cat dog cat

# Combine: replace the 2nd occurrence AND everything after it
echo "cat cat cat cat" | sed 's/cat/dog/2g'
# cat dog dog dog

# Case-insensitive matching — the "i" (or "I") flag
echo "Cat CAT cat" | sed 's/cat/dog/gi'
# dog dog dog

# Print only lines where a substitution occurred — the "p" flag,
# combined with -n to suppress default automatic printing
sed -n 's/error/ERROR/p' logfile.txt

# Use a different delimiter when the pattern contains slashes
sed 's|/usr/local|/opt|' paths.txt
sed 's#/var/www#/srv/web#' paths.txt
```

### Backreferences in the replacement

```bash
# \1, \2, etc. refer to captured groups from the pattern
echo "John Smith" | sed 's/\(\w*\) \(\w*\)/\2 \1/'
# Smith John

# Wrap matched text with something else, referencing the whole match with &
echo "important" | sed 's/important/**&**/'
# **important**

echo "2024-01-15" | sed 's/\([0-9]*\)-\([0-9]*\)-\([0-9]*\)/\3\/\2\/\1/'
# 15/01/2024
```

---

## Addresses — Selecting Which Lines to Act On

By default, a sed command applies to **every** line. Prefixing a command with an **address** restricts it to specific line(s).

```bash
# Act only on line 3
sed '3s/foo/bar/' file.txt

# Act on a range of lines (3 through 7)
sed '3,7s/foo/bar/' file.txt

# Act from line 5 to the end of the file
sed '5,$s/foo/bar/' file.txt

# Act only on lines matching a regex pattern
sed '/^ERROR/s/foo/bar/' file.txt

# Act on a RANGE defined by two patterns (from a "start" line to an "end" line)
sed '/BEGIN/,/END/s/foo/bar/' file.txt

# Negate an address — act on lines that DON'T match
sed '/^#/!s/foo/bar/' file.txt      # skip comment lines starting with #

# Step addressing (GNU extension): every Nth line starting from a given line
sed '0~2s/foo/bar/' file.txt         # every EVEN-numbered line (0, 2, 4, ...)
sed '1~2s/foo/bar/' file.txt         # every ODD-numbered line (1, 3, 5, ...)
```

---

## Common sed Commands

Beyond `s///`, sed has a full set of single-letter commands:

| Command | Meaning | Example |
|---------|---------|---------|
| `s` | Substitute | `sed 's/old/new/'` |
| `d` | Delete the matched line(s) | `sed '/^#/d'` — delete comment lines |
| `p` | Print the pattern space (usually paired with `-n`) | `sed -n '3p'` — print only line 3 |
| `a` | Append text after the matched line | `sed '3a\new line'` |
| `i` | Insert text before the matched line | `sed '3i\new line'` |
| `c` | Change (replace) the matched line entirely | `sed '3c\replacement'` |
| `y` | Transliterate characters (like `tr`) | `sed 'y/abc/xyz/'` |
| `q` | Quit processing immediately | `sed '10q'` — stop after line 10 |
| `Q` | Quit WITHOUT printing the current pattern space | `sed '/STOP/Q'` |
| `n` | Move to the next line, printing the current one first | used in multi-line scripts |
| `N` | Append the next line to the pattern space (multi-line) | `sed 'N;s/\n/ /'` |
| `=` | Print the current line number | `sed '='` |
| `l` | Print the line unambiguously (showing hidden chars) | `sed -n 'l'` |
| `r` | Read and insert content from another file | `sed '3r extra.txt'` |
| `w` | Write matched lines to another file | `sed '/error/w errors.log'` |

```bash
# Print only lines 5 through 10
sed -n '5,10p' file.txt

# Delete blank lines
sed '/^$/d' file.txt

# Delete lines matching a pattern
sed '/DEBUG/d' logfile.txt

# Print line numbers alongside content (like cat -n, roughly)
sed = file.txt | sed 'N;s/\n/\t/'

# Stop processing after the first match, without reading the rest of the file
sed '/STOP_HERE/q' hugefile.txt
```

---

## Regular Expressions in sed (BRE vs ERE)

By default, sed uses **Basic Regular Expressions (BRE)**, where several metacharacters need to be **escaped** to have their special meaning (the opposite of what many people expect coming from other tools):

```bash
# BRE (default): + and ? and | and () need BACKSLASH to be "special"
echo "color colour" | sed 's/colou\?r/COLOR/g'
# COLOR COLOR
# \? means "0 or 1 of the preceding character" in BRE

echo "aaa" | sed 's/a\+/X/'
# X
# \+ means "1 or more" in BRE

echo "cat dog" | sed 's/cat\|dog/pet/g'
# pet pet
# \| means "OR" in BRE

# ERE (Extended Regular Expressions) via -E (or -r on GNU sed): the
# SAME symbols work WITHOUT backslashes, matching Perl/grep -E conventions
sed -E 's/colou?r/COLOR/g' <<< "color colour"
# COLOR COLOR

sed -E 's/a+/X/' <<< "aaa"
# X

sed -E 's/cat|dog/pet/g' <<< "cat dog"
# pet pet
```

| Feature | BRE (default) | ERE (`-E` / `-r`) |
|---------|-----------------|----------------------|
| Grouping `()` | `\(...\)` | `(...)` |
| Alternation `\|` | `\|` (escaped) | `|` |
| One-or-more `+` | `\+` | `+` |
| Zero-or-one `?` | `\?` | `?` |
| Backreference `\1` | `\1` (same in both) | `\1` (same in both) |

**Practical advice:** use `-E` (widely portable across GNU and BSD sed) whenever your pattern needs grouping, alternation, or `+`/`?` quantifiers — it avoids the backslash-heavy BRE syntax entirely and matches what most people expect from `grep -E` or other regex-using tools.

---

## The Hold Space

The **hold space** is a second buffer, separate from the pattern space, that persists across lines — the mechanism sed uses for anything requiring "memory" of previous lines.

| Command | Effect |
|---------|--------|
| `h` | Copy pattern space → hold space (overwrite) |
| `H` | Append pattern space → hold space (with a newline) |
| `g` | Copy hold space → pattern space (overwrite) |
| `G` | Append hold space → pattern space (with a newline) |
| `x` | Exchange pattern space and hold space |

```bash
# Reverse the order of lines in a file (a classic sed "tac" implementation)
sed -n '1!G;h;$p' file.txt
# Explanation:
# 1!G  - for every line EXCEPT the first, append hold space to pattern space
# h    - save the (growing) pattern space into hold space
# $p   - on the LAST line, print the fully accumulated (reversed) result

# Print the last line of a file (like `tail -1`, roughly)
sed '$!d' file.txt

# Double-space a file (insert a blank line after every line)
sed 'G' file.txt

# Join every pair of lines into one (using N + hold space concepts)
sed 'N;s/\n/ /' file.txt
```

---

## In-Place Editing (-i)

```bash
# Edit the file DIRECTLY, no backup (⚠️ irreversible without version control)
sed -i 's/foo/bar/' file.txt

# Edit in place, but keep a backup with a given suffix (GNU sed)
sed -i.bak 's/foo/bar/' file.txt
# creates file.txt.bak (original) and modifies file.txt directly

# Edit multiple files at once, each modified independently
sed -i 's/foo/bar/' *.txt

# ⚠️ macOS/BSD sed REQUIRES an explicit argument to -i (even if empty)
sed -i '' 's/foo/bar/' file.txt        # macOS: empty string = no backup
sed -i.bak 's/foo/bar/' file.txt       # macOS: with a backup suffix — this part matches GNU
```

**Critical portability note:** `sed -i 's/foo/bar/' file.txt` (no argument between `-i` and the script) works on GNU sed but **fails or behaves incorrectly** on macOS/BSD sed, which requires an explicit (even empty) suffix argument — see [GNU sed vs BSD/macOS sed](#gnu-sed-vs-bsdmacos-sed) for the full comparison and safe cross-platform patterns.

---

## All Key Options

| Option | Long | Description |
|--------|------|-------------|
| `-e SCRIPT` | `--expression=SCRIPT` | Add a script/command (allows multiple `-e` for several commands) |
| `-f FILE` | `--file=FILE` | Read sed commands from a script file |
| `-i[SUFFIX]` | `--in-place[=SUFFIX]` | Edit files in place, optional backup suffix |
| `-n` | `--quiet` / `--silent` | Suppress automatic printing (use with explicit `p`) |
| `-E` / `-r` | `--regexp-extended` | Use Extended Regular Expressions (ERE) instead of BRE |
| `-s` | `--separate` | Treat multiple input files as separate streams (not one continuous stream) |
| `-z` | `--null-data` | Treat input as NUL-separated "lines" instead of newline-separated |
| `--posix` | | Disable GNU sed extensions, strict POSIX behavior only |
| `-u` | `--unbuffered` | Minimally buffer output — useful for real-time log filtering |

```bash
sed -n '/error/p' logfile.txt          # print only matching lines
sed -E 's/(foo|bar)/baz/g' file.txt     # ERE syntax
sed -s '$=' file1.txt file2.txt         # treat each file's line numbering separately
```

---

## sed Scripts (Multi-Command Files)

For complex transformations, put multiple commands in a `.sed` script file instead of a long, hard-to-read one-liner:

```bash
# cleanup.sed
s/[ \t]*$//        # remove trailing whitespace
/^$/d              # remove blank lines
s/foo/bar/g        # replace foo with bar globally
```

```bash
sed -f cleanup.sed messy_file.txt
sed -i -f cleanup.sed messy_file.txt    # apply in place
```

Comments in a sed script file start with `#` on their own line (or, in GNU sed, can trail after a command in some contexts, though this varies by implementation).

---

## GNU sed vs BSD/macOS sed

| Feature | GNU sed (Linux) | BSD sed (macOS default) |
|---------|-------------------|------------------------------|
| `-i` without a suffix | Works, edits with no backup | **Requires** an explicit argument (even empty string) |
| Extended regex flag | `-E` or `-r` | `-E` only (no `-r`) |
| `\+`, `\?`, `\|` in BRE | Supported as GNU extensions | Not supported — must use `-E`/ERE instead |
| Step addressing (`0~2`) | Supported | Not supported |
| Case-insensitive flag (`I` on the `s` command) | Supported | Not supported (must restructure the pattern) |
| In-place editing across multiple files | Straightforward | Same basic syntax, but `-i` quirk above still applies |

```bash
# This works on GNU sed (Linux):
sed -i 's/foo/bar/' file.txt

# This FAILS or behaves unexpectedly on macOS/BSD sed:
sed -i 's/foo/bar/' file.txt
# sed: 1: "file.txt": undefined label 'ile.txt'
# (BSD sed interprets 's/foo/bar/' as the SUFFIX argument for -i,
# and 'file.txt' incorrectly as the script itself!)

# The portable, cross-platform-safe form (works identically on both):
sed -i.bak 's/foo/bar/' file.txt        # always specify a suffix explicitly
# then remove the backup afterward if you don't want to keep it:
rm file.txt.bak
```

**Best practice for scripts meant to run on both Linux and macOS:** always provide an explicit suffix (even if you delete the backup immediately after), or detect the platform and branch accordingly, or simply install GNU sed on macOS via Homebrew (`brew install gnu-sed`, available as `gsed`) for consistent behavior.

---

## sed vs awk vs perl

| Feature | `sed` | `awk` | `perl -pe` |
|---------|-------|-------|--------------|
| Best for | Simple line-based substitution, deletion, insertion | Field/column-based processing, calculations, reports | Complex regex, full scripting power inline |
| Regex engine | BRE/ERE (POSIX-ish) | POSIX ERE (mostly) | Full PCRE (most powerful/flexible) |
| Learning curve | Low for basics, cryptic for advanced (hold space) | Moderate | Higher, but most expressive |
| Field/column awareness | None built-in (must use regex tricks) | Native `$1`, `$2`, `NF`, etc. | Available via splitting, but not built-in like awk |
| Multi-line/state across lines | Awkward (hold space) | More natural with arrays/variables | Very natural — full programming language |

```bash
# Same task, three tools: replace "foo" with "bar" everywhere
sed 's/foo/bar/g' file.txt
awk '{gsub(/foo/, "bar"); print}' file.txt
perl -pe 's/foo/bar/g' file.txt

# Task suited to awk's strengths (column-based) — sed CAN do this, but awkwardly
awk '{print $1, $3}' file.txt          # print 1st and 3rd whitespace-separated fields

# Task suited to perl's strengths (complex logic/regex) — sed struggles here
perl -pe 's/(\d{4})-(\d{2})-(\d{2})/$3\/$2\/$1/ if /^\d{4}-\d{2}-\d{2}$/' dates.txt
```

**Rule of thumb:** reach for `sed` for straightforward substitution/deletion/insertion tasks on a line-by-line basis; reach for `awk` the moment you're thinking in terms of columns/fields or need simple calculations; reach for `perl` (or Python) when the transformation logic becomes genuinely complex, involves multi-line lookahead, or needs a real programming language's control flow.

---

## Related Commands

| Command | Relation |
|---------|----------|
| `awk` | Field/column-oriented text processing, often used alongside or instead of sed |
| `grep` | Search/filter lines by pattern (sed can filter too, but grep is the dedicated tool) |
| `tr` | Simple character-by-character transliteration (sed's `y` command overlaps with this) |
| `perl -pe` / `perl -ne` | More powerful regex/scripting alternative for complex transformations |
| `cut` | Extract specific columns/fields by delimiter or byte position |
| `diff` | Often used alongside sed to verify what a transformation actually changed |
| `xargs` | Frequently paired with sed in pipelines for batch operations across many files |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
