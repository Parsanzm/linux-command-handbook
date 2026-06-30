# awk — The Complete Reference

> **Pattern scanning and text processing language**
> awk is not just a command — it's a complete programming language
> designed for structured text processing. One-liners or full programs,
> awk handles both with equal elegance.

---

## Table of Contents

- [What is awk?](#what-is-awk)
- [Where does awk live?](#where-does-awk-live)
- [How awk works internally](#how-awk-works-internally)
- [Syntax & Program Structure](#syntax--program-structure)
- [Built-in Variables](#built-in-variables)
- [Patterns](#patterns)
- [Actions](#actions)
- [Operators](#operators)
- [Control Flow](#control-flow)
- [Built-in Functions](#built-in-functions)
- [Arrays](#arrays)
- [User-Defined Functions](#user-defined-functions)
- [Input/Output](#inputoutput)
- [awk vs gawk vs mawk vs nawk](#awk-vs-gawk-vs-mawk-vs-nawk)
- [Related Commands](#related-commands)

---

## What is awk?

`awk` is a **domain-specific programming language** for text processing. It was created in 1977 at Bell Labs by **A**lf Aho, **P**eter **W**einberger, and Brian **K**ernighan — the name is their initials.

The core idea: read input line by line, split each line into fields, apply pattern-action pairs. Simple in concept, enormously powerful in practice.

**Three core uses:**
1. **Extract fields** from structured text (CSV, logs, `/proc` files)
2. **Transform and reformat** data (reorder columns, change delimiters)
3. **Aggregate and report** (counts, sums, averages, unique values)

---

## Where does awk live?

```
/usr/bin/awk        ← symlink to gawk or mawk depending on distro
/usr/bin/gawk       ← GNU awk (most feature-rich)
/usr/bin/mawk       ← fast awk (default on Debian/Ubuntu)
/usr/bin/nawk       ← new awk (standard on Solaris/macOS)
```

```bash
which awk
awk --version       # shows which implementation
ls -la $(which awk) # see what awk links to

# On Debian/Ubuntu: awk → mawk (fast, POSIX-compliant)
# On RHEL/Fedora:   awk → gawk (GNU, most features)
# On macOS:         awk → nawk (BSD awk)
```

`gawk` (GNU awk) is the reference implementation with the most features. Install on any system:

```bash
apt install gawk     # Debian/Ubuntu
dnf install gawk     # RHEL/Fedora
brew install gawk    # macOS
```

---

## How awk works internally

```
for each input file (or stdin):
  split into records (default: lines)
    for each record:
      split into fields (default: whitespace)
      for each pattern-action pair in the program:
        if pattern matches (or no pattern):
          execute action
```

**Processing model:**

```
┌─────────────────────────────────────────────┐
│  BEGIN { ... }      ← runs once before input │
├─────────────────────────────────────────────┤
│  for each record:                            │
│    split into fields $1 $2 ... $NF          │
│    pattern { action }   ← for each rule      │
│    pattern { action }                        │
├─────────────────────────────────────────────┤
│  END { ... }        ← runs once after input  │
└─────────────────────────────────────────────┘
```

**Field splitting:**
- Read one record (line) at a time
- Split by `FS` (field separator, default: any whitespace)
- `$1` = first field, `$2` = second, ..., `$NF` = last field
- `$0` = the entire record unchanged
- Assigning to a field (`$2 = "new"`) rebuilds `$0`

**Record splitting:**
- Default: newline (`RS = "\n"`)
- Can be changed to split on any character or regex

---

## Syntax & Program Structure

### Basic syntax

```bash
awk 'program' file
awk 'program' file1 file2 ...
awk -f program.awk file
command | awk 'program'
```

### Program structure

```awk
BEGIN { initialization }

/pattern/ { action }
condition { action }
pattern1, pattern2 { action }   # range pattern

END { cleanup / report }
```

### The pattern-action pair

```awk
pattern { action }
```

- **pattern only** (no action) → default action is `print`
- **action only** (no pattern) → runs on every record
- **both** → runs action only when pattern matches
- **neither** → not valid

```bash
# Pattern only — print lines matching "error":
awk '/error/' logfile.txt
# Equivalent to:
awk '/error/ { print }' logfile.txt

# Action only — print field 2 of every line:
awk '{ print $2 }' file.txt

# Both — print field 2 only on lines matching "alice":
awk '/alice/ { print $2 }' file.txt
```

### Multiple rules

```bash
awk '
  BEGIN { print "Start" }
  /error/ { errors++ }
  /warning/ { warnings++ }
  END {
    print "Errors:", errors
    print "Warnings:", warnings
  }
' logfile.txt
```

---

## Built-in Variables

### Record & Field Variables

| Variable | Description |
|----------|-------------|
| `$0` | Entire current record (line) |
| `$1`, `$2`, ..., `$NF` | Fields 1, 2, ..., last |
| `NF` | Number of fields in current record |
| `NR` | Number of records processed so far (total) |
| `FNR` | Number of records in current file only |
| `FS` | Field separator (default: whitespace) |
| `RS` | Record separator (default: newline) |
| `OFS` | Output field separator (default: space) |
| `ORS` | Output record separator (default: newline) |

### File Variables

| Variable | Description |
|----------|-------------|
| `FILENAME` | Name of current input file |
| `ARGC` | Number of command-line arguments |
| `ARGV` | Array of command-line arguments |

### gawk-Specific Variables

| Variable | Description |
|----------|-------------|
| `OFMT` | Output format for numbers (default: `"%.6g"`) |
| `CONVFMT` | Conversion format for numbers to strings |
| `SUBSEP` | Separator for multi-dimensional array subscripts (default: `\034`) |
| `RT` | Record terminator (what matched `RS` when it's a regex) |
| `FIELDWIDTHS` | Fixed-width field splitting |
| `FPAT` | Field pattern (match fields by content, not delimiter) |

```bash
# NR vs FNR (when processing multiple files):
awk '{ print NR, FNR, FILENAME, $0 }' file1.txt file2.txt
# NR increments across all files
# FNR resets to 1 at each new file

# OFS: change output separator
awk 'BEGIN{OFS=","} {print $1, $2, $3}' file.txt

# ORS: change output record separator
awk 'BEGIN{ORS="\n\n"} {print}' file.txt  # double-space output

# RS: read paragraph by paragraph (RS = "")
awk 'BEGIN{RS=""} {print NR, $0}' file.txt

# FILENAME: useful when processing multiple files
awk '{print FILENAME ":" NR ": " $0}' *.log
```

---

## Patterns

### No pattern (matches every record)
```awk
{ print $1 }          # runs on every line
```

### Regex pattern
```awk
/pattern/             # lines matching regex
/^ERROR/              # lines starting with ERROR
/[0-9]+\.[0-9]+/      # lines with decimal numbers
!/pattern/            # lines NOT matching (negated)
```

### Expression pattern
```awk
NR > 5                # records after line 5
NF == 3               # records with exactly 3 fields
$1 == "alice"         # first field equals "alice"
$3 > 100              # third field greater than 100
NR % 2 == 0           # even-numbered lines
length($0) > 80       # lines longer than 80 chars
```

### Range pattern
```awk
/start/, /end/        # from first match of start to first match of end (inclusive)
NR==5, NR==10         # records 5 through 10
/BEGIN/, /END/        # all lines between BEGIN and END markers
```

### BEGIN and END
```awk
BEGIN { ... }         # before any input is read
END   { ... }         # after all input is processed
```

### gawk: BEGINFILE and ENDFILE
```awk
BEGINFILE { print "Starting:", FILENAME }
ENDFILE   { print "Done with:", FILENAME, "(" FNR, "lines)" }
```

---

## Actions

### print vs printf

```awk
# print: adds OFS between args, ORS at end
print $1, $2, $3      # fields separated by OFS (default: space)
print $1 " " $2       # explicit space (string concatenation)
print                 # print $0 (entire record)
print ""              # print empty line

# printf: C-style formatted output, no automatic newline
printf "%s\t%d\t%.2f\n", $1, $2, $3
printf "%-20s %5d\n", name, count
```

### printf format specifiers

| Specifier | Description |
|-----------|-------------|
| `%s` | String |
| `%d` | Integer (decimal) |
| `%f` | Floating point |
| `%e` | Scientific notation |
| `%g` | Shorter of `%e` or `%f` |
| `%o` | Octal |
| `%x` | Hexadecimal |
| `%c` | Character (ASCII code or single char) |
| `%%` | Literal `%` |

Width and precision:
```awk
printf "%10s"     # right-align in 10 chars
printf "%-10s"    # left-align in 10 chars
printf "%05d"     # zero-padded integer
printf "%.2f"     # 2 decimal places
printf "%10.2f"   # width 10, 2 decimal places
```

---

## Operators

### Arithmetic
```awk
+   -   *   /   %   ^      # add, sub, mul, div, mod, exponent
++  --                     # increment, decrement (pre/post)
+=  -=  *=  /=  %=  ^=    # compound assignment
```

### Comparison
```awk
==  !=  <  >  <=  >=       # compare (numeric or string based on context)
```

### String
```awk
$1 " " $2                  # concatenation (juxtaposition)
$1 ~ /pattern/             # matches regex
$1 !~ /pattern/            # does not match regex
```

### Logical
```awk
&&    # AND
||    # OR
!     # NOT
```

### Ternary
```awk
condition ? value_if_true : value_if_false
$3 > 0 ? "positive" : "non-positive"
```

### String vs Numeric comparison
```awk
# awk chooses comparison type based on context:
"10" > "9"    # false: string comparison ("1" < "9")
10 > 9        # true: numeric comparison
$1 > 9        # numeric if $1 looks like a number, string otherwise

# Force numeric:
$1 + 0 > 9    # force numeric comparison
# Force string:
$1 "" > "9"   # force string comparison (concatenate empty string)
```

---

## Control Flow

### if / else
```awk
if (condition) {
  action
} else if (condition2) {
  action2
} else {
  action3
}

# One-liner:
if ($3 > 100) print "high"; else print "low"
```

### while
```awk
while (condition) {
  action
}

# Example: print each character of $1
{
  i = 1
  while (i <= length($1)) {
    print substr($1, i, 1)
    i++
  }
}
```

### do-while
```awk
do {
  action
} while (condition)
```

### for
```awk
# C-style for:
for (i = 1; i <= NF; i++) {
  print i, $i
}

# for-in (iterate over array keys):
for (key in array) {
  print key, array[key]
}
```

### Loop control
```awk
break       # exit loop
continue    # next iteration
next        # skip to next record (like continue for the outer record loop)
nextfile    # skip to next file (gawk)
exit [code] # stop processing, go to END
```

---

## Built-in Functions

### String Functions

| Function | Description |
|----------|-------------|
| `length(s)` | Length of string (or array if no arg: length of $0) |
| `substr(s, start [, len])` | Substring starting at `start` (1-indexed) |
| `index(s, t)` | Position of `t` in `s` (0 if not found) |
| `split(s, arr [, fs])` | Split `s` into array `arr` using `fs` |
| `sub(regex, repl [, target])` | Replace first match of regex in target (default: $0) |
| `gsub(regex, repl [, target])` | Replace all matches |
| `match(s, regex)` | Find regex in s; sets RSTART and RLENGTH |
| `sprintf(fmt, ...)` | Format string (like printf but returns string) |
| `tolower(s)` | Convert to lowercase |
| `toupper(s)` | Convert to uppercase |
| `gensub(regex, repl, how [, target])` | Return modified string (gawk only) |
| `patsplit(s, arr, pat)` | Split by pattern (gawk only) |
| `strftime(fmt [, timestamp])` | Format timestamp (gawk only) |

```bash
# Examples:
awk '{ print length($0) }' file.txt                  # line lengths
awk '{ print substr($0, 1, 10) }' file.txt            # first 10 chars
awk '{ print index($0, "error") }' file.txt           # position of "error"
awk '{ n = split($0, arr, ":"); print arr[1] }' file  # split by :
awk '{ sub(/foo/, "bar"); print }' file.txt           # replace first foo
awk '{ gsub(/foo/, "bar"); print }' file.txt          # replace all foo
awk '{ print toupper($1) }' file.txt                  # uppercase first field
awk 'match($0, /[0-9]+/) { print substr($0, RSTART, RLENGTH) }' f  # extract number
```

### Numeric Functions

| Function | Description |
|----------|-------------|
| `int(x)` | Integer part (truncates toward zero) |
| `sqrt(x)` | Square root |
| `sin(x)` | Sine (radians) |
| `cos(x)` | Cosine (radians) |
| `atan2(y, x)` | Arctangent of y/x |
| `exp(x)` | e^x |
| `log(x)` | Natural logarithm |
| `rand()` | Random number [0, 1) |
| `srand([seed])` | Set random seed; returns previous seed |

```bash
awk 'BEGIN { srand(); print int(rand() * 100) }'    # random 0-99
awk '{ print int($1) }' file.txt                    # truncate to int
awk '{ printf "%.2f\n", sqrt($1) }' file.txt        # square root
```

### I/O Functions

| Function | Description |
|----------|-------------|
| `print ... > "file"` | Write to file (overwrite) |
| `print ... >> "file"` | Append to file |
| `print ... \| "cmd"` | Pipe to command |
| `"cmd" \| getline var` | Read from command into variable |
| `getline` | Read next line from current input |
| `getline var` | Read next line into variable |
| `getline var < "file"` | Read from file |
| `close("file")` | Close file or pipe |
| `fflush([file])` | Flush output buffer |
| `system("cmd")` | Run shell command, return exit status |

---

## Arrays

awk arrays are **associative** (hash maps) — keys can be strings or numbers.

```awk
# Assignment
arr[key] = value
arr["name"] = "alice"
arr[1] = "first"

# Access
print arr["name"]
print arr[1]

# Check existence
if ("name" in arr) { ... }

# Delete
delete arr["name"]
delete arr           # delete entire array

# Iterate
for (key in arr) {
  print key, arr[key]
}

# Multi-dimensional (simulated with SUBSEP)
arr[i, j] = value
if ((i, j) in arr) { ... }
# Key is actually: i SUBSEP j (SUBSEP = \034)
```

### Array patterns

```bash
# Count occurrences
awk '{ count[$1]++ } END { for (w in count) print w, count[w] }' file

# Sum by category
awk '{ sum[$1] += $2 } END { for (k in sum) print k, sum[k] }' file

# Unique values
awk '!seen[$0]++' file.txt

# Frequency table
awk '{ freq[$0]++ } END { for (v in freq) print freq[v], v }' file | sort -rn

# Group by field
awk '{ lines[$1] = lines[$1] "\n" $0 } END { for (k in lines) print k ":" lines[k] }' file
```

---

## User-Defined Functions

```awk
function name(param1, param2,    local1, local2) {
  # body
  return value
}
```

**Note:** Local variables are declared as extra parameters (by convention separated by extra spaces).

```awk
# Example: max function
function max(a, b) {
  return (a > b) ? a : b
}

{ print max($1, $2) }
```

```awk
# Example: trim whitespace
function trim(s) {
  gsub(/^[ \t]+|[ \t]+$/, "", s)
  return s
}

{ print trim($0) }
```

```awk
# Example: local variables (extra params)
function count_words(str,    arr, n) {
  # arr and n are local variables
  n = split(str, arr, " ")
  return n
}

{ print count_words($0) }
```

---

## Input/Output

### Reading files
```bash
# From file
awk '{ print }' file.txt

# From stdin
cat file.txt | awk '{ print }'
awk '{ print }' < file.txt

# Multiple files
awk '{ print FILENAME, $0 }' file1 file2

# With command-line variables
awk -v name="alice" '$1 == name { print }' file.txt
```

### Writing output
```awk
# Redirect to file (truncates on first write, then appends)
{ print $1 > "/tmp/output.txt" }

# Append to file
{ print $1 >> "/tmp/output.txt" }

# Pipe to command
{ print $0 | "sort -u" }

# Multiple output files
{ print > "output_" $1 ".txt" }    # one file per value of $1
```

### getline
```awk
# Read next line from input (advances NR)
{ getline; print }    # print every other line

# Read from file
{ getline line < "/etc/hostname"; print line }

# Read from command
{ "date" | getline today; print today }

# Close after use (important for commands called repeatedly)
{ "date" | getline today; close("date"); print today }
```

### Passing variables from shell
```bash
# -v flag
awk -v threshold=100 '$3 > threshold { print }' file.txt

# Multiple -v flags
awk -v min=10 -v max=100 '$1 >= min && $1 <= max' file.txt

# Shell variable (via -v)
limit=50
awk -v lim="$limit" '$1 > lim' file.txt

# ARGV/ENVIRON
awk 'BEGIN { print ENVIRON["HOME"] }' /dev/null
```

---

## awk vs gawk vs mawk vs nawk

| Feature | POSIX awk | gawk | mawk | nawk |
|---------|-----------|------|------|------|
| Standard | POSIX | GNU | — | AT&T |
| Speed | — | Good | Fastest | Good |
| `gensub()` | ❌ | ✅ | ❌ | ❌ |
| `strftime()` | ❌ | ✅ | ❌ | ❌ |
| `BEGINFILE`/`ENDFILE` | ❌ | ✅ | ❌ | ❌ |
| `nextfile` | ❌ | ✅ | ✅ | ❌ |
| `FPAT` | ❌ | ✅ | ❌ | ❌ |
| Regex intervals `{n,m}` | ❌ | ✅ | ✅ | ❌ |
| TCP/IP (`/inet/`) | ❌ | ✅ | ❌ | ❌ |
| Array sort (`asorti`) | ❌ | ✅ | ❌ | ❌ |
| Profiling | ❌ | ✅ | ❌ | ❌ |
| Available by default | Most systems | RHEL/Fedora | Debian/Ubuntu | macOS |

**Recommendation:** Write portable POSIX awk when possible. Use `gawk` for advanced features. Never rely on `mawk` extensions (it has very few).

---

## Related Commands

| Command | Relation |
|---------|---------|
| `sed` | Stream editor — simpler text transformation, regex substitution |
| `grep` | Search only — no field splitting, no arithmetic |
| `cut` | Simple field extraction — faster but less flexible than awk |
| `sort` | Sort lines — often combined with awk |
| `uniq` | Remove/count duplicates — `awk '!seen[$0]++'` does the same |
| `perl` | More powerful, but heavier — awk is faster for field-based tasks |
| `python` | General purpose — awk beats it for quick one-liners |
| `paste` | Merge files column by column |
| `join` | Relational join on sorted files |
| `xargs` | Pipe awk output as arguments to other commands |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
