# | (Pipe) — The Complete Reference

> **Connect one command's output directly to another command's input**
> Invented by Douglas McIlroy at Bell Labs, introduced in Unix in 1973 — arguably the single
> most influential idea in Unix's design philosophy: small tools, combined freely.

---

## Table of Contents

- [What is the pipe?](#what-is-the-pipe)
- [Where does the pipe "live"?](#where-does-the-pipe-live)
- [How the pipe works internally](#how-the-pipe-works-internally)
- [Syntax](#syntax)
- [Chaining Multiple Pipes](#chaining-multiple-pipes)
- [Pipes vs Redirection](#pipes-vs-redirection)
- [stdin, stdout, stderr — What Actually Flows Through a Pipe](#stdin-stdout-stderr--what-actually-flows-through-a-pipe)
- [Exit Status of a Pipeline](#exit-status-of-a-pipeline)
- [Buffering Behavior](#buffering-behavior)
- [Named Pipes (FIFOs) — Pipes That Live on Disk](#named-pipes-fifos--pipes-that-live-on-disk)
- [The Unix Philosophy Behind Pipes](#the-unix-philosophy-behind-pipes)
- [Related Concepts](#related-concepts)

---

## What is the pipe?

The pipe (`|`) is a shell operator that takes the **standard output** of the command on its left and feeds it directly into the **standard input** of the command on its right — without ever touching disk, without any temporary file, and (mostly) without the two commands needing to know anything about each other at all.

```bash
cat file.txt | grep 'hello'
# cat reads file.txt and writes its content to stdout
# | takes that stdout stream and feeds it as stdin to grep
# grep filters the incoming lines, printing only ones containing "hello"
```

**Why the pipe is considered one of Unix's greatest ideas:** it enables **composability** — instead of writing one large, specialized program that does filtering + sorting + counting all at once, you write (or already have) small, focused tools (`grep`, `sort`, `uniq`, `wc`) and combine them freely, in whatever order a specific task requires, without any of those tools needing to be modified or even aware of each other's existence.

---

## Where does the pipe "live"?

The pipe is **not a program** — it's a syntax feature interpreted directly by the **shell** (bash, zsh, dash, etc.) itself, not something you can `which` or find as a file on disk.

```bash
which |
# (nothing — "|" isn't a command, it's shell syntax)

type |
# bash: syntax error near unexpected token `|'
# (attempting to use `type` on it doesn't even parse correctly, since
# | is a SHELL METACHARACTER, not a word/command name at all)
```

Under the hood, the shell implements a pipe using the `pipe()` **system call**, which the kernel provides — the shell's job is simply to recognize the `|` character in your command line and wire up the two processes accordingly using that kernel facility.

---

## How the pipe works internally

### The pipe() system call and file descriptors

When the shell encounters `cmd1 | cmd2`, it:

1. Calls the `pipe()` system call, which asks the kernel to create an in-memory buffer with **two ends** — a **read end** and a **write end**, each exposed as a file descriptor
2. **Forks** two child processes: one for `cmd1`, one for `cmd2`
3. In `cmd1`'s process, redirects its **stdout** (file descriptor 1) to the pipe's **write end**
4. In `cmd2`'s process, redirects its **stdin** (file descriptor 0) to the pipe's **read end**
5. Runs both processes **concurrently** — `cmd1` doesn't need to fully finish before `cmd2` starts; they run in parallel, with the kernel-managed pipe buffer acting as the connection between them

```bash
# Conceptually, in C, this is roughly what the shell does internally:
# int fd[2];
# pipe(fd);              // fd[0] = read end, fd[1] = write end
# fork() -> child A: dup2(fd[1], STDOUT_FILENO); exec("cmd1");
# fork() -> child B: dup2(fd[0], STDIN_FILENO);  exec("cmd2");
```

### The pipe buffer — a fixed-size, in-kernel queue

The pipe itself is a small, fixed-size buffer maintained entirely **in kernel memory** (typically 64KB on modern Linux) — data never touches the disk. If the writing process produces data faster than the reading process consumes it, the writer **blocks** (pauses) once the buffer fills, until the reader catches up. Conversely, if the reader tries to read from an empty pipe whose writer hasn't produced anything yet, the reader blocks until data arrives.

```bash
# This blocking/flow-control behavior happens automatically and
# invisibly — you don't need to manage buffering yourself, the kernel
# handles synchronizing the two processes' speeds for you:
yes | head -1
# "yes" would produce infinite output, but head only reads ONE line
# then exits — closing its end of the pipe, which causes "yes" to
# receive a SIGPIPE signal and terminate, rather than running forever
```

### Both processes run in parallel, not sequentially

```bash
# It's a common misconception that cmd1 runs to COMPLETION before
# cmd2 starts — in reality, both run CONCURRENTLY:
tail -f /var/log/syslog | grep "error"
# tail -f runs INDEFINITELY, continuously producing new lines as the
# log grows, while grep filters each line AS IT ARRIVES — grep doesn't
# wait for tail to "finish" (which it never will, with -f), it
# processes the stream incrementally, in real time.
```

---

## Syntax

```bash
command1 | command2
command1 | command2 | command3 | ...     # any number of commands can be chained
```

```bash
ls -la | grep ".txt"
ps aux | grep nginx
cat access.log | grep ERROR | wc -l
```

There's no special syntax beyond the `|` character itself between two commands — whitespace around it is optional but conventional for readability.

---

## Chaining Multiple Pipes

```bash
# Each additional | adds another stage to the PIPELINE
cat sales.csv | grep "2024" | cut -d',' -f3 | sort | uniq -c | sort -rn
# Reading left to right:
# 1. cat sales.csv           -> print the file
# 2. grep "2024"             -> keep only lines mentioning 2024
# 3. cut -d',' -f3            -> extract just the 3rd comma-separated field
# 4. sort                     -> sort those values alphabetically
# 5. uniq -c                  -> collapse duplicates, counting occurrences
# 6. sort -rn                 -> sort by count, descending, numerically

# There's no fixed limit on how many stages a pipeline can have —
# each command in the chain runs concurrently, connected by its own
# separate pipe to its immediate neighbors
```

---

## Pipes vs Redirection

| Feature | `command > file` (redirection) | `command1 | command2` (pipe) |
|---------|-----------------------------------|-----------------------------------|
| Destination | A FILE on disk | Another COMMAND's stdin |
| Data touches disk | ✅ Yes, written to the file | ❌ No, stays in kernel memory (unless the receiving command itself writes to disk) |
| Can be chained further | ❌ Not directly (you'd need to `cat file | next_command` separately) | ✅ Yes, pipelines can have arbitrarily many stages |
| Typical use | "Save this output for later" | "Feed this output directly into another tool right now" |

```bash
ls -la > listing.txt              # save to a file
ls -la | grep ".txt"              # feed directly into another command
ls -la | tee listing.txt | grep ".txt"   # do BOTH — save AND continue piping (see tee)
```

---

## stdin, stdout, stderr — What Actually Flows Through a Pipe

### By default, a pipe only connects stdout — NOT stderr

```bash
command_that_errors | grep "something"
# Only command_that_errors's STDOUT is piped into grep.
# Any error messages command_that_errors writes to STDERR still go
# directly to your TERMINAL, completely bypassing grep entirely —
# a very common source of "why isn't my grep catching this error
# message?" confusion.

# To include stderr in what gets piped, redirect it into stdout FIRST,
# explicitly, before the pipe takes effect:
command_that_errors 2>&1 | grep "something"
# 2>&1 merges stderr (fd 2) into stdout (fd 1) BEFORE the pipe grabs
# stdout, so now BOTH streams flow into grep together.
```

### Order matters when combining redirection and piping

```bash
command > output.txt 2>&1 | grep "pattern"
# ⚠️ This does NOT do what many people expect: stdout is redirected to
# output.txt FIRST, then 2>&1 makes stderr ALSO go to output.txt (since
# stdout at that point in the parsing already points to the file) —
# there is NOTHING left flowing through the pipe to grep at all.

command 2>&1 | grep "pattern"
# ✅ This is usually what's intended instead — merge stderr into
# stdout WHILE stdout still points to the pipe, so both streams
# reach grep together.
```

---

## Exit Status of a Pipeline

### $? after a pipeline reflects only the LAST command by default

```bash
false | true
echo $?
# 0     ← this is "true"'s exit status (the LAST command in the pipe),
# completely ignoring that "false" (the FIRST command) failed

# Use bash's PIPESTATUS array to inspect EVERY stage's individual exit code
false | true
echo "${PIPESTATUS[@]}"
# 1 0    ← false's exit code (1), then true's exit code (0)

# Alternatively, enable "pipefail" so the PIPELINE's overall exit
# status reflects the FIRST non-zero exit code among any stage,
# rather than always just the last command's:
set -o pipefail
false | true
echo $?
# 1     ← now reflects false's failure, not true's success
```

---

## Buffering Behavior

### Programs often buffer differently depending on whether output is a terminal or a pipe

```bash
# Many programs use LINE buffering when writing directly to a terminal
# (flushing after every newline, so output appears immediately), but
# switch to FULL/BLOCK buffering when their output is redirected to a
# pipe or file (flushing only when an internal buffer fills up) — a
# performance optimization that can make piped output appear "delayed"
# in real-time monitoring scenarios:

some_program | grep "pattern"
# Output might not appear immediately, even though matching lines
# clearly exist, because some_program is buffering internally before
# grep ever sees the data at all.

# Force line-buffering explicitly when real-time visibility matters:
stdbuf -oL some_program | grep --line-buffered "pattern"
```

---

## Named Pipes (FIFOs) — Pipes That Live on Disk

### Unlike the `|` operator's anonymous, temporary pipe, a FIFO is a persistent, named pipe

```bash
# Create a named pipe (a special file type, visible with ls)
mkfifo mypipe
ls -l mypipe
# prw-r--r-- 1 alice alice 0 ... mypipe    ← the "p" indicates a FIFO

# Two SEPARATE, independent commands (potentially in different
# terminals or scripts) can connect through it, unlike the shell's
# `|` operator which only connects commands written on the SAME line:

# Terminal 1:
cat mypipe
# (blocks, waiting for something to write to the pipe)

# Terminal 2:
echo "hello through a named pipe" > mypipe
# Terminal 1 immediately receives and prints:
# hello through a named pipe

rm mypipe    # clean up when done — FIFOs persist on disk until removed
```

---

## The Unix Philosophy Behind Pipes

The pipe is the practical mechanism behind one of Unix's most famous design principles, often summarized as:

> "Write programs that do one thing and do it well. Write programs to work together. Write programs to handle text streams, because that is a universal interface."
> — commonly attributed to Doug McIlroy

```bash
# Rather than one monolithic "word_frequency_counter" program, Unix
# philosophy composes the SAME result from small, independent,
# general-purpose tools that each do ONE thing well:
cat book.txt | tr -s ' ' '\n' | tr 'A-Z' 'a-z' | sort | uniq -c | sort -rn | head -10
# tr    -> split text into one word per line
# tr    -> normalize to lowercase
# sort  -> group identical words together
# uniq  -> count occurrences of each
# sort  -> order by frequency, descending
# head  -> show just the top 10
# Each tool is independently useful for MANY other tasks too — none
# of them were written specifically "for" word-frequency counting.
```

---

## Related Concepts

| Concept | Relation |
|---------|----------|
| `tee` | Splits a stream to BOTH a file and onward through the pipeline, rather than just passing it along |
| `>` / `>>` | Redirection to a file, the pipe's closest sibling concept but writing to disk instead of another process |
| `2>&1` | Merges stderr into stdout, often combined with a pipe to capture BOTH streams |
| `xargs` | Converts piped input into command-line ARGUMENTS for a command that doesn't read stdin directly |
| `mkfifo` | Creates a persistent, named pipe (FIFO) on disk, unlike the anonymous pipe created by `|` |
| `PIPESTATUS` (bash) | Inspects the individual exit status of each stage in a pipeline |
| `set -o pipefail` | Changes a pipeline's overall exit status to reflect any failing stage, not just the last one |
| Process substitution `<(...)` / `>(...)` | A related but distinct bash feature enabling pipe-like behavior with commands expecting FILE arguments rather than stdin |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
