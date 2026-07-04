# zip / unzip — The Complete Reference

> **The cross-platform archive format everyone's computer already understands**
> Created by Phil Katz (PKWARE) in 1989 as an open alternative to ARC.
> The only common archive format natively readable on Windows, macOS, and Linux without extra software.

---

## Table of Contents

- [What are zip and unzip?](#what-are-zip-and-unzip)
- [Where do they live?](#where-do-they-live)
- [How the ZIP format works internally](#how-the-zip-format-works-internally)
- [zip — Syntax and Usage](#zip--syntax-and-usage)
- [unzip — Syntax and Usage](#unzip--syntax-and-usage)
- [Compression Levels](#compression-levels)
- [Password Protection & Encryption](#password-protection--encryption)
- [All Key zip Options](#all-key-zip-options)
- [All Key unzip Options](#all-key-unzip-options)
- [zip vs tar.gz vs 7z](#zip-vs-targz-vs-7z)
- [Working with Large Files (Zip64)](#working-with-large-files-zip64)
- [Related Commands](#related-commands)

---

## What are zip and unzip?

`zip` creates compressed archive files in the **ZIP format**, and `unzip` extracts them. Unlike `tar`, which is purely an archiver that's typically paired *separately* with a compressor (like `gzip`), ZIP is a single format that does **both** archiving (bundling multiple files/directories into one) and **compression** (shrinking file size) at once, **per-file** rather than for the whole archive as a single stream.

```bash
zip archive.zip file1.txt file2.txt          # create
zip -r archive.zip myfolder/                 # create, recursively
unzip archive.zip                            # extract
unzip -l archive.zip                         # list contents without extracting
```

**Why ZIP is still everywhere in 2026:** it's the *only* archive format with truly universal, built-in OS support — Windows Explorer, macOS Finder, and virtually every Linux file manager can open a `.zip` with zero additional software. This makes it the default choice whenever the recipient's environment is unknown or non-technical (email attachments, downloadable assets, cross-platform data exchange).

---

## Where do they live?

```
/usr/bin/zip
/usr/bin/unzip
```

```bash
which zip unzip
zip --version
unzip -v
```

On many minimal Linux installations, `zip`/`unzip` are **not installed by default** (unlike `tar`, which almost always is) and must be installed separately:

```bash
sudo apt install zip unzip        # Debian/Ubuntu
sudo dnf install zip unzip        # Fedora/RHEL
sudo pacman -S zip unzip          # Arch
brew install zip unzip            # macOS (usually already present)
```

Both are part of the **Info-ZIP** project, the de facto standard open-source implementation.

---

## How the ZIP format works internally

### Per-file compression (a key structural difference from tar.gz)

A ZIP archive stores each file **independently compressed**, with its own local file header, compressed data, and CRC-32 checksum:

```
[Local File Header][Compressed Data][Data Descriptor]   ← file 1
[Local File Header][Compressed Data][Data Descriptor]   ← file 2
...
[Central Directory]                                      ← index of all entries
[End of Central Directory Record]                         ← points to Central Directory
```

This has two major practical consequences compared to `tar.gz`:
1. **Random access** — you can extract, list, or verify a *single* file from a huge ZIP archive without reading through the entire archive first, because the Central Directory at the end tells `unzip` exactly where each file's data starts.
2. **Slightly worse compression ratio** — because each file is compressed in isolation, ZIP can't exploit redundancy *between* files the way `tar.gz` can (which compresses the whole concatenated stream together). Many small, similar files compress noticeably better as `tar.gz` than as `.zip`.

### The Central Directory

Located at the **end** of the file, the Central Directory is why tools like `unzip -l` can instantly list contents of even a multi-gigabyte archive — they seek to the end, read the Central Directory, and never need to scan the compressed data itself just to see filenames.

```bash
# This is instant even on a huge archive, because it only reads the
# Central Directory at the end of the file:
unzip -l huge_archive.zip
```

This also means a ZIP file with a **corrupted or missing Central Directory** (e.g., a download that was cut off) can be much harder to recover from than a similarly-truncated tar.gz, since standard tools expect to find that index at the end.

### Compression method per entry

Each file entry records its own compression method — most commonly **Deflate** (method 8), but entries can also be stored **uncompressed** (method 0, "stored"), which is used automatically for files that don't compress well (already-compressed formats like `.jpg`, `.mp4`, `.zip` itself) to avoid wasting CPU for no size benefit.

---

## zip — Syntax and Usage

```bash
zip [OPTIONS] ARCHIVE.zip FILE...
```

```bash
# Create a new archive from specific files
zip archive.zip file1.txt file2.txt

# Add a directory and everything inside it (recursive)
zip -r archive.zip myfolder/

# Add files to an EXISTING archive
zip archive.zip newfile.txt

# Update only files that are newer than what's already in the archive
zip -u archive.zip file1.txt

# Delete a specific file from an existing archive
zip -d archive.zip unwanted_file.txt

# Exclude specific patterns while creating
zip -r archive.zip myfolder/ -x "*.log" -x "*/.git/*"

# Move files INTO the archive, deleting the originals afterward
zip -rm archive.zip myfolder/
```

---

## unzip — Syntax and Usage

```bash
unzip [OPTIONS] ARCHIVE.zip [FILE...] [-d DESTINATION]
```

```bash
# Extract everything into the current directory
unzip archive.zip

# Extract into a specific destination directory
unzip archive.zip -d /path/to/destination

# List contents WITHOUT extracting
unzip -l archive.zip

# Extract only specific files/patterns
unzip archive.zip "docs/*.pdf"

# Extract, overwriting existing files without prompting
unzip -o archive.zip

# Extract, but NEVER overwrite existing files (skip conflicts silently)
unzip -n archive.zip

# Test archive integrity without extracting anything
unzip -t archive.zip

# Extract quietly (suppress the per-file listing normally printed)
unzip -q archive.zip

# View contents of a file INSIDE the archive without extracting it
unzip -p archive.zip path/inside/archive.txt
```

---

## Compression Levels

`zip` supports compression levels 0 (no compression, fastest) through 9 (maximum compression, slowest):

```bash
zip -0 archive.zip file.txt      # store only, no compression — fastest, largest
zip -1 archive.zip file.txt      # fastest compression
zip -6 archive.zip file.txt      # default balance (used if no level specified)
zip -9 archive.zip file.txt      # maximum compression — slowest, smallest

# Practical trade-off check:
time zip -1 fast.zip bigfile.dat
time zip -9 small.zip bigfile.dat
ls -lh fast.zip small.zip
```

`-0` is particularly useful when you just want to **bundle** files without spending CPU on compression — for example, when the files are already compressed (video, images) and further compression would waste time for negligible size gain.

---

## Password Protection & Encryption

```bash
# Create a password-protected archive (prompts interactively — recommended)
zip -e archive.zip file.txt

# Non-interactive password (⚠️ visible in shell history and process list — avoid in scripts)
zip -P "mypassword" archive.zip file.txt

# Extract a password-protected archive
unzip archive.zip
# [archive.zip] file.txt password:

# Non-interactive extraction with password
unzip -P "mypassword" archive.zip
```

**Important security caveat:** the traditional ZIP encryption (`ZipCrypto`, used by the classic `-e`/`-P` flags in standard `zip`) is **cryptographically weak** and can be broken relatively easily with known-plaintext attacks — it should **not** be relied on to protect genuinely sensitive data. For real security, use **AES-256 encryption**, available via the `-e` flag combined with a build of zip that supports it, or more reliably via the `7z` tool, or by encrypting the archive separately with GPG:

```bash
# Stronger alternative: encrypt with GPG after zipping
zip -r archive.zip myfolder/
gpg -c archive.zip          # prompts for a passphrase, produces archive.zip.gpg
gpg -d archive.zip.gpg > archive.zip   # decrypt later
```

---

## All Key zip Options

| Option | Description |
|--------|-------------|
| `-r` | Recurse into directories |
| `-u` | Update — add new files, replace changed ones, skip unchanged |
| `-d` | Delete specified entries from an existing archive |
| `-m` | Move — delete original files after adding them to the archive |
| `-x PATTERN` | Exclude files matching a pattern |
| `-i PATTERN` | Include only files matching a pattern |
| `-e` | Encrypt entries, prompting for a password |
| `-P PASSWORD` | Encrypt with a password given on the command line (insecure for scripts) |
| `-0` to `-9` | Compression level (0=store, 9=max) |
| `-q` | Quiet — suppress normal output |
| `-v` | Verbose |
| `-j` | Junk paths — store just the file names, discarding directory structure |
| `-y` | Store symlinks as symlinks, not their targets |
| `-sf` | Show files that would be operated on, without doing anything |
| `--exclude=GLOB` | GNU-style exclude pattern (alternative syntax to `-x`) |
| `-T` | Test the archive's integrity after creating it |
| `-A` | Adjust self-extracting archive |

---

## All Key unzip Options

| Option | Description |
|--------|-------------|
| `-l` | List contents without extracting |
| `-t` | Test archive integrity |
| `-d DIR` | Extract into a specific directory |
| `-o` | Overwrite existing files without prompting |
| `-n` | Never overwrite existing files |
| `-q` | Quiet mode |
| `-v` | Verbose listing (more detail than `-l`) |
| `-p` | Extract a file to stdout (pipe-friendly) |
| `-x PATTERN` | Exclude specific files from extraction |
| `-P PASSWORD` | Supply a password non-interactively |
| `-j` | Junk paths — extract files flat, ignoring stored directory structure |
| `-a` | Auto-convert text files' line endings for the current platform |
| `-C` | Case-insensitive filename matching |
| `-L` | Convert extracted filenames to lowercase |
| `-Z` | Invoke `zipinfo`-style listing (used internally by `-l`/`-v`) |

---

## zip vs tar.gz vs 7z

| Feature | `.zip` | `.tar.gz` | `.7z` |
|---------|--------|-----------|-------|
| Universal OS support (no extra tools) | ✅ Windows/macOS/Linux all native | ⚠️ Native on Linux/macOS, needs extra tool on Windows | ❌ Needs 7-Zip installed everywhere |
| Compression ratio | Moderate (per-file) | Better (whole-stream) | Best (LZMA2) |
| Random access to single files | ✅ Fast (Central Directory) | ❌ Must read sequentially from start | ✅ Fast |
| Preserves Unix permissions/symlinks reliably | ⚠️ Partial, inconsistent across tools | ✅ Full fidelity | ✅ Full fidelity |
| Strong built-in encryption | ⚠️ Weak by default (ZipCrypto); AES possible but inconsistent support | ❌ None (needs external gpg/openssl) | ✅ AES-256 by default |
| Best use case | Cross-platform sharing, email attachments, general compatibility | Unix/Linux backups, preserving exact permissions/ownership | Maximum compression, archival storage |

```bash
# Same content, different formats — compare sizes:
zip -r archive.zip myfolder/
tar -czf archive.tar.gz myfolder/
7z a archive.7z myfolder/
ls -lh archive.zip archive.tar.gz archive.7z
```

**Rule of thumb:** choose `.zip` when the recipient's environment is unknown or non-technical; choose `.tar.gz` for Unix-to-Unix transfers where exact permissions/ownership matter; choose `.7z` when squeezing out the smallest possible size matters more than universal compatibility.

---

## Working with Large Files (Zip64)

The original ZIP format has hard limits: **4 GB** per file and **65,535** entries per archive, and a maximum archive size of 4 GB, due to 32-bit fields in the original 1989 specification. **Zip64** is an extension that lifts these limits using 64-bit fields, and is used automatically by modern `zip`/`unzip` when needed:

```bash
# Archiving something over 4GB automatically triggers Zip64 extensions
zip -r bigarchive.zip huge_file_5gb.dat
# No special flag needed on modern Info-ZIP — it's automatic

# Force Zip64 format even for small archives (rarely necessary)
zip -fz archive.zip file.txt
```

**Compatibility caveat:** very old ZIP tools (or minimal/embedded unzip implementations) may not understand Zip64 extensions and could fail on archives that exceed the original limits — worth checking when the recipient's tooling is unknown or aged.

---

## Related Commands

| Command | Relation |
|---------|----------|
| `tar` | Alternative archiver, typically paired with gzip/xz; better for Unix permission fidelity |
| `gzip` / `gunzip` | Single-file compression, often used alongside tar instead of zip |
| `7z` | More modern format with stronger compression and encryption |
| `zipinfo` | Detailed listing of a ZIP archive's contents (what `unzip -v` calls internally) |
| `zipcloak` | Add/remove encryption on an existing (unencrypted) ZIP archive |
| `zipsplit` | Split a large ZIP archive into smaller volume pieces |
| `gpg` | Recommended for real encryption, layered on top of a zip archive |
| `file` | Identify whether a given file is actually a valid ZIP archive |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
