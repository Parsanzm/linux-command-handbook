# tee — Edge Cases & Gotchas

> tee looks trivial, but buffering, exit codes, sudo interactions, and
> permission ordering create real, easy-to-miss bugs in scripts.

---

## Table of Contents

- [tee's Exit Code Hides the Real Command's Failure](#tees-exit-code-hides-the-real-commands-failure)
- [sudo tee vs sudo Redirection — Order of Operations](#sudo-tee-vs-sudo-redirection--order-of-operations)
- [Buffering Delays "Live" Output](#buffering-delays-live-output)
- [Overwriting vs Appending Confusion](#overwriting-vs-appending-confusion)
- [Writing to the Same File You're Reading From](#writing-to-the-same-file-youre-reading-from)
- [tee Doesn't Create Missing Parent Directories](#tee-doesnt-create-missing-parent-directories)
- [Broken Pipe Errors Downstream](#broken-pipe-errors-downstream)
- [Permission Errors Mid-Pipeline Fail Silently](#permission-errors-mid-pipeline-fail-silently)
- [tee with Multiple Files — Partial Failure](#tee-with-multiple-files--partial-failure)
- [Interactive Programs Don't Play Well with tee](#interactive-programs-dont-play-well-with-tee)
- [tee and Very Large Files — Disk Fills Silently](#tee-and-very-large-files--disk-fills-silently)
- [Redirecting tee's Own stdout to /dev/null Confusion](#redirecting-tees-own-stdout-to-devnull-confusion)

---

## tee's Exit Code Hides the Real Command's Failure

### $? after a tee pipeline reflects tee, not the producing command
```bash
false | tee output.txt
echo $?
# 0     ⚠️ this is TEE's exit status (it succeeded at writing an
# empty stream to the file), completely masking that "false"
# (the actual command of interest) exited with failure (1).

# A script checking $? naively after a tee pipeline can proceed as if
# everything succeeded, even when the REAL command failed:
if some_important_command | tee log.txt; then
  echo "Proceeding as if success..."   # ⚠️ may run even if the command failed!
fi

# Fix: use bash's PIPESTATUS array to check the ACTUAL first command's exit code
some_important_command | tee log.txt
if [ "${PIPESTATUS[0]}" -ne 0 ]; then
  echo "The real command actually failed"
  exit 1
fi
```

---

## sudo tee vs sudo Redirection — Order of Operations

### Understanding WHY `sudo command > file` fails but `command | sudo tee file` works
```bash
sudo echo "text" > /etc/protected_file
# Permission denied — even though "echo" itself ran as root via sudo,
# the ">" REDIRECTION is set up by the SHELL, BEFORE echo even runs,
# and the shell is still your normal, unprivileged user — sudo only
# elevated the "echo" PROCESS, not the shell's own redirect operation.

echo "text" | sudo tee /etc/protected_file
# ✅ Works — "echo" runs as your normal user (fine, it's just producing
# text on stdout), and "tee" is the one invoked WITH sudo, so TEE
# itself (which performs the actual file write) runs as root.

# A subtle follow-up gotcha: if you ALSO try to append output redirection
# AFTER sudo tee, that redirection is STILL performed by your unprivileged shell:
echo "text" | sudo tee /etc/protected_file > /etc/another_protected_file
# ❌ Permission denied on the SECOND file — the ">" here is your shell's
# own redirect again, not something sudo/tee elevated at all. tee's
# OWN arguments (the filenames given directly to it) are the ones that
# get root privileges; anything after a bare ">" does not.
```

---

## Buffering Delays "Live" Output

### Output through tee (and pipes generally) isn't always instantaneous
```bash
slow_command_producing_output_gradually | tee output.txt
# Depending on the producing command's OWN buffering behavior (often
# fully-buffered rather than line-buffered when its output isn't a
# terminal), you might see NOTHING appear for a long time, then a
# large chunk all at once — even though tee itself processes data as
# it arrives, the PRODUCER may be batching its own writes internally.

# Force line-buffering on the PRODUCING command to see output as it's
# actually generated, not delayed by internal buffering:
stdbuf -oL slow_command | tee output.txt

# This is a property of many Unix tools' own I/O buffering strategy,
# not something tee itself introduces — but it's frequently blamed on
# tee simply because tee is the visible last step before "nothing
# appears on screen for a while."
```

---

## Overwriting vs Appending Confusion

### Forgetting -a silently destroys previous log content
```bash
echo "Run 1 complete" | tee results.log
echo "Run 2 complete" | tee results.log
cat results.log
# Run 2 complete
# ⚠️ "Run 1 complete" is GONE — without -a, EVERY tee invocation
# overwrites the file fresh, exactly like ">" redirection would.

echo "Run 1 complete" | tee -a results.log
echo "Run 2 complete" | tee -a results.log
cat results.log
# Run 1 complete
# Run 2 complete
# ✅ both entries preserved, because -a appends instead of truncating

# In a loop, forgetting -a is a classic bug that silently discards
# everything except the LAST iteration's output:
for i in 1 2 3; do
  echo "Iteration $i" | tee log.txt    # ⚠️ BUG: only "Iteration 3" survives
done
```

---

## Writing to the Same File You're Reading From

### tee reading and writing the same file can produce corrupted or empty output
```bash
cat data.txt | tee data.txt
# ⚠️ UNDEFINED / dangerous behavior — tee may TRUNCATE data.txt (since
# writing typically opens the file with truncation) BEFORE cat has
# finished reading all of it, potentially resulting in a partially
# empty or corrupted file, similar to the classic `cat file > file` mistake.

# If the goal is to APPEND a transformation of a file to itself,
# always use a TEMPORARY intermediate file instead:
cat data.txt | some_transform > /tmp/temp_output.txt
mv /tmp/temp_output.txt data.txt

# Or, if genuinely appending NEW content to an existing file (not
# reading FROM the same file as part of the same pipeline), -a is safe:
echo "new line" | tee -a data.txt    # ✅ safe — not reading data.txt as input here
```

---

## tee Doesn't Create Missing Parent Directories

### A typo'd or nonexistent path fails silently-ish, without much explanation
```bash
echo "content" | tee /path/to/nonexistent_dir/output.txt
# tee: /path/to/nonexistent_dir/output.txt: No such file or directory
# tee does NOT automatically create missing intermediate directories —
# the target directory must already exist.

# Fix: ensure the directory exists first
mkdir -p /path/to/nonexistent_dir
echo "content" | tee /path/to/nonexistent_dir/output.txt
```

---

## Broken Pipe Errors Downstream

### If the command AFTER tee exits early, tee can receive a "broken pipe" signal
```bash
yes "line" | tee output.txt | head -5
# head -5 exits after reading just 5 lines, closing its end of the
# pipe — "yes" keeps producing infinite output, and tee, still trying
# to write to head's now-closed input, may receive a SIGPIPE and
# terminate abruptly, sometimes printing:
# tee: standard output: Broken pipe

# This is usually harmless (the intended 5 lines were already captured
# in output.txt and printed), but the visible "Broken pipe" message can
# alarm someone unfamiliar with this normal pipe-closing behavior in
# scripts that check for ANY stderr output as a sign of failure.
```

---

## Permission Errors Mid-Pipeline Fail Silently

### tee can fail to write to ONE destination while still succeeding elsewhere
```bash
command | tee writable_file.txt /root/protected_file.txt
# tee: /root/protected_file.txt: Permission denied
# writable_file.txt IS still written successfully — tee continues
# attempting ALL its outputs even if one specific target fails, and by
# DEFAULT only prints a warning to stderr rather than stopping the
# entire operation or returning a clearly failing exit code for that
# one file (tee's own overall exit status reflects whether ANY error
# occurred, but doesn't clearly indicate WHICH specific output failed
# without reading the stderr message).

# For strict handling where any single output failure should stop
# everything immediately, GNU tee's --output-error can be tuned:
command | tee --output-error=exit writable_file.txt /root/protected_file.txt
# exits immediately on the FIRST output error instead of continuing
```

---

## tee with Multiple Files — Partial Failure

### Not all listed files are guaranteed to succeed together as an atomic operation
```bash
command | tee file1.txt file2.txt file3.txt
# If file2.txt's directory doesn't exist or lacks write permission,
# tee reports an error for file2.txt specifically, but STILL attempts
# (and typically succeeds at) writing file1.txt and file3.txt — there's
# no "all or nothing" guarantee across multiple output targets.

# Always verify EVERY target file individually after a multi-file tee
# call in scripts where partial success could cause silent inconsistency:
for f in file1.txt file2.txt file3.txt; do
  [ -s "$f" ] || echo "Warning: $f is missing or empty!"
done
```

---

## Interactive Programs Don't Play Well with tee

### Piping an interactive program's output through tee can break prompts/formatting
```bash
some_interactive_wizard | tee session_log.txt
# Interactive programs often detect whether their output is going to
# a real terminal (a TTY) versus a pipe, and change behavior
# accordingly — disabling colors, progress bars, or interactive
# prompts entirely once piped through tee, since tee's own output is
# technically a pipe from the program's perspective, even though YOUR
# terminal is still what ultimately displays tee's forwarded output.

# Some programs offer a flag to force "interactive-style" output even
# when piped, if logging the session is genuinely needed:
some_program --force-color | tee session_log.txt
# (flag name varies by program; not all programs offer this override)
```

---

## tee and Very Large Files — Disk Fills Silently

### tee doesn't warn you as free disk space runs low
```bash
generate_massive_output | tee /var/log/huge_capture.log
# If /var/log's filesystem fills up mid-write, tee typically reports
# a "No space left on device" error on subsequent write attempts, but
# by that point a PARTIAL, truncated file may already exist, and the
# pipeline's downstream command may have ALSO received truncated data
# — neither side automatically "cleans up" after a disk-full condition.

# Especially risky for tee combined with long-running monitoring loops
# writing to the SAME growing log file indefinitely without rotation:
df -h /var/log    # check available space BEFORE starting a long capture
# Consider log rotation (logrotate) for any long-running tee-based
# logging setup to avoid this scenario entirely.
```

---

## Redirecting tee's Own stdout to /dev/null Confusion

### A common pattern that surprises people expecting to see NOTHING at all
```bash
echo "saved but not shown" | tee output.txt > /dev/null
cat output.txt
# saved but not shown    ✅ the FILE still has the content

# The ">/dev/null" here redirects tee's OWN stdout stream (the copy
# tee would otherwise print to your terminal) into oblivion — it does
# NOT affect tee's separate write to output.txt at all, since that's
# an entirely independent output channel from tee's perspective.
# This is intentional and useful (e.g., silencing sudo tee's echoed
# confirmation while still writing the file), but the two output
# paths being independently controllable is easy to forget.
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
