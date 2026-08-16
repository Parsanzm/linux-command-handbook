# mv — The Complete Reference

> **Move or rename files and directories**
> Two operations that look completely different at the shell but are,
> under the hood, frequently the exact same underlying system call.

---

## Table of Contents

- [What is mv?](#what-is-mv)
- [Where does mv live?](#where-does-mv-live)
- [How mv works internally](#how-mv-works-internally)
- [Syntax](#syntax)
- [Understanding the Output](#understanding-the-output)
- [Renaming vs Moving — Why They're the Same Command](#renaming-vs-moving--why-theyre-the-same-command)
- [Moving Across Filesystems](#moving-across-filesystems)
- [All Key Options](#all-key-options)
- [Overwrite Behavior](#overwrite-behavior)
- [mv vs cp + rm vs rsync --remove-source-files](#mv-vs-cp--rm-vs-rsync---remove-source-files)
- [Related Commands](#related-commands)

---

## What is mv?

`mv` relocates a file or directory to a new path — which, depending on that new path, can look like either "renaming" (same directory, new name) or "moving" (new directory, same or different name). At the shell level, both are simply `mv`; there's no separate "rename" command in standard Unix.

```bash
mv draft.txt final.txt
# "Renaming" — same location, new filename

mv report.pdf archive/report.pdf
# "Moving" — same filename, new location
```

**Why this dual behavior isn't a coincidence:** on a single filesystem, moving and renaming really are the identical underlying operation — updating where a directory entry points, without touching the file's actual data at all. See [How mv works internally](#how-mv-works-internally) for why this matters practically.

---

## Where does mv live?

```
/usr/bin/mv
```

```bash
which mv
mv --version
# mv (GNU coreutils) 9.4
```

Part of **GNU coreutils** on most Linux distributions, and present in essentially identical core form on macOS, BSD, and virtually every Unix-like system.

---

## How mv works internally

When source and destination are on the **same filesystem**, `mv` uses the `rename(2)` system call, which simply updates the directory entry (the filename-to-inode mapping) — the underlying file data never physically moves on disk at all, making same-filesystem moves and renames essentially **instantaneous**, regardless of the file's actual size.

```c
int rename(const char *oldpath, const char *newpath);
```

```bash
# A 50GB file "moved" within the same filesystem completes in
# milliseconds — because nothing is actually being copied
mv huge_file.iso /home/alice/huge_file.iso
```

When source and destination are on **different filesystems** (different disks, different mount points, or a network filesystem), `rename(2)` isn't possible — `mv` transparently falls back to copying the data to the new location and then deleting the original, which takes time proportional to the file's actual size, just like `cp` followed by `rm`.

```bash
# Moving to a DIFFERENT filesystem (e.g., an external USB drive)
# takes real time proportional to size, unlike a same-filesystem move
mv huge_file.iso /mnt/usb_drive/huge_file.iso
```

---

## Syntax

```bash
mv [OPTIONS] SOURCE DEST
mv [OPTIONS] SOURCE... DIRECTORY
```

The second form moves multiple sources into an existing target directory, preserving each source's original filename.

---

## Understanding the Output

```bash
mv old_name.txt new_name.txt
# (no output at all on success)

mv -v old_name.txt new_name.txt
# 'old_name.txt' -> 'new_name.txt'

mv nonexistent.txt somewhere.txt
# mv: cannot stat 'nonexistent.txt': No such file or directory
```

Like most Unix commands, `mv` follows the "silence means success" convention — nothing printed when it works, a clear error (and non-zero exit status) when it fails.

---

## Renaming vs Moving — Why They're the Same Command

```bash
# Same directory, different name → looks like a "rename"
mv report_draft.txt report_final.txt

# Different directory, same name → looks like a "move"
mv report_final.txt /home/alice/documents/report_final.txt

# Different directory AND different name, both at once → both
mv report_final.txt /home/alice/documents/quarterly_report.txt
```

There is genuinely no distinction at the syscall level between these three — `rename(2)` just updates a directory entry's path, whether that path change happens to affect only the filename, only the containing directory, or both simultaneously.

---

## Moving Across Filesystems

```bash
df -T . /mnt/external_drive
# Filesystem     Type
# /dev/sda1       ext4
# /dev/sdb1       ext4
# ⚠️ Even if both show the SAME filesystem TYPE, they can still be
# entirely SEPARATE filesystem instances (different mount points,
# different physical/logical devices) — rename(2) requires the SAME
# filesystem instance, not merely the same filesystem type, to work
# as an instant metadata-only operation.

mv large_file.iso /mnt/external_drive/
# If /mnt/external_drive is a genuinely different filesystem instance
# from the source's, this silently falls back to a full copy + delete
# — noticeably SLOWER than a same-filesystem move, though the command
# and its syntax look identical either way
```

---

## All Key Options

| Option | Long form | Description |
|---|---|---|
| `-i` | `--interactive` | Prompt before overwriting an existing destination file |
| `-n` | `--no-clobber` | Never overwrite an existing destination file |
| `-u` | `--update` | Only move if the source is newer than the destination (or destination doesn't exist) |
| `-v` | `--verbose` | Print each file as it's moved |
| `-f` | `--force` | Never prompt, overwrite without asking (the default behavior anyway, unless `-i` is otherwise implied) |
| `-b` | `--backup` | Make a backup of an existing destination file before overwriting it |
| `-t DIR` | `--target-directory=DIR` | Explicitly specify the target directory, useful for scripted/programmatic invocations |
| `-T` | `--no-target-directory` | Treat DEST as a normal file, never as a directory to move INTO (see edge-cases.md) |

---

## Overwrite Behavior

```bash
mv new_version.txt existing_file.txt
# ⚠️ SILENTLY overwrites existing_file.txt with no warning at all —
# this is mv's DEFAULT behavior

mv -i new_version.txt existing_file.txt
# mv: overwrite 'existing_file.txt'? y
# -i prompts before overwriting

mv -n new_version.txt existing_file.txt
# Silently does NOTHING if existing_file.txt already exists — no
# error, no move, the operation is simply skipped entirely
```

---

## mv vs cp + rm vs rsync --remove-source-files

| Approach | Best for | Key difference from mv |
|---|---|---|
| `mv` | Renaming, or relocating within/across filesystems | Single atomic operation on the same filesystem (via `rename(2)`); falls back to copy+delete across filesystems automatically |
| `cp` then `rm` | Rarely needed directly, since mv already handles both cases | Explicitly two separate steps — a failure between them can leave BOTH a copy and the original present, unlike mv's more atomic same-filesystem behavior |
| `rsync --remove-source-files` | Moving with resume capability, or moving while filtering/excluding specific files | Can resume an interrupted transfer and offers much finer control over what qualifies to be "moved," at the cost of more complexity than a simple mv |

```bash
mv file.txt newlocation/                                # standard move/rename
rsync -a --remove-source-files file.txt newlocation/      # resumable, filterable alternative for large/network transfers
```

---

## Related Commands

| Command | Relation |
|---|---|
| `cp` | Duplicates instead of relocating — the source remains in place afterward |
| `rename` (Perl-based, distinct from mv) | Batch-renames multiple files using pattern/regex substitution, unlike mv's one-at-a-time target |
| `rsync` | Better suited for resumable or filtered moves, especially across networks |
| `ln` | Creates a link instead of relocating — the file appears to exist in two places at once |
| `find -exec mv` | Batch-move files matching specific search criteria |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
