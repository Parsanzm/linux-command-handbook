# rm — The Complete Reference

> **Remove files and directories**
> One of the most powerful — and most dangerous — commands in daily use.
> There is no built-in undo, no recycle bin, and no confirmation by default.

---

## Table of Contents

- [What is rm?](#what-is-rm)
- [Where does rm live?](#where-does-rm-live)
- [How rm works internally](#how-rm-works-internally)
- [Syntax](#syntax)
- [Understanding the Output](#understanding-the-output)
- [Removing Directories](#removing-directories)
- [Interactive and Safety-Oriented Usage](#interactive-and-safety-oriented-usage)
- [All Key Options](#all-key-options)
- [Why Deleted Files Aren't Always Actually Gone](#why-deleted-files-arent-always-actually-gone)
- [rm vs unlink vs shred vs trash-cli](#rm-vs-unlink-vs-shred-vs-trash-cli)
- [Related Commands](#related-commands)

---

## What is rm?

`rm` ("remove") deletes files, and — with the right flag — entire directory trees. Unlike deleting a file through a desktop file manager, there is **no recycle bin, no confirmation prompt by default, and no built-in undo**. Once `rm` succeeds, recovering the data (if possible at all) requires specialized forensic tools working directly against the underlying disk, not `rm` itself.

```bash
rm oldfile.txt
# (no output, and no confirmation asked — the file is simply gone)
```

**Why rm demands more caution than most commands:** it's fast, silent on success, and irreversible through any built-in mechanism — a single mistyped path, an unintended wildcard match, or a misplaced flag can destroy far more than intended, permanently, in under a second.

---

## Where does rm live?

```
/usr/bin/rm
```

```bash
which rm
rm --version
# rm (GNU coreutils) 9.4
```

Part of **GNU coreutils** on most Linux distributions, and present in essentially identical core form on macOS, BSD, and virtually every Unix-like system — a foundational, universally available command.

---

## How rm works internally

For a regular file, `rm` calls the `unlink(2)` system call, which removes the filename's **directory entry** pointing to the underlying data — it does not necessarily erase the actual data on disk immediately (see [Why Deleted Files Aren't Always Actually Gone](#why-deleted-files-arent-always-actually-gone)).

```c
int unlink(const char *pathname);
```

A file's underlying data blocks are only truly freed for reuse when its **link count** drops to zero **and** no process still has it open — a file can have multiple hard links (multiple directory entries pointing to the same underlying data), and `rm`/`unlink` only removes one such link at a time.

```bash
# For directories, rm -r recursively unlinks every file inside first,
# then removes the (now-empty) directory entries themselves, walking
# depth-first through the tree
strace -e trace=unlink,unlinkat,rmdir rm -r somedir/
```

---

## Syntax

```bash
rm [OPTIONS] FILE...
```

Multiple files (or, with `-r`, directories) can be listed in a single invocation, each removed independently.

---

## Understanding the Output

```bash
rm file.txt
# (no output at all on success)

rm nonexistent.txt
# rm: cannot remove 'nonexistent.txt': No such file or directory

rm somedir
# rm: cannot remove 'somedir': Is a directory
# ⚠️ Plain rm refuses to remove a directory at all, without -r or -d
```

`rm` follows the standard "silence means success" Unix convention — nothing printed when it works, and a clear error (with non-zero exit status) when it can't.

---

## Removing Directories

```bash
rm somedir
# rm: cannot remove 'somedir': Is a directory

rmdir somedir
# Works, but ONLY if somedir is completely EMPTY

rm -d somedir
# Also works for an EMPTY directory specifically — a less common
# alternative to rmdir for the same narrow case

rm -r somedir
# Recursively removes the directory AND everything inside it,
# regardless of whether it's empty — the standard way to delete a
# non-empty directory tree

rm -rf somedir
# Same as -r, but ALSO suppresses confirmation prompts and ignores
# nonexistent-file errors entirely — the single most consequential
# and most-feared flag combination in this entire command
```

---

## Interactive and Safety-Oriented Usage

```bash
# Prompt before every single removal
rm -i file1.txt file2.txt
# rm: remove regular file 'file1.txt'? y

# Prompt only once for a recursive removal of 3+ files, rather than
# once per individual file inside it (a middle ground between -i and
# no confirmation at all)
rm -I -r somedir/

# Explain exactly what's being done, useful in scripts for auditing
rm -v file.txt
# removed 'file.txt'
```

Many experienced users configure a shell alias (`alias rm='rm -i'`) specifically because `rm`'s default behavior offers no safety net whatsoever — see [edge-cases.md](edge-cases.md) for why even this common precaution has real limitations.

---

## All Key Options

| Option | Long form | Description |
|---|---|---|
| `-r` / `-R` | `--recursive` | Remove directories and their contents recursively |
| `-f` | `--force` | Never prompt, and ignore nonexistent files/arguments without error |
| `-i` | | Prompt before every individual removal |
| `-I` | | Prompt once before a recursive removal of 3+ files (less intrusive than `-i`) |
| `-d` | `--dir` | Remove empty directories specifically (like `rmdir`, without needing `-r`) |
| `-v` | `--verbose` | Print a message for each file actually removed |
| `--preserve-root` | | Refuse to operate recursively on `/` (this is the DEFAULT behavior on modern coreutils) |
| `--no-preserve-root` | | Explicitly ALLOW recursive operation on `/` — extremely dangerous, essentially never intentional |
| `--one-file-system` | | When recursing, skip directories on a different filesystem/mount than the starting point |

---

## Why Deleted Files Aren't Always Actually Gone

```bash
rm sensitive_file.txt
# ⚠️ This removes the DIRECTORY ENTRY (the filename), not necessarily
# the underlying DATA BLOCKS on disk. Until those blocks are
# overwritten by some future write operation, the actual content can
# often still be recovered using disk-forensics tools, especially on
# traditional filesystems like ext4 — rm provides NO cryptographic or
# forensic guarantee that data is unrecoverable.

# For genuinely sensitive data requiring irreversible destruction,
# a purpose-built secure-deletion tool is the appropriate choice
# instead of plain rm:
shred -u sensitive_file.txt
# -u also removes the file afterward, similar to rm, but first
# overwrites its content multiple times
```

Note that `shred`'s effectiveness itself is reduced on modern SSDs and copy-on-write/journaling filesystems, due to wear-leveling and how writes are actually handled at the hardware/filesystem level — see [edge-cases.md](edge-cases.md) for more on this important caveat.

---

## rm vs unlink vs shred vs trash-cli

| Tool | Best for | Key difference from rm |
|---|---|---|
| `rm` | General-purpose file/directory removal | The standard, most widely used tool for this |
| `unlink` | Removing exactly ONE file, nothing else | A simpler, more restricted tool — no recursive option, no wildcards, no directory support at all — sometimes preferred in scripts specifically to avoid ANY chance of accidentally matching multiple files |
| `shred` | Attempting to make a file's content genuinely unrecoverable | Overwrites data before removing it (though with real limitations on modern SSDs — see edge-cases.md) |
| `trash-cli` (or a desktop trash mechanism) | Recoverable, "soft" deletion with an undo option | Moves files to a trash/recycle location instead of immediately removing them — NOT a default part of most command-line environments |

```bash
rm file.txt                # standard removal, no undo
unlink file.txt              # removes exactly this one file, nothing else, ever
shred -u sensitive.txt        # attempt at secure, unrecoverable removal
trash-put file.txt            # (if installed) recoverable "soft" deletion instead
```

---

## Related Commands

| Command | Relation |
|---|---|
| `rmdir` | Remove an EMPTY directory specifically — a narrower, safer alternative to `rm -r` for that one case |
| `unlink` | Remove exactly one file, with no recursive/wildcard capability at all |
| `shred` | Attempt secure, harder-to-recover deletion by overwriting content first |
| `find -delete` | Remove files matching specific search criteria, often safer than a hand-built `rm` + wildcard combination |
| `trash-cli` | Recoverable, undo-able deletion, if installed |
| `mkdir` / `touch` | The natural creation-side counterparts to rm's removal |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
