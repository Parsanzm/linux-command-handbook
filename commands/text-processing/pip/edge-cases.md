# | (Pipe) — Edge Cases & Gotchas

> Pipes look simple, but stderr handling, exit codes, buffering, and process
> substitution quirks create some of the most common shell scripting bugs.

---

## Table of Contents

- [stderr Doesn't Flow Through the Pipe by Default](#stderr-doesnt-flow-through-the-pipe-by-default)
- [Exit Status Reflects Only the Last Command](#exit-status-reflects-only-the-last-command)
- [Order of Redirection and Pipe Matters](#order-of-redirection-and-pipe-matters)
- [grep Matching Its Own Process Name](#grep-matching-its-own-process-name)
- [Buffering Makes Real-Time Output Appear Delayed](#buffering-makes-real-time-output-appear-delayed)
- [Broken Pipe (SIGPIPE) When the Reader Exits Early](#broken-pipe-sigpipe-when-the-reader-exits-early)
- [Variables Set Inside a Pipeline Don't Survive (Subshell Scoping)](#variables-set-inside-a-pipeline-dont-survive-subshell-scoping)
- [uniq Only Removes ADJACENT Duplicates](#uniq-only-removes-adjacent-duplicates)
- [Piping Into a Command That Doesn't Read stdin](#piping-into-a-command-that-doesnt-read-stdin)
- [Infinite Producers Without a Consuming Limit](#infinite-producers-without-a-consuming-limit)
- [Pipe Buffer Size Limits Causing Deadlock in Rare Cases](#pipe-buffer-size-limits-causing-deadlock-in-rare-cases)
- [Confusing | with || (Logical OR)](#confusing--with--logical-or)
- [Whitespace-Sensitive Output Breaking Downstream Parsing](#whitespace-sensitive-output-breaking-downstream-parsing)

---

## stderr Doesn't Flow Through the Pipe by Default

### A pipe only connects stdout — error messages bypass it entirely
```bash
some_command_with_errors | grep "specific error"
# If some_command_with_errors writes its error to STDERR (which many
# well-behaved programs do, specifically SO that errors aren't
# accidentally captured by something expecting only normal output),
# that error message appears DIRECTLY on your terminal, completely
# bypassing grep — even if the error text would have matched "specific error"!

# Fix: explicitly merge stderr into stdout BEFORE the pipe takes it
some_command_with_errors 2>&1 | grep "specific error"
# Now BOTH streams flow into grep together, so error messages CAN be matched
```

---

## Exit Status Reflects Only the Last Command

### $? after a pipeline can hide an earlier failure completely
```bash
grep "pattern" nonexistent_file.txt | wc -l
# grep: nonexistent_file.txt: No such file or directory
# 0
echo $?
# 0     ⚠️ this is wc's exit status (wc succeeded at counting ZERO
# lines from grep's empty output), NOT grep's actual failure to even
# find the file at all — a script checking $? here would incorrectly
# conclude everything went fine.

# Fix #1: bash's PIPESTATUS array
grep "pattern" nonexistent_file.txt | wc -l
echo "${PIPESTATUS[0]}"
# 2     ← grep's REAL exit code, revealing the actual failure

# Fix #2: enable pipefail for the WHOLE pipeline's exit status
set -o pipefail
grep "pattern" nonexistent_file.txt | wc -l
echo $?
# 2     ← now reflects the FIRST non-zero exit code in the pipeline
```

---

## Order of Redirection and Pipe Matters

### Redirecting stdout to a file BEFORE merging stderr leaves nothing for the pipe
```bash
command > output.txt 2>&1 | grep "pattern"
# ⚠️ Parsed left to right: stdout goes to output.txt FIRST, THEN
# 2>&1 makes stderr ALSO go to wherever stdout currently points
# (which is now output.txt, not the pipe) — the pipe to grep receives
# NOTHING at all, since stdout was already diverted to the file
# before the pipe had a chance to capture it.

command 2>&1 | grep "pattern"
# ✅ Correct order: merge stderr into stdout WHILE stdout still feeds
# the pipe, so grep receives BOTH streams as intended.
```

---

## grep Matching Its Own Process Name

### A classic gotcha when searching for a running process by name
```bash
ps aux | grep nginx
# alice   12345  ...  grep nginx      ← grep's OWN invocation shows up
# root     6789  ...  nginx: master process
# www-data 6790  ...  nginx: worker process

# The command line "grep nginx" ITSELF contains the string "nginx",
# so grep matches its own process entry in ps's output — annoying
# when trying to script something based on the result count.

# Classic fix: bracket one character of the pattern, so grep's own
# invocation no longer literally contains the full search string
ps aux | grep '[n]ginx'
# root     6789  ...  nginx: master process
# www-data 6790  ...  nginx: worker process
# ✅ grep's own line ("grep [n]ginx") doesn't match its own pattern,
# since the regex [n]ginx doesn't literally appear as those exact
# characters in "grep [n]ginx" — a small regex trick, not magic.

# Alternative, arguably clearer: pgrep, purpose-built for this
pgrep nginx
```

---

## Buffering Makes Real-Time Output Appear Delayed

### Programs behave differently when their output is a pipe vs a terminal
```bash
some_verbose_command | grep "important"
# You might see NOTHING for a long stretch, then a burst of matching
# lines all at once — many programs switch from LINE buffering
# (immediate output, common when writing to an interactive terminal)
# to FULL/BLOCK buffering (output withheld until an internal buffer
# fills) the moment their output is redirected to a pipe rather than
# a terminal, purely as a performance optimization on THEIR end.

# Force line-buffered behavior on the PRODUCING command when
# real-time visibility through the pipe actually matters:
stdbuf -oL some_verbose_command | grep --line-buffered "important"
# stdbuf forces the PRODUCER to line-buffer; --line-buffered tells
# grep itself to flush its OWN output after every matching line too
```

---

## Broken Pipe (SIGPIPE) When the Reader Exits Early

### The downstream command finishing before the upstream one is expected, not a bug
```bash
yes "test" | head -3
# test
# test
# test
# head exits after printing exactly 3 lines, closing ITS end of the
# pipe — "yes" (which produces output FOREVER otherwise) then
# receives a SIGPIPE signal on its NEXT write attempt and terminates.

# Sometimes this produces a visible message on stderr:
seq 1 1000000 | head -3
# 1
# 2
# 3
# seq: write error: Broken pipe
# ⚠️ This "error" is expected/harmless in this exact scenario — the
# intended 3 lines were already successfully produced and captured;
# it only looks alarming to someone unfamiliar with normal pipe-closing behavior.
```

---

## Variables Set Inside a Pipeline Don't Survive (Subshell Scoping)

### Each stage of a pipeline typically runs in its OWN subshell
```bash
count=0
cat file.txt | while read -r line; do
  count=$((count + 1))
done
echo "Count: $count"
# Count: 0     ⚠️ NOT the actual number of lines processed!
# The `while` loop runs in a SUBSHELL (because it's the right-hand
# side of a pipe), so any variable changes made INSIDE that subshell
# are completely lost once the subshell exits — the ORIGINAL shell's
# "count" variable was never actually touched.

# Fix #1: avoid the pipe, use input redirection into the loop instead
count=0
while read -r line; do
  count=$((count + 1))
done < file.txt
echo "Count: $count"
# Count: 5     ✅ correct — no pipe/subshell involved this way

# Fix #2 (bash-specific): enable lastpipe so the LAST command in a
# pipeline runs in the current shell instead of a subshell (with caveats)
shopt -s lastpipe
count=0
cat file.txt | while read -r line; do
  count=$((count + 1))
done
echo "Count: $count"
# Count: 5     ✅ works, but has its own subtleties (job control must
# be off, and this is bash-specific, not portable to all shells)
```

---

## uniq Only Removes ADJACENT Duplicates

### Forgetting to sort first produces surprising, incomplete deduplication
```bash
printf "b\na\nb\na\n" | uniq
# b
# a
# b
# a
# ⚠️ NOTHING was removed! uniq only collapses CONSECUTIVE duplicate
# lines — since no two adjacent lines here are identical, uniq has
# nothing to do, even though "b" and "a" are each repeated overall.

printf "b\na\nb\na\n" | sort | uniq
# a
# b
# ✅ sorting FIRST groups identical lines together, so uniq can
# actually find and collapse the now-adjacent duplicates correctly.
```

---

## Piping Into a Command That Doesn't Read stdin

### Some commands expect a FILE ARGUMENT, not piped input, and silently ignore stdin
```bash
cat file.txt | rm
# rm: missing operand
# Try 'rm --help' for more information.
# ⚠️ `rm` doesn't read filenames from stdin at all — it expects
# filenames as command-line ARGUMENTS. Piping into it does nothing
# useful; rm simply ignores the piped data entirely and complains
# about missing arguments instead.

# Fix: use xargs to convert piped LINES into ARGUMENTS for commands
# that don't natively read stdin
cat filelist.txt | xargs rm
# Now each line from filelist.txt becomes a separate ARGUMENT passed
# directly to rm, which is what rm actually expects.
```

---

## Infinite Producers Without a Consuming Limit

### Forgetting a limiting stage can hang a terminal or exhaust resources
```bash
yes | grep "never matches this string"
# ⚠️ "yes" produces the string "y" FOREVER, and since grep never finds
# a match, it never exits either (it just keeps reading and
# discarding forever) — this pipeline runs indefinitely, consuming
# CPU continuously, until manually interrupted with Ctrl+C.

# If a limiting stage IS needed, make sure one is actually present:
yes | head -100 | grep "y"    # bounded — head caps the input at 100 lines
```

---

## Pipe Buffer Size Limits Causing Deadlock in Rare Cases

### Two processes writing to EACH OTHER's pipes can deadlock if buffers fill
```bash
# This is a more advanced/rare scenario, but worth knowing: if TWO
# processes are each simultaneously trying to WRITE more data than
# the OTHER's pipe buffer (commonly 64KB on Linux) can hold, while
# ALSO waiting to READ from each other before continuing, they can
# reach a DEADLOCK — each blocked waiting on a full buffer the other
# side isn't yet reading from, because it's ALSO blocked in the same way.

# This specific scenario is uncommon in typical simple pipelines (which
# flow in one direction), but becomes a real design concern in custom
# bidirectional IPC setups using multiple named pipes (FIFOs) — a
# reminder that pipe buffers are FINITE, not unlimited magic conduits.
```

---

## Confusing | with || (Logical OR)

### A single typo changes the entire meaning of the command
```bash
command1 | command2
# PIPE: command1's OUTPUT becomes command2's INPUT — they're CONNECTED

command1 || command2
# LOGICAL OR: command2 runs ONLY IF command1 FAILS (non-zero exit) —
# they are otherwise COMPLETELY INDEPENDENT, no data flows between them at all

mkdir /tmp/newdir || echo "mkdir failed"
# ✅ correct use of || : run echo ONLY if mkdir fails

mkdir /tmp/newdir | echo "mkdir failed"
# ⚠️ almost certainly a MISTAKE: this pipes mkdir's (usually empty)
# stdout into echo, which then unconditionally prints "mkdir failed"
# REGARDLESS of whether mkdir actually succeeded or failed — the
# intended conditional logic is completely broken by using | instead of ||
```

---

## Whitespace-Sensitive Output Breaking Downstream Parsing

### Extra spaces/tabs in piped output can silently break naive field-splitting
```bash
ps aux | awk '{print $2}'
# Works reliably for MOST ps output, since awk's default field
# splitting handles multiple consecutive spaces as ONE separator —
# but not every command's output is this well-behaved

some_command_with_irregular_spacing | cut -d' ' -f3
# ⚠️ cut, unlike awk, treats EACH individual space character as a
# SEPARATE delimiter — multiple consecutive spaces produce EMPTY
# fields in between, silently shifting which "column" f3 actually
# refers to compared to what a human eye would expect when LOOKING
# at the aligned, space-padded output.

some_command_with_irregular_spacing | awk '{print $3}'
# ✅ awk's default behavior treats runs of whitespace as a SINGLE
# separator, generally matching human visual expectations of
# "column 3" far more reliably than cut does for irregularly-spaced text.
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
