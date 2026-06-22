# cat — Edge Cases & Gotchas

> The things that work differently than you'd expect.
> Know these before you use `cat` in production.

---

## Table of Contents

- [Filename Traps](#filename-traps)
- [stdin Behavior](#stdin-behavior)
- [Binary & Special Files](#binary--special-files)
- [Large Files](#large-files)
- [Permissions & Errors](#permissions--errors)
- [GNU vs BSD differences](#gnu-vs-bsd-differences)
- [Encoding & Locale](#encoding--locale)
- [Newlines & Line Endings](#newlines--line-endings)
- [Race Conditions](#race-conditions)
- [Useless Use of Cat (UUOC)](#useless-use-of-cat-uuoc)
- [Terminal Corruption](#terminal-corruption)

---

## Filename Traps

### Files starting with `-`
```bash
cat -file.txt         # ❌ Error: invalid option -- 'f'

cat -- -file.txt      # ✅ Correct: -- ends option parsing
cat ./-file.txt       # ✅ Also correct
```

### Files with spaces
```bash
cat my file.txt       # ❌ Reads two files: "my" and "file.txt"

cat "my file.txt"     # ✅
cat my\ file.txt      # ✅
```

### Files with newlines in name (exotic but valid on Linux)
```bash
# Don't use cat with xargs for these — xargs splits on newlines
cat file\nname.txt    # Probably not what you expect

# Safe alternative:
find . -name "*pattern*" -exec cat {} \;
find . -name "*pattern*" -print0 | xargs -0 cat
```

### Files with unicode names
```bash
cat données.csv       # Works fine in UTF-8 locale
cat "日本語.txt"       # Works if your locale supports it
# Locale must match filename encoding
```

### Dotfiles (hidden files)
```bash
cat .env              # ✅ Works normally
cat ~/.bashrc         # ✅ Works
cat ~/.ssh/config     # ✅ Works (if permissions allow: 600)
cat .hiddendir/file   # ✅ Works

# cat does NOT expand globs — shell does
cat .*                # ❌ Dangerous: may include . and ..
cat .env .config      # ✅ Explicit list is safe
```

---

## stdin Behavior

### No arguments → waits forever
```bash
cat           # Reads from stdin — blocks until Ctrl+D
              # This is correct behavior, not a hang
```

### Ctrl+D vs Ctrl+C
```bash
cat > file.txt
# Type content
# Ctrl+D → saves the file (sends EOF)
# Ctrl+C → aborts, file may be empty or partial
```

### `-` reads stdin at that position
```bash
cat file1.txt - file2.txt
# Reads file1.txt, then stdin (until Ctrl+D), then file2.txt
# Order matters!
```

### Piping and stdin interaction
```bash
echo "hello" | cat           # stdin = pipe, works fine
echo "hello" | cat - file.txt  # stdin (pipe) first, then file.txt
echo "hello" | cat file.txt -  # file.txt first, then stdin (pipe)
```

---

## Binary & Special Files

### Binary files corrupt terminal
```bash
cat image.png         # ❌ Garbled output, may corrupt terminal
cat /bin/ls           # ❌ Same issue

cat -v image.png      # ✅ Shows non-printing chars safely
xxd image.png         # ✅ Hex dump
strings binary_file   # ✅ Extract printable strings
```

### Recovering a corrupted terminal after binary cat
```bash
reset                 # Full terminal reset
stty sane             # Restore sane terminal settings
printf '\033c'        # ANSI reset sequence
```

### Special files in /proc and /sys
```bash
cat /proc/cpuinfo         # ✅ CPU info (virtual file, always fresh)
cat /proc/meminfo         # ✅ Memory info
cat /proc/$$/status       # ✅ Status of current shell process
cat /proc/net/tcp         # ✅ Active TCP connections (raw)
cat /sys/class/net/eth0/address   # ✅ MAC address
cat /proc/sys/kernel/hostname     # ✅ Current hostname

# These are NOT real files on disk — kernel generates them live
# cat reads them differently (size is always 0 in stat)
```

### /dev/null and /dev/zero
```bash
cat /dev/null         # Outputs nothing (empty file)
cat /dev/zero         # ⚠️  Infinite stream of null bytes — will run forever
cat /dev/random       # ⚠️  Infinite stream of random bytes
cat /dev/urandom      # ⚠️  Same — use with head or pv to limit
```

### /dev/stdin, /dev/stdout
```bash
cat /dev/stdin        # Same as cat with no args
cat file.txt > /dev/stdout    # Explicit stdout
```

---

## Large Files

### cat + large file = bad idea
```bash
cat 50GB.log           # ❌ Dumps everything to terminal, impossible to stop cleanly

# Better alternatives:
less 50GB.log          # Page through it
head -100 50GB.log     # First 100 lines
tail -100 50GB.log     # Last 100 lines
tail -f 50GB.log       # Follow new content
grep "ERROR" 50GB.log  # Search directly without cat

# If you truly need to pipe a large file:
cat 50GB.log | grep "pattern"    # Works but slower
grep "pattern" 50GB.log          # Faster (no extra process)
```

### cat with split files
```bash
# Correctly reassemble split files
cat part.aa part.ab part.ac > original.tar.gz
cat part.?? > original.tar.gz    # glob works if ordered

# Reassemble and extract in one step
cat part.?? | tar xz
```

---

## Permissions & Errors

### Permission denied
```bash
cat /etc/shadow        # ❌ Permission denied (readable only by root)
sudo cat /etc/shadow   # ✅

# Note: sudo tee is better for writing as root:
echo "content" | sudo tee /etc/protected_file
# (sudo cat > file would fail because > runs as current user)
```

### File doesn't exist — cat still processes others
```bash
cat file1.txt missing.txt file3.txt
# file1.txt: displayed ✅
# missing.txt: error to stderr ❌
# file3.txt: displayed ✅
# Exit code: 1 (error)
```

### cat on a directory
```bash
cat /tmp/              # ❌ Error: Is a directory
cat /tmp               # ❌ Same
```

### Symlink to non-existent file
```bash
ln -s /nonexistent broken_link
cat broken_link        # ❌ No such file or directory
```

---

## GNU vs BSD Differences

| Behavior | GNU (Linux) | BSD (macOS) |
|----------|------------|-------------|
| `-A` flag | ✅ Supported | ❌ Not available |
| `--version` | ✅ Shows coreutils version | ❌ Not available |
| `--help` | ✅ | ❌ (use `man cat`) |
| `-e` flag | ✅ (like `-vE`) | ✅ (same) |
| `-l` flag | ❌ | ✅ (lock file) |
| Speed on large files | Uses `splice()` syscall | Standard read/write |

**Write portable scripts:**
```bash
# Avoid GNU-only long flags in scripts meant to run on macOS
cat --show-ends file.txt   # ❌ breaks on macOS
cat -E file.txt            # ✅ works on both
```

---

## Encoding & Locale

### UTF-8 vs Latin-1
```bash
# cat is encoding-agnostic: passes bytes as-is
# Display depends on terminal encoding
cat utf8_file.txt         # Fine in UTF-8 terminal
cat latin1_file.txt       # May show garbled chars in UTF-8 terminal

# -v helps diagnose:
cat -v latin1_file.txt
# Non-ASCII bytes show as M-... notation
```

### BOM (Byte Order Mark)
```bash
# Windows UTF-8 files often start with BOM: EF BB BF
cat -v bom_file.txt | head -1
# Shows: M-oM-;M-? at start of first line

# Remove BOM:
sed -i '1s/^\xEF\xBB\xBF//' file.txt
```

---

## Newlines & Line Endings

### Missing final newline
```bash
printf "no newline" > file.txt
cat file.txt           # Displays fine but prompt appears on same line
cat -A file.txt        # Last line has NO $ at end → no newline
```

### Windows CRLF (\r\n)
```bash
cat windows_file.txt   # Looks normal in most terminals
cat -A windows_file.txt # Shows ^M$ at line endings

# Convert:
sed -i 's/\r//' file.txt
dos2unix file.txt
```

### Concatenating files with/without final newlines
```bash
printf "line1" > a.txt    # no trailing newline
printf "line2" > b.txt
cat a.txt b.txt
# Output: line1line2   ← joined with no separator!

# Fix: ensure all files end with newline, or add one:
(cat a.txt; echo) > a_fixed.txt
```

---

## Race Conditions

### cat on a file being written concurrently
```bash
# Another process writes to active.log
cat active.log         # May read partial data mid-write
# Use tail -f for live log watching instead
```

### cat > file while file is being read
```bash
# DON'T do this:
cat file.txt > file.txt    # ❌ Truncates file BEFORE reading starts → empty file!

# Correct: use a temp file
cat file.txt > file.tmp && mv file.tmp file.txt
# Or: sponge (from moreutils)
cat file.txt | sponge file.txt
```

---

## Useless Use of Cat (UUOC)

A common anti-pattern. `cat` passes data to a pipe, but most commands accept files directly.

```bash
# ❌ Useless cat
cat file.txt | grep "pattern"
cat file.txt | wc -l
cat file.txt | awk '{print $1}'
cat file.txt | sort
cat file.txt | head -20

# ✅ Direct and faster (one fewer process)
grep "pattern" file.txt
wc -l file.txt            # or: wc -l < file.txt
awk '{print $1}' file.txt
sort file.txt
head -20 file.txt
```

**When cat IS appropriate in a pipeline:**
```bash
# When you need to concatenate MULTIPLE files
cat file1.txt file2.txt | grep "pattern"    # ✅ justified

# When you want to be explicit about stdin
cat | process_input     # ✅ makes intent clear
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
