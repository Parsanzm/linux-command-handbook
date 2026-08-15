# touch — Edge Cases & Gotchas

> `touch` looks like the simplest possible command, but its interaction
> with symlinks, build systems, and file permissions still produces
> real, occasionally confusing surprises.

---

## Table of Contents

- [touch on a Symlink Affects the TARGET File by Default, Not the Link Itself](#touch-on-a-symlink-affects-the-target-file-by-default-not-the-link-itself)
- [touch Requires Write Permission on the File — But Only Directory Permission to Create One](#touch-requires-write-permission-on-the-file--but-only-directory-permission-to-create-one)
- [-c Silently Does Nothing If the File Doesn't Exist — Easy to Mistake for a Bug](#-c-silently-does-nothing-if-the-file-doesnt-exist--easy-to-mistake-for-a-bug)
- [A Future Timestamp Is Perfectly Legal and Can Break Build Systems in Confusing Ways](#a-future-timestamp-is-perfectly-legal-and-can-break-build-systems-in-confusing-ways)
- [touch Never Truncates Content — But > (Redirection) Silently Does, and People Confuse the Two](#touch-never-truncates-content--but--redirection-silently-does-and-people-confuse-the-two)
- [ctime Cannot Be Set Directly by touch, Even Though It Changes Every Time touch Runs](#ctime-cannot-be-set-directly-by-touch-even-though-it-changes-every-time-touch-runs)
- [Some Filesystems Don't Actually Track atime by Default](#some-filesystems-dont-actually-track-atime-by-default)
- [Creating a File in a Directory You Can't Write To Fails, Even If You Own the (Nonexistent) File](#creating-a-file-in-a-directory-you-cant-write-to-fails-even-if-you-own-the-nonexistent-file)
- [-d's Flexible Date Parsing Can Interpret Ambiguous Strings Differently Than Expected](#-ds-flexible-date-parsing-can-interpret-ambiguous-strings-differently-than-expected)
- [touch on a File Someone Else Is Actively Writing To Doesn't Interrupt or Lock Anything](#touch-on-a-file-someone-else-is-actively-writing-to-doesnt-interrupt-or-lock-anything)
- [Wildcard touch Can Silently Create Nothing If the Pattern Matches No Files](#wildcard-touch-can-silently-create-nothing-if-the-pattern-matches-no-files)

---

## touch on a Symlink Affects the TARGET File by Default, Not the Link Itself

### A frequent surprise when trying to update a symlink's own timestamp
```bash
ln -s /var/data/real_file.txt mylink.txt
touch mylink.txt
# ⚠️ By default, this updates the timestamp of /var/data/real_file.txt
# — the file the symlink POINTS TO — not the symlink itself. touch
# transparently follows symlinks unless told otherwise, which can be
# unexpected if the actual goal was to modify the LINK's own metadata.

touch -h mylink.txt
# -h (--no-dereference) makes touch affect the SYMLINK itself instead
# of following it to the target — required when the link's own
# timestamp is genuinely what needs changing
```

---

## touch Requires Write Permission on the File — But Only Directory Permission to Create One

### Two different permission requirements depending on whether the file already exists
```bash
touch existing_readonly_file.txt
# touch: setting times of 'existing_readonly_file.txt': Permission denied
# ⚠️ Updating an EXISTING file's timestamp requires WRITE permission
# on that specific file — even though touch isn't changing its
# CONTENT, timestamp modification is still governed by the file's own
# write permission bit.

touch new_file_in_this_dir.txt
# ⚠️ CREATING a new file, by contrast, requires WRITE permission on
# the containing DIRECTORY, not on the (not-yet-existing) file itself
# — a genuinely different permission check applies depending on which
# of touch's two behaviors is actually being triggered.
```

---

## -c Silently Does Nothing If the File Doesn't Exist — Easy to Mistake for a Bug

### No error, no file, and exit code 0 — all at once
```bash
touch -c definitely_does_not_exist.txt
echo $?
# 0
ls definitely_does_not_exist.txt
# ls: cannot access 'definitely_does_not_exist.txt': No such file or directory
# ⚠️ This is INTENTIONAL, documented behavior — -c explicitly disables
# file creation, so touch simply does nothing at all (successfully,
# with no error) when the target doesn't exist. Someone unfamiliar
# with -c's specific purpose might assume this represents a bug or a
# silent failure, when it's actually working exactly as designed.
```

---

## A Future Timestamp Is Perfectly Legal and Can Break Build Systems in Confusing Ways

### touch has no built-in sanity check against "impossible" dates
```bash
touch -d "2099-01-01" important_file.txt
# ⚠️ touch happily accepts and sets a timestamp far in the FUTURE,
# with no warning or refusal — this is technically valid, but can
# cause deeply confusing downstream behavior in tools that assume
# timestamps are never later than "now," most notably build systems
# like make, which compare source/target timestamps to decide what
# needs rebuilding.

make
# make: Warning: File 'important_file.txt' has modification time
# 2.1e+09 s in the future
# A future-dated file can cause make to treat EVERYTHING depending on
# it as perpetually "up to date" (or perpetually stale, depending on
# direction), producing bizarre, hard-to-diagnose build behavior until
# the underlying future timestamp itself is identified and corrected.
```

---

## touch Never Truncates Content — But > (Redirection) Silently Does, and People Confuse the Two

### A dangerous mix-up between two superficially similar-looking operations
```bash
echo "important data" > myfile.txt
touch myfile.txt
cat myfile.txt
# important data     ← content is COMPLETELY UNCHANGED, exactly as expected

echo "important data" > myfile.txt
> myfile.txt
cat myfile.txt
# (empty)     ← ⚠️ shell redirection with NO command before it
# TRUNCATES an existing file to zero bytes IMMEDIATELY — a
# fundamentally different, destructive operation that some people
# mistakenly believe is "the same as touch" for creating an empty
# file, since both CAN result in an empty file, but only one of them
# is safe to run against a file that already has real content.

# touch is the SAFE choice when a file might already exist and its
# content must be preserved; > is only safe for genuinely NEW files
# or when truncation is explicitly, deliberately intended.
```

---

## ctime Cannot Be Set Directly by touch, Even Though It Changes Every Time touch Runs

### A common point of confusion between the three timestamp types
```bash
touch -d "2020-01-01" old_file.txt
stat old_file.txt
# Modify: 2020-01-01 00:00:00
# Change: 2026-08-11 14:32:00   ← ⚠️ ctime shows the CURRENT time,
# NOT 2020, even though -d explicitly requested a 2020 date

# This is expected: ctime ("change time") records when the file's
# METADATA was last altered — and running touch itself IS a metadata
# change, so ctime is automatically updated to reflect WHEN the touch
# command ran, regardless of what mtime/atime value was explicitly
# requested. There is no flag or mechanism to directly set ctime to
# an arbitrary value — it's entirely kernel-controlled as a
# side effect of other operations.
```

---

## Some Filesystems Don't Actually Track atime by Default

### touch -a can appear to silently "not work" depending on mount options
```bash
touch -a somefile.txt
stat somefile.txt
# Access: 2026-08-11 14:32:00   ← may show the SAME value as before,
# seemingly unchanged
# ⚠️ Many modern Linux systems mount filesystems with `relatime` or
# even `noatime` as a performance optimization — under `noatime`,
# access-time updates are DISABLED entirely at the filesystem level,
# and even `touch -a`'s explicit request to update atime may be
# ignored or behave unexpectedly, depending on the exact mount
# configuration.

mount | grep " / "
# check the mount options for relevant atime-related flags
# (noatime, relatime, strictatime) affecting the filesystem in question
```

---

## Creating a File in a Directory You Can't Write To Fails, Even If You Own the (Nonexistent) File

### Ownership of a file that doesn't exist yet is a meaningless concept
```bash
touch /etc/some_new_file.txt
# touch: cannot touch '/etc/some_new_file.txt': Permission denied
# ⚠️ This isn't about "owning" some_new_file.txt (which doesn't exist
# yet, so ownership is moot) — it's purely about whether the CURRENT
# USER has write permission on the /etc DIRECTORY itself, which is
# typically owned/writable only by root. This is the exact same
# underlying requirement covered above, worth restating because the
# error message alone doesn't always make the actual cause obvious.
```

---

## -d's Flexible Date Parsing Can Interpret Ambiguous Strings Differently Than Expected

### Natural-language date parsing is powerful but not infallible
```bash
touch -d "01/02/2026" file.txt
# ⚠️ Is this January 2nd, or February 1st? -d's flexible parser
# generally interprets ambiguous slash-separated dates in
# MM/DD/YYYY order by default (US convention) — but this can differ
# from what someone using DD/MM/YYYY conventions elsewhere might
# instinctively expect, silently setting the WRONG date rather than
# raising an error about the ambiguity.

# Prefer the unambiguous ISO 8601 format to avoid this entirely:
touch -d "2026-01-02" file.txt
# Always unambiguous: YYYY-MM-DD, no regional interpretation involved
```

---

## touch on a File Someone Else Is Actively Writing To Doesn't Interrupt or Lock Anything

### No coordination or locking mechanism involved at all
```bash
# Process A is actively appending to a log file...
tail -f growing_log.txt &

# Process B touches the same file concurrently
touch growing_log.txt
# ⚠️ This has NO effect on process A's ongoing write activity — touch
# only updates timestamp metadata, doesn't lock the file, doesn't
# interrupt any other process's open file descriptor, and doesn't
# affect content in any way. Someone might assume "touching" a file
# implies some kind of safety check or coordination with other
# processes using it, but there is none whatsoever — it's a purely
# independent metadata operation.
```

---

## Wildcard touch Can Silently Create Nothing If the Pattern Matches No Files

### Shell glob expansion behavior, not touch itself, but a common source of confusion
```bash
touch *.txt
# ⚠️ If NO files matching *.txt exist in the current directory, most
# shells leave the glob UNEXPANDED — meaning touch actually receives
# the LITERAL STRING "*.txt" as its argument (not "no arguments"),
# and creates an actual file literally NAMED "*.txt" instead of
# doing nothing as someone might expect.

ls
# '*.txt'
# ⚠️ A literal, oddly-named file appears, which can be confusing
# until the underlying glob-expansion behavior is understood —
# checking `shopt -s nullglob` (bash) changes this specific behavior
# if the goal is genuinely "do nothing when no files match," rather
# than creating a literally-named file from the unexpanded pattern.
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
