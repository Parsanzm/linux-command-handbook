# pwd — Edge Cases & Gotchas

> pwd looks trivial, but symlinks, deleted directories, and $PWD drift
> create real confusion — especially inside scripts that assume too much.

---

## Table of Contents

- [Logical vs Physical Confusion](#logical-vs-physical-confusion)
- [$PWD Can Be Manually Corrupted or Stale](#pwd-can-be-manually-corrupted-or-stale)
- [Deleted / Unlinked Current Directory](#deleted--unlinked-current-directory)
- [pwd Inside a Script vs the Caller's Directory](#pwd-inside-a-script-vs-the-callers-directory)
- [Subshells Silently Isolate Directory Changes](#subshells-silently-isolate-directory-changes)
- [Symlinked Home Directories](#symlinked-home-directories)
- [pwd and Permission-Denied Parent Directories](#pwd-and-permission-denied-parent-directories)
- [Builtin pwd --help Doesn't Behave Like a Normal GNU Tool](#builtin-pwd---help-doesnt-behave-like-a-normal-gnu-tool)
- [NFS / Network Mount Path Ambiguity](#nfs--network-mount-path-ambiguity)
- [Race Condition: Directory Changes Between pwd Calls](#race-condition-directory-changes-between-pwd-calls)
- [Trailing Slash and Root Directory Edge Cases](#trailing-slash-and-root-directory-edge-cases)
- [pwd in a chroot or Container](#pwd-in-a-chroot-or-container)

---

## Logical vs Physical Confusion

### Assuming pwd always shows the "real" path
```bash
ln -s /var/log ~/logs
cd ~/logs
pwd
# /home/alice/logs        ← this is the LOGICAL path (default)

# A script that does string comparison against pwd's output can break
# unexpectedly if it assumed the physical path:
if [ "$(pwd)" = "/var/log" ]; then
  echo "In log directory"
else
  echo "NOT in log directory"    # ⚠️ this branch runs, even though you
fi                                 # actually ARE effectively in /var/log!

# Fix: always be explicit about which you need
if [ "$(pwd -P)" = "/var/log" ]; then
  echo "In log directory (physical match)"
fi
```

---

## $PWD Can Be Manually Corrupted or Stale

### $PWD is just a variable — nothing prevents it from lying
```bash
cd /tmp
echo $PWD
# /tmp

PWD="/completely/wrong/path"     # nothing stops you from doing this
echo $PWD
# /completely/wrong/path          ← now WRONG, but nothing complains

pwd
# /tmp                             ← the actual COMMAND still reports
# the truth, because it queries the kernel via getcwd(), ignoring
# whatever $PWD claims — a script blindly trusting $PWD instead of
# calling `pwd` can silently operate on the WRONG assumed location.

# Best practice in scripts: prefer calling `pwd` (or $(pwd)) over
# reading $PWD directly if there's ANY chance the variable could have
# been altered somewhere earlier in a long or sourced script chain.
```

---

## Deleted / Unlinked Current Directory

### pwd can fail or show stale information if the CWD was removed out from under you
```bash
mkdir /tmp/tempdir
cd /tmp/tempdir
# In ANOTHER terminal/process:
rmdir /tmp/tempdir     # the directory you're currently "in" gets deleted

pwd
# pwd: error retrieving current directory: getcwd: cannot access
# parent directories: No such file or directory
# ⚠️ Your shell doesn't automatically notice the directory vanished —
# you only find out when something (like pwd) actually tries to query it

ls
# ls: cannot access '.': No such file or directory
# Nearly every command relying on the current directory starts failing

# Recovery: you MUST cd elsewhere; there's no way to "undo" this —
# the directory is genuinely gone from the shell's perspective
cd /tmp
pwd
# /tmp    ✅ working again, but /tmp/tempdir is permanently gone
```

---

## pwd Inside a Script vs the Caller's Directory

### A script can't know its own file location just by calling pwd
```bash
# ~/scripts/deploy.sh
#!/bin/bash
echo "$(pwd)/config.sh"

# Called from the user's home directory:
cd ~
./scripts/deploy.sh
# /home/alice/config.sh    ⚠️ WRONG — config.sh doesn't live in ~, it
# lives in ~/scripts/, right next to deploy.sh itself

# pwd reports where the CALLER happens to be standing, not where the
# SCRIPT FILE is located on disk — these are only the same by coincidence.

# Fix: derive the script's own directory explicitly
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
echo "$SCRIPT_DIR/config.sh"
# /home/alice/scripts/config.sh    ✅ correct, regardless of caller's CWD
```

---

## Subshells Silently Isolate Directory Changes

### A cd inside $(...) or (...) never affects the outer shell
```bash
cd /home/alice
RESULT=$(cd /tmp && pwd)
echo "$RESULT"
# /tmp

pwd
# /home/alice     ← UNCHANGED — the cd happened inside a command
# substitution's subshell, which is a completely separate process;
# once it finishes, its directory change simply evaporates with it.

# This is often exactly what you WANT (isolation), but surprises
# people who expect `$(cd dir && somecommand)` to leave them in `dir`
# afterward — it never does, by design.
```

---

## Symlinked Home Directories

### Logging in through a symlinked home directory produces mismatched pwd output
```bash
# /home/alice is actually a symlink to /mnt/storage/alice_home
ls -l /home
# lrwxrwxrwx ... alice -> /mnt/storage/alice_home

# After logging in:
pwd
# /home/alice                    ← logical (matches $HOME as configured)
pwd -P
# /mnt/storage/alice_home         ← physical (the real underlying location)

# Scripts or tools comparing pwd's output against a hardcoded path can
# fail in confusing, environment-specific ways depending on whether
# they expected the logical or physical form:
if [ "$(pwd)" = "$HOME" ]; then
  echo "matches HOME"    # this comparison is fragile if HOME itself
fi                          # was also set inconsistently (logical vs
                            # physical) somewhere in the login process
```

---

## pwd and Permission-Denied Parent Directories

### Restrictive permissions on a PARENT directory can break pwd, even if you own the CWD itself
```bash
# Directory structure: /restricted (mode 700, owned by root) contains
# /restricted/shared (mode 777, world-accessible)
cd /restricted/shared        # succeeds if you have execute on BOTH
                              # /restricted (to traverse it) AND
                              # /restricted/shared itself

# But if /restricted's execute bit is later revoked for you WHILE
# you're still sitting inside /restricted/shared:
pwd
# pwd: error retrieving current directory: Permission denied
# ⚠️ pwd (in LOGICAL mode) may need to read/traverse parent directory
# entries to construct the full path string, and losing execute
# permission on an ancestor directory can break that reconstruction
# even though you're technically still validly "inside" the directory
# and files there remain otherwise accessible to you.
```

---

## Builtin pwd --help Doesn't Behave Like a Normal GNU Tool

### Muscle memory from other GNU tools doesn't transfer
```bash
pwd --help
# bash: pwd: --help: invalid option
# pwd: usage: pwd [-LP]
# ⚠️ The shell BUILTIN doesn't support the familiar --help/--version
# convention most GNU command-line tools follow, because it's
# implemented as a lightweight POSIX builtin, not a full GNU coreutils binary.

/bin/pwd --help
# Usage: pwd [OPTION]...
# Print the full filename of the current working directory.
# ✅ the STANDALONE binary DOES support the expected GNU-style flags —
# the confusing part is that plain "pwd" almost always resolves to the
# BUILTIN first, not the binary, so --help silently "fails" unless you
# explicitly invoke the full path.
```

---

## NFS / Network Mount Path Ambiguity

### Physical resolution can reveal server-side mount details you didn't expect
```bash
cd /home/alice     # actually an NFS-mounted directory
pwd
# /home/alice                        ← logical, matches what you expect

pwd -P
# /net/fileserver01/export/home/alice   ← physical, reveals the ACTUAL
# NFS server's export path structure, which can look completely
# different from the client-side mount point and may expose internal
# infrastructure details you didn't intend to display (e.g., in a
# script's log output or an error message shown to an end user)
```

---

## Race Condition: Directory Changes Between pwd Calls

### Two "simultaneous" pwd calls in a script aren't guaranteed to agree
```bash
# In a script where something ELSE (a background job, a signal handler,
# a sourced script) could conceivably change directory between two
# points in your own script's execution:

DIR1=$(pwd)
some_function_that_might_cd_internally
DIR2=$(pwd)

if [ "$DIR1" != "$DIR2" ]; then
  echo "Warning: working directory changed unexpectedly during execution!"
fi

# This is a real risk when sourcing third-party scripts (`source
# some_library.sh`) that might contain an unexpected `cd` with no
# corresponding cd back — sourced code runs in YOUR shell's own
# process, so its directory changes are NOT isolated like a subshell's would be.
```

---

## Trailing Slash and Root Directory Edge Cases

### The root directory is a special, minimal case
```bash
cd /
pwd
# /                    ← just a single slash, no trailing content after it

cd /var/
pwd
# /var                 ← trailing slash from `cd /var/` is normalized away;
# pwd never reports a trailing slash except for the root directory itself,
# which is inherently just "/" with nothing to strip
```

---

## pwd in a chroot or Container

### The path shown is relative to the (cha)root, not the host's real filesystem
```bash
# Inside a chroot or container at /var/lib/mycontainer on the HOST:
chroot /var/lib/mycontainer /bin/bash
pwd
# /                    ← as far as THIS process can tell, it's at the
# root of its OWN filesystem view — it has no knowledge of, or way to
# report, the host's actual /var/lib/mycontainer prefix at all, because
# getcwd() operates entirely within the process's own chroot'd namespace

# This is expected and by design — pwd (and getcwd()) are fundamentally
# process-relative, not "absolute" in some host-wide sense, so the
# SAME literal path ("/") means something completely different
# depending on which chroot/container/namespace the reporting process
# is actually running inside of.
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
