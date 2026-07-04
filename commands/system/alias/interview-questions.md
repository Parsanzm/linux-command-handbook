# alias — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [Expansion Mechanics](#expansion-mechanics)
- [alias vs Functions](#alias-vs-functions)
- [Scripting Implications](#scripting-implications)
- [Configuration & Persistence](#configuration--persistence)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What is an alias, and is it a program or a shell feature?**
> `alias` is a **shell builtin**, not an external program — it's implemented inside the shell itself (bash, zsh, etc.), not as a file somewhere on disk. It defines a shortcut name that gets textually substituted for a longer command before the shell parses and executes the line.

---

**Q2 🔥 At what stage of command processing does alias substitution happen?**
> Alias expansion happens very early — the shell checks if the **first word** of a command line matches a defined alias name, and if so, substitutes the alias's replacement text **before** the rest of parsing, variable expansion, and execution proceeds. It's a simple text substitution, not a function call or a separate process.

---

**Q3. Why can't you see an alias by running `which aliasname`?**
> `which` looks for **executable files** on disk in `$PATH`. Since an alias is purely an in-memory shell construct (not a file), `which` finds nothing. Use `type aliasname` or `alias aliasname` instead, both of which understand shell-internal constructs like aliases, functions, and builtins.

---

## Basic Usage

**Q4 🔥 How do you create an alias, and why must the replacement usually be quoted?**
> ```bash
> alias ll='ls -alF'
> ```
> Quoting is necessary whenever the replacement contains spaces or special characters — without quotes, the shell would try to interpret each word after the `=` as a separate argument to the `alias` command itself, rather than as one single replacement string.

---

**Q5. How do you list all currently active aliases, and how do you remove one?**
> ```bash
> alias              # lists every alias defined in this session
> unalias ll         # removes the specific "ll" alias
> unalias -a         # removes ALL aliases in the current session
> ```

---

**Q6 🔥 How do you run the real, un-aliased version of a command just once, without removing the alias?**
> Three common ways:
> ```bash
> \rm file.txt          # backslash prefix skips alias lookup
> "rm" file.txt          # quoting the command name also bypasses aliases
> command rm file.txt    # the "command" builtin explicitly bypasses aliases/functions
> ```

---

## Expansion Mechanics

**Q7 🔥 If you define `alias ll='ls -alF'` and then run `ll /tmp`, what actual command executes?**
> `ls -alF /tmp`. The alias substitutes only the first word (`ll` → `ls -alF`); any arguments typed after the alias name are simply appended to the end of the expanded text — they aren't inserted anywhere else.

---

**Q8. Why does `alias sudo='sudo '` (with a trailing space) behave differently from `alias sudo='sudo'` (without one) when running `sudo someotheralias`?**
> A trailing space after an alias's replacement text tells the shell to **also** check the next word on the line for alias expansion. Without the trailing space, only the word `sudo` itself is checked (and simply passes through unchanged); the following word (`someotheralias`) is passed to the real `sudo` command literally, without being expanded — which fails if `someotheralias` isn't a real command outside your aliased shell.

---

**Q9. Can an alias reference `$1` or other positional parameters the way a function can?**
> No — not meaningfully. Any `$1`-style reference inside an alias definition is evaluated (or left empty/unset) at the time the alias is **defined**, not when it's later invoked with arguments. Arguments given when calling the alias are always just appended to the end of the expanded text, never substituted into a specific position. Genuine positional-argument handling requires a shell **function** instead.

---

**Q10 🔥 Does aliasing `ls` to `ls --color=auto` cause infinite recursive expansion?**
> No. When an alias's own replacement text starts with the same name as the alias itself, the shell specifically avoids re-expanding that occurrence again, running the real underlying command instead after one level of substitution. This prevents the most obvious infinite-loop case, though deeply chained aliases pointing to each other can still be confusing to trace manually.

---

## alias vs Functions

**Q11 🔥 When would you use a shell function instead of an alias?**
> Whenever you need to: reference positional arguments meaningfully (`$1`, `$2`), use conditional logic or loops, perform multiple dependent steps based on input, or have the shortcut work reliably inside non-interactive scripts. Aliases are best reserved for simple, static, no-logic shortcuts.

---

**Q12. If both a function and an alias exist with the same name, which one runs?**
> In bash, the alias generally takes priority over the function of the same name within an interactive session — the function is effectively shadowed and never invoked, with no warning that this conflict exists. Checking with `type name` reveals which one is actually active.

---

## Scripting Implications

**Q13 🔥 Why does an alias that works perfectly in your interactive terminal often fail inside a shell script?**
> By default, **non-interactive** bash shells (which is what runs when you execute a script file) do not perform alias expansion at all, even if the exact same alias is defined and working in the interactive shell that launched the script. This is a deliberate bash default, not a bug. Scripts needing this kind of shortcut should use a shell function instead, which works identically in both interactive and non-interactive contexts.

---

**Q14. Is there a way to force alias expansion inside a non-interactive bash script?**
> Yes: `shopt -s expand_aliases` enables it, but it must appear in the script **before** the alias is defined and used, and this approach is rarely considered good practice — using a function is the more portable, idiomatic solution for logic that needs to run inside scripts.

---

**Q15 🔥 Can you export an alias so it's available in a subshell or a script called by your interactive session, the way `export -f` works for functions?**
> No. There is no equivalent of `export` for aliases in bash — they exist only within the specific interactive shell session that defined them and cannot be inherited by child processes, subshells, or separately-launched scripts. Functions, by contrast, can be exported with `export -f functionname` and will then be available inside subshells spawned from that session.

---

## Configuration & Persistence

**Q16. Why does an alias disappear after you close your terminal, and how do you make it permanent?**
> Aliases defined directly at the command line only exist in that shell process's memory — closing the terminal ends the process and discards them. To persist an alias across sessions, add the `alias` line to a shell startup file that's read automatically on every new shell, most commonly `~/.bashrc` for bash or `~/.zshrc` for zsh.

---

**Q17 🔥 What's the difference between .bashrc and .bash_profile, and why does it matter for aliases?**
> `.bashrc` is read for interactive **non-login** shells — the common case for most terminal emulator windows on Linux. `.bash_profile` (or `.profile`) is read for **login** shells — such as a fresh SSH session or a text-console login. An alias placed only in `.bash_profile` won't be available in a typical new terminal window on many Linux setups, because that terminal opens a non-login shell that never reads `.bash_profile` at all. Best practice is either to define aliases in `.bashrc`, or have `.bash_profile` explicitly source `.bashrc`.

---

**Q18. After adding a new alias to ~/.bashrc, why doesn't it work immediately in your currently open terminal?**
> The running shell process already read and executed `.bashrc` once when it started — editing the file afterward has no automatic effect on that already-running process. You must either run `source ~/.bashrc` (or its shorthand `. ~/.bashrc`) to re-read the file into the current session, or open a brand-new terminal window, which reads the updated file fresh.

---

## Scenario-Based

**Q19 🔥 A user has `alias rm='rm -i'` set up for safety, but a cron job that runs `find /tmp -name "*.tmp" -exec rm {} \;` deletes files without any confirmation prompt ever appearing. Why doesn't the alias protect this case?**
> `find`'s `-exec` option calls the `rm` binary **directly**, bypassing the shell's alias lookup entirely — aliases only apply when a command is typed and parsed through an **interactive shell's** command-line processing, not when a program execs another binary programmatically. The alias only provides protection for someone directly typing `rm` at an interactive prompt; it offers zero protection against automated or scripted invocations of the real command.

---

**Q20. You defined a useful alias directly in your terminal, used it successfully all afternoon, then closed the terminal and it's gone the next day. What happened, and how should this have been avoided?**
> Aliases defined ad-hoc on the command line exist only in that shell process's memory and are discarded the moment the shell exits — nothing about typing `alias name='...'` writes it to any file automatically. To make it persist, the definition needed to be added to a startup file like `~/.bashrc` (or a dedicated `~/.bash_aliases` sourced from it), not just typed directly into the running terminal.

---

**Q21 🔥 A developer defines both a function `deploy() { ... }` with real logic and an alias `alias deploy='echo "not ready"'` earlier in their `.bashrc`, then wonders why running `deploy` never executes the function's logic. What's happening, and how would you diagnose it?**
> Because the alias and function share the same name, bash gives priority to the **alias**, silently shadowing the function entirely — the function's code is still defined and technically exists, but is never actually reached when `deploy` is typed. Diagnosis: run `type deploy`, which will report that `deploy` is aliased to the echo command, immediately revealing the conflict without needing to read through the entire `.bashrc` file line by line.

---

**Q22. Two different config files (`~/.bashrc` and `~/.bash_aliases`, sourced from it) both define an alias named `gs` with different commands. Which one wins, and how would you find out why `gs` isn't behaving as expected?**
> Whichever definition is read **last** during shell startup wins — if `.bash_aliases` is sourced after the `.bashrc` line that defines its own `gs`, the `.bash_aliases` version silently overrides the earlier one, with no warning about the conflict. To diagnose, search every file that could plausibly define it, in the actual order they're sourced: `grep -rn "alias gs=" ~/.bashrc ~/.bash_aliases ~/.zshrc`, and check `alias gs` in a live shell to see which definition is currently active.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
