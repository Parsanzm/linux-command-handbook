# sed — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Substitution Command](#substitution-command)
- [Addresses](#addresses)
- [Regular Expressions](#regular-expressions)
- [In-Place Editing](#in-place-editing)
- [Hold Space & Multi-line](#hold-space--multi-line)
- [sed vs Other Tools](#sed-vs-other-tools)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does sed stand for, and what's its core execution model?**
> **Stream EDitor.** sed reads input one line at a time into a temporary buffer called the **pattern space**, applies its script of commands to that line, prints the result (unless suppressed), then clears the buffer and repeats for the next line. It never loads the entire file into memory, making it efficient for very large files or streaming input.

---

**Q2 🔥 Why is sed described as "non-interactive," and why does that matter?**
> Unlike an editor like `vim`, sed requires no human interaction during its run — the entire transformation is specified upfront as a script, so the exact same command reliably produces the exact same output every time. This makes it ideal for automation: build scripts, CI pipelines, batch file processing, and anywhere a repeatable, scriptable transformation is needed.

---

**Q3. What is the "pattern space" in sed?**
> A temporary buffer holding the **current line** being processed. sed's commands operate on whatever is currently in the pattern space; by default, after all commands in the script run against it, its contents are printed to output and it's cleared before the next line is read in.

---

## Substitution Command

**Q4 🔥 Write a sed command to replace all occurrences of "cat" with "dog" in a file, printing to stdout.**
> ```bash
> sed 's/cat/dog/g' file.txt
> ```
> The `g` flag is essential — without it, only the **first** occurrence per line is replaced, not all of them.

---

**Q5. What's the difference between `sed 's/cat/dog/'` and `sed 's/cat/dog/g'`?**
> Without `g`, sed replaces only the **first** match of "cat" on each line. With `g` (global), it replaces **every** match on each line, not just the first.

---

**Q6 🔥 How would you replace only the 2nd occurrence of a pattern on each line, and also every occurrence after that?**
> ```bash
> sed 's/cat/dog/2g'
> ```
> The number specifies the starting occurrence, and combining it with `g` means "from the Nth occurrence onward," not just that single one.

---

**Q7. How do you perform a case-insensitive substitution in GNU sed?**
> Append the `I` flag: `sed 's/error/ERROR/gI'` matches "error", "Error", "ERROR", etc., regardless of case.

---

**Q8 🔥 Why might you use a delimiter other than `/` in a sed substitution, and how do you do it?**
> When the pattern or replacement itself contains slashes (very common with file paths), using `/` as the delimiter requires escaping every literal slash, hurting readability. sed allows almost any character to serve as the delimiter instead: `sed 's#/usr/local#/opt#'` or `sed 's|/var/www|/srv/web|'` — no escaping needed for the paths themselves.

---

## Addresses

**Q9 🔥 How do you apply a sed command to only a specific range of lines?**
> ```bash
> sed '2,5s/foo/bar/' file.txt     # lines 2 through 5 only
> sed '10,$s/foo/bar/' file.txt    # line 10 through the end of the file
> ```

---

**Q10. How would you apply a substitution only to lines matching a pattern, rather than by line number?**
> ```bash
> sed '/START/s/foo/bar/' file.txt
> ```
> This restricts the substitution to lines that contain "START," using a pattern-based address instead of (or in addition to) a numeric one.

---

**Q11 🔥 How do you apply a command to every line EXCEPT ones matching a certain pattern?**
> Use `!` to negate the address: `sed '/DEBUG/!s/foo/bar/'` applies the substitution to every line that does **not** contain "DEBUG".

---

**Q12. How would you delete every line between two markers, e.g., everything between a line containing "BEGIN" and a line containing "END" (inclusive)?**
> ```bash
> sed '/BEGIN/,/END/d' file.txt
> ```
> This uses a pattern-to-pattern range address, deleting from the first matching "BEGIN" line through the next matching "END" line.

---

## Regular Expressions

**Q13 🔥 What's the difference between Basic Regular Expressions (BRE) and Extended Regular Expressions (ERE) in sed?**
> In BRE (sed's default mode), metacharacters like `+`, `?`, `|`, and grouping parentheses need a backslash to have their special meaning (`\+`, `\?`, `\|`, `\( \)`). In ERE (enabled with `-E` or `-r`), those same characters work directly without escaping (`+`, `?`, `|`, `( )`), matching the syntax most people expect from modern regex.

---

**Q14. Write a sed command using ERE to replace one or more digits with the word "NUM".**
> ```bash
> sed -E 's/[0-9]+/NUM/g' file.txt
> ```
> Without `-E`, the equivalent in BRE would require escaping the plus sign: `sed 's/[0-9]\+/NUM/g'`.

---

**Q15 🔥 What does `&` represent in a sed replacement string, and give an example use.**
> `&` refers to the **entire matched text**, letting you wrap or reuse it without retyping the pattern. Example: `echo "hello" | sed 's/hello/[&]/'` produces `[hello]`.

---

**Q16. How do capture groups and backreferences work in sed? Give an example that swaps two words.**
> Groups are captured with `\( \)` (BRE) or `( )` (ERE), and referenced in the replacement with `\1`, `\2`, etc.
> ```bash
> echo "John Smith" | sed -E 's/(.*) (.*)/\2 \1/'
> # Smith John
> ```

---

## In-Place Editing

**Q17 🔥 What does `sed -i` do, and why should you almost always create a backup when using it?**
> `-i` edits the file **in place**, overwriting the original rather than just printing the transformed result to stdout. Since this permanently modifies the file with no built-in undo, it's good practice to always create a backup: `sed -i.bak 's/foo/bar/' file.txt` writes the original untouched content to `file.txt.bak` before applying changes to `file.txt`.

---

**Q18. Why does `sed -i 's/foo/bar/' file.txt` fail or behave unexpectedly on macOS, even though it works fine on Linux?**
> macOS ships BSD sed, not GNU sed. BSD sed's `-i` **requires an explicit argument** (even an empty string for "no backup"), while GNU sed treats the suffix as optional. On BSD sed, `sed -i 's/foo/bar/' file.txt` is misinterpreted — BSD sed tries to use `'s/foo/bar/'` itself as the required backup suffix argument, then treats `file.txt` as the script, causing an error or unintended behavior. The portable fix across both is always supplying an explicit suffix: `sed -i.bak 's/foo/bar/' file.txt` (or `sed -i '' 's/foo/bar/' file.txt` specifically on BSD/macOS for no backup).

---

## Hold Space & Multi-line

**Q19 🔥 What is the hold space, and how does it differ from the pattern space?**
> The pattern space holds the **current line** being processed and is cleared every cycle. The hold space is a **separate, persistent buffer** that survives across cycles, letting sed "remember" data from earlier lines — used for operations that need context beyond a single line, like reversing a file's line order or joining lines.

---

**Q20. What do the sed commands h, H, g, G, and x do with respect to the hold space?**
> - `h` — copy pattern space **to** hold space (overwrite)
> - `H` — append pattern space **to** hold space
> - `g` — copy hold space **to** pattern space (overwrite)
> - `G` — append hold space **to** pattern space
> - `x` — exchange/swap the pattern space and hold space entirely

---

**Q21 🔥 How would you reverse the order of all lines in a file using only sed (no `tac`)?**
> ```bash
> sed -n '1!G;h;$p' file.txt
> ```
> For every line except the first, it appends the hold space before the current line, saves the growing result back to the hold space, and only prints the fully accumulated result once the last line is reached.

---

**Q22. How would you join every pair of consecutive lines into one line using sed?**
> ```bash
> sed 'N;s/\n/ /' file.txt
> ```
> `N` appends the next line to the pattern space (separated internally by a newline), and the substitution then replaces that newline with a space, merging the two lines together.

---

## sed vs Other Tools

**Q23 🔥 When would you choose awk over sed for a text processing task?**
> When the task requires **field/column awareness** (like summing a specific CSV column) or arithmetic — awk has native support for splitting lines into fields (`$1`, `$2`, etc.) and performing calculations, while sed has no native arithmetic and only limited field-like behavior via regex capture groups. sed is generally better suited to straightforward line-based substitution, deletion, or insertion.

---

**Q24. Why might someone use grep before or instead of sed for a given task?**
> `grep` is purpose-built for **finding and filtering** lines matching a pattern — if the goal is simply "show me lines containing X," grep is simpler and clearer than using sed's print (`-n ... p`) mechanism to achieve the same result. sed becomes necessary the moment you need to actually **transform** the matched (or surrounding) text, not just display it.

---

## Scenario-Based

**Q25 🔥 You need to update a configuration value across 50 files, but want a safety net in case the change is wrong. What sed command achieves this, and why?**
> ```bash
> sed -i.bak 's/old_value/new_value/g' *.conf
> ```
> `-i.bak` performs the in-place edit while automatically preserving the original content of each file with a `.bak` suffix, so any mistake can be immediately reverted by restoring from the backups before deciding they're no longer needed.

---

**Q26. A colleague's sed command `sed 's/(.*)-(.*)/\2-\1/' data.txt` isn't swapping the two halves of each line as expected, even though the pattern looks correct. What's the likely issue?**
> They're using parentheses for grouping without enabling Extended Regular Expressions — in sed's default BRE mode, plain `(` and `)` are treated as **literal characters**, not grouping metacharacters; grouping in BRE requires escaping them as `\( \)`. The fix is either escaping them (`sed 's/\(.*\)-\(.*\)/\2-\1/'`) or adding `-E` to use ERE syntax, where unescaped parentheses work as intended: `sed -E 's/(.*)-(.*)/\2-\1/' data.txt`.

---

**Q27 🔥 You need to delete every line in a config file that is either empty or starts with a `#` (comment), in one command. How would you do it?**
> ```bash
> sed -e '/^#/d' -e '/^$/d' config.txt
> ```
> Or equivalently, combined into a single address alternation with ERE: `sed -E '/^(#|$)/d' config.txt`. Both delete lines starting with `#` and entirely blank lines in one pass.

---

**Q28. A script written and tested on Linux using `sed -i` fails when a teammate runs it on their Mac. What's happening, and what's the most portable fix?**
> This is the classic GNU sed vs BSD sed `-i` incompatibility — BSD sed (macOS's default) requires an explicit suffix argument to `-i`, while GNU sed treats it as optional; running the exact same GNU-style command on BSD sed causes an error or misinterprets the script as the suffix argument. The most portable fix is to always supply an explicit suffix in scripts meant to run on both platforms: `sed -i.bak 's/foo/bar/' file.txt`, which behaves consistently (and safely, with an automatic backup) on both GNU and BSD sed.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
