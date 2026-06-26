# tar — The Complete Reference

> **Tape ARchive: pack, compress, and extract file archives**
> Originally designed for magnetic tape backup in 1979.
> Today the universal archive format on Linux — from backups to software distribution.

---

## Table of Contents

- [What is tar?](#what-is-tar)
- [Where does tar live?](#where-does-tar-live)
- [How tar works internally](#how-tar-works-internally)
- [Syntax — Old vs New Style](#syntax--old-vs-new-style)
- [Core Operations](#core-operations)
- [Compression Options](#compression-options)
- [All Key Options](#all-key-options)
- [Archive Format Internals](#archive-format-internals)
- [tar vs zip vs 7z vs cpio](#tar-vs-zip-vs-7z-vs-cpio)
- [Related Commands](#related-commands)

---

## What is tar?

`tar` originally stood for **Tape ARchive**. It was designed in 1979 to write files sequentially to magnetic tape drives. Modern `tar` still uses the same fundamental approach: pack files one after another into a single stream, with headers describing each file.

`tar` itself does **not compress** — it only bundles. Compression is handled by a separate tool (`gzip`, `bzip2`, `xz`, `zstd`) either piped or via built-in flags. This separation is the Unix philosophy in action.

**Three core uses:**
1. **Create** archives (`tar -c`) — bundle files/dirs into one `.tar` file
2. **Extract** archives (`tar -x`) — unpack files from an archive
3. **List** contents (`tar -t`) — inspect without extracting

---

## Where does tar live?

```
/bin/tar            ← most Linux systems
/usr/bin/tar        ← some distros
```

```bash
which tar
tar --version
# tar (GNU tar) 1.34
```

On Linux: **GNU tar** (part of `tar` package). On macOS: **BSD tar** (with significant differences). GNU tar has far more features.

```bash
# Install GNU tar on macOS (via Homebrew):
brew install gnu-tar    # installs as gtar
```

---

## How tar works internally

### The tar stream format

```
┌─────────────────┐
│   512-byte      │  ← File header (name, size, permissions, timestamps, owner...)
│   header block  │
├─────────────────┤
│                 │
│   File data     │  ← File contents (padded to 512-byte boundary)
│                 │
├─────────────────┤
│   512-byte      │  ← Next file header
│   header block  │
├─────────────────┤
│   ...           │
├─────────────────┤
│  Two 512-byte   │  ← End-of-archive marker (two zero blocks)
│  zero blocks    │
└─────────────────┘
```

Each file in a tar archive has:
1. A **512-byte header** containing: filename (100 bytes), permissions, UID/GID, size, mtime, checksum, type flag, link name, and extended info (USTAR format)
2. The **file data** padded to a multiple of 512 bytes

**No central directory** — unlike ZIP, tar has no index at the end. To find a file, you must read from the beginning. This is why listing (`-t`) and extracting specific files from large archives is slow compared to ZIP.

**Compression pipeline:**
```
tar -czf archive.tar.gz files/
# Internally:
tar -c files/ | gzip > archive.tar.gz

tar -xzf archive.tar.gz
# Internally:
gzip -d < archive.tar.gz | tar -x
```

**Exit codes:**
| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Some files differ (with `--compare`) |
| `2` | Fatal error |

---

## Syntax — Old vs New Style

tar has **two syntax styles** that confuse beginners:

### Old style (traditional, no dash)
```bash
tar cvf archive.tar files/     # no leading dash
tar xvf archive.tar
tar tvf archive.tar
```

### New style (POSIX, with dash)
```bash
tar -cvf archive.tar files/
tar -xvf archive.tar
tar -tvf archive.tar
```

### Long options (GNU tar)
```bash
tar --create --verbose --file=archive.tar files/
tar --extract --verbose --file=archive.tar
tar --list --verbose --file=archive.tar
```

All three styles are equivalent. **Old style** (no dash) is most common in documentation and scripts. **Long options** are most readable.

---

## Core Operations

The **operation flag** is mandatory and must come first. Only one operation per command.

| Flag | Long | Action |
|------|------|--------|
| `-c` | `--create` | Create new archive |
| `-x` | `--extract` | Extract files from archive |
| `-t` | `--list` | List contents of archive |
| `-r` | `--append` | Append files to end of archive |
| `-u` | `--update` | Append only newer files |
| `-A` | `--concatenate` | Append another archive |
| `--delete` | | Delete from archive (not on tape) |
| `-d` | `--diff` / `--compare` | Compare archive with filesystem |

```bash
# CREATE
tar -cvf archive.tar dir/              # create, verbose, filename
tar -czf archive.tar.gz dir/           # + gzip compression
tar -cjf archive.tar.bz2 dir/          # + bzip2 compression
tar -cJf archive.tar.xz dir/           # + xz compression
tar --zstd -cf archive.tar.zst dir/    # + zstandard

# EXTRACT
tar -xvf archive.tar                   # extract all, verbose
tar -xvf archive.tar -C /target/       # extract to specific directory
tar -xvf archive.tar file.txt          # extract specific file only
tar -xvzf archive.tar.gz              # extract gzip archive

# LIST
tar -tvf archive.tar                   # list contents, verbose (ls-style)
tar -tf archive.tar                    # list without details

# APPEND (uncompressed archives only)
tar -rvf archive.tar newfile.txt       # append file
tar -uvf archive.tar dir/             # append if newer

# COMPARE
tar -dvf archive.tar                   # compare archive vs filesystem
```

---

## Compression Options

| Flag | Long | Tool | Extension | Speed | Ratio |
|------|------|------|-----------|-------|-------|
| `-z` | `--gzip` | gzip | `.tar.gz` / `.tgz` | Fast | Good |
| `-j` | `--bzip2` | bzip2 | `.tar.bz2` / `.tbz2` | Slow | Better |
| `-J` | `--xz` | xz | `.tar.xz` / `.txz` | Very slow | Best |
| `--zstd` | `--zstd` | zstd | `.tar.zst` | Fast | Very good |
| `-Z` | `--compress` | compress | `.tar.Z` | Fast | Poor (legacy) |
| `-a` | `--auto-compress` | auto-detect from extension | any | — | — |

```bash
# Auto-detect compression from filename extension
tar -caf archive.tar.gz dir/     # -a: uses gzip because of .gz
tar -caf archive.tar.xz dir/    # -a: uses xz because of .xz
tar -caf archive.tar.zst dir/   # -a: uses zstd because of .zst

# Speed vs ratio comparison (same data):
tar -czf  out.tar.gz  dir/   # fast, moderate compression
tar -cjf  out.tar.bz2 dir/   # 2-3x slower, ~10% better ratio
tar -cJf  out.tar.xz  dir/   # 5-10x slower, ~20% better ratio
tar --zstd -cf out.tar.zst dir/ # similar to gzip speed, near-xz ratio

# Multi-threaded compression (pigz, pbzip2, pixz):
tar -c dir/ | pigz -p 4 > archive.tar.gz    # gzip with 4 threads
tar -c dir/ | pbzip2 -p4 > archive.tar.bz2  # bzip2 with 4 threads
tar -c dir/ | xz -T 4 > archive.tar.xz      # xz with 4 threads
```

---

## All Key Options

### File Selection

| Option | Description |
|--------|-------------|
| `-f file` | Archive filename (required; use `-` for stdin/stdout) |
| `-C dir` | Change to directory before operation |
| `--exclude=PATTERN` | Exclude files matching shell pattern |
| `--exclude-from=FILE` | Read exclude patterns from file |
| `--exclude-vcs` | Exclude VCS dirs (`.git`, `.svn`, `.hg`, `.bzr`) |
| `--exclude-vcs-ignores` | Exclude files listed in VCS ignore files |
| `-X file` | Exclude patterns from file (same as `--exclude-from`) |
| `--include=PATTERN` | Include only matching files |
| `-T file` | Read list of files to archive from file |
| `--files-from=FILE` | Same as `-T` |
| `--no-recursion` | Don't recurse into directories |
| `--recursion` | Recurse into directories (default) |
| `-h` / `--dereference` | Follow symlinks (archive target, not link) |
| `--hard-dereference` | Follow hard links |

### Permissions & Ownership

| Option | Description |
|--------|-------------|
| `-p` / `--preserve-permissions` | Preserve all permissions (default for root) |
| `--no-same-permissions` | Apply umask to permissions |
| `--same-owner` | Preserve ownership (default for root) |
| `--no-same-owner` | Don't restore ownership |
| `--numeric-owner` | Use numeric UID/GID instead of names |
| `--strip-components=N` | Strip N leading path components on extract |

### Output & Verbosity

| Option | Description |
|--------|-------------|
| `-v` | Verbose: list files processed |
| `-vv` | Very verbose: like `ls -l` output |
| `-w` | Ask for confirmation for each file |
| `-k` | Don't overwrite existing files on extract |
| `--overwrite` | Overwrite existing files (default) |
| `--keep-newer-files` | Don't replace files newer than archive copy |
| `--skip-old-files` | Skip files already on filesystem |

### Timestamps & Special Files

| Option | Description |
|--------|-------------|
| `-m` | Don't restore modification times |
| `--mtime=DATE` | Set modification time to DATE |
| `--delay-directory-restore` | Defer directory permissions until end |
| `--sparse` | Handle sparse files efficiently |
| `--sparse-version=M.N` | Set sparse format version |

### Tape & Splitting

| Option | Description |
|--------|-------------|
| `-L size` | Tape length (size in blocks) |
| `-M` | Multi-volume archive |
| `--tape-length=N` | Tape length in 1024-byte blocks |
| `--new-volume-script=CMD` | Run CMD at end of each volume |

### Format Selection

| Option | Description |
|--------|-------------|
| `--format=FORMAT` | Archive format: `gnu`, `oldgnu`, `posix`, `ustar`, `v7` |
| `-H FORMAT` | Same as `--format` |
| `--posix` | POSIX (pax) format |

---

## Archive Format Internals

### USTAR (default, POSIX)
- Filename limit: **100 bytes** (hard limit)
- Max file size: **8 GiB**
- UID/GID limit: 21-bit (max 2097151)
- Supports: symlinks, hard links, device files, directories

### GNU tar format (gnu)
- Filename limit: **unlimited** (uses extension headers)
- Max file size: **unlimited**
- Supports: long names, long symlinks, sparse files, incremental backups

### PAX / POSIX.1-2001 (posix)
- Most modern, most portable
- Filename limit: **unlimited**
- Supports: extended headers, sub-second timestamps, large UIDs

```bash
# Check what format an archive uses
tar --list --verbose --verbose -f archive.tar | head -1
# or:
file archive.tar
# archive.tar: POSIX tar archive (GNU)
```

### The 100-character filename trap
```bash
# USTAR has 100-char filename limit (including path)
# GNU tar handles this automatically with extensions
# But some tools (old tar, Python's tarfile with format=USTAR) will truncate!

# Check if any filenames are long:
find . -type f | awk 'length > 100'

# Use --format=posix for maximum compatibility with long names:
tar --format=posix -czf archive.tar.gz dir/
```

---

## tar vs zip vs 7z vs cpio

| Feature | tar (+gz/xz) | zip | 7z | cpio |
|---------|-------------|-----|----|------|
| Unix permissions | ✅ Full | Limited | Limited | ✅ Full |
| Symlinks | ✅ | Limited | Limited | ✅ |
| Hard links | ✅ | ❌ | ❌ | ✅ |
| Special files | ✅ | ❌ | ❌ | ✅ |
| Streaming | ✅ | ❌ | ❌ | ✅ |
| Random access | ❌ | ✅ | ✅ | ❌ |
| Per-file compression | ❌ | ✅ | ✅ | ❌ |
| Encryption | ❌ native | Limited | ✅ AES-256 | ❌ |
| Windows support | Via tools | ✅ Native | ✅ | ❌ |
| Self-extracting | ❌ | ✅ | ✅ | ❌ |
| Split volumes | ✅ | ✅ | ✅ | ✅ |
| Standard on Linux | ✅ | ✅ | Optional | ✅ |

**Rule:** Use `tar` for Unix-to-Unix transfers (preserves everything). Use `zip` for cross-platform or random access. Use `7z` for maximum compression or encryption.

---

## Related Commands

| Command | Relation |
|---------|---------|
| `gzip` / `gunzip` | Compress/decompress `.gz` files |
| `bzip2` / `bunzip2` | Compress/decompress `.bz2` files |
| `xz` / `unxz` | Compress/decompress `.xz` files |
| `zstd` / `unzstd` | Compress/decompress `.zst` files |
| `zip` / `unzip` | Cross-platform archive format |
| `7z` | High-compression archive with encryption |
| `cpio` | Alternative streaming archive (used by rpm, initramfs) |
| `ar` | Archive for static libraries (`.a` files) |
| `find` | Select files to archive with `-T` |
| `rsync` | Network-aware archive/sync (uses tar internally for `--no-whole-file`) |
| `pigz` | Parallel gzip for faster compression |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
