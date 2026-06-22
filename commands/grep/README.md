# grep — The Complete Reference

> **Search for patterns in files using regular expressions**
> The most used text-search tool in Unix. Mastering grep means mastering regex —
> and regex appears everywhere: editors, languages, databases, cloud tools.

---

## Table of Contents

- [What is grep?](#what-is-grep)
- [Where does grep live?](#where-does-grep-live)
- [How grep works internally](#how-grep-works-internally)
- [Syntax](#syntax)
- [All Options](#all-options)
- [Regex Engines: BRE vs ERE vs PCRE](#regex-engines-bre-vs-ere-vs-pcre)
- [Regular Expression Reference](#regular-expression-reference)
- [grep Family: egrep, fgrep, rgrep, zgrep](#grep-family)
- [grep vs ripgrep vs ag vs ack](#grep-vs-ripgrep-vs-ag-vs-ack)
- [Related Commands](#related-commands)

---

## What is grep?

`grep` reads input line by line and prints lines that match a given pattern. The pattern can be a fixed string or a regular expression.

The name stands for **g/re/p** — a command from the `ed` line editor meaning "**g**lobally search for a **r**egular **e**xpression and **p**rint matching lines." Ken Thompson ported it to Unix as a standalone tool in 1973.

**Three core uses:**
1. Search file contents for a pattern
2. Filter lines from a pipeline
3. Find files that contain (or don't contain) a pattern

---

## Where does grep live?

```
/bin/grep           ← most Linux systems
/usr/bin/grep       ← some distros and macOS
```

```bash
which grep
type grep
grep --version      # GNU grep version (Linux)
```

On Linux, `grep` is **GNU grep** — the fastest and most feature-rich version, supporting BRE, ERE, and PCRE. On macOS it's BSD grep (with PCRE via `-P` on newer versions).

```bash
grep --version
# grep (GNU grep) 3.8
```

---

## How grep works internally

```
open file → read line → apply regex engine → if match → write line to stdout
```

1. Reads input line by line (delimiter: `\n` by default)
2. Applies the regex/pattern against each line
3. If the pattern matches → prints the line (or acts per flags)
4. Moves to next line

**Regex engine selection:**
- `-G` (default): BRE engine — POSIX Basic Regular Expressions
- `-E`: ERE engine — POSIX Extended Regular Expressions
- `-F`: fixed-string engine — no regex, raw string comparison (fastest)
- `-P`: PCRE engine — Perl-Compatible Regular Expressions (most powerful)

**Performance internals:**
GNU grep uses the **Boyer-Moore** algorithm for fixed strings and **NFA/DFA** for regex — among the fastest implementations available. For very large files, `-F` (fixed string) is significantly faster than regex.

**Buffering:**
- When output goes to a terminal: **line-buffered** (each line appears immediately)
- When output goes to a pipe: **block-buffered** (written in chunks)
- Use `--line-buffered` to force line buffering in pipelines

**Exit codes:**
| Code | Meaning |
|------|---------|
| `0` | Match found |
| `1` | No match found |
| `2` | Error (file not found, invalid regex...) |

Exit code matters in scripts: `if grep -q "pattern" file; then ...`

---

## Syntax

```
grep [OPTIONS] PATTERN [FILE...]
grep [OPTIONS] -e PATTERN... [FILE...]
grep [OPTIONS] -f PATTERNFILE... [FILE...]
```

- With no FILE → reads stdin
- Multiple `-e` patterns → OR logic (matches any)
- `-f` reads patterns from a file (one per line)

---

## All Options

### Matching & Pattern

| Option | Description |
|--------|-------------|
| `-e pattern` | Specify pattern (allows multiple patterns and patterns starting with `-`) |
| `-f file` | Read patterns from file (one per line) |
| `-i` | Case-insensitive matching |
| `-v` | Invert match — print lines that do NOT match |
| `-w` | Match whole words only (surrounded by word boundaries) |
| `-x` | Match whole line only (entire line must match) |
| `-F` | Treat pattern as fixed string, not regex (fastest) |
| `-E` | Use Extended Regular Expressions (ERE) |
| `-G` | Use Basic Regular Expressions (BRE) — default |
| `-P` | Use Perl-Compatible Regular Expressions (PCRE) |

### Output Control

| Option | Description |
|--------|-------------|
| `-c` | Print count of matching lines only |
| `-l` | Print only filenames that contain a match |
| `-L` | Print only filenames that do NOT contain a match |
| `-o` | Print only the matched part of each line (not the whole line) |
| `-q` | Quiet — no output, just exit code (0=match, 1=no match) |
| `-s` | Suppress error messages about nonexistent/unreadable files |
| `--color=auto` | Highlight matching text (default in most aliases) |
| `--color=always` | Always highlight (even when piping) |
| `--color=never` | Disable color |

### Line Numbers & Context

| Option | Description |
|--------|-------------|
| `-n` | Prefix each output line with line number |
| `-b` | Print byte offset of each match |
| `-A n` | Print n lines **After** each match (context) |
| `-B n` | Print n lines **Before** each match (context) |
| `-C n` | Print n lines before **and** after each match (context) |
| `--group-separator=SEP` | Separator between context groups (default: `--`) |

### File & Directory

| Option | Description |
|--------|-------------|
| `-r` | Recursive — search directories recursively |
| `-R` | Like `-r` but follows symlinks |
| `-l` | List filenames with matches only |
| `-L` | List filenames without matches only |
| `--include=GLOB` | Only search files matching glob (use with `-r`) |
| `--exclude=GLOB` | Skip files matching glob (use with `-r`) |
| `--exclude-dir=GLOB` | Skip directories matching glob |
| `-d action` | How to handle directories: `read`/`recurse`/`skip` |

### Multiplexing

| Option | Description |
|--------|-------------|
| `-h` | Suppress filename prefix (when searching multiple files) |
| `-H` | Always print filename prefix (even for single file) |
| `-T` | Align output with tabs when `-n` is used |
| `-Z` | Print null byte after filename (for xargs -0) |

### Other

| Option | Description |
|--------|-------------|
| `-m n` | Stop after n matching lines |
| `-a` | Process binary files as if they were text |
| `-I` | Ignore binary files (treat as having no matches) |
| `--binary-files=TYPE` | `binary`, `text`, `without-match` |
| `-U` | Treat file as binary (don't strip CR on Windows) |
| `--line-buffered` | Force line buffering of output |
| `--null` | Same as `-Z` |
| `-z` | Input is NUL-delimited instead of newline-delimited |

---

## Regex Engines: BRE vs ERE vs PCRE

This is the most important conceptual section in grep.

### BRE — Basic Regular Expressions (`grep`, `grep -G`)

Metacharacters must be **escaped** to be special:

| Pattern | Meaning |
|---------|---------|
| `.` | Any character |
| `*` | Zero or more of preceding |
| `^` | Start of line |
| `$` | End of line |
| `[abc]` | Character class |
| `[^abc]` | Negated class |
| `\(` `\)` | Grouping (escaped!) |
| `\|` | Alternation (escaped!) — GNU extension |
| `\+` | One or more (escaped!) — GNU extension |
| `\?` | Zero or one (escaped!) — GNU extension |
| `\{n,m\}` | Quantifier (escaped!) |
| `\1` | Backreference |

### ERE — Extended Regular Expressions (`grep -E`, `egrep`)

Metacharacters work **without escaping**:

| Pattern | Meaning |
|---------|---------|
| `.` | Any character |
| `*` | Zero or more |
| `+` | One or more |
| `?` | Zero or one |
| `^` | Start of line |
| `$` | End of line |
| `[abc]` | Character class |
| `[^abc]` | Negated class |
| `(` `)` | Grouping |
| `\|` or `|` | Alternation |
| `{n,m}` | Quantifier |
| `\1` | Backreference (in GNU ERE) |

### PCRE — Perl-Compatible Regular Expressions (`grep -P`)

Everything in ERE plus:

| Pattern | Meaning |
|---------|---------|
| `\d` | Digit (= `[0-9]`) |
| `\D` | Non-digit |
| `\w` | Word character (= `[a-zA-Z0-9_]`) |
| `\W` | Non-word character |
| `\s` | Whitespace |
| `\S` | Non-whitespace |
| `\b` | Word boundary |
| `\B` | Non-word boundary |
| `(?:...)` | Non-capturing group |
| `(?=...)` | Lookahead |
| `(?!...)` | Negative lookahead |
| `(?<=...)` | Lookbehind |
| `(?<!...)` | Negative lookbehind |
| `\n` `\t` | Newline, tab |
| `.*?` | Non-greedy match |
| `\K` | Reset match start (GNU PCRE) |

### Quick Comparison

```bash
# Match one or more digits:
grep '[0-9][0-9]*' file.txt    # BRE
grep -E '[0-9]+' file.txt      # ERE
grep -P '\d+' file.txt         # PCRE

# Match email-like pattern:
grep -E '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' file.txt

# Lookahead (PCRE only):
grep -P 'foo(?=bar)' file.txt  # "foo" followed by "bar", prints "foo"
```

---

## Regular Expression Reference

### Anchors
```
^       start of line
$       end of line
\b      word boundary (PCRE / GNU ERE)
\B      non-word boundary
```

### Character Classes
```
.       any character except newline
[abc]   a, b, or c
[a-z]   any lowercase letter
[^abc]  any character except a, b, c
[[:alpha:]]   letters (POSIX)
[[:digit:]]   digits (POSIX)
[[:alnum:]]   letters and digits (POSIX)
[[:space:]]   whitespace (POSIX)
[[:upper:]]   uppercase letters (POSIX)
[[:lower:]]   lowercase letters (POSIX)
[[:punct:]]   punctuation (POSIX)
```

### Quantifiers
```
*       zero or more
+       one or more (ERE/PCRE; \+ in BRE)
?       zero or one (ERE/PCRE; \? in BRE)
{n}     exactly n times
{n,}    n or more times
{n,m}   between n and m times
*?      non-greedy (PCRE only)
+?      non-greedy (PCRE only)
```

### Groups & Alternation
```
(abc)       group (ERE/PCRE)
\(abc\)     group (BRE)
a|b         a or b (ERE/PCRE)
\|          a or b (BRE GNU extension)
\1          backreference to group 1
(?:abc)     non-capturing group (PCRE)
```

### POSIX Character Classes (portable across locales)
```
[[:alpha:]]   [a-zA-Z]
[[:digit:]]   [0-9]
[[:alnum:]]   [a-zA-Z0-9]
[[:space:]]   [ \t\n\r\f\v]
[[:blank:]]   [ \t]
[[:upper:]]   [A-Z]
[[:lower:]]   [a-z]
[[:punct:]]   !"#$%&'()*+,-./:;<=>?@[\]^_`{|}~
[[:graph:]]   non-space printable chars
[[:print:]]   printable chars including space
[[:cntrl:]]   control characters
[[:xdigit:]]  [0-9a-fA-F]
```

---

## grep Family

### egrep = grep -E
```bash
egrep "pattern+" file.txt
grep -E "pattern+" file.txt   # equivalent
# egrep is deprecated — use grep -E
```

### fgrep = grep -F
```bash
fgrep "literal.string" file.txt
grep -F "literal.string" file.txt   # equivalent
# -F treats pattern as literal — no regex interpretation
# Much faster for fixed string searches
# fgrep is deprecated — use grep -F
```

### rgrep = grep -r
```bash
rgrep "pattern" /path/
grep -r "pattern" /path/   # equivalent
# rgrep is deprecated — use grep -r
```

### zgrep — search compressed files
```bash
zgrep "pattern" file.gz        # gzip
zgrep "ERROR" /var/log/syslog.1.gz

# Variants for other formats:
bzgrep "pattern" file.bz2      # bzip2
xzgrep "pattern" file.xz       # xz
zstdgrep "pattern" file.zst    # zstandard (if installed)
```

### grep -r vs grep -R
```bash
grep -r "pattern" .    # recursive, does NOT follow symlinks
grep -R "pattern" .    # recursive, DOES follow symlinks (can loop!)
```

---

## grep vs ripgrep vs ag vs ack

| Feature | `grep` | `ripgrep (rg)` | `ag` (silver searcher) | `ack` |
|---------|--------|----------------|------------------------|-------|
| Speed | Fast | Fastest | Very fast | Fast |
| Respects .gitignore | No | Yes | Yes | Yes |
| Default recursive | No (`-r`) | Yes | Yes | Yes |
| PCRE | `-P` flag | `-P` flag | Yes | Yes |
| Unicode | Partial | Full | Full | Full |
| Binary files | Skips | Skips | Skips | Skips |
| Install needed | No | Yes | Yes | Yes |
| Syntax | Standard | Similar to grep | Similar to grep | Similar |

```bash
# ripgrep examples
rg "pattern"               # recursive by default
rg -i "pattern"            # case-insensitive
rg -t py "pattern"         # only Python files
rg -g "*.txt" "pattern"    # only txt files
rg -l "pattern"            # filenames only
rg --no-ignore "pattern"   # ignore .gitignore
rg -A 3 -B 3 "pattern"    # context lines
```

---

## Related Commands

| Command | Relation |
|---------|---------|
| `sed` | Stream editor — transform text (uses regex) |
| `awk` | Pattern scanning and processing (uses regex) |
| `find` | Find files — combine with grep for content search |
| `sort` | Often used after grep to organize output |
| `wc` | Count lines matching: `grep -c` or `grep \| wc -l` |
| `less` | View grep output with paging: `grep ... \| less` |
| `rg` | ripgrep — faster modern replacement |
| `ag` | Silver Searcher — fast code-aware search |
| `diff` | Compare files — uses similar line-based model |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
