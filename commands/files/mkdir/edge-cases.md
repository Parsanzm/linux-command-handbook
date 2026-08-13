# mkdir — Edge Cases & Gotchas

> `mkdir` looks like the simplest possible command, but umask
> interactions, race conditions, and trailing-slash quirks still
> produce real, occasionally confusing surprises.

---

## Table of Contents

- [mkdir Without -p Fails If ANY Parent Directory Is Missing](#mkdir-without--p-fails-if-any-parent-directory-is-missing)
- [Plain mkdir Errors on an Already-Existing Directory — -p Does Not](#plain-mkdir-errors-on-an-already-existing-directory---p-does-not)
- [The Actual Resulting Permissions Depend on umask, Not Just What You Typed](#the-actual-resulting-permissions-depend-on-umask-not-just-what-you-typed)
- [-m Only Affects the FINAL Directory, Not Intermediate Ones Created Along the Way](#-m-only-affects-the-final-directory-not-intermediate-ones-created-along-the-way)
- [A Race Condition: Checking Existence Then Creating Isn't Atomic](#a-race-condition-checking-existence-then-creating-isnt-atomic)
- [mkdir Succeeding Doesn't Mean You Can Actually Write Files Into It](#mkdir-succeeding-doesnt-mean-you-can-actually-write-files-into-it)
- [A File (Not a Directory) Already Existing at That Path Causes a Different Error](#a-file-not-a-directory-already-existing-at-that-path-causes-a-different-error)
- [Trailing Slashes Are Generally Harmless, But Not Universally Guaranteed To Be](#trailing-slashes-are-generally-harmless-but-not-universally-guaranteed-to-be)
- [mkdir -p Partially Succeeding Mid-Chain Can Leave Some Directories Created and Others Not](#mkdir--p-partially-succeeding-mid-chain-can-leave-some-directories-created-and-others-not)
- [Case Sensitivity Differs Between Filesystems — "Dir" and "dir" May or May Not Collide](#case-sensitivity-differs-between-filesystems--dir-and-dir-may-or-may-not-collide)
- [Creating a Directory Doesn't Automatically Create It Anywhere Special — Relative Paths Depend Entirely on cwd](#creating-a-directory-doesnt-automatically-create-it-anywhere-special--relative-paths-depend-entirely-on-cwd)

---

## mkdir Without -p Fails If ANY Parent Directory Is Missing

### The single most common mkdir beginner error
```bash
mkdir projects/2026/reports
# mkdir: cannot create directory 'projects/2026/reports': No such file or directory
# ⚠️ This fails as long as "projects" OR "projects/2026" doesn't
# already exist — mkdir (without -p) only ever creates the FINAL
# path component; every parent segment must already be present.

mkdir -p projects/2026/reports
# Succeeds, creating "projects", "projects/2026", and
# "projects/2026/reports" all in one command, filling in whatever
# was missing along the chain
```

---

## Plain mkdir Errors on an Already-Existing Directory — -p Does Not

### A very common source of scripts unexpectedly halting
```bash
mkdir output
mkdir output
# mkdir: cannot create directory 'output': File exists
# ⚠️ Running mkdir a SECOND time on a directory that already exists
# is treated as an ERROR by default — a script that assumes plain
# mkdir is safe to call repeatedly (e.g., inside a loop, or run
# multiple times across separate invocations) can halt unexpectedly
# on the second run if it isn't checking for this specifically.

mkdir -p output
mkdir -p output
# (no error either time) — -p is idempotent: it succeeds silently if
# the directory already exists, which is why -p is the standard,
# SAFE default for "ensure this directory exists" in scripts, even
# when nested/parent creation isn't actually needed at all
```

---

## The Actual Resulting Permissions Depend on umask, Not Just What You Typed

### A directory's real permissions can differ from what someone might assume
```bash
umask
# 0022

mkdir shared_folder
ls -ld shared_folder
# drwxr-xr-x    ← NOT drwxrwxrwx, even though mkdir's underlying
# syscall requested full (0777) permissions by default — the process's
# CURRENT umask subtracts bits from that request, and the actual
# resulting permissions can genuinely surprise someone who assumed
# a "default" mkdir call grants unrestricted access.

# To get an EXACT, predictable permission set regardless of the
# current umask, specify it explicitly:
mkdir -m 777 shared_folder
# -m bypasses the umask calculation for exactly the bits specified
```

---

## -m Only Affects the FINAL Directory, Not Intermediate Ones Created Along the Way

### A subtle scoping detail when combining -m with -p
```bash
mkdir -pm 700 a/b/c
ls -ld a a/b a/b/c
# drwxr-xr-x  a         ← created with DEFAULT/umask permissions
# drwxr-xr-x  a/b        ← also default/umask permissions
# drwx------  a/b/c       ← ONLY this final directory gets the
#                            explicitly requested 700 permissions
# ⚠️ When combining -m with -p to create a nested chain, the
# specified mode applies ONLY to the last (target) directory in the
# chain — any intermediate parent directories created along the way
# still use the default permission calculation (mode minus umask),
# NOT the value passed to -m. This can be surprising if the intent
# was for the ENTIRE new chain to share the same restrictive
# permissions.

# To apply consistent permissions across the whole chain, set them
# explicitly afterward:
mkdir -p a/b/c
chmod -R 700 a
```

---

## A Race Condition: Checking Existence Then Creating Isn't Atomic

### A common but flawed defensive pattern in scripts
```bash
if [ ! -d "$DIR" ]; then
  mkdir "$DIR"
fi
# ⚠️ Between the existence CHECK and the actual mkdir CALL, another
# process (a parallel instance of the same script, a concurrent job)
# could create that same directory first — causing this mkdir call to
# fail with "File exists" anyway, despite the check having just
# reported it didn't exist moments earlier. This is a classic
# time-of-check-to-time-of-use (TOCTOU) race condition.

# mkdir -p sidesteps this issue entirely, since it doesn't error on
# an already-existing target regardless of WHEN it was created:
mkdir -p "$DIR"
# Safe even under concurrent execution, since -p tolerates the
# directory already existing by the time this specific call runs
```

---

## mkdir Succeeding Doesn't Mean You Can Actually Write Files Into It

### Directory creation and directory writability are separate concerns
```bash
mkdir -m 555 readonly_dir
# Succeeds — the DIRECTORY itself was created fine

touch readonly_dir/newfile.txt
# touch: cannot touch 'readonly_dir/newfile.txt': Permission denied
# ⚠️ A directory created with read+execute permissions but WITHOUT
# write permission (555 = r-xr-xr-x) can be successfully CREATED and
# even listed/entered, but nothing can be written INTO it afterward
# — a script that only checks "did mkdir succeed" without also
# considering the resulting permissions may proceed assuming it can
# write files there, then fail on the very next step.
```

---

## A File (Not a Directory) Already Existing at That Path Causes a Different Error

### Worth distinguishing from the "directory already exists" case
```bash
touch config
mkdir config
# mkdir: cannot create directory 'config': File exists
# ⚠️ This EXACT SAME error message appears whether "config" is
# already an existing DIRECTORY or an existing plain FILE at that
# path — the message alone doesn't disambiguate which case you're in,
# which matters because the appropriate fix differs (removing a
# conflicting file is very different from just reusing an existing
# directory).

ls -ld config
# -rw-r--r-- 1 alice alice 0 Aug 11 14:32 config
# Checking explicitly reveals it's a FILE, not a directory, requiring
# it to be removed/renamed before a directory can occupy that path
```

---

## Trailing Slashes Are Generally Harmless, But Not Universally Guaranteed To Be

### A minor but occasionally relevant portability note
```bash
mkdir newdir/
# Generally behaves identically to:
mkdir newdir
# On virtually all modern systems and shells, a trailing slash on the
# target path makes no practical difference for mkdir specifically.
# ⚠️ This is unlike some OTHER commands (rsync's SOURCE trailing
# slash meaningfully changes behavior, for instance) — don't
# generalize trailing-slash sensitivity learned from a different tool
# onto mkdir, which doesn't share that particular quirk.
```

---

## mkdir -p Partially Succeeding Mid-Chain Can Leave Some Directories Created and Others Not

### A permissions failure partway through a nested creation isn't fully atomic
```bash
mkdir -p a/b/c
# ⚠️ If, say, "a" gets created successfully but the process then
# lacks permission to create "a/b" (perhaps due to a permission
# restriction on "a" itself, or a quota limit hit partway through),
# mkdir -p can leave "a" existing on disk while "a/b" and "a/b/c"
# were never created — a PARTIAL result, not an all-or-nothing
# atomic operation across the whole chain.

echo $?
# non-zero — reflects that the OVERALL command failed, but doesn't
# by itself tell you exactly WHICH parts of the chain succeeded
# before the failure occurred; checking afterward is necessary if
# that distinction matters for cleanup or retry logic
ls -ld a a/b 2>/dev/null
```

---

## Case Sensitivity Differs Between Filesystems — "Dir" and "dir" May or May Not Collide

### The same command can behave differently depending on the underlying filesystem
```bash
mkdir Reports
mkdir reports
# On a standard Linux ext4/xfs filesystem: succeeds — these are TWO
# genuinely SEPARATE directories, since the filesystem is case-sensitive.

# ⚠️ On a case-INSENSITIVE filesystem (the default on many macOS
# installs, or certain mounted network shares/exFAT/NTFS
# configurations), the SECOND mkdir call would instead fail with
# "File exists," treating "Reports" and "reports" as the SAME
# directory — a script written and tested on Linux can behave
# genuinely differently when run against a case-insensitive filesystem
# elsewhere.
```

---

## Creating a Directory Doesn't Automatically Create It Anywhere Special — Relative Paths Depend Entirely on cwd

### A very common source of "where did my directory actually go" confusion
```bash
cd /tmp
mkdir output
# ⚠️ This creates /tmp/output — NOT wherever the person might have
# been assuming (their home directory, a project root, etc.) — a
# RELATIVE path (no leading /) is always resolved against the
# CURRENT working directory at the moment mkdir runs, which is easy
# to lose track of, especially inside a longer script that has
# already `cd`'d somewhere unexpected earlier.

pwd
# Always worth confirming the current directory before running a
# relative-path mkdir in an unfamiliar or scripted context — or use
# an explicit absolute path to remove the ambiguity entirely:
mkdir /home/alice/projects/output
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
