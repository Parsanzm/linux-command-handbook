# rm — Practical Examples

> Real-world patterns for removing files and directories safely and
> deliberately.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Removing Directories](#removing-directories)
- [Safer, Confirmation-Based Removal](#safer-confirmation-based-removal)
- [Removing by Pattern](#removing-by-pattern)
- [Scripting with rm](#scripting-with-rm)
- [Combining rm with Other Tools](#combining-rm-with-other-tools)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Remove a single file
rm oldfile.txt

# Remove several files at once
rm file1.txt file2.txt file3.txt

# Remove with a confirmation message for each file
rm -v file1.txt file2.txt
# removed 'file1.txt'
# removed 'file2.txt'
```

---

## Removing Directories

```bash
# Remove an empty directory
rmdir empty_folder

# Remove a directory and everything inside it
rm -r project_backup/

# Remove, forcefully, with no prompts and no error on missing files
rm -rf temp_build/

# Remove several directories at once
rm -r dir1/ dir2/ dir3/
```

---

## Safer, Confirmation-Based Removal

```bash
# Prompt individually before removing each file
rm -i *.log

# A gentler prompt for a large recursive removal — asks ONCE rather
# than once per file
rm -Ir old_project/

# Preview what WOULD be removed before actually doing it, using find
find . -name "*.tmp" -print
# review the output carefully, THEN:
find . -name "*.tmp" -delete
```

---

## Removing by Pattern

```bash
# Remove all .log files in the current directory
rm *.log

# Remove all .tmp files recursively under a directory
find . -name "*.tmp" -delete

# Remove all files older than 30 days
find /tmp/cache -type f -mtime +30 -delete

# Remove all empty directories under a path
find . -type d -empty -delete
```

---

## Scripting with rm

```bash
# Clean up a temp directory at the end of a script, regardless of
# whether earlier steps succeeded
cleanup() {
  rm -rf "$TMPDIR"
}
trap cleanup EXIT

TMPDIR=$(mktemp -d)
# ... do work using $TMPDIR ...

# Remove a file only if it exists, without erroring if it doesn't
rm -f maybe_exists.txt

# Confirm a path is what's expected before removing it in a script —
# a basic sanity check against an empty/unset variable
if [ -n "$TARGET_DIR" ] && [ -d "$TARGET_DIR" ]; then
  rm -rf "$TARGET_DIR"
else
  echo "Refusing to remove: TARGET_DIR is unset or invalid"
  exit 1
fi
```

---

## Combining rm with Other Tools

```bash
# Remove files matching a pattern, listing them first for review
ls *.bak
rm *.bak

# Remove all files NOT matching a specific pattern
find . -type f ! -name "*.keep" -delete

# Remove build artifacts as part of a clean step
rm -rf build/ dist/ *.egg-info

# Securely remove a sensitive file, overwriting content first
shred -u credentials.txt
```

---

## Real-World Recipes

```bash
# --- Clean Up Old Log Files Older Than a Week ---
find /var/log/myapp -name "*.log" -mtime +7 -delete

# --- Remove a Directory Tree, But Only After Confirming It's the Right One ---
echo "About to permanently delete: /data/old_project"
read -p "Type YES to confirm: " confirm
[ "$confirm" = "YES" ] && rm -rf /data/old_project

# --- Clean Build Artifacts Before a Fresh Build ---
rm -rf node_modules dist build
npm install && npm run build

# --- Remove Temporary Files Created by a Script, on Exit ---
TMPFILE=$(mktemp)
trap 'rm -f "$TMPFILE"' EXIT
# ... use $TMPFILE ...

# --- Purge a Cache Directory While Keeping the Directory Itself ---
rm -rf /var/cache/myapp/*
# note: the trailing /* leaves the cache DIRECTORY itself intact,
# only removing its contents

# --- Remove Everything Except a Few Specific Files ---
find . -maxdepth 1 -type f ! -name "keep_this.txt" ! -name "and_this.txt" -delete
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
