# wc — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [Lines, Words, Characters, Bytes](#lines-words-characters-bytes)
- [Locale & Encoding](#locale--encoding)
- [Scripting with wc](#scripting-with-wc)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does wc stand for, and what does it do?**
> **Word Count.** It counts lines, words, characters, and bytes in a file or piped input, printing the results as simple numbers — one of the most fundamental and widely-used text-processing utilities in Unix.

---

**Q2 🔥 Why is wc so commonly used in shell scripts despite its simplicity?**
> Counting things — matching lines, files, results — is one of the most frequent needs in shell scripting, and `wc` provides the fastest, most direct way to get a count from virtually any command's output via a pipe (`command | wc -l`), without needing to write custom counting logic.

---

## Basic Usage

**Q3 🔥 What's the default output of running `wc file.txt` with no options?**
> Four values: the line count, word count, byte count, and the filename, in that order (e.g., `  42  350 2103 file.txt`).

---

**Q4. How do you get JUST the line count of a file, without the word/byte counts or the filename?**
> ```bash
> wc -l < file.txt
> ```
> Using `<` (input redirection) instead of passing the file as an argument suppresses the filename from the output, leaving just the number — often cleaner for capturing into a script variable than `wc -l file.txt`, which would include the filename alongside the count.

---

**Q5. If you run `wc` on multiple files at once, what extra information appears in the output?**
> An automatic **total** row is appended after each individual file's counts, summing the lines/words/bytes across all the listed files.

---

## Lines, Words, Characters, Bytes

**Q6 🔥 What does `wc -l` actually count — "lines of text" or something more specific?**
> It counts **newline characters** (`\n`), not "lines" in a purely visual/intuitive sense. This distinction matters because a file whose last line lacks a trailing newline will have that final line **excluded** from the count entirely, even though it's clearly visible as a line of text when viewing the file.

---

**Q7. Given a file containing "line1\nline2\nline3" (no trailing newline), what does `wc -l` report, and why?**
> `2`, not `3` — because `wc -l` counts newline characters, and there are only two `\n` characters in that string (after "line1" and after "line2"); "line3" has nothing following it, so it isn't counted at all.

---

**Q8 🔥 What's the difference between `wc -c` and `wc -m`?**
> `-c` counts **bytes**; `-m` counts **characters**, which is locale-aware and can differ from the byte count whenever multibyte encodings (like UTF-8) are involved — a single non-ASCII character (e.g., "é" or "ü") may take 2 or more bytes but still counts as exactly 1 character.

---

**Q9. Give an example where `wc -c` and `wc -m` produce different results for the same text.**
> ```bash
> echo "café" | wc -c
> # 6   (é takes 2 bytes in UTF-8, plus c-a-f + newline)
> echo "café" | wc -m
> # 5   (4 actual characters + newline)
> ```

---

**Q10 🔥 How does `wc -w` define a "word"? Does punctuation split a word into multiple words?**
> A word is simply a maximal sequence of **non-whitespace characters** — punctuation attached to other characters does NOT create a split. For example, `echo "hello, world!" | wc -w` reports `2`, since "hello," and "world!" are each treated as one whitespace-delimited token, punctuation and all.

---

**Q11. What does `wc -L` report, and when would it be useful?**
> The length (in characters) of the **longest line** in the input. It's useful for checking whether any line in a file exceeds a certain width — for example, enforcing a maximum line length as part of a coding style check.

---

## Locale & Encoding

**Q12 🔥 Why might `wc -m` give an incorrect character count in some environments?**
> If the active **locale** doesn't understand multibyte encodings (such as the plain `C`/POSIX locale, which has no concept of UTF-8 sequences), `-m` can silently fall back to essentially byte-counting behavior instead of true character-counting — producing the same result as `-c` even for genuinely multibyte text, without any warning that this degradation occurred.

---

**Q13. How would you ensure accurate character counting for non-ASCII text regardless of the environment's default locale?**
> Explicitly set a UTF-8-aware locale when invoking wc: `echo "$text" | LC_ALL=en_US.UTF-8 wc -m`, ensuring `-m` correctly interprets multibyte sequences as single characters rather than degrading to byte-counting behavior.

---

## Scripting with wc

**Q14 🔥 Why is `wc -l < file.txt` generally preferred over `wc -l file.txt` when capturing a count into a shell variable?**
> `wc -l file.txt` (passing the file as an argument) includes the filename alongside the number in its output, which then requires additional parsing (e.g., piping through `awk '{print $1}'`) before the value can be used safely in an arithmetic comparison. `wc -l < file.txt` (input redirection) produces just the bare number with no filename, making it directly usable in a variable assignment without further cleanup.

---

**Q15. Why might `grep pattern file.txt | wc -l` be considered less efficient than `grep -c pattern file.txt` for the same task?**
> `grep -c` counts matching lines **internally** within a single grep process, while `grep pattern file.txt | wc -l` requires spawning a **second**, separate `wc` process and piping data between the two — functionally equivalent in result, but `grep -c` avoids the extra process-spawning and pipe overhead entirely.

---

**Q16 🔥 What's a robust way to count files in a directory that avoids issues with unusual filenames (like ones containing embedded newlines)?**
> Standard `ls -1 | wc -l` can be fooled by a filename containing an actual embedded newline character, since `wc -l` simply counts newline occurrences regardless of what they represent — such a filename could get counted more than once. A more robust approach uses NUL-separated output (immune to this, since NUL cannot appear in a filename): `find . -maxdepth 1 -type f -print0 | tr -cd '\0' | wc -c`.

---

## Scenario-Based

**Q17 🔥 A script uses `wc -l file.txt` to check a file's line count, but `if [ "$COUNT" -eq 42 ]` fails with a "too many arguments" error. What's wrong, and how do you fix it?**
> `wc -l file.txt` outputs both the count AND the filename, so `$COUNT` actually contains two "words" (the number and the filename) rather than a single value the numeric comparison can handle — bash's `[` interprets this as too many arguments. The fix is using input redirection instead of a file argument to suppress the filename entirely: `COUNT=$(wc -l < file.txt)`.

---

**Q18. A validation script uses `wc -c` to enforce a "200 character limit" on a user bio field, but users entering perfectly reasonable non-English text (with accented characters) are being incorrectly rejected as "too long." What's the bug, and how do you fix it?**
> `wc -c` counts **bytes**, not characters — any text containing multibyte UTF-8 characters (accented letters, non-Latin scripts, etc.) will have a byte count higher than its actual character count, causing legitimate text well within the intended character limit to be rejected. The fix is switching to `wc -m` (character count, locale-aware), and ensuring the script runs in a UTF-8-aware locale so `-m` behaves correctly rather than silently degrading to byte-counting.

---

**Q19 🔥 A developer expects `printf "a\nb\nc" | wc -l` to report 3, but it reports 2. Why, and does this indicate a bug in wc?**
> Not a bug — `wc -l` counts newline characters, and the input here has only two `\n` characters (after "a" and after "b"); "c" has no trailing newline, so it's simply never counted by this metric. This is standard, well-defined `wc` behavior, though it frequently surprises people expecting a more intuitive "count of visual lines" regardless of trailing newline presence.

---

**Q20. A script counts files with `ls | wc -l` and occasionally reports a count one higher than the actual number of files present, only in directories where files were created by a script that programmatically generates unusual filenames. What's the likely root cause?**
> One or more filenames likely contain an embedded newline character (rare, but technically valid on most filesystems) — `ls`'s default listing displays such a filename across what visually looks like two lines, and since `wc -l` simply counts newline occurrences without any awareness of what they represent, that single file gets counted twice. The robust fix is using NUL-delimited output throughout the pipeline (`find ... -print0`) instead of relying on newline-separated `ls` output for counting.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
