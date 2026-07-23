# env — The Complete Reference

> **Print the current environment, or run a program in a modified one**
> Present since early BSD/Unix, standardized in POSIX, and now the mechanism
> behind nearly every `#!/usr/bin/env` shebang line in existence.

---

## Table of Contents

- [What is env?](#what-is-env)
- [Where does env live?](#where-does-env-live)
- [How env works internally](#how-env-works-internally)
- [Syntax](#syntax)
- [Understanding the Output](#understanding-the-output)
- [The Environment Block — How Variables Flow to a Child Process](#the-environment-block--how-variables-flow-to-a-child-process)
- [Modifying the Environment Correctly](#modifying-the-environment-correctly)
- [All Key Options](#all-key-options)
- [env and /proc/self/environ](#env-and-procselfenviron)
- [env vs printenv vs export vs set vs unset](#env-vs-printenv-vs-export-vs-set-vs-unset)
- [Related Commands](#related-commands)

---

## What is env?

`env`, invoked with **no arguments**, prints every variable in the calling process's environment, one `NAME=VALUE` pair per line. Invoked **with a command**, it runs that command with a modified copy of the environment — variables added, overridden, or stripped entirely — without touching the invoking shell's own environment at all.

```bash
env
# HOME=/home/alice
# PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
# SHELL=/bin/bash
# LANG=en_US.UTF-8
# TERM=xterm-256color
# ...
```

**Why `env` exists at all, rather than just invoking a command directly:**

1. **Shebang interpreter resolution.** `#!/usr/bin/env python3` locates `python3` by searching `PATH`, instead of hardcoding a fixed interpreter path (`/usr/bin/python3`) that may not exist or may be at a different location on another machine.
2. **Controlled, reproducible environments.** `env -i FOO=bar somecmd` runs `somecmd` with an exact, minimal, known environment — useful for testing, sandboxing, and CI reproducibility.
3. **One-off variable injection without shell state pollution.** Setting a variable for a single child process without `export`-ing it into the current shell's future.

---

## Where does env live?

```
/usr/bin/env
```

```bash
which env
env --version
# env (GNU coreutils) 9.4
```

Part of **GNU coreutils** on most Linux distributions (BusyBox and BSD/macOS ship their own, more limited implementations — flag support differs; see [All Key Options](#all-key-options)).

---

## How env works internally

`env` builds a **modified copy of the environment array** in memory, then replaces its own process image with the target command via `execvp()` — it does not fork an additional child; the `env` process itself becomes the target command.

Build order:

1. Start from the inherited environment (or an **empty** one if `-i`/`--ignore-environment` is given).
2. Apply any `-u NAME` removals.
3. Apply any `NAME=VALUE` assignment arguments (added or overriding existing values).
4. `execvp(COMMAND, ARGS, resulting_environment)` — replace the process image.
5. If no `COMMAND` was given, skip step 4 entirely and print the resulting environment to stdout instead, one entry per line, then exit `0`.

```bash
strace -f -e trace=execve env FOO=bar printenv FOO
# execve("/usr/bin/env", ["env", "FOO=bar", "printenv", "FOO"], ...) 
# execve("/usr/bin/printenv", ["printenv", "FOO"], ["FOO=bar", ...]) 
```

Because it's a single `execve()` replacing the process, `ps` will show `printenv` in that example — not a lingering `env` parent — once the exec completes.

---

## Syntax

```bash
env [OPTION]... [-] [NAME=VALUE]... [COMMAND [ARG]...]
```

- The optional `-` is equivalent to `-i` for POSIX compatibility.
- `NAME=VALUE` arguments must come **before** `COMMAND`; anything after `COMMAND` is passed through as its own arguments untouched.

---

## Understanding the Output

```bash
env
# HOME=/home/alice
# LANG=en_US.UTF-8
# PATH=/usr/local/bin:/usr/bin:/bin
```

| Property | Behavior |
|---|---|
| Format | One `NAME=VALUE` pair per line |
| Ordering | **Not sorted** — reflects internal `environ` array order (inheritance/insertion order), not alphabetical |
| Values containing `=` | Only the *first* `=` is the delimiter; everything after is part of the value |
| Values containing newlines | Break naive line-based parsing — see [edge-cases.md](edge-cases.md) |
| Exit status | `0` on success when just printing; command's own exit status when a command was run |

---

## The Environment Block — How Variables Flow to a Child Process

A child process's environment is a **copy** of the environment array handed to it at `exec` time — not a live link back to the parent. `env` lets you construct that copy explicitly before the final `exec`:

```bash
env                     # inherited environment, unmodified, just printed
env FOO=bar cmd         # inherited environment + FOO=bar override, then exec cmd
env -u FOO cmd          # inherited environment minus FOO, then exec cmd
env -i cmd              # completely empty environment, then exec cmd
env -i FOO=bar cmd      # empty environment containing ONLY FOO=bar, then exec cmd
```

Critically: **none of this ever affects the invoking shell.** `env` is not a shell builtin — it is a separate process, and its environment modifications are scoped only to the command it execs.

---

## Modifying the Environment Correctly

Combine `-i` with explicit `VAR=value` pairs to build a minimal, fully controlled environment — the standard pattern for reproducible testing or for running an untrusted/third-party binary without leaking unrelated inherited state (proxy variables, credentials, `LD_PRELOAD`, etc.):

```bash
env -i PATH=/usr/bin HOME=/home/alice bash -c 'echo "$PATH"; echo "$SECRET_TOKEN"'
# /usr/bin
#                          ← SECRET_TOKEN is unset; nothing inherited survived
```

Use `-u` to remove only specific sensitive variables while keeping everything else intact:

```bash
env -u AWS_SECRET_ACCESS_KEY -u AWS_ACCESS_KEY_ID ./deploy-script.sh
```

---

## All Key Options

| Option | Long form | Description |
|---|---|---|
| `-i` | `--ignore-environment` | Start the child from a completely empty environment |
| `-u NAME` | `--unset=NAME` | Remove `NAME` from the environment before exec (repeatable) |
| `-0` | `--null` | NUL-terminate output entries instead of newline (safe for values containing embedded newlines) |
| `-C DIR` | `--chdir=DIR` | `chdir(DIR)` before running the command |
| `-S STRING` | `--split-string=STRING` | Split `STRING` on whitespace into multiple arguments (GNU coreutils ≥ 8.30 — enables multi-argument shebangs) |
| `-v` | `--verbose` | Print each action (unset/set) to stderr as it happens |
| — | `--default-signal[=SIG]` | Reset a signal's disposition to the default before exec (coreutils ≥ 8.30) |
| — | `--ignore-signal[=SIG]` | Set a signal to be ignored before exec |
| `-P DIR` | `--search-path=DIR` | Use `DIR` list instead of `PATH` for locating the command (recent GNU addition) |
| `-V` | `--version` | Print version |
| — | `--help` | Print usage help |

> Portability note: `-S`, `--default-signal`, `--ignore-signal`, and `-P` are **GNU-only** additions from relatively recent coreutils releases. BusyBox's `env` and BSD/macOS `env` support only the POSIX-baseline set (`-i`, `-u`, and plain assignments) — do not assume these exist on non-GNU targets.

---

## env and /proc/self/environ

```bash
cat /proc/self/environ | tr '\0' '\n'
# HOME=/home/alice
# PATH=/usr/local/bin:/usr/bin:/bin
# ...
```

`/proc/[pid]/environ` is the kernel's raw, **NUL-separated** record of a process's exec-time environment — contrast with `env`'s own default newline-separated, human-readable formatting. `env -0` produces output in the same NUL-delimited format, which is the correct choice when environment values may contain embedded newlines and the output needs to be parsed reliably by another program (e.g., piped into `xargs -0`).

Access to `/proc/[pid]/environ` is restricted to the process's own UID (or root) under normal `ptrace_scope` settings — a relevant boundary to know when reasoning about whether environment-stored secrets are exposed to other users on a shared host. See [edge-cases.md](edge-cases.md) for the security implications in more depth.

---

## env vs printenv vs export vs set vs unset

| Tool | Type | Scope | Behavior |
|---|---|---|---|
| `env` | External binary | Only the command it execs | Prints full environment, or launches a child with a modified copy |
| `printenv [NAME]` | External binary | Read-only | Prints one named variable, or all variables if none given — no modification capability |
| `export NAME=VALUE` | Shell builtin | Current shell + all its **future** children | Marks a shell variable for inheritance by subsequently spawned processes |
| `set` (bash, no args) | Shell builtin | Current shell | Lists **all** shell variables and functions — exported and non-exported — much broader than `env` |
| `unset NAME` | Shell builtin | Current shell | Permanently removes a variable from the running shell itself |

The distinction that trips people up most: `env -u NAME cmd` only affects the **one child process being launched** — it does not touch the invoking shell's own variables. `unset NAME` in the shell, by contrast, removes it from the shell permanently going forward.

---

## Related Commands

| Command | Relation |
|---|---|
| `printenv` | Simpler, read-only variant for inspecting variables |
| `export` | Shell builtin for making a variable inherited by future children of the *current* shell |
| `set` / `unset` | Shell builtins for listing/removing shell variables (broader scope than `env`) |
| `xargs -0` | Commonly paired with `env -0` for NUL-safe processing |
| `chroot` / `nsenter` | Heavier isolation tools, often combined with `env -i` for sandboxed execution |
| `sudo -E` | Preserves the caller's environment across a privilege change — the inverse concern of `env -i` |
| `su -` | Starts a login shell with a *fresh* environment, conceptually similar to `env -i` at a larger scope |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
