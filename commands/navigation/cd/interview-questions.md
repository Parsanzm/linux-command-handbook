# cd — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [Symlinks & Paths](#symlinks--paths)
- [Scripting](#scripting)
- [Environment Variables](#environment-variables)
- [Scenario-Based](#scenario-based)
- [Advanced](#advanced)

---

## Conceptual

**Q1 🔥 Why is `cd` a shell builtin and not an external program?**
> Every Linux process has its own working directory stored in the kernel. When a shell runs an external command, it **forks a child process**. The child can call `chdir()` to change its own working directory, but this change **only affects the child** — the parent shell remains unchanged. Since the whole point of `cd` is to change the shell's working directory, it must run inside the shell process itself using `chdir()` directly. That's what a builtin is.
>
> ```bash
> type cd        # cd is a shell builtin
> /usr/bin/cd    # this exists but is useless as a standalone program
> ```

---

**Q2. What syscall does `cd` use internally?**
> `chdir(path)` — changes the calling process's working directory. On success returns 0; on failure returns -1 with errno set (ENOENT, ENOTDIR, EACCES, etc.). The shell then updates `$PWD` and `$OLDPWD`.

---

**Q3 🔥 What are `$PWD` and `$OLDPWD`?**
> - `$PWD` — current working directory, maintained by the shell (not the kernel). Updated by `cd` on every successful directory change.
> - `$OLDPWD` — the previous working directory, set by `cd` before changing to the new one. Used by `cd -`.
>
> The kernel also tracks the real (physical) path. `$PWD` may differ if symlinks are involved — `pwd -P` asks the kernel for the real path.

---

**Q4. What is the difference between a shell builtin and an external command?**
> A **builtin** runs inside the shell process (no fork). An **external command** forks a child process, execs the binary, and returns. Builtins are necessary for commands that must affect the shell's own state: `cd` (cwd), `export` (environment), `set` (shell options), `source` (run in current shell), etc.

---

**Q5. What does `cd` with no argument do? Why?**
> Goes to `$HOME`. POSIX specifies that `cd` with no operand is equivalent to `cd $HOME`. `$HOME` is set at login by PAM from `/etc/passwd`.

---

## Basic Usage

**Q6. What does `cd -` do?**
> Changes to the previous directory (`$OLDPWD`) and prints the destination. Toggling `cd -` repeatedly alternates between two directories.

---

**Q7. How do you go two levels up?**
> ```bash
> cd ../..
> cd ../../         # same
> ```

---

**Q8. What does `cd ~username` do?**
> Changes to that user's home directory (read from `/etc/passwd`). Requires execute permission on the target directory.
> ```bash
> cd ~root      # /root
> cd ~alice     # /home/alice
> ```

---

**Q9 🔥 What is the difference between `cd -L` and `cd -P`?**
> - `-L` (default, logical): follows the logical path. Symlink names are preserved in `$PWD`.
> - `-P` (physical): resolves all symlinks. `$PWD` reflects the real filesystem path.
>
> ```bash
> ln -s /var/log /tmp/logs
> cd /tmp/logs && pwd        # /tmp/logs   (logical)
> cd -P /tmp/logs && pwd     # /var/log    (physical)
> ```
>
> Affects `cd ..` behavior: logical `..` goes to `/tmp`; physical `..` goes to `/var`.

---

**Q10. What does `cd .` do?**
> Nothing — stays in the current directory. But it does re-evaluate `$PWD` from the kernel, which can fix a stale `$PWD` if the directory was deleted and re-created, or if `$PWD` was manually corrupted.

---

## Symlinks & Paths

**Q11 🔥 You're in `/tmp/link` (a symlink to `/var/log`). You run `cd ..`. Where are you?**
> `/tmp` — not `/var`. The default `-L` mode preserves the logical path. `cd ..` goes to the logical parent of `/tmp/link`, which is `/tmp`.
>
> To get to `/var` (the real parent), use `cd -P ..` while in the symlinked directory.

---

**Q12. How do you find the real (physical) path of your current directory when symlinks are involved?**
> ```bash
> pwd -P          # shell builtin: asks kernel for real path
> realpath .      # external command: resolves all symlinks
> readlink -f .   # another option
> ```

---

**Q13. Can `$PWD` lie to you? How?**
> Yes — in two ways:
> 1. `$PWD` preserves symlink names (logical path) even though the kernel knows the real path.
> 2. `$PWD` can be directly overwritten: `PWD=/fake/path`. After this, `pwd` shows the fake path but `pwd -P` shows the truth.
>
> Run `cd .` to resync `$PWD` from the kernel.

---

## Scripting

**Q14 🔥 What's the bug in this script?**
```bash
#!/bin/bash
cd /tmp/workdir
rm -rf *
```
> If `cd /tmp/workdir` fails (directory doesn't exist), the script continues and runs `rm -rf *` in whatever directory the script was called from — potentially deleting everything in the caller's working directory.
>
> Fix:
> ```bash
> cd /tmp/workdir || exit 1
> rm -rf *
> ```
> Or with `set -e` at the top, a failed `cd` exits the script immediately.

---

**Q15 🔥 How do you temporarily change directory in a script and return afterwards?**
> Three approaches:
> ```bash
> # 1. Subshell (safest — parent unaffected automatically)
> (cd /tmp && do_something)
>
> # 2. pushd/popd
> pushd /tmp > /dev/null
> do_something
> popd > /dev/null
>
> # 3. Save and restore
> original=$(pwd)
> cd /tmp
> do_something
> cd "$original"
> ```

---

**Q16. How do you make a script always run relative to its own location?**
> ```bash
> #!/bin/bash
> script_dir="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
> cd "$script_dir"
> # Now ./config, ./lib, etc. are relative to the script
> ```
> This works even if the script is called from a different directory.

---

**Q17 🔥 Why doesn't this work?**
```bash
$(cd /tmp)
```
> `$(...)` runs in a **subshell**. The `cd` changes the subshell's working directory, but the subshell exits immediately after. The parent shell's cwd is unchanged. This is a no-op.
>
> `cd` must be called directly in the current shell (not in a subshell, pipe, or xargs) to have any effect.

---

**Q18. How do you handle a path with spaces in a script?**
> Always quote the variable:
> ```bash
> dir="my documents"
> cd "$dir"       # ✅ quotes prevent word splitting
> cd $dir         # ❌ word splits: tries cd "my" "documents"
> ```

---

## Environment Variables

**Q19. What is `$CDPATH` and when is it useful?**
> A colon-separated list of directories. When `cd` receives a relative path that doesn't exist locally, it searches each `$CDPATH` entry in order.
> ```bash
> export CDPATH=".:$HOME:$HOME/projects"
> cd myapp    # searches ./myapp, ~/myapp, ~/projects/myapp
> ```
> Useful for developers who jump between project directories frequently. Always put `.` first to prefer local directories.

---

**Q20. What's the danger of `$CDPATH` in scripts?**
> If the user has `$CDPATH` set, a script's `cd config` might go to `~/projects/config` instead of `./config`. Always use `unset CDPATH` or explicit relative paths (`cd ./config`) in scripts.

---

**Q21. What environment variable does `cd -` use?**
> `$OLDPWD` — set by `cd` to the previous working directory before each directory change.
> ```bash
> cd /var/log    # OLDPWD = (previous cwd), PWD = /var/log
> cd /etc        # OLDPWD = /var/log, PWD = /etc
> cd -           # uses OLDPWD → goes to /var/log
> ```

---

## Scenario-Based

**Q22 🔥 You deleted a directory you were currently in. What happens and how do you recover?**
> The shell still shows the old `$PWD` (stale). The kernel still has the inode open so some operations may work, others won't. You'll see errors like "No such file or directory" on most operations.
>
> Recovery:
> ```bash
> cd        # go home — always works
> cd /tmp   # or any known good directory
> ```

---

**Q23. How do you `cd` to the output of a command?**
> ```bash
> cd "$(git rev-parse --show-toplevel)"    # git repo root
> cd "$(dirname "$(which python3)")"       # python3 binary's directory
> cd "$(find . -type d -name "src" | head -1)"
> ```
> The `$()` subshell captures stdout; the outer `cd` runs in the current shell.

---

**Q24. What's the difference between these two?**
```bash
cd /tmp && ls
(cd /tmp && ls)
```
> - `cd /tmp && ls` — changes the current shell's directory to `/tmp`, then lists it. After this command, your shell is in `/tmp`.
> - `(cd /tmp && ls)` — runs in a **subshell**. Lists `/tmp` but the parent shell's cwd is unchanged.

---

**Q25 🔥 A junior developer's deploy script fails randomly. The script does `cd $DEPLOY_DIR` and then `rm -rf *`. What could go wrong and how would you fix it?**
> Several problems:
> 1. If `$DEPLOY_DIR` is unset or empty, `cd` goes to `$HOME`, then `rm -rf *` deletes everything in home.
> 2. If `$DEPLOY_DIR` doesn't exist, `cd` fails silently (without `set -e`), and `rm -rf *` runs in the current directory.
> 3. `rm -rf *` doesn't remove hidden files (dotfiles).
>
> Fix:
> ```bash
> set -e
> set -u   # error on unset variables
> cd "${DEPLOY_DIR:?DEPLOY_DIR is not set}" || exit 1
> rm -rf ./*  # more explicit; consider rm -rf "${DEPLOY_DIR:?}"/* instead
> ```

---

## Advanced

**Q26. How does `cd` handle the path resolution according to POSIX?**
> POSIX specifies this algorithm for `cd path`:
> 1. If path begins with `/`, use it as-is.
> 2. If path is `.` or starts with `./` or `../`, use `$PWD/path`.
> 3. Otherwise, search `$CDPATH` entries in order.
> 4. Call `chdir()` with the resolved path.
> 5. Update `$PWD` (logical or physical depending on `-L`/`-P`).

---

**Q27. What is the directory stack and which commands manage it?**
> A LIFO stack of directories maintained by the shell. Commands:
> - `pushd dir` — cd to dir and push current to stack
> - `popd` — return to top of stack
> - `dirs` — show stack contents
> - `dirs -v` — show with index numbers
> - `dirs -c` — clear stack
> - `pushd +N` — rotate stack (bash/zsh)
>
> In zsh, `setopt AUTO_PUSHD` makes every `cd` automatically push to the stack — giving you a full navigation history.

---

**Q28. You need execute but not read permission on a directory. What can and can't you do?**
> ```bash
> chmod 111 /secret    # execute only, no read
> cd /secret           # ✅ can enter the directory
> ls /secret           # ❌ Permission denied (need r to list)
> cat /secret/file.txt # ✅ can access files IF you know the name
> ```
> Execute (`x`) on a directory = permission to traverse/enter and stat known files.
> Read (`r`) on a directory = permission to list filenames.
> They are completely independent.

---

**Q29 🔥 Why does `cd` in a shell function affect the caller, but `cd` in a script doesn't?**
> Shell **functions** run in the current shell process — they share the shell's state including cwd, variables, and environment. A `cd` inside a function changes the calling shell's directory.
>
> A **script** runs in a separate child process (fork + exec). Its `cd` only affects the child. When the script exits, the parent shell's cwd is unchanged.
>
> Exception: if you `source` (`. script.sh`) a script, it runs in the current shell — so `cd` inside it DOES affect your shell.

---

**Q30. What is `zoxide` and how does it improve on `cd`?**
> `zoxide` is a smarter `cd` written in Rust. It tracks which directories you visit and how often (frecency = frequency + recency). You can jump to any previously visited directory by typing part of its name:
> ```bash
> z proj      # jumps to ~/projects/myapp if you've been there before
> zi proj     # interactive fuzzy selection with fzf
> ```
> It learns your patterns over time and improves. Drop-in replacement: `eval "$(zoxide init bash)"` adds `z` as an alias.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
