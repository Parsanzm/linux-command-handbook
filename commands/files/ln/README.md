# ln — The Complete Reference

> **Create links between files**
> Two genuinely different mechanisms — hard links and symbolic links —
> share one command, and confusing the two is the source of nearly
> every ln-related surprise.

---

## Table of Contents

- [What is ln?](#what-is-ln)
- [Where does ln live?](#where-does-ln-live)
- [How ln works internally](#how-ln-works-internally)
- [Syntax](#syntax)
- [Understanding the Output](#understanding-the-output)
- [Hard Links vs Symbolic Links](#hard-links-vs-symbolic-links)
- [Creating Each Type](#creating-each-type)
- [All Key Options](#all-key-options)
- [Inspecting Existing Links](#inspecting-existing-links)
- [ln vs cp vs mount --bind](#ln-vs-cp-vs-mount---bind)
- [Related Commands](#related-commands)

---

## What is ln?

`ln` creates a **link** — an additional way to reference a file — without duplicating its actual data. It supports two fundamentally different mechanisms: **hard links**, which are essentially a second name for the exact same underlying data, and **symbolic links** (symlinks), which are a separate, small file that simply points to another path by name.

```bash
ln original.txt hardlink.txt
# hardlink.txt is now ANOTHER NAME for the exact same underlying data

ln -s original.txt symlink.txt
# symlink.txt is a NEW, SEPARATE file that just contains the TEXT
# "original.txt" as a pointer to follow
```

**Why the distinction matters so much:** a hard link and its "original" are genuinely indistinguishable from each other after creation — there's no way to tell which one was created first, and removing either one leaves the data fully intact through the other. A symlink, by contrast, is a fundamentally different, separate object that can break, point to nothing, or be told apart from its target trivially.

---

## Where does ln live?

```
/usr/bin/ln
```

```bash
which ln
ln --version
# ln (GNU coreutils) 9.4
```

Part of **GNU coreutils** on most Linux distributions, and present in essentially identical core form on macOS, BSD, and virtually every Unix-like system.

---

## How ln works internally

A **hard link** is created via the `link(2)` system call, which adds a new directory entry pointing to the **same inode** (the underlying data structure holding a file's actual content and metadata) as an existing file — both names are now equally valid, equally "real" references to that one inode.

```c
int link(const char *oldpath, const char *newpath);
```

A **symbolic link** is created via `symlink(2)`, which creates a genuinely new, small file — a distinct inode of its own — whose content is simply the **text of a path string**. When the kernel encounters a symlink while resolving a path, it transparently substitutes that stored path and continues resolution from there.

```c
int symlink(const char *target, const char *linkpath);
```

```bash
ls -i original.txt hardlink.txt
# 1234567 original.txt
# 1234567 hardlink.txt    ← IDENTICAL inode number — same underlying file

ls -i original.txt symlink.txt
# 1234567 original.txt
# 7654321 symlink.txt      ← DIFFERENT inode number — a separate file
# that merely references the first one's PATH
```

---

## Syntax

```bash
ln [OPTIONS] TARGET LINK_NAME
ln [OPTIONS] TARGET... DIRECTORY
```

By default (no `-s`), `ln` creates a **hard link**. `-s` is required for a symbolic link — this default is a frequent point of confusion, since symlinks are far more commonly used in everyday practice.

---

## Understanding the Output

```bash
ln original.txt hardlink.txt
# (no output at all on success)

ln -s original.txt symlink.txt
# (no output)

ls -l symlink.txt
# lrwxrwxrwx 1 alice alice 12 Aug 11 14:32 symlink.txt -> original.txt
# ⚠️ Note the leading "l" and the "-> original.txt" — these are how
# `ls -l` visibly distinguishes a symlink from a regular file or hard link

ls -l hardlink.txt
# -rw-r--r-- 1 alice alice 1024 Aug 11 14:32 hardlink.txt
# ⚠️ A hard link shows NO special indicator at all — it looks
# completely identical to any other regular file, because it IS one
```

---

## Hard Links vs Symbolic Links

| Property | Hard link | Symbolic link |
|---|---|---|
| What it actually is | Another directory entry for the SAME inode | A separate file containing a PATH STRING |
| Works across filesystems? | No — must be on the same filesystem as the target | Yes — can point anywhere, even a different filesystem |
| Can link to a directory? | No (with rare, discouraged exceptions requiring root) | Yes |
| Can point to something that doesn't exist? | No — impossible; it IS the data | Yes — a "broken" or "dangling" symlink is entirely valid to create |
| Distinguishable from the original? | No — completely indistinguishable, equally "real" | Yes — clearly a separate, small pointer file |
| Effect of removing the "original" | None — the data persists via the remaining hard link(s) | The symlink itself remains, but now points at nothing (broken) |
| Size shown by `ls -l` | The actual file's full size | The length of the target PATH STRING, not the target's size |

---

## Creating Each Type

```bash
# Hard link (the DEFAULT, no flag needed)
ln important_data.db important_data_backup.db

# Symbolic link (requires -s explicitly)
ln -s /opt/myapp/releases/v2.3.0 /opt/myapp/current

# Symlink to a directory
ln -s /mnt/large_disk/shared_data ~/data

# Force-overwrite an existing link/file at the destination
ln -sf /opt/myapp/releases/v2.4.0 /opt/myapp/current
```

---

## All Key Options

| Option | Long form | Description |
|---|---|---|
| `-s` | `--symbolic` | Create a symbolic link instead of the default hard link |
| `-f` | `--force` | Remove an existing destination file first, if needed, rather than erroring |
| `-i` | `--interactive` | Prompt before overwriting an existing destination |
| `-n` | `--no-dereference` | When the destination is itself a symlink to a directory, treat it as the file to replace, not as a directory to link INTO |
| `-v` | `--verbose` | Print each link as it's created |
| `-r` | `--relative` | Create a symlink using a relative path to the target, computed automatically |
| `-T` | `--no-target-directory` | Always treat LINK_NAME as the literal link name, never as an existing directory to link into |

---

## Inspecting Existing Links

```bash
# See a symlink's target directly
ls -l symlink.txt
# lrwxrwxrwx ... symlink.txt -> original.txt

# Resolve a symlink (or chain of symlinks) to its final real path
readlink -f symlink.txt
# /home/alice/original.txt

# Check how many hard links point to the same underlying data
ls -l hardlink.txt
# -rw-r--r-- 2 alice alice 1024 ... hardlink.txt
#           ↑ this "2" is the LINK COUNT — how many directory
# entries currently reference this exact inode

stat hardlink.txt
# shows the Inode number and Links count explicitly
```

---

## ln vs cp vs mount --bind

| Tool | Best for | Key difference from ln |
|---|---|---|
| `ln` | A lightweight reference/alias without duplicating data | Instant, near-zero extra disk space; changes to one hard link's content are visible through all others |
| `cp` | A genuinely independent duplicate | Actually duplicates the data — takes real disk space and time; the copy and original are fully independent afterward |
| `mount --bind` | Making an entire directory tree appear at another path | A filesystem-level mechanism, not a per-file link — requires root, and typically used for whole directories, not individual files |

```bash
ln -s /data/shared ~/shared          # lightweight symlink, common everyday use
cp -r /data/shared ~/shared_copy       # genuine independent duplicate, uses real extra space
sudo mount --bind /data/shared /mnt/shared_here   # filesystem-level directory mirroring
```

---

## Related Commands

| Command | Relation |
|---|---|
| `readlink` | Resolve a symlink to the actual path it points to |
| `stat` | Inspect a file's inode number and hard link count directly |
| `cp` | Creates an independent duplicate instead of a reference |
| `unlink` | Removes exactly one link (file or symlink) — the natural inverse of `ln` |
| `find -type l` | Locate symbolic links specifically within a directory tree |
| `find -xtype l` | Locate BROKEN symbolic links specifically (pointing to something that no longer exists) |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
