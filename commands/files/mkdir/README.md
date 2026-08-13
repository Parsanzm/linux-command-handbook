# mkdir — The Complete Reference

> **Create a new directory**
> One of the simplest commands in Unix, and one of the very first
> anyone learns — but with a few genuinely useful options beyond the basics.

---

## Table of Contents

- [What is mkdir?](#what-is-mkdir)
- [Where does mkdir live?](#where-does-mkdir-live)
- [How mkdir works internally](#how-mkdir-works-internally)
- [Syntax](#syntax)
- [Understanding the Output](#understanding-the-output)
- [Creating Nested Directories with -p](#creating-nested-directories-with--p)
- [Setting Permissions at Creation Time](#setting-permissions-at-creation-time)
- [All Key Options](#all-key-options)
- [mkdir and Directory Permissions](#mkdir-and-directory-permissions)
- [mkdir vs install -d vs mkdirat](#mkdir-vs-install--d-vs-mkdirat)
- [Related Commands](#related-commands)

---

## What is mkdir?

`mkdir` ("make directory") creates one or more new, empty directories at the specified path(s). It's a foundational, almost universally first-learned command — but its `-p` flag (create any missing parent directories along the way) is genuinely useful well beyond beginner usage, especially in scripts.

```bash
mkdir projects
# Creates a new, empty directory named "projects" in the current location
```

**Why mkdir is so ubiquitous in scripts:** nearly every setup script, build process, or deployment routine needs to ensure a directory exists before writing files into it — `mkdir -p` in particular is a near-universal first line in that kind of script.

---

## Where does mkdir live?

```
/usr/bin/mkdir  (or /bin/mkdir on some systems)
```

```bash
which mkdir
mkdir --version
# mkdir (GNU coreutils) 9.4
```

Part of **GNU coreutils** on most Linux distributions, and present in essentially identical form (with a smaller POSIX-baseline flag set) on macOS, BSD, and virtually every other Unix-like system — `mkdir` is about as universally portable as commands get.

---

## How mkdir works internally

`mkdir` is a thin wrapper around the `mkdir(2)` system call, which asks the kernel's filesystem layer to create a new directory entry at the given path, with the specified (or default) permission bits.

```c
int mkdir(const char *pathname, mode_t mode);
```

The kernel handles the actual filesystem-level work — allocating an inode, setting up the initial `.` and `..` entries, and applying permissions (adjusted by the process's current `umask`, covered below). `mkdir` itself does essentially nothing beyond parsing arguments and making this syscall once per directory being created (or, with `-p`, once per missing directory in a chain).

```bash
strace -e trace=mkdir mkdir newdir
# mkdir("newdir", 0777) = 0
# Note: the syscall requests 0777, but the ACTUAL resulting
# permissions are further restricted by the process's umask —
# see "mkdir and Directory Permissions" below
```

---

## Syntax

```bash
mkdir [OPTIONS] DIRECTORY...
```

Multiple directory names can be given in a single invocation, each created independently:

```bash
mkdir dir1 dir2 dir3
# Creates all three, as siblings, in one command
```

---

## Understanding the Output

```bash
mkdir projects
# (no output at all on success)

mkdir projects
# mkdir: cannot create directory 'projects': File exists
# ⚠️ Running it again on an already-existing directory is an ERROR
# by default, not silently ignored

mkdir a/b/c
# mkdir: cannot create directory 'a/b/c': No such file or directory
# ⚠️ Without -p, mkdir refuses to create intermediate/parent
# directories that don't already exist — see below
```

`mkdir` follows the standard Unix convention of "silence means success" — no output at all when it works correctly, and a clear error message (with a non-zero exit code) when it fails.

---

## Creating Nested Directories with -p

```bash
mkdir a/b/c
# mkdir: cannot create directory 'a/b/c': No such file or directory
# ⚠️ Without -p, ALL parent directories in the path must already
# exist — mkdir refuses to create "b" and "c" on the fly if "a"
# doesn't already exist first.

mkdir -p a/b/c
# Succeeds — creates "a", then "a/b", then "a/b/c", all in one command,
# creating whatever intermediate directories are missing along the way

# -p is ALSO idempotent — running it again on an already-existing
# path does NOT error, unlike plain mkdir:
mkdir -p a/b/c
# (no error, even though a/b/c already exists) — this makes -p the
# standard, safe choice for "ensure this directory exists" in scripts
```

---

## Setting Permissions at Creation Time

```bash
# Create a directory with specific permissions directly, rather than
# creating it and then running a separate chmod afterward
mkdir -m 700 private_dir
ls -ld private_dir
# drwx------ 2 alice alice 4096 Aug 11 14:32 private_dir

# Combine with -p for a full nested creation with specific permissions
# on the FINAL directory (intermediate ones still use the default/umask)
mkdir -pm 750 shared/team/reports
```

---

## All Key Options

| Option | Long form | Description |
|---|---|---|
| `-p` | `--parents` | Create missing parent directories as needed; no error if the target already exists |
| `-m MODE` | `--mode=MODE` | Set specific permissions at creation time, instead of relying on the default/umask |
| `-v` | `--verbose` | Print a message for each directory actually created |
| `-Z` | | Set SELinux security context (on SELinux-enabled systems) |
| — | `--help` | Print usage help |
| — | `--version` | Print version |

---

## mkdir and Directory Permissions

```bash
mkdir newdir
ls -ld newdir
# drwxr-xr-x 2 alice alice 4096 Aug 11 14:32 newdir
```

The resulting permissions come from the **requested mode** (typically `0777`, i.e., all permissions for everyone) **minus** whatever bits the process's current `umask` masks out:

```bash
umask
# 0022     ← the current umask
# Resulting default directory permissions: 0777 & ~0022 = 0755
# (rwxr-xr-x) — owner gets full access, group/others get read+execute
# but not write, which is the typical, sensible default for new directories

# Temporarily create a directory with a different effective permission
# set by adjusting umask first, or more directly with -m:
mkdir -m 700 private_dir
# -m bypasses the umask calculation for the SPECIFIED bits entirely,
# setting exactly what was requested
```

---

## mkdir vs install -d vs mkdirat

| Tool | Best for | Key difference from mkdir |
|---|---|---|
| `mkdir -p` | The default, standard choice for "ensure this directory exists" | The simplest, most universally recognized option |
| `install -d` | Creating a directory with specific ownership AND permissions in one step, common in install scripts/packaging | Can set owner/group directly at creation time (`install -d -o user -g group -m 755 dir`), which plain mkdir cannot do in a single command |
| `mkdirat()` (C API, not a shell command) | Programmatic directory creation relative to an open file descriptor rather than a path string | Relevant only when writing C/systems code directly, not something invoked from the shell |

```bash
mkdir -p /opt/myapp/data           # standard, everyday directory creation
install -d -o appuser -g appgroup -m 750 /opt/myapp/data   # creation + ownership + permissions in one step, common in packaging/install scripts
```

---

## Related Commands

| Command | Relation |
|---|---|
| `rmdir` | Remove an EMPTY directory (the natural inverse of mkdir for simple cases) |
| `rm -r` | Remove a directory and everything inside it, even if non-empty |
| `chmod` | Change a directory's permissions after creation (or use `mkdir -m` to set them at creation time) |
| `chown` | Change a directory's owner/group after creation |
| `install -d` | Create a directory with ownership and permissions set in a single step |
| `find -type d` | Locate existing directories matching criteria, as opposed to creating new ones |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
