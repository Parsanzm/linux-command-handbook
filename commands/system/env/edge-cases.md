# env — Edge Cases & Gotchas

> `env` looks trivial — print or forge an environment — until shebang
> interpreter resolution, PATH hijacking, and environment-as-code history
> (Shellshock) all quietly become attack surface.

---

## Table of Contents

- [`#!/usr/bin/env` Shebangs Search PATH — And That's Both the Point and the Risk](#usrbinenv-shebangs-search-path--and-thats-both-the-point-and-the-risk)
- [`-S`/`--split-string` Is a GNU-Only Fix for a Kernel-Level Shebang Limitation](#-s--split-string-is-a-gnu-only-fix-for-a-kernel-level-shebang-limitation)
- [Environment Values Containing Function Definitions — the Shellshock Case Study](#environment-values-containing-function-definitions--the-shellshock-case-study)
- [`env -i` Wipes PATH Too — "Command Not Found" Confusion](#env--i-wipes-path-too--command-not-found-confusion)
- [env's Output Order Is Not Sorted or Guaranteed Stable](#envs-output-order-is-not-sorted-or-guaranteed-stable)
- [Embedded Newlines in Values Break Naive Parsing — Use `-0`](#embedded-newlines-in-values-break-naive-parsing--use--0)
- [`env VAR=value command` Only Affects the Child, Never Your Shell](#env-varvalue-command-only-affects-the-child-never-your-shell)
- [`VAR=value command` Without `env` Behaves Differently Depending on Whether `command` Is Present](#varvalue-command-without-env-behaves-differently-depending-on-whether-command-is-present)
- [`/proc/[pid]/environ` Exposes a Running Process's Full Environment — Including Secrets](#procpidenviron-exposes-a-running-processs-full-environment--including-secrets)
- [A Writable Directory Earlier in PATH Can Hijack What env Resolves To](#a-writable-directory-earlier-in-path-can-hijack-what-env-resolves-to)
- [GNU env's Signal-Handling Flags Are Recent and Non-Portable](#gnu-envs-signal-handling-flags-are-recent-and-non-portable)
- [An Empty env Output Might Mean a Prior `env -i` Ran, Not That the Feature Is Broken](#an-empty-env-output-might-mean-a-prior-env--i-ran-not-that-the-feature-is-broken)

---

## `#!/usr/bin/env` Shebangs Search PATH — And That's Both the Point and the Risk

### Portability vs. predictability is a real trade-off, not a solved problem
```bash
#!/usr/bin/env python3
# Portable: resolves wherever python3 happens to live on THIS machine —
# no hardcoded /usr/bin/python3 assumption.

# ⚠️ But this ALSO means the script's interpreter is determined entirely
# by whatever "python3" resolves to via the CALLER's PATH at execution
# time — not a fixed, auditable, absolute path. If PATH has been
# tampered with (a malicious entry prepended, a user-writable directory
# earlier in the search order, a wrapper shimmed in via a package
# manager), the shebang will silently run a DIFFERENT interpreter binary
# than the one the script's author intended — without any error or warning.

# Contrast with a hardcoded shebang:
#!/usr/bin/python3
# Less portable across machines, but not subject to PATH-based substitution.

# For security-sensitive scripts (setuid-adjacent tooling, anything run
# by a higher-privileged account, CI runners executing untrusted PATHs),
# treat PATH-based shebang resolution as an explicit trust boundary —
# don't assume "#!/usr/bin/env X" resolves to the X you expect without
# verifying PATH first.
which -a python3
```

---

## `-S`/`--split-string` Is a GNU-Only Fix for a Kernel-Level Shebang Limitation

### The kernel's binfmt_script parser only ever split a shebang line into interpreter + ONE argument
```bash
#!/usr/bin/env python3 -u
# ⚠️ Historically, the Linux kernel passes the ENTIRE remainder of the
# shebang line ("python3 -u") as a SINGLE literal argument to env — env
# then tries to find a program literally named "python3 -u" (with a
# space in it) and fails with "No such file or directory", because it
# never gets the chance to split that string into separate arguments itself.

# The fix (GNU coreutils >= 8.30) is the -S flag, which tells env to
# split its own argument string on whitespace before exec'ing:
#!/usr/bin/env -S python3 -u
# This now correctly resolves to: python3 -u <script args...>

# ⚠️ -S is a GNU-only addition. BusyBox env and BSD/macOS env do NOT
# support it — a shebang relying on -S is non-portable and will fail
# outright on those systems. Don't assume -S is universally available.
```

---

## Environment Values Containing Function Definitions — the Shellshock Case Study

### A historical illustration of why environment variables are not inert data
```bash
# CVE-2014-6271 ("Shellshock"): certain bash versions, upon STARTING,
# scanned inherited environment variables for values that LOOKED like a
# function definition, and — critically — continued parsing and
# EXECUTING any shell commands trailing after that function body, rather
# than treating the value as inert string data.

# The classic detection/reproduction pattern against a vulnerable bash:
env x='() { :;}; echo VULNERABLE' bash -c "echo this is a test"
# On a PATCHED bash: only "this is a test" prints — the env value is
# treated as inert data, exactly as expected.
# On an (old, unpatched) VULNERABLE bash: "VULNERABLE" ALSO prints,
# because bash executed the trailing "echo VULNERABLE" as a real command
# purely because it arrived as part of an environment variable's value.

# ⚠️ The broader lesson for a security engineer: environment variables
# are attacker-influenceable input whenever they cross a trust boundary
# (CGI scripts, subprocess-spawning services, containers built from
# less-trusted base layers) — and any downstream interpreter that
# "imports" or evaluates env values as anything other than opaque
# strings can turn that data into arbitrary code execution. Modern bash
# is not vulnerable to this specific CVE, but the underlying category of
# bug (environment content re-parsed as code) is worth threat-modeling
# generally, not just as a historical curiosity.
```

---

## `env -i` Wipes PATH Too — "Command Not Found" Confusion

### Stripping the environment strips PATH along with everything else
```bash
env -i ls
# env: 'ls': No such file or directory
# ⚠️ PATH is gone along with every other inherited variable, so the
# shell/exec mechanism has nothing to search — "ls" can't be located
# even though it obviously exists on the system.

# Either supply the absolute path directly:
env -i /bin/ls

# Or explicitly restore a minimal PATH as part of the -i invocation:
env -i PATH=/usr/bin:/bin ls
```

---

## env's Output Order Is Not Sorted or Guaranteed Stable

### Don't rely on line position when scripting against `env`'s output
```bash
env
# Order reflects the internal `environ` array (roughly inheritance /
# insertion order across the process's ancestry), NOT alphabetical order
# and NOT necessarily identical across separate invocations or machines.

# ⚠️ A script that does something like "the 3rd line of env output is
# always PATH" is fragile and will silently break. If you need a
# specific variable, filter for it explicitly rather than assuming
# position:
env | grep '^PATH='
# or, more robustly, sort first if a stable comparison is genuinely needed:
env | sort
```

---

## Embedded Newlines in Values Break Naive Parsing — Use `-0`

### A variable's value can legally contain a literal newline character
```bash
export MULTILINE_VAR=$'first line
second line'

env | grep -A1 MULTILINE_VAR
# MULTILINE_VAR=first line
# second line
# ⚠️ Line-based tools (grep, awk, a naive `while read` loop) can no
# longer tell where one variable's entry ends and the next begins,
# because the embedded newline looks identical to the delimiter between
# separate NAME=VALUE entries.

# The NUL-delimited form removes the ambiguity entirely, since NUL bytes
# cannot legally appear inside an environment variable's value:
env -0 | tr '\0' '\n'   # for human inspection only — reintroduces the
                        # ambiguity visually, but is fine for manual reading

env -0 | xargs -0 -n1 echo   # for actual machine parsing, keep it NUL-safe
```

---

## `env VAR=value command` Only Affects the Child, Never Your Shell

### A common point of confusion for people newer to the distinction between processes
```bash
env DEBUG=1 ./some-script.sh    # some-script.sh sees DEBUG=1

echo "$DEBUG"
# (empty) — your CURRENT shell never had DEBUG set at all; env's
# modification was scoped entirely to the one child process it exec'd,
# which is now a completely separate, already-finished process.

# To make a variable available to your shell (and everything it spawns
# from now on), you need the shell's own `export` builtin instead:
export DEBUG=1
./some-script.sh   # now sees DEBUG=1, because it inherited it normally
```

---

## `VAR=value command` Without `env` Behaves Differently Depending on Whether `command` Is Present

### The shell's own assignment-prefix syntax looks identical to a permanent variable assignment — but isn't, as long as a command follows
```bash
FOO=bar echo "$FOO"
# bar    ← FOO is set ONLY for this one command's environment (a POSIX
# shell feature, not env itself), then gone immediately afterward

echo "$FOO"
# (empty) — confirms it did NOT leak into the current shell

# ⚠️ But drop the trailing command entirely, and the exact same-looking
# syntax becomes a PERMANENT shell variable assignment instead:
FOO=bar
echo "$FOO"
# bar    ← this time it PERSISTS in the current shell indefinitely,
# because there was no command for the assignment to be scoped to —
# it fell through to being an ordinary shell variable assignment.
```

---

## `/proc/[pid]/environ` Exposes a Running Process's Full Environment — Including Secrets

### A concrete, frequently-cited reason not to pass credentials via environment variables
```bash
cat /proc/1234/environ | tr '\0' '\n'
# Any process's full exec-time environment — including API keys,
# database passwords, or tokens passed in via env vars (a common
# antipattern) — is readable through this interface.

# ⚠️ Access is restricted to the process's own UID (or root) under
# normal ptrace_scope settings, but that's a MUCH weaker boundary than
# it sounds in shared/multi-tenant contexts: any other process running
# as that same user (a compromised sibling process, a debugger attached
# by an operator, a core dump written to disk and later read by someone
# else, or a container escape that lands you at the same UID) can read
# it just as easily as the owning process itself.

# This is a primary reason security guidance generally recommends
# dedicated secrets-management mechanisms (files with restrictive
# permissions, a secrets manager, short-lived injected tokens via a
# vault agent) over long-lived credentials sitting in plain environment
# variables for the lifetime of a process.
```

---

## A Writable Directory Earlier in PATH Can Hijack What env Resolves To

### Classic PATH-hijacking, made directly relevant because shebangs depend on PATH resolution
```bash
echo "$PATH"
# .:/usr/local/bin:/usr/bin:/bin
# ⚠️ If "." (the current directory) or any world/group-writable
# directory appears BEFORE the legitimate system directories in PATH,
# an attacker who can drop a file into that directory — named the same
# as a common interpreter or utility (python3, sh, ls) — can cause any
# PATH-resolving invocation (including env's own command lookup, and by
# extension every "#!/usr/bin/env X" shebang) to silently execute their
# planted binary instead of the real one.

# Practical mitigations: never include "." in PATH (especially for
# root or any privileged account), audit PATH ordering in any script
# or shebang-security review, and prefer absolute-path shebangs over
# PATH-resolving ones in genuinely privileged execution contexts.
```

---

## GNU env's Signal-Handling Flags Are Recent and Non-Portable

### `--default-signal` and `--ignore-signal` only exist on modern GNU coreutils
```bash
env --default-signal=PIPE somecmd
# Resets SIGPIPE's disposition to the default before exec'ing somecmd —
# useful in cases where the invoking shell had altered a signal's
# disposition in a way the child doesn't expect (a known class of subtle
# bug when piping output from a shell-launched utility).

# ⚠️ These flags were added in coreutils >= 8.30 / refined further in
# 9.x releases. They are absent from BusyBox env, and absent from
# BSD/macOS env entirely — a script relying on them will fail with an
# "unrecognized option" error on those systems. Verify target platform
# coreutils version before depending on this behavior in portable scripts.
```

---

## An Empty env Output Might Mean a Prior `env -i` Ran, Not That the Feature Is Broken

### Context matters when a nested/wrapped process's environment looks unexpectedly bare
```bash
# If a wrapper script or CI step already did something like:
#   env -i PATH=/usr/bin ./launch-inner-tool.sh
# then INSIDE launch-inner-tool.sh, running a plain `env` will show only
# that minimal, deliberately-stripped set — NOT the full environment you
# might expect from the top-level invocation.

# ⚠️ Don't conclude "env isn't working" or "something ate my
# environment variables" without first checking whether an EARLIER
# stage in the invocation chain intentionally stripped it via `env -i`
# (or an equivalent mechanism like `sudo` without `-E`, or a container
# entrypoint that resets the environment). Trace the full process
# ancestry, not just the immediate invocation, when environment content
# looks unexpectedly sparse.
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
