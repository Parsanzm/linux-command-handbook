# man — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Manual Sections](#manual-sections)
- [Searching & Discovery](#searching--discovery)
- [Internals & Rendering](#internals--rendering)
- [man vs Other Documentation Tools](#man-vs-other-documentation-tools)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What is `man`, and why has it remained relevant despite the availability of internet search?**
> `man` displays the manual page for a command, system call, library function, or file format, stored locally on the system. It remains valuable because it's **authoritative** (matches the exact software actually installed), **version-accurate** (reflects your specific system, not a generic or differently-versioned answer from the web), and works **entirely offline** — essential for remote administration, air-gapped systems, or whenever exact local behavior matters more than a general answer.

---

**Q2 🔥 Where do man pages actually live on the filesystem, and in what format are they typically stored?**
> They're stored under directories like `/usr/share/man/man1/`, `/usr/share/man/man5/`, etc. (one subdirectory per section), typically as **gzip-compressed** files (e.g., `ls.1.gz`) to save disk space across potentially thousands of installed pages. `man` decompresses the relevant file on the fly whenever it's viewed.

---

**Q3. What markup language are man pages written in, and what tool renders them into readable terminal text?**
> Man pages are authored using **groff's `man` macro package**, built on top of the general-purpose `troff` typesetting system. When you run `man`, it locates the source file, decompresses it if needed, and pipes it through `groff` (or the older `nroff`) to render plain, formatted text, which is then displayed through a pager (`less` by default).

---

## Manual Sections

**Q4 🔥 Why does the manual have numbered sections instead of just one flat list of pages?**
> Because the same name can refer to entirely different things depending on context — for example, `printf` is both a shell command (section 1) and a C library function (section 3), and `passwd` is both a command (section 1) to change a password and a file format (section 5) describing `/etc/passwd`'s structure. Sections disambiguate these cases so the correct, specific documentation can be requested explicitly.

---

**Q5. What's the difference between `man 1 crontab` and `man 5 crontab`?**
> `man 1 crontab` documents the **crontab command** — how to edit, list, or remove a user's scheduled jobs from the command line. `man 5 crontab` documents the **file format** of a crontab file itself — the syntax of the schedule fields, special strings, and how entries are structured — completely different content answering a completely different question.

---

**Q6 🔥 If you just run `man printf` without specifying a section, which page do you get, and why might that not be what you wanted?**
> By default, `man` searches sections in a predefined priority order (commonly checking section 1 before 3), so `man printf` typically returns the **shell command** documentation, not the C library function's exact format-string behavior and signature — which is what a C programmer usually actually needs. Explicitly requesting `man 3 printf` gets the correct library function page.

---

**Q7. Name at least four of the standard manual sections and what each contains.**
> - Section 1: user commands (e.g., `ls`, `grep`)
> - Section 2: system calls (e.g., `open`, `fork`)
> - Section 3: library calls (e.g., `printf()`, `malloc()`)
> - Section 5: file formats and conventions (e.g., `/etc/passwd`'s structure)
> - Section 8: system administration commands, typically requiring root (e.g., `iptables`, `useradd`)

---

## Searching & Discovery

**Q8 🔥 What's the difference between `whatis` and `apropos`?**
> `whatis` gives a one-line description for an **exact** command name, showing every section that has a matching page (`whatis printf` shows both the section 1 and section 3 entries). `apropos` (equivalent to `man -k`) searches the description text of **every** installed man page for a keyword, useful when you don't know the exact command name but know roughly what you're looking for (`apropos "compress files"`).

---

**Q9. Why might `apropos` or `whatis` return nothing at all, even for a command you know has a man page installed?**
> Both rely on a **pre-built index database** managed by `mandb`, not a live search of every man page's content on each invocation. If that index was never built (common on a freshly minimal-installed system, or after manually placing man page files outside the normal package manager), keyword and exact-name searches return empty even though direct lookups like `man commandname` work fine. Running `sudo mandb` rebuilds the index and fixes this.

---

**Q10 🔥 How would you search the full text (not just descriptions) of every installed man page for a specific phrase?**
> ```bash
> man -K "recursive delete"
> ```
> `-K` performs a full-text search across every man page's actual content, opening each match one at a time — slower than `apropos`'s indexed description search, but thorough, since it isn't limited to just the one-line NAME/description entries.

---

## Internals & Rendering

**Q11. What pager does man use by default, and how would you check or change it?**
> By default, `man` pipes rendered output through `less`. The pager can be checked/overridden via the `$MANPAGER` environment variable (falls back to `$PAGER` if unset), or with the `-P` command-line option (`man -P more ls`).

---

**Q12 🔥 Why does redirecting `man` output to a file (e.g., `man ls > out.txt`) sometimes produce garbled text with backspace characters instead of clean plain text?**
> `man`'s rendered output encodes bold and underlined text using classic terminal overstrike sequences (a character, a backspace, then the character again) — perfectly interpreted correctly by an interactive terminal, but ugly and confusing when captured to a plain file and viewed with a tool that doesn't process those control sequences the same way. Piping through `col -bx` strips these formatting artifacts, producing clean plain text: `man ls | col -bx > out.txt`.

---

**Q13. How does `man` decide where to look for man pages, and how would you add a custom directory to that search path?**
> It uses a configured search path (visible via the `manpath` command, influenced by `/etc/manpath.config` and derived partly from `$PATH`), optionally overridden by the `$MANPATH` environment variable. To add a custom directory (e.g., for a locally built tool's man page), prepend it to `$MANPATH`: `export MANPATH="$HOME/myapp/man:$(manpath)"`.

---

**Q14 🔥 Why might `man cd` fail or show unexpected results?**
> `cd` is a **shell builtin**, not a standalone executable file — there's no separate `/usr/bin/cd` binary for `man` to have a dedicated page for. Many systems either show "No manual entry for cd" or redirect toward the shell's own comprehensive man page (e.g., `man bash`), which documents its builtins (including `cd`) in a dedicated section within that larger page. Bash's own `help cd` command is a faster way to get builtin-specific documentation directly.

---

## man vs Other Documentation Tools

**Q15 🔥 When would you reach for `--help` instead of `man`, and vice versa?**
> `--help` is best for a quick reminder of exact flag syntax when you already broadly know the command and just need a fast, no-pager flag list. `man` is best when you need the complete, authoritative picture — every flag with full explanation, exit status meanings, related files, environment variables, and edge-case behavior — information that's usually far more extensive than a `--help` listing provides.

---

**Q16. In what situation might `info` be preferable to `man` for the same command?**
> For GNU tools with especially extensive documentation (classic examples include `coreutils`, `tar`, and `sed`), the `info` system sometimes contains more detailed, hyperlinked, and better-organized documentation than the corresponding man page, which may be comparatively terse. `info` is GNU-specific and not universally available or as consistently maintained across all tools as `man`, but worth checking for GNU utilities specifically.

---

## Scenario-Based

**Q17 🔥 A developer runs `man printf` while writing C code and gets confused because the flag descriptions don't match what they expected for the `printf()` function. What went wrong, and how do they fix it?**
> `man printf` without a section number defaulted to section 1 — the **shell command** `printf`, not the C standard library function they actually needed. The fix is to explicitly request the correct section: `man 3 printf`, which documents the C library function's actual signature, format specifiers, and return value behavior.

---

**Q18. A sysadmin runs `apropos network` on a freshly provisioned minimal server and gets "nothing appropriate" even though `man ss` and `man netstat` both work fine when called directly. What's the likely cause, and how do they fix it?**
> The `mandb` search index (which `apropos`/`whatis` depend on) was likely never built on this fresh system — direct `man` lookups work because they don't need the index at all, they just locate and render the specific requested file, but keyword search across descriptions requires the pre-built database. Running `sudo mandb` builds/rebuilds the index, after which `apropos network` should return results correctly.

---

**Q19 🔥 Inside a minimal Docker container, running `man ls` returns "command not found," even though `ls` itself works perfectly. Is this a bug, and what's the fix?**
> Not a bug — minimal container base images very commonly strip out the `man-db` package and all man page files entirely to reduce image size, since documentation isn't needed for a container's actual runtime function. This is expected, intentional slimming, not a sign that anything is broken with `ls` or the container image. The fix, if documentation access inside the container is genuinely needed, is to explicitly install it: `apt-get install -y man-db manpages` (or the equivalent for the container's package manager).

---

**Q20. A team member says a man page's documented behavior for a flag doesn't match what they're observing when actually running the tool. Before assuming the man page is simply wrong, what should they check first?**
> Whether the installed binary and the installed man page actually come from the **same version/source** — it's common for a man page to lag behind if the binary was manually built from source, updated via a language-specific package manager (pip/npm/cargo, which typically don't install or update man pages at all), or if a newer binary was dropped into `/usr/local/bin` while the older distro-packaged man page remained in place. Cross-checking `--version`, `--help`, and `man -w toolname` (to see exactly where the displayed page is coming from) helps confirm whether there's a genuine version mismatch rather than an actual documentation error.

---

**Q21. Why might the same `man ls` command display different languages on two different servers, and how would you force English output on demand without changing the whole system's locale settings?**
> `man` selects a translated page based on the active locale (`$LANG`/`$LC_MESSAGES`) if a translation is installed for that language; different servers with different locale configurations (or different sets of installed language packs) can therefore show different languages for the identical command. Forcing English for a single lookup without altering broader system settings: `LANG=C man ls` (overriding the locale just for that one command invocation).

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
