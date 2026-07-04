# zip / unzip — Edge Cases & Gotchas

> ZIP's universal compatibility comes at the cost of inconsistent implementations —
> what "works" in one tool can silently misbehave in another.

---

## Table of Contents

- [Zip Slip — Path Traversal via Malicious Archive Names](#zip-slip--path-traversal-via-malicious-archive-names)
- [ZipCrypto Is Not Real Security](#zipcrypto-is-not-real-security)
- [Password on the Command Line Leaks Everywhere](#password-on-the-command-line-leaks-everywhere)
- [Unix Permissions Are Not Reliably Preserved](#unix-permissions-are-not-reliably-preserved)
- [Symlinks: Stored as Symlinks vs Followed](#symlinks-stored-as-symlinks-vs-followed)
- [Filename Encoding Mismatches (Mojibake)](#filename-encoding-mismatches-mojibake)
- [Case-Sensitivity Collisions](#case-sensitivity-collisions)
- [Silent Overwrite vs Prompt Behavior](#silent-overwrite-vs-prompt-behavior)
- [Nested Zip Bombs](#nested-zip-bombs)
- [4GB / 65535-File Limits Without Zip64](#4gb--65535-file-limits-without-zip64)
- [Corrupted Central Directory](#corrupted-central-directory)
- [Trailing Garbage / Self-Extracting Archives](#trailing-garbage--self-extracting-archives)
- [Line Ending Conversion Surprises](#line-ending-conversion-surprises)
- [-x Exclude Patterns Need Quoting](#-x-exclude-patterns-need-quoting)

---

## Zip Slip — Path Traversal via Malicious Archive Names

### A maliciously crafted archive can write files OUTSIDE the extraction directory
```bash
# An attacker crafts a zip entry with a path like:
#   ../../../../etc/cron.d/malicious
# stored inside what looks like an innocent archive.

unzip untrusted.zip -d /tmp/safe_extraction/
# Depending on the unzip version and OS, this can write a file to
# /etc/cron.d/malicious instead of staying inside /tmp/safe_extraction/
# — a well-known vulnerability class called "Zip Slip."

# Modern Info-ZIP unzip has mitigations and will usually refuse or warn:
# "warning: skipped '../../../etc/cron.d/malicious' path traversal attempt"
# But NOT ALL zip-handling libraries (especially in application code, not
# the command-line tool) implement this protection correctly.

# Never extract an untrusted archive with an old/unpatched unzip, and
# never extract as root unless absolutely necessary:
unzip -t untrusted.zip                 # test integrity first
unzip untrusted.zip -d /tmp/sandbox/   # extract as an unprivileged user, in isolation
```

---

## ZipCrypto Is Not Real Security

### The classic zip -e / -P password is trivially breakable
```bash
zip -e secrets.zip passwords.txt
# Uses "ZipCrypto" by default in standard Info-ZIP — a stream cipher
# from 1989 with known, practical known-plaintext and known-ciphertext
# attacks. Tools exist that can recover ZipCrypto passwords in minutes
# to hours depending on available plaintext, NOT centuries.

# If anything inside the archive has a predictable format (e.g., a PDF
# header, a common file type with known bytes), the "encryption" is
# effectively broken from the start.

# For real protection, don't rely on zip's built-in password feature:
zip -r archive.zip sensitive/
gpg -c archive.zip                # AES-256 via GPG symmetric encryption
rm archive.zip                    # remove the weakly-protected original
```

---

## Password on the Command Line Leaks Everywhere

### -P PASSWORD is visible to anyone who can see running processes
```bash
zip -P "MySecretPass123" archive.zip file.txt
# While this command runs, ANY other user on the same machine can see
# the full password by running:
ps aux | grep zip
# ... zip -P MySecretPass123 archive.zip file.txt

# It's ALSO saved in your shell history file:
history | grep MySecretPass123
cat ~/.bash_history | grep zip

# Fix: use the interactive prompt instead, which never appears in
# process listings or history:
zip -e archive.zip file.txt
# Enter password: (typed, not echoed, not logged)

# If scripting is unavoidable, read the password from a protected file
# or environment variable set outside the command line itself:
zip -P "$(cat /run/secrets/zip_password)" archive.zip file.txt
```

---

## Unix Permissions Are Not Reliably Preserved

### ZIP was designed on DOS/Windows — Unix metadata support is a later bolt-on
```bash
chmod 700 script.sh
zip archive.zip script.sh
unzip archive.zip
ls -l script.sh
# Permissions may come back as something generic like 644 or 755,
# NOT necessarily the original 700 — depends on the specific zip/unzip
# implementation and whether "extra field" Unix permission data was
# written and correctly interpreted on both ends.

# tar preserves permissions with full fidelity because it was designed
# for Unix from the start; zip's Unix permission support is an
# extension, not part of the core format, and support varies:
tar -czf archive.tar.gz script.sh    # preferred when exact perms matter
tar -xzf archive.tar.gz
ls -l script.sh                       # 700 preserved reliably

# Always verify critical permissions after extracting from zip,
# especially for executable scripts or restricted-access files.
chmod 700 script.sh   # re-apply explicitly after extraction if needed
```

---

## Symlinks: Stored as Symlinks vs Followed

### Without -y, zip may follow symlinks and store the TARGET's content
```bash
ln -s /etc/hostname mylink
zip archive.zip mylink
# Depending on version/defaults, this may store a COPY of /etc/hostname's
# actual content under the name "mylink", rather than storing "mylink"
# as a symlink pointing to /etc/hostname.

# Force zip to store symlinks AS symlinks (matches tar's default behavior):
zip -y archive.zip mylink

unzip archive.zip
ls -l mylink
# lrwxrwxrwx ... mylink -> /etc/hostname   (correctly restored as a symlink)

# Without -y, extracting the archive elsewhere would instead produce a
# REGULAR FILE containing a snapshot of what /etc/hostname held at
# archive-creation time — silently different semantics than the original.
```

---

## Filename Encoding Mismatches (Mojibake)

### Non-ASCII filenames can turn into garbage across platforms
```bash
# A zip created on Windows with a filename like "résumé.docx" using
# the legacy CP437 or system codepage encoding...
unzip archive.zip
ls
# r├®sum├®.docx    ⚠️ mangled — encoding mismatch between creator and extractor

# Modern zip supports UTF-8 filename flags (bit 11 in the general
# purpose flag), but older zip tools or archives created without it
# may not signal UTF-8 correctly, leading to mojibake on extraction.

# Force UTF-8 interpretation when extracting (Info-ZIP unzip):
unzip -O UTF-8 archive.zip

# When CREATING, ensure your zip build/version defaults to UTF-8 filename
# encoding (most modern Info-ZIP builds do this automatically since
# roughly the 6.0 era, but very old systems may not).
```

---

## Case-Sensitivity Collisions

### An archive created on a case-insensitive filesystem can produce collisions on Linux
```bash
# On macOS (default HFS+/APFS case-insensitive mode) or Windows,
# "File.txt" and "file.txt" are the SAME file — but a maliciously or
# carelessly constructed zip could contain BOTH as separate entries.

unzip archive.zip
# Archive:  archive.zip
#  extracting: File.txt
#  extracting: file.txt   ← on Linux (case-sensitive), both extract separately
# On a case-insensitive extraction target, the second overwrites the first
# silently, and the outcome depends on extraction ORDER, not anything
# you can predict just by looking at the archive listing.

# If cross-platform consistency matters, audit archive contents first:
unzip -l archive.zip | awk '{print tolower($4)}' | sort | uniq -d
# Lists any names that only differ by case — a red flag before extracting
```

---

## Silent Overwrite vs Prompt Behavior

### Default behavior differs from what many people expect
```bash
unzip archive.zip
# If files already exist in the destination, unzip PROMPTS interactively:
# replace existing_file.txt? [y]es, [n]o, [A]ll, [N]one, [r]ename:
# In a NON-interactive script (e.g., cron, CI), this prompt has no
# terminal to answer it and the process can hang indefinitely waiting
# for input that will never come.

# Always be explicit in scripts:
unzip -o archive.zip     # force overwrite, never prompt
# or:
unzip -n archive.zip     # never overwrite, skip conflicts silently, never prompt

# A script without -o or -n that hits an existing file will simply HANG
# in a cron job or CI pipeline with no visible error — a classic silent
# failure mode that's confusing to debug from logs alone.
```

---

## Nested Zip Bombs

### A tiny file can expand to an enormous, disk-filling size
```bash
# A specially crafted zip (or worse, a zip containing zips containing
# zips, recursively) can have an extremely high compression ratio —
# a few KB on disk that decompresses into gigabytes or more.

unzip -l suspicious.zip
# Compressed size and uncompressed size are BOTH shown by -l —
# a wildly disproportionate ratio (e.g., 1KB compressed → 10GB uncompressed)
# is a strong warning sign before you actually extract it.

unzip -v suspicious.zip
# Shows compression ratio per file explicitly — check for absurd values

# NEVER extract an untrusted archive without checking this first,
# especially on a system with limited disk space, or inside an
# automated pipeline that extracts archives unattended.
```

---

## 4GB / 65535-File Limits Without Zip64

### Old tools silently truncate or fail on very large archives
```bash
# Modern Info-ZIP automatically uses Zip64 extensions when needed, but
# an OLDER unzip implementation (or certain minimal/embedded tools,
# some older Java libraries, some older Windows built-in zip support)
# may not understand Zip64 at all.

unzip old_tool_output.zip
# error: expected central file header signature not found
# (or) reports fewer files than actually exist, silently truncating the listing

# If you know the archive will be opened by unknown/older tooling,
# consider splitting into multiple smaller archives instead of relying
# on Zip64 support being universal:
zip -r -s 100m split_archive.zip large_folder/
# Creates split_archive.z01, .z02, ... .zip (100MB volumes)
```

---

## Corrupted Central Directory

### A truncated download can be far harder to recover than a truncated tar.gz
```bash
# If a zip file's download is interrupted partway through, the Central
# Directory (which lives at the very END of the file) may be missing
# or incomplete entirely.
unzip -l truncated.zip
# error:  zipfile is empty -- OR --
# End-of-central-directory signature not found

# Because standard unzip relies on seeking to the END first, a
# truncated zip can appear completely unreadable even if 99% of the
# actual file data is intact and undamaged earlier in the file.

# Recovery tools exist for exactly this scenario:
zip -FF broken.zip --out fixed.zip     # "fix" mode, attempts full recovery
# or:
unzip -FI truncated.zip                # some builds support a "find" mode

# tar.gz, by contrast, is a sequential stream and can often still yield
# whatever files were fully written before the truncation point, even
# without any special "repair" step — a meaningful practical difference
# when dealing with unreliable transfers.
```

---

## Trailing Garbage / Self-Extracting Archives

### Not every "corrupt" zip warning means real data loss
```bash
unzip archive.zip
# warning [archive.zip]:  128 extra bytes at beginning or within zipfile
# (attempting to process anyway)

# This commonly happens with SELF-EXTRACTING archives (an .exe stub
# prepended to valid zip data) or files that had extra bytes appended/
# prepended by an email gateway or upload tool. The Central Directory
# offsets are relative to where the ACTUAL zip data starts, and unzip
# is usually smart enough to locate it despite the extra bytes —
# but always verify contents after a warning like this appears:
unzip -t archive.zip     # confirm every entry actually extracts cleanly
```

---

## Line Ending Conversion Surprises

### -a can silently rewrite binary files if misapplied
```bash
# -a auto-converts text file line endings (CRLF <-> LF) based on a
# heuristic that inspects file content — but that heuristic isn't perfect.
unzip -a archive.zip
# A binary file that happens to LOOK text-like in its first few bytes
# could have bytes altered during "conversion," subtly corrupting it.

# Safer: only use -a when you KNOW the archive contains text files
# originating from a different platform, and verify binary files
# afterward with a checksum comparison if in doubt:
unzip archive.zip                              # without -a, no conversion
sha256sum extracted_file.bin                   # compare against a known-good hash
```

---

## -x Exclude Patterns Need Quoting

### Unquoted glob patterns get expanded by the SHELL before zip ever sees them
```bash
# Intended: exclude all .log files anywhere under the tree
zip -r archive.zip myproject/ -x *.log
# ⚠️ If any .log files exist in the CURRENT directory, the shell expands
# *.log into actual filenames BEFORE zip runs, so zip receives something
# like "-x file1.log file2.log" instead of the literal pattern "*.log" —
# and files elsewhere in the tree won't be excluded as intended.

# Fix: always QUOTE exclude/include patterns so zip receives the raw
# glob string and applies it itself, recursively:
zip -r archive.zip myproject/ -x "*.log"
zip -r archive.zip myproject/ -x "*/.git/*" -x "*.tmp"
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
