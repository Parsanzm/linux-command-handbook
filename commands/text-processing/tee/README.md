# tee — The Complete Reference

> **Split a stream: write to a file AND pass it through to the next command, simultaneously**
> Named after the plumbing "T-shaped" pipe fitting — one input, two outputs.
> Present since Version 5 Unix (1974), designed specifically to solve the "I need to both save AND continue piping this" problem.

---

## Table of Contents

- [What is tee?](#what-is-tee)
- [Where does tee live?](#where-does-tee-live)
- [How tee works internally](#how-tee-works-internally)
- [Syntax](#syntax)
- [Basic Usage](#basic-usage)
- [All Key Options](#all-key-options)
- [tee and Pipelines](#tee-and-pipelines)
- [tee with sudo — Writing to Protected Files](#tee-with-sudo--writing-to-protected-files)
- [Writing to Multiple Files at Once](#writing-to-multiple-files-at-once)
- [tee and Exit Codes](#tee-and-exit-codes)
- [tee vs Redirection (> and >>)](#tee-vs-redirection--and-)
- [Related Commands](#related-commands)

---

## What is tee?

`tee` reads from standard input and writes that same data to **both** standard output and one or more files, simultaneously. The name comes from a plumbing "T" fitting, which splits a single pipe into two directions — exactly what `tee` does with a stream of data: one copy continues down the pipeline, another copy is saved to disk.

```bash
echo "hello" | tee output.txt
# hello                    ← printed to stdout (visible in the terminal)
cat output.txt
# hello                    ← ALSO written to the file
```

**Why tee exists as a distinct tool:** without it, you're forced to choose between **seeing** a command's output live and **saving** it to a file — normal redirection (`>`) does only one of those. `tee` lets you do both at once, and additionally lets a saved copy be captured **partway through** a longer pipeline, not just at the very end.

---

## Where does tee live?

```
/usr/bin/tee
```

```bash
which tee
tee --version
# tee (GNU coreutils) 9.4
```

Part of **GNU coreutils** on Linux; also present (with slightly different option support) on macOS/BSD and virtually every POSIX-compliant system, since `tee` is part of the POSIX standard utilities.

---

## How tee works internally

### A simple, single-purpose data duplicator

`tee` reads from stdin in a loop, and for every chunk of data read, writes it to:
1. **stdout** (so the next command in a pipeline receives it, or it appears in the terminal)
2. **every file** named as an argument (opened for writing, or appending with `-a`)

```bash
command_producing_output | tee saved.txt | next_command_in_pipeline
# command_producing_output's output flows through tee, which:
#   - writes an identical copy to saved.txt
#   - simultaneously passes the SAME data onward to next_command_in_pipeline
# Neither downstream destination "blocks" or delays the other significantly;
# tee performs both writes as it reads each chunk of input.
```

### Line-buffered vs block-buffered behavior

By default, `tee`'s buffering behavior can differ depending on whether its output is going to a terminal (line-buffered, appears immediately) or to a pipe/file (potentially block-buffered, so output might not appear until a buffer fills or the command finishes) — this occasionally surprises people expecting truly instantaneous output in every context.

```bash
# Force line-buffered output explicitly, useful in monitoring pipelines
# where you want to see each line as soon as it's produced, not delayed:
some_long_running_command | stdbuf -oL tee output.txt
```

---

## Syntax

```bash
tee [OPTIONS] [FILE...]
```

```bash
command | tee file.txt                  # write to file.txt AND pass through to stdout
command | tee file1.txt file2.txt       # write to MULTIPLE files simultaneously
command | tee -a file.txt               # APPEND instead of overwrite
command | tee file.txt | next_command   # write to file.txt, continue piping onward
```

---

## Basic Usage

```bash
# Save a command's output to a file WHILE still seeing it in the terminal
ls -la | tee directory_listing.txt

# Append to an existing log file instead of overwriting it
echo "New entry" | tee -a activity.log

# Discard stdout entirely, only save to the file (redirect stdout to /dev/null)
long_command | tee output.txt > /dev/null

# Use tee purely to "peek" at data flowing through a longer pipeline,
# without altering the pipeline's ultimate behavior at all
cat data.csv | tee /tmp/debug_snapshot.csv | awk -F',' '{print $1}' | sort | uniq -c
```

---

## All Key Options

| Option | Long | Description |
|--------|------|--------------|
| `-a` | `--append` | Append to the given file(s) instead of overwriting them |
| `-i` | `--ignore-interrupts` | Ignore SIGINT (Ctrl+C) signals |
| `-p` | | Diagnose errors writing to non-pipe outputs (GNU extension, useful with process substitution) |
| `--output-error=MODE` | | Control behavior when writing to an output fails (`warn`, `warn-nopipe`, `exit`, `exit-nopipe`) |
| `--help` | | Show usage help |
| `--version` | | Show version information |

```bash
command | tee -a log.txt                   # append instead of overwrite
command | tee -i output.txt                # keep writing even if Ctrl+C is pressed
command | tee --output-error=exit out.txt  # stop immediately if the write fails,
                                            # rather than silently continuing
```

---

## tee and Pipelines

### Saving output at multiple stages of a long pipeline

```bash
cat access.log \
  | grep "ERROR" \
  | tee /tmp/all_errors.log \
  | grep "500" \
  | tee /tmp/server_errors.log \
  | wc -l
# Saves a snapshot of ALL errors, a further-filtered snapshot of just
# 500-status errors, AND still produces the final count — all in ONE
# pipeline execution, without needing to run the underlying command
# multiple times to capture each intermediate stage separately.
```

### Debugging a pipeline without disrupting it

```bash
# Insert tee temporarily to inspect what's flowing at a specific point,
# without changing the pipeline's final behavior at all
producer_command | tee /tmp/debug_stage1.txt | transform_command | consumer_command
# /tmp/debug_stage1.txt now contains exactly what transform_command received,
# letting you verify assumptions about intermediate data without
# modifying the actual processing logic.
```

---

## tee with sudo — Writing to Protected Files

### The classic reason tee exists in modern shell usage: fixing sudo + redirection
```bash
# This does NOT work as beginners often expect:
sudo echo "new setting" > /etc/protected_config
# Permission denied — "echo" runs as root, but the ">" REDIRECTION is
# performed by the SHELL, which is still running as your normal,
# unprivileged user — the shell, not echo, is the one denied write access.

# tee solves this cleanly, because tee ITSELF (not the shell) performs
# the file write, and tee can be the thing that's run with sudo:
echo "new setting" | sudo tee /etc/protected_config
# echo runs as your normal user (fine, it's just producing text)
# tee runs AS ROOT (via sudo), and IT performs the actual file write

# Append instead of overwrite, still as root
echo "another setting" | sudo tee -a /etc/protected_config

# Write to the file AND still see the content printed to your terminal
# (tee's normal dual behavior, now combined with sudo)
echo "new setting" | sudo tee /etc/protected_config
# new setting                    ← also printed, confirming what was written
```

---

## Writing to Multiple Files at Once

```bash
# Save the SAME output to several files in a single pass
command | tee file1.txt file2.txt file3.txt

# Combine with sudo to write the same content to multiple protected locations
echo "config_value=true" | sudo tee /etc/app1/config /etc/app2/config

# Use with process substitution for even more advanced fan-out
# (send the same stream to a file AND to another command simultaneously)
command | tee >(gzip > output.gz) | wc -l
# The process substitution >(gzip > output.gz) receives a COPY of the
# stream and compresses it in the background, while wc -l still
# receives and counts the original, uncompressed stream in parallel.
```

---

## tee and Exit Codes

### tee's own exit status vs the exit status of what it's piped to
```bash
false | tee output.txt
echo $?
# 0     ← this is TEE's OWN exit status, not "false"'s!
# tee itself succeeded (it successfully wrote the — empty — input to
# the file), completely independent of whether the PRODUCING command
# earlier in the pipeline succeeded or failed.

# To check the exit status of the ORIGINAL command instead of tee's,
# use PIPESTATUS (bash-specific) to inspect every stage of the pipeline:
false | tee output.txt
echo "${PIPESTATUS[0]}"
# 1     ← the ACTUAL exit status of "false", the first command in the pipe

echo "${PIPESTATUS[@]}"
# 1 0    ← false's exit status (1), then tee's own exit status (0)
```

---

## tee vs Redirection (> and >>)

| Feature | `command > file` | `command | tee file` |
|---------|---------------------|--------------------------|
| See output in terminal | ❌ No (goes only to file) | ✅ Yes (also printed) |
| Save output to file | ✅ Yes | ✅ Yes |
| Continue piping to another command | ❌ No (redirection ends the flow to a file) | ✅ Yes (`tee file | next_command`) |
| Write to MULTIPLE files at once | ❌ No (only one target) | ✅ Yes (`tee file1 file2 file3`) |
| Works cleanly with sudo for protected files | ❌ No (shell performs the redirect, not the sudo'd command) | ✅ Yes (tee itself performs the write, as root) |
| Overwrite vs append | `>` overwrites, `>>` appends | Overwrites by default, `-a` appends |

```bash
command > file.txt          # save ONLY, nothing shown in terminal
command | tee file.txt      # save AND show in terminal simultaneously
command >> file.txt         # append, nothing shown
command | tee -a file.txt   # append AND show in terminal simultaneously
```

**Rule of thumb:** use plain redirection (`>`/`>>`) when you only care about the saved result and don't need to watch it live or continue piping; reach for `tee` the moment you need to **also** see the output, **also** continue processing it further in a pipeline, or write to a **protected** location via `sudo`.

---

## Related Commands

| Command | Relation |
|---------|----------|
| `cat` | Often paired with tee for simple file duplication tasks |
| `sudo` | Commonly combined with tee to write to root-owned files |
| `xargs` | Sometimes used alongside tee in more complex pipeline constructions |
| `script` | Records an ENTIRE terminal session, a much broader capture than tee's single-stream duplication |
| `PIPESTATUS` (bash) | Used to inspect the real exit status of commands earlier in a tee pipeline |
| `>` / `>>` | Simple redirection — tee's closest conceptual sibling, but without the "also show/continue" behavior |
| `mkfifo` | Named pipes, sometimes used together with tee for more advanced stream-splitting scenarios |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
