# pwd — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Logical vs Physical Paths](#logical-vs-physical-paths)
- [Internals](#internals)
- [Builtin vs Binary](#builtin-vs-binary)
- [Scripting](#scripting)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does pwd do, and what is it actually reporting?**
> `pwd` (**Print Working Directory**) prints the absolute path of the calling shell or process's **current working directory (CWD)** — a value tracked by the kernel for every running process, which relative paths are resolved against.

---

**Q2 🔥 Is pwd a shell builtin, an external binary, or both?**
> Both. Nearly every shell (bash, zsh, ksh, dash) implements `pwd` as a **builtin** for speed, and a standalone `/bin/pwd` or `/usr/bin/pwd` binary also exists on the filesystem — used by scripts or shells without a builtin version, or when explicitly bypassing the builtin. `type pwd` shows which one runs by default (usually the builtin); `which pwd` shows the path to the standalone binary regardless.

---

**Q3. What system call does pwd rely on internally?**
> `getcwd()`. At its core, `pwd` (whether the builtin or the standalone binary) is a thin wrapper around this system call, which asks the kernel to return the calling process's currently recorded working directory as an absolute path string.

---

## Logical vs Physical Paths

**Q4 🔥 What's the difference between `pwd -L` and `pwd -P`?**
> `-L` (logical, the default in most shells) prints the path **as navigated**, preserving any symlinks you `cd`'d through without resolving them. `-P` (physical) prints the path with **every symlink fully resolved** to its actual target — the true, canonical location on disk with no symbolic links remaining.

---

**Q5. If `/var/run` is a symlink to `/run`, what does `cd /var/run; pwd` print by default, and what does `pwd -P` print instead?**
> By default (logical), `pwd` prints `/var/run` — exactly the path you navigated to, symlink intact. `pwd -P` prints `/run` — the fully resolved, real location the symlink actually points to.

---

**Q6 🔥 Why would a script prefer `pwd -P` over the default `pwd -L`?**
> When the script needs the **true, canonical path** for reliable string comparisons, avoiding ambiguity from symlinks, or when the exact real on-disk location matters (e.g., generating a path to embed in a config file, or comparing against another already-resolved path). Using the logical path in such cases can cause silent mismatches if a symlink is involved anywhere in the navigated path.

---

## Internals

**Q7. What is $PWD, and how does it differ from actually running the pwd command?**
> `$PWD` is an environment variable that most shells automatically maintain and update every time `cd` (or `pushd`/`popd`) runs, caching what `pwd` would report without needing to spawn a new process. Running the actual `pwd` command instead performs a fresh `getcwd()` system call every time, which is authoritative even if `$PWD` were somehow manually altered or became stale — `$PWD` is just a variable and nothing prevents a script or user from overwriting it with an incorrect value.

---

**Q8 🔥 What is $OLDPWD, and what built-in behavior relies on it?**
> `$OLDPWD` is automatically set to the **previous** working directory every time `cd` successfully changes directories. The shorthand `cd -` relies on it directly — it's equivalent to `cd "$OLDPWD"`, and additionally **prints** the directory it switches to (unlike a normal silent `cd`), making it the classic way to toggle back and forth between two directories.

---

**Q9. If someone manually runs `PWD="/some/fake/path"` in their shell, what happens when they subsequently run the actual `pwd` command?**
> `pwd` (the command) still reports the **correct, real** current directory, because it queries the kernel directly via `getcwd()` rather than trusting the `$PWD` variable at all. Only code that reads `$PWD` directly (instead of calling the `pwd` command) would be fooled by the manually corrupted value.

---

## Builtin vs Binary

**Q10 🔥 Why might `pwd --help` produce an error, while `/bin/pwd --help` works fine?**
> Plain `pwd` resolves to the shell's own lightweight **builtin** implementation, which follows the minimal POSIX specification (supporting only `-L`/`-P`) and doesn't implement the `--help`/`--version` conventions common to full GNU coreutils tools. The standalone `/bin/pwd` binary, being a proper GNU coreutils program, does support the familiar `--help` and `--version` flags — the confusing part is that unqualified `pwd` almost always runs the builtin first, not the binary.

---

**Q11. How would you force the standalone /bin/pwd binary to run instead of the shell's builtin?**
> Call it by its full path directly: `/bin/pwd`. Using `command pwd` typically still invokes the builtin in most shells (since `command` mainly bypasses aliases/functions, not builtins), so the explicit full path is the most reliable way to guarantee the actual binary runs.

---

## Scripting

**Q12 🔥 A script uses `$(pwd)/config.sh` to locate a config file next to itself, but it breaks when called from a different directory. What's wrong, and how do you fix it?**
> `pwd` reports where the **caller** currently is, not where the **script file itself** is located on disk — these only coincide by chance. The reliable fix is deriving the script's own directory explicitly: `SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"`, then referencing files relative to `$SCRIPT_DIR` instead of `$(pwd)`.

---

**Q13. Does running `cd somewhere` inside a `$(...)` command substitution or `(...)` subshell affect the outer/calling shell's working directory?**
> No. Both command substitution and explicit subshells run in a **separate child process** — any `cd` performed inside is entirely isolated to that subshell and evaporates once it exits. The calling shell's own working directory (and its `pwd` output) remains completely unaffected, which is often useful for temporarily operating in a different directory without needing to manually `cd` back afterward.

---

**Q14 🔥 What's the safest pattern for temporarily changing directory in a script and guaranteeing you return to the original location, even if an error occurs partway through?**
> ```bash
> ORIGINAL_DIR="$(pwd)"
> trap 'cd "$ORIGINAL_DIR"' EXIT
> cd /tmp/work
> # ... any error, signal, or normal exit still triggers the trap, restoring the directory
> ```
> Alternatively, wrapping the entire risky section in a subshell — `(cd /tmp/work && do_something)` — achieves the same isolation without needing an explicit trap, since the subshell's directory change can never leak back to the caller in the first place.

---

## Scenario-Based

**Q15 🔥 A user cd's into a directory, then that same directory is deleted by another process in a separate terminal. What happens the next time they run `pwd` or `ls` in that shell?**
> Both commands fail — typically with something like "error retrieving current directory: No such file or directory" for `pwd`, and a similar "cannot access '.'" error for `ls`. The shell has no built-in mechanism to detect or react to its current directory vanishing out from under it; the failure only becomes visible the next time something actually tries to query or use the now-nonexistent path. Recovery requires explicitly `cd`-ing elsewhere — there's no way to "undo" the deletion or reconstruct the shell's sense of location.

---

**Q16. A script does `if [ "$(pwd)" = "/var/log" ]; then ...` but the branch never triggers even though the user is clearly working with files in /var/log via a symlinked path. What's the likely cause?**
> The user likely navigated there through a symlink (e.g., `cd ~/logs_shortcut` where `~/logs_shortcut -> /var/log`), so the default logical `pwd` reports the symlinked path (`/home/alice/logs_shortcut`), not the physical target (`/var/log`) the script's comparison expects. Fixing the comparison to use `pwd -P` instead of plain `pwd` resolves the mismatch by comparing against the fully-resolved physical path.

---

**Q17 🔥 Why might `pwd -P` reveal internal infrastructure details (like an NFS server's export path) that surprised a developer expecting to just see their home directory?**
> If the home directory is actually an NFS-mounted or otherwise symlinked location, the logical path (`/home/alice`) is the clean, client-side view, but the physical/resolved path can expose the real underlying server export structure (e.g., `/net/fileserver01/export/home/alice`). This is expected behavior for `-P`, but worth being cautious about when physical paths might end up in logs, error messages, or output shown to end users, since they can leak infrastructure details unintentionally.

---

**Q18. Inside a chrooted process or a container, running `pwd` right after entering shows `/`, even though the actual directory on the host filesystem is something like `/var/lib/mycontainer`. Is this a bug?**
> No — this is expected and correct behavior. `pwd` and the underlying `getcwd()` system call are fundamentally **process-relative**: within a chroot or container's own filesystem namespace, `/` genuinely is the root as far as that process can see or report, and it has no visibility into (or way to express) the host's real absolute path prefix. The identical string `"/"` legitimately means different real locations depending on which namespace the reporting process is running inside.

---

**Q19. A developer sources a third-party shell script (`source vendor_lib.sh`) that internally does a `cd` but never changes back, and afterward their own script's subsequent commands unexpectedly operate on the wrong directory. Why did this happen, and how could it have been prevented?**
> Unlike a subshell or command substitution, `source`d code runs directly within the **calling shell's own process** — any `cd` performed inside a sourced script is NOT isolated and persists in the caller's environment exactly as if the caller had typed it themselves. This could have been prevented by capturing the working directory before sourcing (`BEFORE=$(pwd)`) and explicitly restoring it afterward (`cd "$BEFORE"`), or by auditing/wrapping untrusted sourced code's directory-changing behavior before relying on it inside a larger script.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
