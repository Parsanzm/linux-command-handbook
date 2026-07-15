# diff — The Complete Reference

> **Show the differences between two files, line by line**
> Created by Douglas McIlroy at Bell Labs in 1974, based on an elegant longest-common-subsequence algorithm.
> The foundation underneath version control, patching, and code review as we know them today.

---

## Table of Contents

- [What is diff?](#what-is-diff)
- [Where does diff live?](#where-does-diff-live)
- [How diff works internally](#how-diff-works-internally)
- [Syntax](#syntax)
- [Output Formats](#output-formats)
- [Normal Format (Default)](#normal-format-default)
- [Unified Format (-u)](#unified-format--u)
- [Context Format (-c)](#context-format--c)
- [Side-by-Side Format (-y)](#side-by-side-format--y)
- [All Key Options](#all-key-options)
- [Comparing Directories (-r)](#comparing-directories--r)
- [diff and Exit Status](#diff-and-exit-status)
- [Creating and Applying Patches](#creating-and-applying-patches)
- [diff vs Other Comparison Tools](#diff-vs-other-comparison-tools)
- [Related Commands](#related-commands)

---

## What is diff?

`diff` compares two files (or two directories) line by line and reports the differences between them — which lines were added, removed, or changed. It's the tool underlying `git diff`, code review interfaces, patch files, and most "compare versions" functionality across the entire software development world.

```bash
diff file1.txt file2.txt
# 2c2
# < original line
# ---
# > modified line
```

**Why diff matters far beyond its simple description:** the algorithm diff uses (finding the **longest common subsequence** between two sequences) is a foundational piece of computer science, applied everywhere from version control systems to DNA sequence alignment. Nearly every "what changed?" feature in modern software — git, code review tools, document version history — ultimately traces back to diff's core algorithm and output conventions.

---

## Where does diff live?

```
/usr/bin/diff
```

```bash
which diff
diff --version
# diff (GNU diffutils) 3.10
```

Part of **GNU diffutils** on Linux, alongside related tools `diff3`, `cmp`, and `sdiff`. Present in some form on virtually every Unix-like system, though output formatting details can vary slightly between GNU diff and BSD/macOS diff.

---

## How diff works internally

### The longest common subsequence (LCS) algorithm

At its core, `diff` finds the **longest common subsequence** of lines between the two files — the largest set of lines that appear in both files, in the same relative order (though not necessarily contiguously). Everything else — lines in one file but not the other — is reported as an addition or deletion relative to that common subsequence.

```bash
# Conceptually: diff finds the MINIMAL set of additions/deletions
# needed to transform file1 into file2, by identifying the LARGEST
# shared subsequence of unchanged lines between them
```

This is why diff's output tends to be **surprisingly good at localizing changes** — even if you insert a single line in the middle of a 10,000-line file, diff correctly reports just that one addition, rather than treating everything after the insertion point as "changed," because the algorithm is specifically designed to find the minimal edit distance.

### Myers diff algorithm

Modern implementations (including GNU diff) typically use an efficient variant of this idea known as the **Myers diff algorithm**, which finds a shortest edit script (SES) between two sequences in roughly O(ND) time, where N is the total length of both sequences and D is the size of the actual difference — meaning diff is fast even on huge files that differ only slightly.

---

## Syntax

```bash
diff [OPTIONS] FILE1 FILE2
diff [OPTIONS] DIR1 DIR2
```

```bash
diff old.txt new.txt              # basic comparison, "normal" format
diff -u old.txt new.txt           # unified format (most common in practice)
diff -c old.txt new.txt           # context format
diff -y old.txt new.txt           # side-by-side format
diff -r dir1/ dir2/               # recursively compare two directories
```

---

## Output Formats

diff supports several output formats, each suited to different purposes:

| Format | Flag | Best for |
|--------|------|----------|
| Normal (default) | (none) | Quick, terse machine-readable summary |
| Unified | `-u` | Human review, git/patch standard, most widely used today |
| Context | `-c` | More verbose, shows surrounding lines for context (older convention) |
| Side-by-side | `-y` | Visual comparison in a wide terminal, two columns |

---

## Normal Format (Default)

```bash
diff old.txt new.txt
```

```
2c2
< This is the original line.
---
> This is the modified line.
5d4
< This line was deleted.
6a6
> This line was added.
```

**Reading normal format:**
- `2c2` — line 2 in file1 was **c**hanged to line 2 in file2
- `5d4` — line 5 in file1 was **d**eleted (file2's corresponding position is after line 4)
- `6a6` — a line was **a**dded after line 6 (in file1's numbering), becoming line 6 in file2
- `<` marks lines from the FIRST file
- `>` marks lines from the SECOND file
- `---` separates the "before" and "after" for a change block

---

## Unified Format (-u)

```bash
diff -u old.txt new.txt
```

```
--- old.txt    2024-01-15 10:00:00
+++ new.txt    2024-01-15 10:05:00
@@ -1,6 +1,6 @@
 unchanged line
-This is the original line.
+This is the modified line.
 unchanged line
 unchanged line
-This line was deleted.
+This line was added.
 unchanged line
```

**Reading unified format:**
- `---` / `+++` — headers identifying the two files being compared
- `@@ -1,6 +1,6 @@` — the **hunk header**: "starting at line 1, 6 lines shown from file1; starting at line 1, 6 lines shown from file2"
- Lines starting with `-` — present in file1, removed in file2
- Lines starting with `+` — present in file2, added compared to file1
- Lines with no prefix (just a leading space) — unchanged, shown for **context**

```bash
# Control how much surrounding context is shown (default is 3 lines)
diff -u3 old.txt new.txt     # 3 lines of context (default)
diff -u0 old.txt new.txt     # NO context, just the changed lines themselves
diff -u10 old.txt new.txt    # 10 lines of context around each change
```

**Why unified format dominates modern tooling:** it's compact (shows only a window of context around each change, not the entire surrounding block), and it's the format `git diff`, GitHub/GitLab pull requests, and the `patch` command all standardize on.

---

## Context Format (-c)

```bash
diff -c old.txt new.txt
```

```
*** old.txt    2024-01-15 10:00:00
--- new.txt    2024-01-15 10:05:00
***************
*** 1,6 ****
  unchanged line
! This is the original line.
  unchanged line
  unchanged line
! This line was deleted.
  unchanged line
--- 1,6 ----
  unchanged line
! This is the modified line.
  unchanged line
  unchanged line
! This line was added.
  unchanged line
```

Context format shows the "before" and "after" blocks **separately** (rather than interleaved like unified format), each with `!` marking changed lines — more verbose than unified format, and largely superseded by it in modern usage, though still occasionally seen in older codebases and some formal patch-review workflows.

---

## Side-by-Side Format (-y)

```bash
diff -y old.txt new.txt
```

```
unchanged line                    unchanged line
This is the original line.      | This is the modified line.
unchanged line                    unchanged line
unchanged line                    unchanged line
This line was deleted.          <
                                 > This line was added.
unchanged line                    unchanged line
```

- `|` — the line differs between both files (shown side by side)
- `<` — the line exists only in the LEFT file
- `>` — the line exists only in the RIGHT file

```bash
# Control the output WIDTH (important since this format needs space
# for BOTH files' content plus the separator column)
diff -y --width=200 old.txt new.txt

# Suppress lines that are IDENTICAL in both files, showing only differences
diff -y --suppress-common-lines old.txt new.txt
```

---

## All Key Options

| Option | Long | Description |
|--------|------|--------------|
| `-u [N]` | `--unified[=N]` | Unified format, with N lines of context (default 3) |
| `-c [N]` | `--context[=N]` | Context format, with N lines of context |
| `-y` | `--side-by-side` | Side-by-side two-column format |
| `-r` | `--recursive` | Recursively compare directories |
| `-i` | `--ignore-case` | Ignore case differences when comparing |
| `-w` | `--ignore-all-space` | Ignore ALL whitespace differences |
| `-b` | `--ignore-space-change` | Ignore differences in AMOUNT of whitespace (but not its presence entirely) |
| `-B` | `--ignore-blank-lines` | Ignore changes that consist ONLY of blank lines |
| `-q` | `--brief` | Just report WHETHER files differ, not the actual differences |
| `-s` | `--report-identical-files` | Explicitly report when compared files are IDENTICAL |
| `-N` | `--new-file` | Treat a MISSING file as empty (useful when comparing directories with added/removed files) |
| `--suppress-common-lines` | | With `-y`, hide lines that are identical in both files |
| `--color[=WHEN]` | | Colorize output (added/removed lines) |
| `-e` | `--ed` | Output in `ed` script format, directly usable as editing commands |

```bash
diff -q file1.txt file2.txt          # just "Files differ" or nothing (if identical)
diff -w file1.txt file2.txt          # ignore whitespace-only differences entirely
diff -i file1.txt file2.txt          # case-insensitive comparison
diff --color=always old.txt new.txt | less -R   # colorized diff in a pager
```

---

## Comparing Directories (-r)

```bash
# Recursively compare two entire directory trees
diff -r dir1/ dir2/
# Only in dir1/: extra_file.txt
# Only in dir2/: new_file.txt
# diff -r dir1/subfolder dir2/subfolder
# Files dir1/config.txt and dir2/config.txt differ

# Compare directories, treating a MISSING file (present in one dir
# but not the other) as if it were EMPTY, generating a full diff for
# it instead of just "Only in X: filename"
diff -rN dir1/ dir2/

# Combine with -q for a QUICK summary of which files differ, without
# showing the full line-by-line differences for each
diff -rq dir1/ dir2/
# Files dir1/app.py and dir2/app.py differ
# Only in dir2/: new_module.py

# Exclude certain files/patterns from the comparison
diff -r --exclude="*.pyc" --exclude="__pycache__" dir1/ dir2/
```

---

## diff and Exit Status

```bash
diff file1.txt file2.txt
echo $?
# 0   ← files are IDENTICAL
# 1   ← files DIFFER
# 2   ← an ERROR occurred (e.g., a file doesn't exist)

# This makes diff directly useful in CONDITIONAL scripts:
if diff -q config.txt config.txt.bak > /dev/null; then
  echo "No changes since backup"
else
  echo "Config has been modified"
fi

# Common pattern: verify a generated file matches an expected reference
if ! diff -q generated_output.txt expected_output.txt > /dev/null; then
  echo "Test FAILED: output doesn't match expected result"
  diff -u expected_output.txt generated_output.txt
  exit 1
fi
```

---

## Creating and Applying Patches

```bash
# Generate a patch file capturing the difference between two versions
diff -u original.txt modified.txt > changes.patch

# Apply that patch to the ORIGINAL file elsewhere, reproducing the
# same changes on a different copy
patch original.txt < changes.patch
# original.txt now matches what "modified.txt" contained

# Generate a patch comparing two directory trees (common for sharing
# a set of code changes without a full version control diff)
diff -ruN original_project/ modified_project/ > project_changes.patch

# Apply a directory-wide patch
cd original_project/
patch -p1 < ../project_changes.patch

# Preview what a patch WOULD do without actually applying it
patch --dry-run < changes.patch
```

---

## diff vs Other Comparison Tools

| Tool | Best for | Key difference from diff |
|------|----------|------------------------------|
| `diff` | General-purpose line-by-line file comparison | The baseline; produces the classic normal/unified/context formats |
| `cmp` | Byte-by-byte comparison, especially BINARY files | Reports the FIRST differing byte's position, not line-by-line text differences |
| `comm` | Comparing two ALREADY-SORTED files' unique/common lines | Requires sorted input; shows set-like unique/common lines, not an edit script |
| `git diff` | Comparing versions within a git repository | Built on the SAME underlying diff algorithm, with git-specific conveniences (commit ranges, file rename detection) |
| `vimdiff` / `meld` / `kdiff3` | Visual, interactive diffing and merging | GUI/TUI wrappers providing visual highlighting and INTERACTIVE merge capability, often using diff or a similar algorithm internally |

```bash
diff file1.txt file2.txt        # line-based text comparison
cmp file1.bin file2.bin          # byte-based comparison, good for BINARY files
comm sorted1.txt sorted2.txt     # set operations on TWO SORTED files
```

---

## Related Commands

| Command | Relation |
|---------|----------|
| `patch` | Applies a diff's output to reproduce the changes it describes |
| `cmp` | Byte-level comparison, better suited to binary files than diff |
| `comm` | Compares two SORTED files' unique/common lines (a different kind of comparison) |
| `diff3` | Three-way diff, useful for comparing a common ancestor against two diverged versions (merge conflict detection) |
| `sdiff` | Interactive side-by-side diff with merge capability |
| `git diff` | Version-control-aware diff, built on the same underlying concepts |
| `vimdiff` | Visual, interactive diff/merge inside vim |
| `md5sum` / `sha256sum` | Quick "are these files identical?" check without showing WHERE they differ |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
