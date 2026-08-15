# touch — The Complete Reference

> **Create an empty file, or update an existing file's timestamps**
> A small, simple command with two genuinely distinct uses — and the
> second one is why it's called "touch" in the first place.

---

## Table of Contents

- [What is touch?](#what-is-touch)
- [Where does touch live?](#where-does-touch-live)
- [How touch works internally](#how-touch-works-internally)
- [Syntax](#syntax)
- [Understanding the Output](#understanding-the-output)
- [Two Timestamps: Access Time vs Modification Time](#two-timestamps-access-time-vs-modification-time)
- [Setting a Specific Timestamp](#setting-a-specific-timestamp)
- [All Key Options](#all-key-options)
- [touch and File Creation](#touch-and-file-creation)
- [touch vs > vs install vs cp --no-clobber](#touch-vs--vs-install-vs-cp---no-clobber)
- [Related Commands](#related-commands)

---

## What is touch?

`touch` has two related but distinct effects, depending on whether the target file already exists:

1. If the file **doesn't exist**, `touch` creates it as a new, empty (zero-byte) file.
2. If the file **already exists**, `touch` updates its access and modification timestamps to the current time, **without changing its content at all**.

```bash
touch newfile.txt
# Creates newfile.txt, empty, if it doesn't already exist

touch existing_file.txt
# Updates existing_file.txt's timestamps to "now" — content untouched
```

**Why it's called "touch":** the core purpose was always about timestamps — "touching" a file to mark it as recently accessed/modified — and file creation is really just a side effect of that same operation applied to a file that doesn't yet exist.

---

## Where does touch live?

```
/usr/bin/touch
```

```bash
which touch
touch --version
# touch (GNU coreutils) 9.4
```

Part of **GNU coreutils** on most Linux distributions, and present in essentially identical core form on macOS, BSD, and virtually every Unix-like system — one of the more universally portable commands, though some newer GNU-specific flags aren't available on every variant.

---

## How touch works internally

For an existing file, `touch` calls the `utimensat(2)` (or the older `utime(2)`) system call, which lets a process directly set a file's access and modification timestamps without touching its actual content.

```c
int utimensat(int dirfd, const char *pathname,
              const struct timespec times[2], int flags);
```

For a **non-existent** file, `touch` first calls `open(2)` with the `O_CREAT` flag (create if missing) to bring the file into existence as an empty file, then applies the same timestamp-setting logic as above.

```bash
strace -e trace=openat,utimensat touch newfile.txt
# openat(AT_FDCWD, "newfile.txt", O_WRONLY|O_CREAT|O_NOCTTY|O_NONBLOCK, 0666) = 3
# utimensat(0, NULL, NULL, 0) = 0
```

---

## Syntax

```bash
touch [OPTIONS] FILE...
```

Multiple files can be given at once, each created (if missing) or timestamp-updated (if existing) independently in a single invocation.

---

## Understanding the Output

```bash
touch newfile.txt
# (no output at all on success)

ls -l newfile.txt
# -rw-r--r-- 1 alice alice 0 Aug 11 14:32 newfile.txt
# ⚠️ Size is exactly 0 bytes — touch never writes any content

touch /root/protected.txt
# touch: cannot touch '/root/protected.txt': Permission denied
```

Like most Unix commands, `touch` follows the "silence means success" convention — nothing is printed when it works, and a clear error appears (with a non-zero exit code) when it can't create the file or update its timestamps.

---

## Two Timestamps: Access Time vs Modification Time

Every file tracks (at minimum) these timestamps, both of which `touch` can update:

| Timestamp | Meaning | Shown by |
|---|---|---|
| Access time (`atime`) | When the file's contents were last read | `ls -lu` |
| Modification time (`mtime`) | When the file's contents were last changed | `ls -l` (the default) |
| Change time (`ctime`) | When the file's metadata (permissions, ownership, or content) was last changed — NOT settable by touch directly | `ls -lc` |

```bash
touch -a file.txt      # update ONLY the access time
touch -m file.txt       # update ONLY the modification time
touch file.txt            # update BOTH access and modification time (the default)
```

`ctime` is a kernel-maintained record of the last metadata change and **cannot** be set directly by `touch` or any other userspace tool — it's automatically updated as a side effect whenever `atime`/`mtime` themselves are changed, among other things.

---

## Setting a Specific Timestamp

```bash
# Set to a specific date and time
touch -t 202601151430.00 file.txt
# Format: [[CC]YY]MMDDhhmm[.ss]

# Set using a more readable date string
touch -d "2026-01-15 14:30:00" file.txt
touch -d "yesterday" file.txt
touch -d "3 days ago" file.txt
touch -d "next monday" file.txt

# Copy another file's timestamp instead of using "now"
touch -r reference_file.txt target_file.txt
# target_file.txt's timestamps now match reference_file.txt's exactly
```

---

## All Key Options

| Option | Long form | Description |
|---|---|---|
| `-a` | | Change only the access time |
| `-m` | | Change only the modification time |
| `-c` | `--no-create` | Do NOT create the file if it doesn't already exist |
| `-t STAMP` | | Set a specific timestamp, in `[[CC]YY]MMDDhhmm[.ss]` format |
| `-d STRING` | `--date=STRING` | Set a specific timestamp, parsed from a flexible human-readable string |
| `-r FILE` | `--reference=FILE` | Copy the timestamp from another existing file, instead of using "now" |
| `-h` | `--no-dereference` | Affect a symbolic link itself, rather than the file it points to |

---

## touch and File Creation

```bash
touch newfile.txt
ls -l newfile.txt
# -rw-r--r-- 1 alice alice 0 Aug 11 14:32 newfile.txt
```

The permissions on a newly created file follow the same default-minus-umask logic as `mkdir` for directories:

```bash
umask
# 0022
# Resulting file permissions: 0666 & ~0022 = 0644 (rw-r--r--)
# — note the DEFAULT REQUEST for a new file is 0666 (no execute bit
# at all), unlike mkdir's 0777 request, since a plain new file isn't
# expected to be executable by default
```

`touch -c` explicitly prevents file creation, making it purely a "update timestamps if it exists, otherwise do nothing (and don't error either)" operation:

```bash
touch -c does_not_exist.txt
echo $?
# 0 — no error, and no file was created either
ls does_not_exist.txt
# ls: cannot access 'does_not_exist.txt': No such file or directory
```

---

## touch vs > vs install vs cp --no-clobber

| Approach | Best for | Key difference from touch |
|---|---|---|
| `touch file` | Creating an empty file, or refreshing timestamps on an existing one | Never modifies content; safe to run repeatedly on the same existing file |
| `> file` | Creating an empty file (or TRUNCATING an existing one to zero bytes!) | ⚠️ On an EXISTING file, `>` destructively erases its content — a critical difference from touch, which never does this |
| `install -m 644 /dev/null file` | Creating an empty file with specific permissions in one step | Less common for this specific purpose; more typical for install/packaging scripts |
| `cp --no-clobber` | Copying a file only if the destination doesn't already exist | A different operation entirely (copying content), not directly comparable except in the shared "don't disturb an existing file" spirit |

```bash
touch existing_file.txt    # SAFE — content preserved, only timestamps change
> existing_file.txt          # DESTRUCTIVE — content is immediately WIPED to zero bytes
```

---

## Related Commands

| Command | Relation |
|---|---|
| `mkdir` | The directory-creation equivalent of touch's file-creation behavior |
| `stat` | Inspect a file's full timestamp details (atime/mtime/ctime) that touch can modify |
| `ls -l` / `ls -lu` / `ls -lc` | View a file's mtime, atime, or ctime respectively |
| `find -newer` | Locate files based on timestamp comparisons, often used alongside touch-created reference files |
| `make` | Build systems like `make` rely heavily on file modification timestamps — a common practical reason to use `touch` deliberately |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
