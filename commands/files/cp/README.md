# cp — The Complete Reference

> **Copy files and directories**
> A foundational command everyone learns early — with a handful of
> flags (`-r`, `-p`, `-a`, `-n`) that quietly matter far more than
> they first appear to.

---

## Table of Contents

- [What is cp?](#what-is-cp)
- [Where does cp live?](#where-does-cp-live)
- [How cp works internally](#how-cp-works-internally)
- [Syntax](#syntax)
- [Understanding the Output](#understanding-the-output)
- [Copying Directories](#copying-directories)
- [Preserving Attributes](#preserving-attributes)
- [All Key Options](#all-key-options)
- [Overwrite Behavior](#overwrite-behavior)
- [cp vs rsync vs install vs dd](#cp-vs-rsync-vs-install-vs-dd)
- [Related Commands](#related-commands)

---

## What is cp?

`cp` copies files and, with the right flag, entire directory trees, creating an independent duplicate at the destination — changes to the copy afterward have no effect on the original, and vice versa.

```bash
cp report.txt report_backup.txt
# report.txt is now duplicated as report_backup.txt — two fully
# independent files with identical content
```

**Why cp's simplicity is deceptive:** the plain, no-flags version genuinely is that simple, but real usage almost always involves at least one modifier — recursion for directories, attribute preservation for backups, or overwrite protection for anything touching existing data — and getting these wrong is a common, quiet source of lost permissions, lost timestamps, or overwritten files.

---

## Where does cp live?

```
/usr/bin/cp
```

```bash
which cp
cp --version
# cp (GNU coreutils) 9.4
```

Part of **GNU coreutils** on most Linux distributions, and present in essentially identical core form on macOS, BSD, and virtually every Unix-like system — one of the most universally available and portable commands, though some GNU-specific flags (like `--reflink`) aren't present everywhere.

---

## How cp works internally

For a straightforward file copy, `cp` opens the source file for reading, creates (or truncates) the destination file, and reads/writes the content across in chunks until complete — conceptually a loop of `read()`/`write()` calls, though modern implementations often use more efficient mechanisms like `copy_file_range(2)` when the underlying filesystem supports it.

```bash
strace -e trace=openat,read,write cp source.txt dest.txt
# openat(AT_FDCWD, "source.txt", O_RDONLY) = 3
# openat(AT_FDCWD, "dest.txt", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 4
# read(3, ..., 131072) = ...
# write(4, ..., ...) = ...
```

On filesystems supporting **copy-on-write** (Btrfs, XFS with reflink support, APFS on macOS), `cp --reflink=auto` can create a near-instant "copy" that initially shares the same underlying data blocks as the original, only actually duplicating storage for blocks that are later modified in either copy — a significant efficiency gain over always physically duplicating every byte immediately.

---

## Syntax

```bash
cp [OPTIONS] SOURCE DEST
cp [OPTIONS] SOURCE... DIRECTORY
```

The second form copies multiple sources into an existing target directory, preserving each source's original filename.

---

## Understanding the Output

```bash
cp file.txt copy.txt
# (no output at all on success)

cp file.txt existing_directory/
# (no output — file.txt is copied INTO the directory, keeping its name)

cp -v file.txt copy.txt
# 'file.txt' -> 'copy.txt'

cp nonexistent.txt copy.txt
# cp: cannot stat 'nonexistent.txt': No such file or directory
```

Like most Unix commands, `cp` follows the "silence means success" convention — nothing printed when it works, a clear error (and non-zero exit status) when it fails.

---

## Copying Directories

```bash
cp project/ backup/
# cp: -r not specified; omitting directory 'project/'
# ⚠️ Plain cp refuses to copy a directory's contents at all without -r

cp -r project/ backup/
# Recursively copies project/ and everything inside it into backup/
```

The presence or absence of a trailing slash, and whether the destination already exists, both affect the exact resulting structure — see [edge-cases.md](edge-cases.md) for the details that trip people up here.

---

## Preserving Attributes

By default, a copied file gets **new** timestamps (the copy's creation time) and permissions derived from the current process's umask — **not** an exact clone of the original's metadata, unless explicitly told to preserve it:

```bash
cp -p original.txt copy.txt
# -p preserves modification/access times, ownership (if permitted),
# and permission mode from the original — the copy's metadata now
# matches the source's, rather than reflecting "just now" and the
# current umask

cp -a project/ backup/
# -a ("archive") is effectively -dR --preserve=all — recursive, with
# ALL attributes preserved (timestamps, ownership, permissions,
# and symlinks kept as symlinks rather than followed) — the standard
# choice for a genuinely faithful backup/clone of a directory tree
```

---

## All Key Options

| Option | Long form | Description |
|---|---|---|
| `-r` / `-R` | `--recursive` | Copy directories and their contents recursively |
| `-p` | `--preserve` | Preserve timestamps, ownership, and permissions |
| `-a` | `--archive` | Full archive mode: recursive + all attributes preserved + symlinks kept as symlinks |
| `-i` | `--interactive` | Prompt before overwriting an existing destination file |
| `-n` | `--no-clobber` | Never overwrite an existing destination file (silently skip instead) |
| `-u` | `--update` | Only copy if the source is newer than the destination (or destination doesn't exist) |
| `-v` | `--verbose` | Print each file as it's copied |
| `-f` | `--force` | Remove an existing destination first if it can't be opened for writing directly |
| `-l` | `--link` | Create a hard link instead of actually copying the data |
| `-s` | `--symbolic-link` | Create a symbolic link instead of copying |
| `--reflink=auto` | | Use a copy-on-write reflink where the filesystem supports it, for a near-instant, space-efficient copy |

---

## Overwrite Behavior

```bash
cp new_version.txt existing_file.txt
# ⚠️ SILENTLY overwrites existing_file.txt with no warning or
# confirmation at all — this is cp's DEFAULT behavior

cp -i new_version.txt existing_file.txt
# cp: overwrite 'existing_file.txt'? y
# -i prompts before overwriting, giving a chance to back out

cp -n new_version.txt existing_file.txt
# Silently does NOTHING if existing_file.txt already exists —
# no error, no overwrite, no prompt; the copy is simply skipped
```

---

## cp vs rsync vs install vs dd

| Tool | Best for | Key difference from cp |
|---|---|---|
| `cp` | A quick, one-off copy of a known file or directory | Simple, always re-copies the full content on every run |
| `rsync` | Repeated syncs, large directory trees, resumable/efficient transfers | Only copies the actual DIFFERENCES on repeated runs; can also operate across the network, which cp cannot |
| `install` | Copying a file into place with specific ownership/permissions as one atomic-ish step (common in packaging/build scripts) | Combines copying with explicit ownership/mode setting in a single command |
| `dd` | Low-level, byte-for-byte copying, often of raw block devices or disk images | Operates below the filesystem abstraction cp works at; used for imaging, not everyday file copying |

```bash
cp file.txt backup/                          # quick, everyday copy
rsync -a ./project/ backup-server:/backups/    # efficient sync, works across a network too
install -m 755 script.sh /usr/local/bin/        # copy + set permissions in one step
dd if=/dev/sda of=disk_image.img bs=4M          # raw, low-level block copy
```

---

## Related Commands

| Command | Relation |
|---|---|
| `mv` | Moves (renames) a file instead of duplicating it — the source no longer exists afterward |
| `rsync` | The more efficient choice for repeated syncs, large trees, or network transfers |
| `install` | Copy combined with explicit ownership/permission setting, common in install scripts |
| `scp` | Network-aware copy over SSH, conceptually similar to cp but between hosts |
| `ln` | Create a hard or symbolic link instead of an independent physical copy |
| `dd` | Low-level, block-device-aware copying |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
