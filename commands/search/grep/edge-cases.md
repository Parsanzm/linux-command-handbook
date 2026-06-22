# grep — Edge Cases & Gotchas

> grep has more subtle traps than almost any command.
> Most come from regex, locale, and exit code behavior.

---

## Table of Contents

- [Exit Code Confusion](#exit-code-confusion)
- [BRE vs ERE Traps](#bre-vs-ere-traps)
- [Anchor Behavior](#anchor-behavior)
- [Character Class Surprises](#character-class-surprises)
- [Greedy Matching & Dot](#greedy-matching--dot)
- [Locale & Encoding Issues](#locale--encoding-issues)
- [Binary File Behavior](#binary-file-behavior)
- [Filename Traps](#filename-traps)
- [Pipeline & Buffering Issues](#pipeline--buffering-issues)
- [-w and Word Boundary Surprises](#-w-and-word-boundary-surprises)
- [Recursive Search Gotchas](#recursive-search-gotchas)
- [Performance Traps](#performance-traps)
- [GNU vs BSD Differences](#gnu-vs-bsd-differences)
- [Quoting in Shell](#quoting-in-shell)

---

## Exit Code Confusion

### grep returns 1 (failure) when no match — even if that's expected
```bash
grep "pattern" file.txt
echo $?   # 0 = match found, 1 = no match, 2 = error

# In a script with set -e, no match causes script to exit!
set -e
grep "error" clean.log   # exits script if no errors found!

# Fix: use || true
grep "error" clean.log || true

# Fix: explicit check
grep "error" clean.log && echo "found" || echo "not found"

# Fix: use -q and if
if grep -q "error" clean.log; then
  echo "found"
else
  echo "not found"
fi
```

### Exit code 2 is often confused with exit code 1
```bash
grep "pattern" nonexistent_file
echo $?   # 2 = error (file not found)

grep "pattern" file.txt
echo $?   # 1 = no match (not an error per se)

# Distinguish:
grep "pattern" file.txt
case $? in
  0) echo "match" ;;
  1) echo "no match" ;;
  2) echo "error" ;;
esac
```

### -c returns 0 count but exit 1
```bash
grep -c "error" file.txt
# If no matches: prints "0" but exits with code 1
# This surprises people who check exit code after -c

count=$(grep -c "error" file.txt || true)   # prevent set -e from firing
```

---

## BRE vs ERE Traps

### `+`, `?`, `|`, `()` are LITERAL in BRE
```bash
# BRE — these are NOT metacharacters:
grep "a+b" file.txt      # matches literal "a+b" (not "one or more a")
grep "a|b" file.txt      # matches literal "a|b" (not "a or b")
grep "(ab)" file.txt     # matches literal "(ab)" (not a group)

# To use them as metacharacters in BRE, escape with \:
grep "a\+b" file.txt     # one or more 'a' followed by 'b' (GNU extension)
grep "a\|b" file.txt     # 'a' or 'b' (GNU extension)
grep "\(ab\)\+" file.txt # one or more "ab"

# In ERE (grep -E), no escaping needed:
grep -E "a+b" file.txt   # ✅ one or more 'a' followed by 'b'
grep -E "a|b" file.txt   # ✅ 'a' or 'b'
grep -E "(ab)+" file.txt # ✅ one or more "ab"

# The confusion:
grep "a+" file.txt    # matches literal "a+"
grep -E "a+" file.txt # matches "a", "aa", "aaa"...
```

### Escaping in ERE reverses meaning
```bash
# In ERE:
grep -E "(" file.txt    # ❌ error: unmatched parenthesis
grep -E "\(" file.txt   # ✅ literal "("

grep -E "a+" file.txt   # metacharacter: one or more a
grep -E "a\+" file.txt  # literal "a+"

# In BRE:
grep "\(" file.txt      # metacharacter: start of group
grep "(" file.txt       # literal "("
```

---

## Anchor Behavior

### `^` and `$` match start/end of LINE, not string
```bash
# grep is line-based — each line is processed independently
grep "^start" file.txt   # lines starting with "start"
grep "end$" file.txt     # lines ending with "end"

# There's no way to match start/end of file in standard grep
# Use awk or sed for multi-line patterns
```

### `^` inside character class is negation, not anchor
```bash
grep "^abc" file.txt     # lines starting with "abc" (anchor)
grep "[^abc]" file.txt   # any char that is NOT a, b, or c (negation)

# Common mistake:
grep "[^0-9]" file.txt   # NOT "lines starting with a digit range"
                         # It matches lines with any non-digit character
```

### Empty pattern matches every line
```bash
grep "" file.txt         # matches ALL lines (empty pattern = always true)
grep -c "" file.txt      # counts total lines (same as wc -l)

# Useful:
grep -c "" file.txt      # fast line count
wc -l file.txt           # same but different output format
```

---

## Character Class Surprises

### `-` must be first or last inside `[]`
```bash
grep "[a-z]" file.txt   # ✅ range: a through z
grep "[az-]" file.txt   # ✅ literal '-' (last position)
grep "[-az]" file.txt   # ✅ literal '-' (first position)
grep "[a-]" file.txt    # ✅ 'a' or literal '-'
grep "[a-z-]" file.txt  # ✅ a-z range plus literal '-'

grep "[a-z-A]" file.txt # ⚠️ ambiguous — avoid
```

### `]` inside `[]` must be first
```bash
grep "[]abc]" file.txt  # ✅ matches ], a, b, or c
grep "[abc]" file.txt   # ✅ matches a, b, or c
grep "[a]bc]" file.txt  # ⚠️ treats first ] as end of class
```

### POSIX classes are locale-aware
```bash
grep "[a-z]" file.txt       # depends on locale! May include é, ñ, etc.
grep "[[:lower:]]" file.txt # more portable: POSIX class

# In C locale (safe, predictable):
LANG=C grep "[a-z]" file.txt   # strictly ASCII a-z

# In UTF-8 locale:
grep "[a-z]" file.txt   # may match non-ASCII lowercase letters
```

### `.` does NOT match newline
```bash
grep "a.b" file.txt   # matches a, any char EXCEPT newline, b
                      # Won't match multi-line patterns

# grep is line-based — you can't span multiple lines
# For multi-line: use perl -0 or pcregrep -M
```

---

## Greedy Matching & Dot

### `.*` is greedy — matches as much as possible
```bash
echo "foo bar baz" | grep -o "f.*z"
# Output: foo bar baz   (greedy: matches everything to last z)

echo "<tag>content</tag>" | grep -oE "<.*>"
# Output: <tag>content</tag>   (greedy: matches outermost <>)

# Non-greedy requires PCRE:
echo "<tag>content</tag>" | grep -oP "<.*?>"
# Output: <tag>    (non-greedy: stops at first >)
#         </tag>
```

### `.` is greedy and matches spaces, tabs, special chars
```bash
grep "a.b" file.txt   # matches: aXb, a b, a\tb, a1b — anything between a and b
                      # Does NOT match: ab (. requires exactly one char)
grep "a.*b" file.txt  # matches: ab, aXb, aXXXb, a   b — even empty between
```

---

## Locale & Encoding Issues

### Locale affects regex character classes
```bash
# In en_US.UTF-8:
grep "[A-Z]" file.txt   # might match non-ASCII uppercase too
LANG=C grep "[A-Z]" file.txt   # strictly A-Z only

# Safe cross-locale patterns:
grep "[[:upper:]]" file.txt     # POSIX class — locale-aware but explicit
LANG=C grep "[A-Z]" file.txt    # force ASCII interpretation
```

### UTF-8 multi-byte characters
```bash
# grep handles UTF-8 in modern versions
grep "café" file.txt          # ✅ works in UTF-8 locale

# But byte-level operations can surprise:
grep -b "café" file.txt       # byte offset may be > char offset

# Grep with non-UTF-8 files:
LANG=C grep "pattern" latin1_file.txt  # treat as bytes
```

### `\w` behavior differs by locale
```bash
grep -P "\w" file.txt         # in UTF-8: may match Unicode word chars
LANG=C grep -P "\w" file.txt  # strictly [a-zA-Z0-9_]
```

---

## Binary File Behavior

### grep skips binary files by default
```bash
grep "password" /bin/program
# Binary file /bin/program matches
# ← No actual content shown!

grep -a "password" /bin/program   # treat as text, show content
grep -I "password" /bin/program   # treat binary as no match (silent skip)
```

### Binary detection is heuristic
```bash
# grep detects binary by looking for null bytes (\x00) in the first few bytes
# A text file with embedded nulls will be treated as binary
# A binary file with no nulls may be treated as text

# Force text:
grep --text "pattern" file
grep -a "pattern" file
```

### Compressed files look like binary
```bash
grep "pattern" file.gz     # Binary file file.gz matches (or no output)
zgrep "pattern" file.gz    # ✅ correct: decompress then grep
```

---

## Filename Traps

### Files starting with `-`
```bash
grep "pattern" -file.txt    # ❌ treats -file.txt as an option!

grep "pattern" -- -file.txt  # ✅ use -- to end options
grep "pattern" ./-file.txt   # ✅ also works
```

### No files given + no stdin = hangs
```bash
grep "pattern"    # ❌ waits for stdin input forever
                  # Looks like it's hung but it's reading from terminal
# Press Ctrl+C or Ctrl+D to exit
```

### Glob expands to nothing
```bash
grep "pattern" *.nonexistent
# If no files match *.nonexistent:
# bash: no matches found (with nullglob off — default)
# zsh: similar error
# Use: ls *.nonexistent 2>/dev/null | xargs grep "pattern"
```

---

## Pipeline & Buffering Issues

### grep buffers output in pipes (block buffering)
```bash
tail -f app.log | grep "error"
# grep may not show output immediately — it's block-buffered

# Fix: force line buffering
tail -f app.log | grep --line-buffered "error"

# Or use stdbuf:
tail -f app.log | stdbuf -oL grep "error"
```

### SIGPIPE with head
```bash
grep "pattern" huge.log | head -5
# grep may print "Broken pipe" error when head exits after 5 lines

# Suppress:
grep "pattern" huge.log 2>/dev/null | head -5
# Or:
grep "pattern" huge.log | head -5 2>/dev/null
```

### Color codes break downstream processing
```bash
# If ls or other tool aliases grep with --color=always:
grep --color=always "error" log | wc -c   # counts ANSI escape codes too!
grep --color=never "error" log | wc -c    # ✅ clean count

# Strip color codes:
grep --color=always "error" log | sed 's/\x1b\[[0-9;]*m//g'
```

---

## -w and Word Boundary Surprises

### `-w` uses `\b` which is locale-dependent
```bash
grep -w "log" file.txt
# Matches: "log", "the log file"
# No match: "logger", "syslog"

# But what counts as a word character depends on locale
# In UTF-8: accented chars may be word chars
# In C locale: only [a-zA-Z0-9_]
```

### `-w` with special regex chars
```bash
grep -w "error[0-9]" file.txt
# The word boundary applies to the whole pattern, not just the last char
# Matches: "error5" surrounded by non-word chars
```

### `-w` is NOT the same as `\bword\b`
```bash
grep -w "foo" file.txt
# Equivalent to: grep "\bfoo\b" with GNU grep
# But -w is not available in all grep versions
# Use \b in PCRE for portability: grep -P "\bfoo\b"
```

---

## Recursive Search Gotchas

### `-r` does NOT follow symlinks; `-R` does (and can loop)
```bash
grep -r "pattern" .    # safe: won't follow symlinks
grep -R "pattern" .    # follows symlinks: can loop on circular symlinks!

# grep detects some loops but not all
# Prefer -r with explicit symlink handling if needed
```

### Without `--include`, grep searches ALL files
```bash
grep -r "pattern" /src/
# Searches .git, node_modules, *.pyc, *.jpg — everything!

grep -r --include="*.py" "pattern" /src/   # ✅ only Python files
grep -r --exclude-dir=".git" --exclude-dir="node_modules" "pattern" /src/
```

### `--exclude-dir` only works with `-r`/`-R`
```bash
grep --exclude-dir=".git" "pattern" file.txt   # ⚠️ --exclude-dir ignored (no -r)
grep -r --exclude-dir=".git" "pattern" .       # ✅ correct
```

---

## Performance Traps

### `-E` or `-P` is slower than `-F` for fixed strings
```bash
grep -E "literal_string" huge.log    # uses regex engine
grep -F "literal_string" huge.log    # uses Boyer-Moore, 2-10x faster

# Rule: if your pattern has no regex metacharacters, use -F
grep -F "error: connection refused" app.log   # ✅ faster
```

### `.*` at start/end of pattern forces full scan
```bash
grep ".*pattern.*" file.txt    # .* at start = scan entire line for every line
grep "pattern" file.txt        # faster: searches for "pattern" directly
# grep already searches for pattern ANYWHERE in line by default
```

### Anchoring improves performance
```bash
grep "pattern" huge.log        # scans entire line
grep "^pattern" huge.log       # can stop at first char mismatch
grep "pattern$" huge.log       # can scan from end
```

### `-m 1` stops after first match
```bash
grep "error" huge.log          # scans entire file
grep -m 1 "error" huge.log     # stops after first match — much faster
```

### Recursive grep on / is very slow
```bash
grep -r "pattern" /            # traverses entire filesystem!
grep -r "pattern" / --exclude-dir={proc,sys,dev}   # a bit better
# Use find + grep or locate for filesystem-wide search
```

---

## GNU vs BSD Differences

| Feature | GNU grep (Linux) | BSD grep (macOS) |
|---------|-----------------|-----------------|
| `-P` (PCRE) | ✅ Full support | ✅ (newer macOS via libpcre) |
| `--include` | ✅ | ✅ |
| `--exclude-dir` | ✅ | ✅ (newer versions) |
| `-R` (follow symlinks) | ✅ | ✅ |
| `--line-buffered` | ✅ | ✅ |
| `-Z` (null after filename) | ✅ | ✅ |
| `--color` | `auto/always/never` | `auto/always/never` |
| `\w \d \s` in BRE/ERE | GNU extension | ❌ not supported |
| `\b` word boundary | `-E` and `-P` | `-E` and `-P` |
| `-o` with `-P` lookahead | ✅ | ✅ |
| `\K` reset match | ✅ | ❌ |

**Portable patterns (work on both):**
```bash
grep -E "[0-9]+" file.txt        # ✅ both
grep -P "\d+" file.txt           # ✅ both (modern macOS)
grep "[[:digit:]]" file.txt      # ✅ most portable (POSIX)
grep -E "\w+" file.txt           # ⚠️ GNU extension in ERE
grep -P "\w+" file.txt           # ✅ PCRE — works on both
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
