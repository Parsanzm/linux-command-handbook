# tee — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [sudo and Permissions](#sudo-and-permissions)
- [Exit Codes & Pipelines](#exit-codes--pipelines)
- [Buffering & Behavior](#buffering--behavior)
- [tee vs Redirection](#tee-vs-redirection)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does tee do, and where does its name come from?**
> `tee` reads from standard input and writes that same data to **both** standard output and one or more specified files simultaneously. It's named after a plumbing "T" fitting, which splits a single pipe's flow into two directions — exactly analogous to how tee splits a data stream into two destinations at once.

---

**Q2 🔥 What core problem does tee solve that plain redirection (>) cannot?**
> Plain redirection sends output to only ONE destination — either the terminal (default) or a file (`>`), but never both. `tee` lets you **simultaneously** see a command's output live in the terminal AND save a copy to a file, and additionally allows a copy to be captured mid-pipeline while the original data continues flowing to further commands.

---

## Basic Usage

**Q3 🔥 Write a command that runs `ls -la`, shows the output in the terminal, and also saves it to a file called listing.txt.**
> ```bash
> ls -la | tee listing.txt
> ```

---

**Q4. How do you make tee append to an existing file instead of overwriting it?**
> Use the `-a` (`--append`) flag: `command | tee -a logfile.txt`. Without `-a`, every invocation of tee truncates and overwrites the target file, exactly like `>` redirection would.

---

**Q5 🔥 How would you write a command's output to three different files at once?**
> ```bash
> command | tee file1.txt file2.txt file3.txt
> ```
> tee accepts any number of file arguments and writes an identical copy of the input to each one.

---

## sudo and Permissions

**Q6 🔥 Why does `sudo echo "text" > /etc/protected_file` fail with Permission denied, even though sudo is used?**
> The `>` redirection is set up by the **shell itself**, before `echo` ever runs, and the shell process performing that redirect is still your normal, unprivileged user — `sudo` only elevated the `echo` command's own execution, not the shell's separate act of opening the output file for writing.

---

**Q7. How does `echo "text" | sudo tee /etc/protected_file` correctly solve the problem above?**
> Here, `echo` runs as your normal user (harmless, since it's just producing text on stdout), while `tee` — the program that actually performs the file write — is the one invoked **with** `sudo`. Since tee itself runs as root, it has the necessary permission to write to the protected file, sidestepping the shell-redirection permission problem entirely.

---

**Q8 🔥 If you need to suppress tee's normal terminal output while still writing to a protected file via sudo, how would you do it, and does this affect the actual file write?**
> ```bash
> echo "text" | sudo tee /etc/protected_file > /dev/null
> ```
> The `> /dev/null` here redirects tee's **own stdout stream** (the copy it would otherwise print to the terminal) to nowhere — it does not affect tee's separate, independent write to the named file at all, since writing to stdout and writing to the listed file(s) are two distinct output channels for tee.

---

## Exit Codes & Pipelines

**Q9 🔥 After running `false | tee output.txt`, what does `$?` report, and why might this surprise someone?**
> `$?` reports **tee's own** exit status (typically `0`, since tee succeeded at writing the — empty — input to the file), not `false`'s exit status (`1`). This surprises people because they intend to check whether the **original** command in the pipeline succeeded, but `$?` after a pipeline always reflects the **last** command in that pipeline, which is tee.

---

**Q10. How would you correctly check whether the command BEFORE tee in a pipeline actually succeeded?**
> Use bash's `PIPESTATUS` array, which holds the exit status of every stage in the most recently executed pipeline:
> ```bash
> some_command | tee output.txt
> if [ "${PIPESTATUS[0]}" -ne 0 ]; then
>   echo "some_command actually failed"
> fi
> ```

---

## Buffering & Behavior

**Q11 🔥 Why might output "through tee" appear delayed or arrive in large chunks rather than immediately, line by line?**
> This is usually caused by the **producing command's own buffering**, not tee itself — many programs switch to fully-buffered (rather than line-buffered) output when their stdout isn't a terminal (as is the case once piped into tee), batching writes internally before tee ever receives them. Forcing line-buffering on the producer (e.g., with `stdbuf -oL producing_command | tee output.txt`) typically resolves this.

---

**Q12. What happens if the command AFTER tee in a pipeline (e.g., `head -5`) exits before tee has finished writing all its input?**
> The downstream command closing its end of the pipe early can cause tee to receive a `SIGPIPE` when it next tries to write, often terminating tee with a "Broken pipe" message on stderr. This is generally harmless — the data that WAS already passed through before the pipe closed is preserved — but the visible error message can be misinterpreted as an actual failure by scripts that treat any stderr output as a hard error.

---

## tee vs Redirection

**Q13 🔥 When would you choose `tee` over `>>` for appending to a log file?**
> When you also need to **see** the output being appended live in the terminal at the same time it's being saved, or when you need to continue **piping** that same output onward to another command for further processing — plain `>>` only saves to the file and produces no terminal output at all.

---

**Q14. Give a scenario where tee is functionally necessary and plain redirection genuinely cannot achieve the same result.**
> Writing to a root-owned file via `sudo`: `echo "setting" | sudo tee /etc/config` works because tee itself (running as root via sudo) performs the write, whereas `sudo echo "setting" > /etc/config` fails because the redirection is performed by the unprivileged calling shell, not by the sudo'd command.

---

## Scenario-Based

**Q15 🔥 A CI pipeline script does `npm run build | tee build.log` and then checks `$?` to decide whether the build succeeded, but the check always passes even when the build actually fails. What's wrong, and how do you fix it?**
> `$?` after this pipeline reflects **tee's** exit status, not `npm run build`'s — tee almost always succeeds at writing to `build.log` regardless of whether the build itself failed, so the check is effectively meaningless as written. Fix: check `${PIPESTATUS[0]}` instead, which holds `npm run build`'s actual exit code: `if [ "${PIPESTATUS[0]}" -ne 0 ]; then exit 1; fi`.

---

**Q16. A developer runs a loop that's supposed to accumulate log entries with `echo "$msg" | tee log.txt`, but at the end, log.txt only contains the LAST loop iteration's message. What went wrong?**
> Without the `-a` (append) flag, every single invocation of `tee` **overwrites** the target file from scratch, exactly like `>` redirection — so each loop iteration destroys the previous iteration's content, leaving only the final one. The fix is adding `-a`: `echo "$msg" | tee -a log.txt`.

---

**Q17 🔥 Someone tries `cat data.txt | some_transform | tee data.txt` intending to save the transformed result back into the same file, but ends up with a corrupted or empty file. What happened?**
> Reading from and writing to the **same file** within one pipeline is unsafe — `tee` typically opens its output file with truncation, which can happen **before** `cat` has finished reading all of the original content, since both are running concurrently as part of the same pipeline. This can result in a partially empty or corrupted file, similar to the classic `cat file > file` mistake. The safe fix is writing to a temporary file first, then moving it into place: `cat data.txt | some_transform > /tmp/tmp_output.txt && mv /tmp/tmp_output.txt data.txt`.

---

**Q18. A script uses `command | tee file1.txt /root/restricted_file.txt` and later assumes both files were written successfully because the script didn't "crash." Is this assumption safe?**
> No. If tee cannot write to one of the listed files (e.g., due to a permissions error on `/root/restricted_file.txt`), it prints an error to stderr for that specific file but by default **continues** writing to any other files it can still access — it doesn't necessarily halt the entire operation or make it obvious which specific output failed unless stderr is actively checked. Scripts relying on multiple tee output targets should verify each file individually afterward, or use `--output-error=exit` (GNU tee) to make any single write failure abort the whole operation immediately instead of silently continuing.

---

**Q19 🔥 Why might an interactive command lose its colored output or progress bar once piped through `tee`, even though the terminal itself clearly supports color?**
> Many programs detect whether their **own** stdout is connected to a real terminal (a TTY) versus a pipe, and disable color/progress-bar formatting automatically when it's a pipe — from that program's perspective, its output is going into `tee`, which is technically a pipe, regardless of what tee eventually forwards it to. Some programs offer an explicit flag (varies by tool) to force colorized/interactive-style output even when piped, if preserving that formatting in a saved log is genuinely needed.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
