# sed — Edge Cases & Gotchas

> sed's terse syntax hides real traps — greedy matching, portability differences,
> and in-place editing mistakes can silently corrupt data if you're not careful.

---

## Table of Contents

- [GNU vs BSD -i Behaves Completely Differently](#gnu-vs-bsd--i-behaves-completely-differently)
- [Greedy Matching Grabs More Than Expected](#greedy-matching-grabs-more-than-expected)
- [Special Characters in the Replacement String](#special-characters-in-the-replacement-string)
- [Special Characters in Variables Used as Patterns](#special-characters-in-variables-used-as-patterns)
- [In-Place Editing Loses the Original on Interruption](#in-place-editing-loses-the-original-on-interruption)
- [sed Doesn't Match Across Lines by Default](#sed-doesnt-match-across-lines-by-default)
- [BRE vs ERE Silently Changes Meaning](#bre-vs-ere-silently-changes-meaning)
- [Delimiter Collision with Path-Like Patterns](#delimiter-collision-with-path-like-patterns)
- [The Last Line Without a Trailing Newline](#the-last-line-without-a-trailing-newline)
- [-n Without Any p Produces No Output At All](#-n-without-any-p-produces-no-output-at-all)
- [Address Ranges That Never Close](#address-ranges-that-never-close)
- [Locale Affecting Character Classes](#locale-affecting-character-classes)
- [Modifying a File While Reading It (Same File In and Out)](#modifying-a-file-while-reading-it-same-file-in-and-out)
- [Unescaped Ampersand in Replacement](#unescaped-ampersand-in-replacement)

---

## GNU vs BSD -i Behaves Completely Differently

### The single most common cross-platform sed bug
```bash
# On Linux (GNU sed) — works fine, no backup created:
sed -i 's/foo/bar/' file.txt

# The EXACT SAME command on macOS (BSD sed):
sed -i 's/foo/bar/' file.txt
# sed: 1: "file.txt": undefined label 'ile.txt'
# ⚠️ BSD sed interprets 's/foo/bar/' as the REQUIRED suffix argument
# for -i, then tries to use "file.txt" as the SCRIPT itself, which
# obviously isn't valid sed syntax, and fails cryptically.

# Cross-platform-safe pattern (works identically on both):
sed -i.bak 's/foo/bar/' file.txt
rm -f file.txt.bak       # clean up the backup if you don't want it

# Or, in a script, detect the platform explicitly:
if [[ "$OSTYPE" == "darwin"* ]]; then
  sed -i '' 's/foo/bar/' file.txt
else
  sed -i 's/foo/bar/' file.txt
fi
```

---

## Greedy Matching Grabs More Than Expected

### `.*` matches as much as possible, not the "obvious" minimal chunk
```bash
echo '<b>bold</b> and <i>italic</i>' | sed 's/<.*>/TAG/'
# TAG
# ⚠️ Probably NOT what was expected — .* is greedy and matched
# EVERYTHING from the first < all the way to the LAST >, collapsing
# both tags and the text between them into a single "TAG".

# Fix: match the SHORTEST possible sequence by avoiding the greedy
# wildcard and being more specific about what's NOT allowed inside:
echo '<b>bold</b> and <i>italic</i>' | sed 's/<[^>]*>/TAG/g'
# TAGboldTAG and TAGitalicTAG
# [^>]* only matches characters that are NOT '>', preventing the
# match from accidentally spanning across multiple tags.
```

---

## Special Characters in the Replacement String

### Unescaped /, &, and \ in the REPLACEMENT break the command
```bash
echo "value" | sed 's/value/a/b/'
# sed: -e expression #1, char 7: unknown option to `s'
# ⚠️ The replacement "a/b" contains an UNESCAPED delimiter character (/),
# which sed interprets as ending the substitution early, leaving
# "b/" as unexpected trailing garbage.

echo "value" | sed 's/value/a\/b/'
# a/b     ✅ escaping the delimiter inside the replacement fixes it

# The & character in a replacement means "the whole match" — using a
# LITERAL & requires escaping it too:
echo "price: 50" | sed 's/[0-9]*/&%/'
# price: 50%    (& expanded to the matched "50")

echo "A & B" | sed 's/&/and/'
# sed: interprets & as "the match" here too if unescaped in the PATTERN
# context is different, but a literal & appearing in a REPLACEMENT
# string always needs escaping if you want the literal character:
echo "50" | sed 's/50/Cost: \&50/'
# Cost: &50    ✅ \& produces a literal ampersand in the output
```

---

## Special Characters in Variables Used as Patterns

### Building a sed command from a shell variable can break unexpectedly
```bash
SEARCH="a.b"
echo "axb" | sed "s/$SEARCH/found/"
# found      ⚠️ the "." in $SEARCH was interpreted as regex "any character",
# so "axb" matched even though the LITERAL string "a.b" wasn't actually present!

# If you need a LITERAL string match (not a regex), escape special
# regex characters in the variable first:
SEARCH="a.b"
ESCAPED=$(printf '%s\n' "$SEARCH" | sed 's/[.[\*^$/]/\\&/g')
echo "axb" | sed "s/$ESCAPED/found/"
# axb        ✅ no match now, because "a.b" is treated LITERALLY

# This is especially important when the variable comes from user input,
# a filename, or any other source that might contain regex metacharacters
# (., *, [, ], ^, $, /) that you don't want sed to interpret specially.
```

---

## In-Place Editing Loses the Original on Interruption

### A power loss or Ctrl+C mid-operation can leave a corrupted or empty file
```bash
sed -i 's/foo/bar/g' important_config.txt
# If the process is killed (system crash, disk full, Ctrl+C at exactly
# the wrong moment) WHILE sed is writing the temporary file that
# eventually replaces important_config.txt, you can be left with:
# - a truncated/incomplete file
# - in rare cases depending on implementation details, an empty file
# - or, if a backup wasn't requested, NO way to recover the original at all

# Always keep a backup when editing anything important in place:
sed -i.bak 's/foo/bar/g' important_config.txt
# Or better yet, rely on version control (git) rather than sed's
# backup mechanism as your actual safety net for anything meaningful:
git diff important_config.txt    # review the change
git checkout important_config.txt   # revert if something went wrong
```

---

## sed Doesn't Match Across Lines by Default

### A pattern spanning a newline simply never matches
```bash
printf "hello\nworld\n" | sed 's/hello\nworld/greeting/'
# hello
# world
# ⚠️ NO substitution happened — by default, sed loads and processes
# ONE LINE at a time, so a pattern containing a literal \n can never
# match anything, since the pattern space never contains more than
# a single line's worth of content at once.

# Fix: explicitly join lines into the pattern space first with N
printf "hello\nworld\n" | sed 'N;s/hello\nworld/greeting/'
# greeting
# N appends the NEXT line to the pattern space (creating a genuine
# embedded newline inside it), so the multi-line pattern can now match.

# For matching across MANY lines, a common pattern is looping N until
# the whole file (or a delimited block) is loaded:
sed ':a;N;$!ba;s/\n/ /g' file.txt
# Joins the ENTIRE file into one line by repeatedly appending (N) and
# looping (:a ... ba) until the last line ($!) is reached.
```

---

## BRE vs ERE Silently Changes Meaning

### The SAME symbol means something different depending on mode
```bash
echo "a+b" | sed 's/a+b/MATCH/'
# MATCH
# In BRE (default), "+" is a LITERAL character, not a quantifier —
# this matched the literal string "a+b" exactly.

echo "aaab" | sed 's/a+b/MATCH/'
# aaab      ⚠️ NO match — "+" is literal in BRE, so this pattern only
# matches the exact three characters "a+b", not "one or more a's".

echo "aaab" | sed -E 's/a+b/MATCH/'
# MATCH     ✅ WITH -E (ERE), "+" now means "one or more of the
# preceding character", correctly matching "aaab"

# This is a very common source of confusion for people coming from
# tools (grep -E, most other languages' regex) where + is ALWAYS a
# quantifier — in plain default sed, it silently is NOT, unless \+ is
# used explicitly, or -E/-r mode is turned on.
```

---

## Delimiter Collision with Path-Like Patterns

### Using / as the delimiter when the pattern ALSO contains slashes gets messy
```bash
sed 's/\/usr\/local\/bin/\/opt\/bin/' file.txt
# Technically WORKS, but painfully hard to read with all those
# escaped slashes cluttering both the pattern and replacement.

# Fix: use a DIFFERENT delimiter character — ANY character can serve
# as the delimiter in sed, not just "/"
sed 's|/usr/local/bin|/opt/bin|' file.txt
# ✅ much more readable, no escaping needed for the slashes themselves

sed 's#/var/www#/srv/web#' file.txt
sed 's,/old/path,/new/path,' file.txt
# Any of these work identically — just pick whatever character doesn't
# appear inside your actual pattern/replacement text.
```

---

## The Last Line Without a Trailing Newline

### Files missing a final newline can produce subtly different output
```bash
printf "line1\nline2" > no_trailing_newline.txt    # note: NO \n after line2
sed 's/line/LINE/' no_trailing_newline.txt
# LINE1
# LINE2    (no newline character after this — matches the input exactly)

# This usually isn't a PROBLEM per se, but can cause subtle issues
# when concatenating sed's output with other content, or when a
# downstream tool assumes every line (including the last) ends with \n:
sed 's/line/LINE/' no_trailing_newline.txt >> another_file.txt
# The next append could end up jammed onto the same line as the
# previous file's last line, with no newline separating them.

# Check whether a file is missing its final newline:
tail -c1 file.txt | wc -l
# 0 = missing trailing newline; 1 = has one
```

---

## -n Without Any p Produces No Output At All

### A common beginner mistake when building up a sed command incrementally
```bash
sed -n 's/foo/bar/' file.txt
# (no output at all!)
# ⚠️ -n suppresses sed's DEFAULT automatic printing of every line —
# without an explicit `p` flag on the substitute command (or a
# separate p command), NOTHING gets printed, even though the
# substitution technically still happened internally on matching lines.

sed -n 's/foo/bar/p' file.txt
# ✅ now prints only the lines where the substitution actually occurred,
# which is usually what people intend when reaching for -n in the
# first place (to filter output down to just the relevant changed lines).
```

---

## Address Ranges That Never Close

### A range whose END pattern never appears consumes the rest of the file
```bash
sed '/START/,/END/d' file.txt
# If "START" appears in the file but "END" is NEVER found afterward
# (typo, wrong pattern, or the END marker simply doesn't exist),
# the range silently extends to the END OF THE FILE — deleting far
# more content than intended, with no warning or error message that
# the closing pattern was never actually matched.

# Always verify both boundary patterns actually exist and match as
# expected before running a range-based deletion, especially with -i:
grep -n "START\|END" file.txt    # confirm both patterns are present
sed -n '/START/,/END/p' file.txt  # PREVIEW what the range would select, first
sed '/START/,/END/d' file.txt     # only THEN actually delete it
```

---

## Locale Affecting Character Classes

### [a-z] doesn't always mean what you expect in every locale
```bash
export LC_ALL=en_US.UTF-8
echo "Hello" | sed 's/[a-z]/X/g'
# HXXXX     (only lowercase a-z matched, as expected in most locales)

export LC_ALL=C
echo "Hello" | sed 's/[a-z]/X/g'
# HXXXX     (same result here, but SOME locales can affect range/collation
# order, occasionally causing [a-z] to include unexpected characters, or
# case-related surprises, particularly with certain non-English locales)

# When strict, predictable, byte-level behavior matters (e.g., in
# a script meant to run identically on any machine regardless of
# locale settings), explicitly force the C locale:
LC_ALL=C sed 's/[a-z]/X/g' file.txt
```

---

## Modifying a File While Reading It (Same File In and Out)

### Redirecting sed's own output back into its input file destroys the data
```bash
sed 's/foo/bar/' file.txt > file.txt
# ⚠️ CATASTROPHIC — the shell TRUNCATES file.txt to empty (as part of
# setting up the > redirect) BEFORE sed even starts reading from it,
# so sed ends up reading from an already-empty file, and file.txt is
# now permanently blank, with the ORIGINAL CONTENT COMPLETELY LOST.

# NEVER redirect sed's output back to the SAME file it's reading from.
# Use sed's own -i flag instead, which handles this safely internally
# (via a temporary file, then an atomic rename):
sed -i 's/foo/bar/' file.txt
# ✅ correct and safe — sed's -i implementation avoids this exact trap
```

---

## Unescaped Ampersand in Replacement

### & means "the whole matched text" — surprising if you wanted a literal &
```bash
echo "Tom and Jerry" | sed 's/and/\&/'
# Tom & Jerry     ✅ correct, using \& for a LITERAL ampersand

echo "Tom and Jerry" | sed 's/Tom/&my/'
# Tommy and Jerry (& here means "Tom", the matched text, so &my → Tommy)

echo "50" | sed 's/[0-9]*/[&]/'
# [50]            (wrapping the ENTIRE match in brackets, a common
# and USEFUL pattern — but easy to trip over if you forgot & has this
# special meaning and just wanted a literal ampersand character)
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
