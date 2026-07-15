# wc — The Complete Reference

> **Word Count: count lines, words, characters, and bytes in text**
> Present since Version 1 Unix (1971) — one of the original, foundational Unix utilities.
> Small, simple, and quietly essential in countless scripts and pipelines.

---

## Table of Contents

- [What is wc?](#what-is-wc)
- [Where does wc live?](#where-does-wc-live)
- [How wc works internally](#how-wc-works-internally)
- [Syntax](#syntax)
- [Basic Usage](#basic-usage)
- [Counting Lines, Words, Characters, and Bytes](#counting-lines-words-characters-and-bytes)
- [All Key Options](#all-key-options)
- [wc with Multiple Files](#wc-with-multiple-files)
- [wc in Pipelines](#wc-in-pipelines)
- [Characters vs Bytes — Multibyte/UTF-8 Nuances](#characters-vs-bytes--multibytutf-8-nuances)
- [What Counts as a "Line" and a "Word"](#what-counts-as-a-line-and-a-word)
- [wc vs Other Counting Approaches](#wc-vs-other-counting-approaches)
- [Related Commands](#related-commands)

---

## What is wc?

`wc` stands for **Word Count**. It counts the number of **lines**, **words**, **characters**, and **bytes** in a file or piped input, printing the results as simple numbers. Despite its tiny feature set, `wc` is one of the most frequently used tools in shell scripting and command-line workflows — anywhere you need a quick count of "how many things," `wc` is usually the fastest way to get it.

```bash
wc file.txt
#   42  350 2103 file.txt
# (lines, words, bytes, filename — default output order)

wc -l file.txt
#   42 file.txt
# (just the line count)

echo "hello world" | wc -w
# 2
```

**Why wc is everywhere in shell scripts:** counting the number of matching lines, files, or results is one of the most common things a script needs to do — `grep pattern file | wc -l` (count matches), `ls | wc -l` (count files), `find . -name "*.py" | wc -l` (count Python files) are all extremely common idioms built directly on `wc`.

---

## Where does wc live?

```
/usr/bin/wc
```

```bash
which wc
wc --version
# wc (GNU coreutils) 9.4
```

Part of **GNU coreutils** on Linux, and part of the POSIX standard utilities — present in essentially identical form on every Unix-like system, including macOS/BSD, with only minor output-formatting differences.

---

## How wc works internally

### A simple streaming counter

`wc` reads its input **once, sequentially**, incrementing counters as it goes:
- A **newline character** (`\n`) increments the line counter
- A transition from whitespace to non-whitespace increments the word counter
- Every byte read increments the byte counter
- Depending on the current locale, character counting may differ from byte counting (see [Characters vs Bytes](#characters-vs-bytes--multibyutf-8-nuances) below)

Because it only needs a single pass and minimal memory (just a few counters, not the actual content), `wc` is extremely fast and memory-efficient even on enormous files — it never needs to hold the file's content in memory at all.

```bash
# wc can count lines in a multi-gigabyte file almost instantly,
# because it's just streaming through once, incrementing counters:
wc -l huge_10gb_logfile.txt
```

### Why "lines" means "newline characters," not "visual lines"

```bash
printf "no newline at the end" | wc -l
# 0     ← because wc counts NEWLINE CHARACTERS, and there isn't one here!

printf "no newline at the end\n" | wc -l
# 1     ← now there's exactly one newline character, so the count is 1

# This trips people up constantly: wc -l literally counts '\n'
# occurrences, NOT the number of "lines of text" in a more intuitive,
# visual sense — a file's LAST line, if it lacks a trailing newline,
# won't be counted at all by wc -l.
```

---

## Syntax

```bash
wc [OPTIONS] [FILE...]
command | wc [OPTIONS]
```

```bash
wc file.txt                 # lines, words, bytes, filename
wc -l file.txt               # just line count
wc -w file.txt               # just word count
wc -c file.txt               # just byte count
wc -m file.txt               # just character count (locale-aware)
wc file1.txt file2.txt       # counts for EACH file, plus a combined total
```

---

## Basic Usage

```bash
# Default output: lines, words, bytes, filename
wc report.txt
#   120  843 5234 report.txt

# Count lines in piped output — one of the MOST common wc usages overall
ls | wc -l
grep "ERROR" logfile.txt | wc -l
find . -name "*.py" | wc -l

# Count without a filename argument shown (piped input has no "filename")
cat file.txt | wc -l
#   120
# (no filename printed, since wc doesn't know/have one for piped stdin)
```

---

## Counting Lines, Words, Characters, and Bytes

```bash
# -l : LINES (count of newline characters)
wc -l file.txt

# -w : WORDS (whitespace-separated tokens)
wc -w file.txt

# -c : BYTES (raw byte count, regardless of encoding)
wc -c file.txt

# -m : CHARACTERS (locale-aware — may differ from byte count with
# multibyte encodings like UTF-8, see below)
wc -m file.txt

# -L : LENGTH of the LONGEST LINE (in characters, not bytes)
wc -L file.txt
# Useful for checking if any line in a file exceeds a certain width
# (e.g., enforcing a coding style's max line length)

# Combine multiple flags to see just the specific counts you want,
# in the ORDER you request them
wc -lw file.txt
#   120  843 file.txt    (lines, then words — bytes/chars omitted since not requested)
```

---

## All Key Options

| Option | Long | Description |
|--------|------|--------------|
| `-l` | `--lines` | Count lines (newline characters) |
| `-w` | `--words` | Count words (whitespace-separated tokens) |
| `-c` | `--bytes` | Count bytes |
| `-m` | `--chars` | Count characters (locale-aware, may differ from `-c`) |
| `-L` | `--max-line-length` | Print the length of the longest line |
| `--files0-from=FILE` | | Read the list of input files from FILE, NUL-separated (useful with `find -print0`) |

```bash
wc -l -w -c file.txt           # explicitly request all three (same as default order)
wc -L file.txt                  # longest line length
wc --files0-from=filelist.txt   # count across files listed (NUL-separated) in filelist.txt
```

---

## wc with Multiple Files

```bash
wc file1.txt file2.txt file3.txt
#    10   50  300 file1.txt
#    20   80  450 file2.txt
#     5   20  120 file3.txt
#    35  150  870 total
# ✅ Each file gets its own row, PLUS an automatic "total" summary row
# at the end when multiple files are given.

# Combine with a glob to count across many files at once
wc -l *.log
#   120 access.log
#    45 error.log
#   165 total

# Count total lines across an entire directory tree of a certain file type
find . -name "*.py" -exec wc -l {} + | tail -1
# Shows just the GRAND TOTAL line count across all matched Python files
```

---

## wc in Pipelines

```bash
# Count how many processes match a pattern
ps aux | grep nginx | wc -l

# Count how many files exist in a directory
ls -1 | wc -l

# Count unique values in a column
cut -d',' -f2 data.csv | sort -u | wc -l

# Count how many commits a git repo has
git log --oneline | wc -l

# Count how many lines changed in a diff
git diff | wc -l

# Quick "is this empty?" check for command output
if [ "$(some_command | wc -l)" -eq 0 ]; then
  echo "No output — nothing to process"
fi
```

---

## Characters vs Bytes — Multibyte/UTF-8 Nuances

### -c (bytes) and -m (characters) can differ significantly with non-ASCII text

```bash
echo "héllo" | wc -c
# 7     ← BYTE count: "é" takes 2 BYTES in UTF-8 encoding, plus the
# other 4 ASCII characters (h,l,l,o) at 1 byte each, plus the newline = 7

echo "héllo" | wc -m
# 6     ← CHARACTER count: "héllo" is 5 CHARACTERS (h-é-l-l-o) plus 1
# for the newline = 6, correctly counting "é" as ONE character
# regardless of how many bytes it takes to ENCODE it

# This distinction matters a lot for text in languages with extensive
# non-ASCII characters (Persian, Arabic, Chinese, Cyrillic, emoji, etc.)
echo "سلام" | wc -c
# 9     ← 4 Persian characters, each 2 bytes in UTF-8, plus newline = 9 bytes
echo "سلام" | wc -m
# 5     ← 4 characters + newline = 5, correctly counting actual characters
```

### Locale affects whether -m behaves correctly at all

```bash
# If the locale is set to something that doesn't understand UTF-8
# (like the plain "C" locale), -m may fall back to essentially
# byte-counting behavior instead of true character-counting:
LC_ALL=C echo "héllo" | wc -m
# 7     ⚠️ In the "C" locale, -m may behave the SAME as -c, since the
# C locale has no concept of multibyte UTF-8 sequences at all —
# always ensure a UTF-8-aware locale (like en_US.UTF-8) is active if
# accurate character counting for non-ASCII text matters.
```

---

## What Counts as a "Line" and a "Word"

### Lines: strictly newline-delimited

```bash
printf "line1\nline2\nline3" | wc -l
# 2     ⚠️ NOT 3! The last "line3" has NO trailing newline character,
# so wc -l (which counts '\n' occurrences) only counts 2, even though
# a human looking at the text would say there are clearly 3 lines.

printf "line1\nline2\nline3\n" | wc -l
# 3     ✅ now correct, because EVERY line (including the last) ends
# with a newline character
```

### Words: whitespace-separated tokens, nothing smarter

```bash
echo "hello,world foo-bar" | wc -w
# 2     ← wc's definition of "word" is simply "a maximal sequence of
# non-whitespace characters" — punctuation like commas and hyphens
# do NOT split a word further; "hello,world" counts as ONE word, and
# "foo-bar" also counts as ONE word, since neither contains actual
# whitespace internally.

echo "one   two    three" | wc -w
# 3     ← MULTIPLE consecutive spaces between words are treated as a
# single separator, not counted as extra "empty" words
```

---

## wc vs Other Counting Approaches

| Task | wc-based approach | Alternative |
|------|----------------------|--------------|
| Count lines in a file | `wc -l file.txt` | `awk 'END {print NR}' file.txt` |
| Count matching lines | `grep pattern file \| wc -l` | `grep -c pattern file` (built-in count flag, often faster) |
| Count words | `wc -w file.txt` | `awk '{n+=NF} END {print n}' file.txt` |
| Count files in a directory | `ls \| wc -l` | `find . -maxdepth 1 -type f \| wc -l` (more robust with unusual filenames) |
| Count unique values | `sort -u file.txt \| wc -l` | `awk '!seen[$0]++' file.txt \| wc -l` |

```bash
# grep -c is often preferable to grep | wc -l for the specific case of
# "count matching lines," since it avoids spawning a SEPARATE wc
# process entirely and grep computes the count internally:
grep -c "ERROR" logfile.txt
# 42
# vs the functionally equivalent but slightly less efficient:
grep "ERROR" logfile.txt | wc -l
# 42
```

---

## Related Commands

| Command | Relation |
|---------|----------|
| `grep -c` | Counts MATCHING lines directly, often preferable to `grep \| wc -l` |
| `awk` | Can replicate and extend wc's counting logic with full programmability |
| `du` | Counts disk USAGE (bytes on disk), a different kind of "counting" than wc's text-based counts |
| `find` | Frequently piped into `wc -l` to count matching files |
| `nl` | Numbers lines of a file (complementary to wc, which just COUNTS them) |
| `cat -A` | Reveals hidden characters (like missing trailing newlines) that explain wc -l surprises |
| `stat` | Shows a single file's exact byte size, an alternative to `wc -c` for ONE file |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
