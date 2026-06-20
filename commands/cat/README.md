# cat — The Complete Reference

> **Concatenate files and print to standard output**
> One of the oldest and most used commands in Unix — simple on the surface, powerful underneath.

---

## Table of Contents

- [What is cat?](#what-is-cat)
- [Where does cat live?](#where-does-cat-live)
- [How cat works internally](#how-cat-works-internally)
- [Syntax](#syntax)
- [All Options](#all-options)
- [Variants: gzcat, bzcat, zcat, lzcat](#variants)
- [Quick Examples](#quick-examples)
- [Related Commands](#related-commands)

---

## What is cat?

`cat` stands for **concatenate**. It reads files sequentially and writes them to standard output. Despite its simplicity, it's the backbone of countless shell pipelines and scripts.

Originally written by Ken Thompson for Unix Version 1 (1971), `cat` has remained virtually unchanged for over 50 years.

**Three core uses:**
1. Display the contents of a file
2. Merge multiple files into one
3. Create files from standard input

---

## Where does cat live?

```
/bin/cat          ← on most Linux systems (Debian, Ubuntu, Arch)
/usr/bin/cat      ← on some distros and macOS
```

Verify on your system:

```bash
which cat          # → /bin/cat  or  /usr/bin/cat
type cat           # → cat is /bin/cat
command -v cat     # → /bin/cat
ls -lh $(which cat)
```

On Linux, `cat` is part of **GNU coreutils**. On macOS/BSD it's the **BSD version** — slightly different behavior on some flags.

Check your version:
```bash
cat --version      # GNU coreutils version (Linux only)
```

---

## How cat works internally

`cat` is a thin wrapper around three system calls:

```
open(file)  →  read(fd, buffer)  →  write(stdout, buffer)
```

1. Opens each file in order
2. Reads chunks (usually 4096 or 65536 bytes) into a buffer
3. Writes that buffer to stdout (file descriptor 1)
4. Moves to the next file
5. Closes when all files are done or EOF is reached on stdin

**stdin/stdout model:**
- If no file is given → reads from stdin (fd 0)
- Output always goes to stdout (fd 1)
- Errors go to stderr (fd 2)

**Kernel-level shortcut:** On modern Linux, `cat` uses `splice()` or `sendfile()` syscalls to pass data between file descriptors without copying into userspace — making it extremely fast for large files.

**Exit codes:**
| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Error (file not found, permission denied, etc.) |

---

## Syntax

```
cat [OPTION]... [FILE]...
```

- Files are processed **in the order given**
- `-` means **read from stdin** at that position
- With **no arguments** → reads stdin until Ctrl+D (EOF)

---

## All Options

### GNU cat (Linux)

| Option | Long form | Description |
|--------|-----------|-------------|
| `-A` | `--show-all` | Equivalent to `-vET` |
| `-b` | `--number-nonblank` | Number non-empty output lines (overrides `-n`) |
| `-e` | | Equivalent to `-vE` |
| `-E` | `--show-ends` | Show `$` at end of each line |
| `-n` | `--number` | Number all output lines |
| `-s` | `--squeeze-blank` | Suppress repeated empty output lines |
| `-t` | | Equivalent to `-vT` |
| `-T` | `--show-tabs` | Show tabs as `^I` |
| `-u` | | (Ignored — output is always unbuffered) |
| `-v` | `--show-nonprinting` | Show non-printing chars using `^` and `M-` notation |
| | `--help` | Display help |
| | `--version` | Output version information |

### BSD cat (macOS)

Same as GNU but adds:

| Option | Description |
|--------|-------------|
| `-l` | Lock output file with flock(2) |

---

## Variants

These are separate programs that work like `cat` but for **compressed files** — they decompress on-the-fly and print to stdout without creating a temporary file.

| Command | Format | Package |
|---------|--------|---------|
| `zcat` | `.gz` (gzip) | `gzip` |
| `gzcat` | `.gz` (gzip) | `gzip` (BSD/macOS alias) |
| `bzcat` | `.bz2` (bzip2) | `bzip2` |
| `xzcat` | `.xz` (xz/lzma) | `xz-utils` |
| `lzcat` | `.lz` (lzip) | `lzip` |
| `lzopcat` | `.lzo` (lzop) | `lzop` |
| `zstdcat` | `.zst` (zstandard) | `zstd` |

**Usage:**
```bash
zcat file.txt.gz            # decompress + print
bzcat archive.bz2           # bzip2 compressed
xzcat data.xz               # xz compressed
zstdcat backup.zst          # zstandard compressed

# Pipe directly:
zcat server.log.gz | grep "ERROR"
bzcat data.bz2 | wc -l
```

**`zcat` on macOS** behaves like `gzcat`. On Linux it may also handle `.Z` (compress) files.

---

## Quick Examples

```bash
# Display a file
cat file.txt

# Show with line numbers
cat -n file.txt

# Number only non-blank lines
cat -b file.txt

# Show hidden characters (tabs, line endings)
cat -A file.txt

# Squeeze multiple blank lines into one
cat -s file.txt

# Concatenate files
cat file1.txt file2.txt file3.txt

# Merge into a new file
cat file1.txt file2.txt > merged.txt

# Append to existing file
cat extra.txt >> main.txt

# Create a file from stdin
cat > newfile.txt
# (type content, press Ctrl+D to save)

# Stdin in the middle of files
cat header.txt - footer.txt < body.txt

# Empty a file without deleting it
cat /dev/null > logfile.txt
# or:
> logfile.txt
```

---

## Related Commands

| Command | Relation |
|---------|---------|
| `tac` | Reverse of cat — prints lines in reverse order |
| `less` | Pager — view file one screen at a time |
| `more` | Simpler pager |
| `head` | Print first N lines |
| `tail` | Print last N lines (also `-f` for follow) |
| `strings` | Extract printable strings from binary |
| `xxd` | Hex dump of file |
| `od` | Octal/hex dump |
| `nl` | Number lines (more flexible than `cat -n`) |
| `rev` | Reverse characters in each line |
| `paste` | Merge lines of files side by side |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
