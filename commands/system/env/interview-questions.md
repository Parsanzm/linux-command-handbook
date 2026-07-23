# env — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Environment Manipulation](#environment-manipulation)
- [Internals](#internals)
- [env vs Other Tools](#env-vs-other-tools)
- [Security-Focused](#security-focused)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does `env`, run with no arguments, do?**
> Prints every variable in the calling process's environment, one `NAME=VALUE` pair per line, in whatever order the internal environment array happens to hold them.

---

**Q2 🔥 What does `env` do differently when given a command as an argument?**
> It builds a modified copy of the environment (applying any `-i`, `-u`, and `NAME=VALUE` arguments), then replaces its own process image with that command via `execvp()`, running it under the modified environment — without ever affecting the invoking shell's own environment.

---

**Q3. Why is `#!/usr/bin/env python3` preferred over `#!/usr/bin/python3` in many scripts?**
> It resolves `python3` by searching `PATH` at execution time rather than hardcoding a fixed absolute path — making the script portable across systems where the interpreter lives in a different location (a different OS, a virtualenv, a version manager's shim directory, etc.).

---

## Environment Manipulation

**Q4 🔥 What's the difference between `env FOO=bar somecmd` and running `export FOO=bar` then `somecmd`?**
> `env FOO=bar somecmd` scopes `FOO` only to that single child process — the current shell never has `FOO` set at all. `export FOO=bar` sets it in the current shell and makes it inherited by every subsequently spawned child from that point forward, persisting beyond just one command.

---

**Q5. What does `env -i` do, and what's the most common mistake people make when using it?**
> It starts the child process from a completely empty environment, discarding everything inherited — including `PATH`. The common mistake is forgetting that `PATH` is gone too, so subsequent commands referenced by name (not absolute path) fail with "No such file or directory" unless `PATH` is explicitly restored as part of the same invocation.

---

**Q6 🔥 How would you remove just one specific inherited variable before running a command, while keeping the rest of the environment intact?**
> `env -u VARNAME command` — this unsets only `VARNAME` from the copy of the environment passed to the child, leaving every other inherited variable untouched.

---

## Internals

**Q7. Does `env` fork a child process, or does something else happen mechanically?**
> Neither in the usual "spawn and wait" sense — `env` calls `execvp()`, which **replaces its own process image** with the target command. There's no separate lingering `env` process in the tree once the exec completes; `ps` will show the target command itself, not `env` as a parent.

---

**Q8 🔥 Where does the kernel/OS actually store a process's environment, and how can you inspect it directly without `env`?**
> `/proc/[pid]/environ` on Linux holds the raw, NUL-separated environment a process was exec'd with. It can be read directly, e.g. `cat /proc/1234/environ | tr '\0' '\n'`, bypassing `env` entirely.

---

**Q9. Why does `env`'s default output use newlines to separate entries, and when is that insufficient?**
> Newline separation is human-readable and fine for values that don't themselves contain a newline character. It becomes ambiguous/unsafe for machine parsing when a variable's value legitimately contains an embedded newline — at that point `env -0` (NUL-terminated output) is the correct, unambiguous choice, since NUL bytes can't legally appear inside an environment variable value.

---

## env vs Other Tools

**Q10 🔥 What's the difference between `env` and `printenv`?**
> `printenv` is strictly read-only — it prints one named variable (or all, if none specified) and has no capability to modify or launch a process with a changed environment. `env` can both print the environment *and* launch a command under a modified copy of it.

---

**Q11. What's the difference between `env` and the shell's `set` builtin (in bash)?**
> `set` (with no arguments, in bash) lists **all** shell variables and functions currently defined — exported or not. `env` shows only variables that are actually part of the process environment (i.e., exported), and reflects that scope only, not the shell's full internal variable/function namespace.

---

**Q12 🔥 If `unset FOO` is a shell builtin, why would anyone use `env -u FOO command` instead?**
> `unset FOO` permanently removes `FOO` from the *current, running shell* going forward. `env -u FOO command` only strips `FOO` for that one child process being launched — the current shell's own `FOO` (if any) is completely untouched and still exists afterward.

---

## Security-Focused

**Q13 🔥 Why is storing secrets (API keys, passwords, tokens) in environment variables considered risky practice?**
> A process's full environment is generally readable via `/proc/[pid]/environ` by anything running as the same UID (or root) — a compromised sibling process, an attached debugger, or a core dump written to disk can all read it. Environment variables also tend to be inherited by every child process and can leak into logs, crash reports, or error-tracking tools that dump process state. Dedicated secrets-management mechanisms (short-lived injected tokens, restrictively-permissioned files, a vault agent) are generally preferred.

---

**Q14. What was the "Shellshock" vulnerability (CVE-2014-6271), and what does it illustrate about environment variables generally?**
> Certain bash versions, on startup, would scan inherited environment variables for values resembling a function definition and — critically — continue executing any shell commands trailing after that function body, rather than treating the value as inert data. It illustrates that environment variables are attacker-influenceable input wherever they cross a trust boundary, and that any downstream interpreter re-parsing env values as something other than opaque strings can turn that data into code execution.

---

**Q15 🔥 How could a malicious PATH entry turn a harmless `#!/usr/bin/env python3` shebang into a code-execution vector?**
> If a writable-by-attacker directory appears earlier in `PATH` than the legitimate interpreter's location (including a dangerous but historically common case: `.` present in `PATH`), an attacker who can place a file named `python3` in that directory causes the shebang's PATH-based resolution to silently execute their planted binary instead of the real interpreter — with no error or warning, since resolution succeeded, just against the wrong file.

---

**Q16. Why is `#!/usr/bin/env -S python3 -u` required instead of `#!/usr/bin/env python3 -u` on some systems, and what's the security-adjacent implication of getting it wrong?**
> The Linux kernel's shebang parser (`binfmt_script`) historically splits the line into only the interpreter plus a single argument — so `"python3 -u"` arrives at `env` as one literal string, which `env` then fails to locate as a program name. `-S` (GNU coreutils ≥ 8.30) tells `env` to split that string into separate arguments itself. Getting this wrong doesn't usually create a vulnerability on its own, but it's a common source of "why does this script silently fail differently on macOS vs. Linux" confusion when auditing portability across environments.

---

## Scenario-Based

**Q17 🔥 A teammate wants to run a third-party install script but is worried it might read cloud credentials from the environment. What `env` invocation would you recommend, and why?**
> `env -i PATH=/usr/bin:/bin HOME=/tmp/sandbox-home ./install-script.sh` (adjusting the minimal variable set to whatever the script genuinely needs to function) — this ensures the script starts from a deliberately empty environment containing only explicitly-approved variables, so any inherited cloud credentials, tokens, or proxy settings simply aren't present for it to read, regardless of what the script's code attempts to do.

---

**Q18. During an incident review, you find that a script used `FOO=bar echo test` at one point and, a few lines later, `FOO=bar` on its own with no trailing command — and the second one caused unexpected behavior downstream. What happened?**
> The first form (`FOO=bar echo test`) is a POSIX shell environment-assignment prefix scoped only to that single command's execution — it never persists. The second form, with no trailing command, is simply an ordinary, **permanent** shell variable assignment that persists in that shell for the rest of its life (and is inherited by everything it subsequently spawns) — almost certainly not what was intended if the author was trying to mimic the scoped behavior of the first line.

---

**Q19 🔥 You're auditing a shared multi-tenant host and want to check whether any currently-running process has an obvious secret sitting in its environment. How would you check, and what's the caveat?**
> `tr '\0' '\n' < /proc/<pid>/environ | grep -iE 'key|secret|token|password'` for a given PID (looping over PIDs owned by the account in question). The caveat: this only works for processes you already have permission to inspect (matching UID, or root) — it's a real audit technique for accounts/processes you're authorized to review, not a way to read arbitrary other users' process environments, since that boundary is enforced by the kernel under normal `ptrace_scope` settings.

---

**Q20. A CI pipeline behaves inconsistently across two build agents — one succeeds, one fails with the exact same script. `env` output on both machines looks nearly identical except for ordering. What's your first hypothesis, and why is that a red herring?**
> `env`'s output ordering is not guaranteed stable or alphabetical — it reflects internal environment-array order, which can legitimately differ between machines without any actual functional difference in the variables present. The apparent "ordering difference" is very likely a red herring; the actual investigation should focus on presence/absence/value differences of specific variables (diff the sorted output of both: `diff <(env | sort)` on each host) rather than raw line position.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
