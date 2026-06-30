# awk — Edge Cases & Gotchas

> awk's flexibility comes with subtle type coercion rules, field-rebuilding
> quirks, and portability traps that catch even experienced users.

---

## Table of Contents

- [String vs Numeric Comparison Traps](#string-vs-numeric-comparison-traps)
- [Field Reassignment Rebuilds $0](#field-reassignment-rebuilds-0)
- [NF Manipulation Surprises](#nf-manipulation-surprises)
- [FS Regex vs Literal Confusion](#fs-regex-vs-literal-confusion)
- [Uninitialized Variables](#uninitialized-variables)
- [print vs printf Pitfalls](#print-vs-printf-pitfalls)
- [Quoting & Shell Escaping](#quoting--shell-escaping)
- [getline Side Effects](#getline-side-effects)
- [Array Key Coercion](#array-key-coercion)
- [Locale & Sorting Surprises](#locale--sorting-surprises)
- [mawk vs gawk Feature Gaps](#mawk-vs-gawk-feature-gaps)
- [Large Number Precision](#large-number-precision)
- [Regex Greediness & Anchoring](#regex-greediness--anchoring)
- [Performance Traps](#performance-traps)

---

## String vs Numeric Comparison Traps

### awk guesses the comparison type — and guesses wrong
```bash
# String comparison when BOTH operands look like strings:
echo "abc" | awk '{ print ($1 == "abc") }'   # 1 (true) — fine

# Numeric comparison when fields come from input (awk treats as "numeric strings"):
echo "10 9" | awk '{ print ($1 > $2) }'      # 1 (true) — numeric: 10 > 9 ✅

# But hardcoded strings in the program are ALWAYS compared as strings:
awk 'BEGIN { print ("10" > "9") }'           # 0 (false)! String compare: "1" < "9"

# The rule: a field/input value that LOOKS like a number is a "numeric string"
# and compared numerically. A quoted literal in the program is always a string.
```

### Leading/trailing whitespace breaks numeric detection
```bash
echo " 10" | awk '{ print ($1 == 10) }'      # 1 — awk trims and converts
echo "10abc" | awk '{ print ($1 == 10) }'    # 0 — not a pure number, string compare
echo "1e2" | awk '{ print ($1 == 100) }'     # 1 — scientific notation recognized
echo "0x10" | awk '{ print ($1 == 16) }'     # 0 — awk doesn't parse hex by default!
echo "0x10" | awk '{ print ($1+0) }'         # 0 — hex string evaluates to 0 in math context
```

### Force a specific comparison type
```bash
# Force numeric (add zero):
awk '{ print ($1+0 > $2+0) }' file.txt

# Force string (concatenate empty string):
awk '{ print ($1"" > $2"") }' file.txt
```

---

## Field Reassignment Rebuilds $0

### Assigning to any field rebuilds $0 using OFS
```bash
echo "a b c" | awk '{ $2 = "X"; print }'
# Output: a X c   ← rebuilt using OFS (default space)

echo "a:b:c" | awk -F: '{ $2 = "X"; print }'
# Output: a X c   ← ⚠️ OFS is SPACE by default, even though FS was colon!
# The colons are LOST because OFS wasn't changed

# Fix: set OFS to match FS if you want to preserve the delimiter
echo "a:b:c" | awk -F: 'BEGIN{OFS=":"} { $2 = "X"; print }'
# Output: a:X:c   ✅
```

### Forcing a rebuild without changing anything
```bash
echo "a   b    c" | awk '{ print }'         # a   b    c  (unchanged, multiple spaces preserved)
echo "a   b    c" | awk '{ $1=$1; print }'  # a b c       (rebuilt — squeezed to single OFS)
# $1=$1 is a common trick to "normalize" whitespace
```

### NF++ or $(NF+1) adds a field
```bash
echo "a b c" | awk '{ $(NF+1) = "d"; print }'
# Output: a b c d

echo "a b c" | awk '{ $5 = "e"; print }'
# Output: a b c  e     ← fields 4 and 5 created, field 4 is empty, $0 rebuilt
# NF is now 5, with $4 = "" (empty string)
```

---

## NF Manipulation Surprises

### Decreasing NF truncates fields
```bash
echo "a b c d e" | awk '{ NF=3; print }'
# Output: a b c     ← fields 4 and 5 are GONE, $0 is rebuilt without them
```

### Increasing NF pads with empty fields
```bash
echo "a b c" | awk '{ NF=5; print }'
# Output: a b c       ← (2 trailing spaces — $4 and $5 are empty strings)
echo "a b c" | awk '{ NF=5; print NF, "["$4"]", "["$5"]" }'
# Output: 5 [] []
```

### NF inside END reflects the LAST record processed
```bash
printf "a b\nc d e\n" | awk 'END { print NF }'
# Output: 3   ← NF from the last line read (c d e), not a sum or special value
```

---

## FS Regex vs Literal Confusion

### Single character FS is literal; multi-character FS is regex
```bash
# Single char: treated literally (even regex special chars)
echo "a.b.c" | awk -F. '{ print $2 }'    # b  — '.' as FS is literal, not "any char"

# Multi-char: treated as REGEX
echo "a..b..c" | awk -F'..' '{ print $2 }'   # ⚠️ '..' as regex means "any two chars"
# This does NOT mean "split on the literal string '..'"

# To split on a literal multi-character string, escape regex metacharacters:
echo "a..b..c" | awk -F'\\.\\.' '{ print $2 }'   # b  — correctly escaped

# Common trap: splitting on "|" (regex OR metacharacter)
echo "a|b|c" | awk -F'|' '{ print $2 }'      # b — single char, literal, works fine
echo "a||b||c" | awk -F'||' '{ print $2 }'   # ⚠️ unpredictable — '||' is invalid regex alternation
```

### Default FS (unset) splits on runs of whitespace AND trims leading/trailing
```bash
echo "   a   b   c   " | awk '{ print NF }'
# Output: 3   ← leading/trailing whitespace ignored, multiple spaces = one separator

echo "   a   b   c   " | awk -F' ' '{ print NF }'
# Output: 7   ← ⚠️ explicit FS=" " is a SPECIAL CASE same as default... 
# Actually FS=" " (single space, the default value) behaves like unset FS
# But if you do FS="[ ]" (regex), behavior changes:
echo "   a   b   c   " | awk -F'[ ]' '{ print NF }'
# Output: 9 or more ← now every individual space is a separator, creating empty fields!
```

---

## Uninitialized Variables

### Uninitialized variables are "" (string) AND 0 (number) simultaneously
```bash
awk 'BEGIN { print x }'         # prints nothing (empty string)
awk 'BEGIN { print x + 0 }'     # prints 0
awk 'BEGIN { if (x == "") print "empty" }'   # true
awk 'BEGIN { if (x == 0) print "zero" }'     # also true!
# Both conditions are true — uninitialized vars are simultaneously "" and 0
```

### Typos create silent new variables (no error!)
```bash
awk '{ total += $1 } END { print totl }' file.txt
# Typo: "totl" instead of "total"
# awk does NOT error — it just prints an uninitialized variable (empty/0)
# This is one of the most dangerous awk gotchas — no compile-time variable checking

# Use --lint (gawk) to catch this:
gawk --lint '{ total += $1 } END { print totl }' file.txt
# gawk: warning: reference to uninitialized variable `totl'
```

### Array vs scalar confusion
```bash
awk 'BEGIN { x = 5; x[1] = "a" }'
# gawk: fatal: attempt to use scalar `x' as an array
# Using a variable as both scalar and array is an error (good — caught at runtime)
```

---

## print vs printf Pitfalls

### print with parentheses and commas can be misread as function call
```bash
awk '{ print ($1, $2) }' file.txt
# ⚠️ This is NOT print($1, $2) like a function call
# It's print (expression-list) which evaluates ($1, $2) oddly in some contexts
# Safer: omit parentheses for print
awk '{ print $1, $2 }' file.txt    # ✅ correct
```

### printf without \n doesn't add a newline
```bash
awk '{ printf "%s", $1 }' file.txt
# Output: all values run together on one line — no \n!
# Must add explicitly:
awk '{ printf "%s\n", $1 }' file.txt
```

### printf with mismatched format specifiers
```bash
awk '{ printf "%d\n", $1 }' file.txt
# If $1 is "abc" (not numeric): prints 0 (silently, no error)
echo "abc" | awk '{ printf "%d\n", $1 }'
# Output: 0
```

### print redirection truncates on EVERY first occurrence within a run
```bash
awk '{ print $1 > "out.txt" }' file.txt
# This does NOT truncate on every line — awk opens "out.txt" ONCE
# (the first time this statement executes) and appends afterward
# To truly overwrite each time, you'd need close() + reopen, which defeats the purpose

# Common gotcha: running the SAME awk script twice appends old content's file handle state
# is per-execution, so each run starts fresh — but within one run, the file is opened once.
```

---

## Quoting & Shell Escaping

### Single quotes protect awk programs from shell expansion — almost always required
```bash
# ❌ Without quotes, shell interprets $1, *, etc.
awk { print $1 } file.txt
# bash: print: command not found (and worse with $1 expanding to empty)

# ✅ Always single-quote the awk program:
awk '{ print $1 }' file.txt
```

### Double quotes inside single-quoted awk programs
```bash
awk '{ print "Hello, " $1 }' file.txt        # ✅ fine, double quotes are awk string literals

# Passing a shell variable into awk safely: use -v, not string interpolation
name="alice"
awk -v user="$name" '$1 == user { print }' file.txt   # ✅ safe

# ❌ Dangerous: interpolating shell variables directly into the awk program string
awk "{ print \"$name\" }" file.txt
# Works but fragile — breaks if $name contains quotes, awk syntax chars, etc.
# Always prefer -v for passing shell values into awk
```

### Backslashes in regex need extra escaping from shell
```bash
# Matching a literal dot in awk regex: /\./ 
awk '/\./ { print }' file.txt      # ✅ works directly in single quotes

# But via -v or double-quoted program, backslashes may need doubling:
awk -v pat='\\.' '$0 ~ pat' file.txt   # need \\. because -v string isn't a regex literal
```

---

## getline Side Effects

### getline without redirection consumes from the SAME input stream
```bash
# This pairs lines (line 1+2, then 3+4, etc.)
awk '{ getline next_line; print $0, next_line }' file.txt
# ⚠️ getline ADVANCES NR and FNR — easy to lose track of position

# If file has odd number of lines, last getline fails silently:
# the last unpaired line may be dropped or behave unexpectedly
```

### getline return value should always be checked
```bash
# getline returns: 1 (success), 0 (EOF), -1 (error)
awk '{
  if ((getline line < "external.txt") > 0) {
    print line
  } else {
    print "no more lines or file error"
  }
}' file.txt

# ❌ Ignoring return value can process garbage on EOF/error:
awk '{ getline line < "external.txt"; print line }' file.txt
# On EOF, "line" retains its PREVIOUS value (not cleared!) — silent bug
```

### Forgetting close() exhausts file descriptors
```bash
# If you open many files via getline in a loop without closing:
awk '{
  while ((getline line < $1) > 0) { ... }
  # forgot close($1) here!
}' file_list.txt
# Eventually: "too many open files" error
# Always close() what you open with getline < file or | getline
```

---

## Array Key Coercion

### Array subscripts are always strings internally
```bash
awk 'BEGIN {
  a[1] = "one"
  a["1"] = "ONE"
  print a[1]      # "ONE" — same key! 1 and "1" are the same array index
}'
# Numbers used as array subscripts are converted to their string representation
```

### Numeric formatting affects array keys
```bash
awk 'BEGIN {
  a[1] = "int-key"
  a[1.0] = "float-key"
  print a[1]
}'
# Output: float-key — 1.0 converts to string "1" (same as integer 1), OVERWRITES previous entry

awk 'BEGIN {
  a[100000000000] = "big"
  print (100000000000 in a)
}'
# May be false! Very large numbers can be formatted in scientific notation as strings,
# making the lookup key different from what you expect (depends on CONVFMT)
```

### Iterating "for (k in arr)" has undefined order (unless gawk PROCINFO sorting is used)
```bash
awk 'BEGIN {
  a["zebra"]=1; a["apple"]=2; a["mango"]=3
  for (k in a) print k
}'
# Order is NOT guaranteed to be insertion order or alphabetical — implementation-defined!

# gawk: force sorted iteration
gawk 'BEGIN {
  a["zebra"]=1; a["apple"]=2; a["mango"]=3
  PROCINFO["sorted_in"] = "@ind_str_asc"
  for (k in a) print k
}'
# Output: apple, mango, zebra (alphabetical)
```

---

## Locale & Sorting Surprises

### String comparison depends on LC_COLLATE
```bash
# In en_US.UTF-8 locale, comparison may not be pure ASCII order:
LC_ALL=en_US.UTF-8 awk 'BEGIN { print ("apple" < "Banana") }'
# Result may differ from:
LC_ALL=C awk 'BEGIN { print ("apple" < "Banana") }'
# In C locale: uppercase letters sort BEFORE lowercase (ASCII order)
# In en_US locale: case-insensitive-ish collation may apply

# For predictable byte-order comparison, force C locale:
LC_ALL=C awk '...' file.txt
```

### Number formatting depends on locale (decimal point vs comma)
```bash
# In some European locales, awk might expect "," as decimal separator for input
LC_NUMERIC=de_DE.UTF-8 awk 'BEGIN { print 3.14 + 1 }'
# Behavior varies by awk implementation — POSIX awk typically uses "." regardless
# gawk respects locale for OUTPUT formatting in some cases — test carefully

# Always force C locale for numeric scripts to avoid surprises:
LC_NUMERIC=C awk '{ sum += $1 } END { print sum }' file.txt
```

---

## mawk vs gawk Feature Gaps

### Scripts using gawk-only features fail silently or error on mawk
```bash
# This works on gawk but fails on mawk (Debian/Ubuntu default 'awk'):
awk 'BEGIN { print strftime("%Y-%m-%d") }'
# mawk: awk: calling undefined function strftime
# gawk: 2024-06-15

# Check which awk you have:
awk --version | head -1

# Force gawk explicitly in scripts for portability of advanced features:
gawk 'BEGIN { print strftime("%Y-%m-%d") }'
```

### Regex interval expressions {n,m} disabled by default in old awk
```bash
# POSIX awk and old implementations may not support {n,m} without a flag:
echo "aaa" | awk '/a{2,3}/'    # may not match on old/strict awk

# gawk: works by default
# mawk: works by default (since mawk 1.3.4)
# POSIX mode in gawk may require --re-interval (older gawk versions)
```

---

## Large Number Precision

### awk uses double-precision floating point — large integers lose precision
```bash
awk 'BEGIN { print 9007199254740993 }'
# Output: 9007199254740992   ← off by one! Exceeds float64 precision (2^53)

awk 'BEGIN { print 123456789012345678 + 1 }'
# Output: wrong — precision lost far before this

# For exact large integer math, awk is NOT suitable — use python, bc, or perl with bigint
```

### Integer division truncation surprises
```bash
awk 'BEGIN { print 7/2 }'       # 3.5 — awk does float division by default
awk 'BEGIN { print int(7/2) }'  # 3   — must explicitly truncate
```

---

## Regex Greediness & Anchoring

### awk regex is always "leftmost longest" — no non-greedy option in POSIX
```bash
echo "<a><b><c>" | awk '{ gsub(/<.*>/, "X"); print }'
# Output: X   ← entire string consumed (greedy), not <a>X<b>X<c>

# To match the shortest each time, restrict the character class:
echo "<a><b><c>" | awk '{ gsub(/<[^>]*>/, "X"); print }'
# Output: XXX   ← matches each tag individually
```

### ^ and $ anchor to the WHOLE STRING, not each line in multi-line records
```bash
# When RS is changed to read multi-line records (e.g., RS=""), 
# ^ and $ still refer to start/end of the ENTIRE record, not each embedded line
awk 'BEGIN{RS=""} /^Name:/' file.txt
# Only matches if "Name:" is at the very start of the record,
# even if "Name:" appears at the start of a later line within that record

# gawk extension: use multiline mode carefully, or split manually
```

---

## Performance Traps

### Using system() or external pipes in a loop is extremely slow
```bash
# ❌ Spawns a new process for EVERY line — very slow on large files
awk '{ "date" | getline d; close("date"); print d, $0 }' huge_file.txt

# ✅ Call once in BEGIN if the value doesn't change per-line
awk 'BEGIN { "date" | getline d; close("date") } { print d, $0 }' huge_file.txt
```

### Repeatedly rebuilding $0 via field assignment is costlier than it looks
```bash
# Each field assignment triggers a full $0 rebuild internally
awk '{ $1=$1; $2=$2; $3=$3; print }' huge_file.txt
# Slower than necessary — only assign fields that actually change

awk '{ if ($2 != "X") $2 = "X"; print }' huge_file.txt   # rebuild only if needed
```

### Unanchored regex on every line is slower than necessary
```bash
# Scanning entire line for an unanchored pattern:
awk '/pattern/' huge_file.txt

# If you know the pattern is always at the start, anchor it — faster:
awk '/^pattern/' huge_file.txt
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
