# | (Pipe) — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Internals](#internals)
- [stdin/stdout/stderr](#stdinstdoutstderr)
- [Exit Codes](#exit-codes)
- [Subshells & Variable Scope](#subshells--variable-scope)
- [Buffering](#buffering)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does the pipe operator (|) do?**
> It connects the **standard output** of the command on its left directly to the **standard input** of the command on its right, allowing data to flow between the two processes without any temporary file or disk I/O involved.

---

**Q2 🔥 Is the pipe a separate program, or part of the shell?**
> It's shell **syntax**, interpreted directly by the shell itself (bash, zsh, etc.) — not an external program you can locate with `which` or invoke on its own. The shell recognizes `|` in a command line and sets up the connection between the two processes using the kernel's `pipe()` system call.

---

**Q3. Why is the pipe considered one of Unix's most influential design ideas?**
> It enables **composability** — small, focused, single-purpose tools (`grep`, `sort`, `uniq`, `wc`, etc.) can be freely combined in any order to accomplish complex tasks, without any of those tools needing to be modified or even aware of each other. This directly embodies the Unix philosophy of "do one thing well" combined with "make tools work together via a universal text-stream interface."

---

## Internals

**Q4 🔥 What system call does the shell use to implement a pipe, and what does it actually create?**
> The `pipe()` system call, which asks the kernel to create an in-memory buffer exposed as two file descriptors — a **read end** and a **write end**. The shell then forks the two commands as separate processes, connecting one's stdout to the write end and the other's stdin to the read end.

---

**Q5. Do the two commands in a pipeline run sequentially (one after the other) or concurrently?**
> **Concurrently.** Both processes start and run in parallel, connected by the kernel-managed pipe buffer — the left-hand command doesn't need to finish before the right-hand command starts processing. This is why something like `tail -f logfile | grep error` works: `tail -f` runs indefinitely while `grep` processes each line as it arrives, in real time.

---

**Q6 🔥 What happens if the writing process in a pipeline produces data faster than the reading process consumes it?**
> The pipe's underlying kernel buffer (typically 64KB on modern Linux) fills up, and the writer **blocks** (pauses) until the reader consumes enough data to free up buffer space. This flow-control behavior is automatic and transparent — neither process needs to manage it manually.

---

## stdin/stdout/stderr

**Q7 🔥 Does a pipe capture a command's error messages (stderr) by default?**
> No. By default, a pipe only connects **stdout** — any output the left-hand command writes to **stderr** goes directly to the terminal, completely bypassing the pipe and the receiving command entirely.

---

**Q8. How would you make sure BOTH stdout and stderr are piped into the next command?**
> Merge stderr into stdout with `2>&1` **before** the pipe takes effect: `command 2>&1 | grep "pattern"`. This causes both streams to be combined into stdout while it still feeds the pipe.

---

**Q9. Why does `command > output.txt 2>&1 | grep "pattern"` fail to actually pipe anything meaningful to grep?**
> Because of left-to-right parsing order: stdout is redirected to `output.txt` first, and only then does `2>&1` merge stderr into whatever stdout currently points to — which is now the file, not the pipe. By the time the pipe would have received data, stdout has already been diverted entirely to the file, leaving nothing flowing to `grep`.

---

## Exit Codes

**Q10 🔥 After running `false | true`, what does `$?` report, and why might that be misleading?**
> `$?` reports `0` — the exit status of `true`, the **last** command in the pipeline — completely masking that `false` (the first command) actually failed. By default, a pipeline's exit status only reflects its final stage, regardless of what happened earlier in the chain.

---

**Q11. How do you check the exit status of every individual command in a pipeline, not just the last one?**
> Use bash's `PIPESTATUS` array immediately after the pipeline runs: `echo "${PIPESTATUS[@]}"` shows the exit code of each stage, in order (e.g., `1 0` for a failing first command followed by a succeeding second one).

---

**Q12 🔥 What does `set -o pipefail` change about pipeline behavior?**
> It changes a pipeline's overall exit status (`$?`) to reflect the **first non-zero exit code** among any of its stages, rather than always defaulting to just the last command's exit status — making failure detection in scripts far more reliable when any stage of a pipeline might fail silently otherwise.

---

## Subshells & Variable Scope

**Q13 🔥 Why does a variable modified inside a `while read` loop that's part of a pipeline not retain its updated value afterward?**
> Each stage of a pipeline (other than sometimes the last, depending on shell settings) typically runs in its own **subshell** — a separate child process. Variable changes made inside that subshell are local to it and are discarded once the subshell exits; the original, calling shell's variable was never actually modified, since it lives in a completely different process.

---

**Q14. Give two ways to fix a `count` variable not persisting after `cat file.txt | while read -r line; do count=$((count+1)); done`.**
> (1) Avoid the pipe entirely by redirecting the file directly into the loop instead: `while read -r line; do count=$((count+1)); done < file.txt` — no subshell is created this way. (2) In bash specifically, enable `shopt -s lastpipe`, which runs the **last** command of a pipeline in the current shell rather than a subshell (with some caveats around job control).

---

## Buffering

**Q15 🔥 Why might output flowing through a pipe appear "delayed," arriving in large bursts rather than line by line, even though the producing command is clearly generating output continuously?**
> Many programs switch from line-buffered output (flushing after every newline, typical when writing directly to an interactive terminal) to fully block-buffered output the moment their stdout is redirected to a pipe rather than a terminal — a performance optimization on the producing program's part that can make real-time monitoring through a pipe appear delayed.

---

**Q16. How would you force a command to produce line-buffered output even when piped, if real-time visibility matters?**
> Wrap the producing command with `stdbuf -oL`: `stdbuf -oL some_command | grep --line-buffered "pattern"`. `stdbuf` forces the producer's output buffering mode; pairing it with a consumer flag like grep's `--line-buffered` (where available) ensures the entire chain stays responsive in real time.

---

## Scenario-Based

**Q17 🔥 A script pipes a command's output into grep to search for an error message, but the error clearly printed on screen never gets matched by grep. What's the most likely explanation?**
> The error message is almost certainly being written to **stderr**, not stdout — since a plain pipe only connects stdout to the next command, stderr output bypasses the pipe entirely and goes straight to the terminal, invisible to grep. The fix is merging stderr into stdout before the pipe: `command 2>&1 | grep "error message"`.

---

**Q18. A developer runs `ps aux | grep myapp` expecting to see only the running "myapp" process, but the grep command's own invocation also shows up in the results. Why, and how is this typically avoided?**
> The literal text "myapp" appears in the process list simply because it's part of the `grep myapp` command line itself, which `ps aux` also lists as a running process — and grep, searching for "myapp," matches its own entry too. The classic fix is bracketing one character of the search pattern (`grep '[m]yapp'`), so the regex no longer literally matches grep's own command-line text, though the more explicit tool for this exact purpose is `pgrep myapp`.

---

**Q19 🔥 A script checks `$?` immediately after `grep "pattern" missing_file.txt | wc -l` to decide whether the file was found and processed successfully, but the check always reports success even when the file doesn't exist. What's wrong, and how do you fix it?**
> `$?` reflects only `wc`'s exit status (which succeeds at counting zero lines from grep's empty output), not grep's actual failure to find the file — the real error from grep (file not found) is completely hidden by the pipeline's default exit-code behavior. Fix: check `${PIPESTATUS[0]}` instead of `$?` to see grep's actual exit code, or enable `set -o pipefail` so the pipeline's overall exit status reflects the earliest failure.

---

**Q20. Someone writes `mkdir /data/newfolder | echo "creation failed"` intending to print a message only if mkdir fails, but the message prints every single time regardless of whether mkdir actually succeeds. What's the bug?**
> They used `|` (pipe) instead of `||` (logical OR). A pipe unconditionally connects mkdir's stdout to echo's stdin and runs echo regardless of mkdir's success or failure — there's no conditional logic involved at all. The intended behavior requires `||`: `mkdir /data/newfolder || echo "creation failed"`, which only runs the echo command if mkdir actually returns a non-zero (failure) exit status.

---

**Q21 🔥 A pipeline `cat huge_data.txt | sort | uniq -c | sort -rn > report.txt` seems to hang indefinitely on an extremely large file, using significant memory, while a colleague's simpler `grep` step earlier in development ran instantly on the same file. What's different about `sort` specifically?**
> Unlike `grep` or `cat`, which can process and emit output line-by-line in a streaming fashion without waiting for the entire input, `sort` fundamentally must see the **entire input** before it can produce any correctly sorted output at all (it can't know a line is truly "in order" until every subsequent line has also been seen) — for extremely large files, this forces `sort` to buffer significant portions of data (spilling to temporary disk files if needed), unlike the earlier streaming stages of the same pipeline. This is expected behavior inherent to any correct sorting algorithm operating on unbounded input, not a bug in the pipe mechanism itself.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
