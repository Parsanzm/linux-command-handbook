# mv — Practical Examples

> Real-world patterns for renaming, relocating, and batch-moving
> files and directories.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Renaming Files and Directories](#renaming-files-and-directories)
- [Moving Multiple Files](#moving-multiple-files)
- [Safe, Overwrite-Aware Moving](#safe-overwrite-aware-moving)
- [Batch Renaming with mv](#batch-renaming-with-mv)
- [Scripting with mv](#scripting-with-mv)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Move a file into a different directory
mv report.txt archive/

# Rename a file in place
mv draft.txt final.txt

# Move and rename in a single command
mv draft.txt archive/final_report.txt

# Verbose confirmation
mv -v draft.txt final.txt
# 'draft.txt' -> 'final.txt'
```

---

## Renaming Files and Directories

```bash
# Rename a directory
mv old_project_name/ new_project_name/

# Rename a file, changing its extension
mv notes.txt notes.md

# Fix a typo in a filename
mv recieved_files/ received_files/
```

---

## Moving Multiple Files

```bash
# Move several files into one destination directory
mv file1.txt file2.txt file3.txt destination/

# Move all .log files into an archive directory
mv *.log logs_archive/

# Move everything from one directory into another
mv source_dir/* destination_dir/

# Explicitly specify the target directory (useful in scripted contexts)
mv -t destination/ file1.txt file2.txt file3.txt
```

---

## Safe, Overwrite-Aware Moving

```bash
# Prompt before overwriting an existing destination file
mv -i new_config.yaml /etc/myapp/config.yaml

# Never overwrite — silently skip if the destination already exists
mv -n new_config.yaml /etc/myapp/config.yaml

# Only move if the source is actually newer than an existing destination
mv -u updated_data.csv /backups/data.csv

# Keep a numbered backup of anything that would otherwise be overwritten
mv -b important.conf /etc/myapp/important.conf
# creates important.conf~ as a backup of the previous version
```

---

## Batch Renaming with mv

```bash
# Rename all .txt files to .md in a loop
for f in *.txt; do
  mv "$f" "${f%.txt}.md"
done

# Add a prefix to every file in a directory
for f in *.jpg; do
  mv "$f" "vacation_$f"
done

# Remove spaces from filenames, replacing with underscores
for f in *' '*; do
  mv "$f" "${f// /_}"
done
```

---

## Scripting with mv

```bash
# Move a file only if it doesn't already exist at the destination
[ -f dest/file.txt ] || mv source/file.txt dest/file.txt

# Fail the script clearly if a move operation itself fails
mv important_file.txt /backup/ || { echo "Move failed"; exit 1; }

# Move a completed output file into place atomically, avoiding a
# reader ever seeing a partially-written file
process_data > output.tmp
mv output.tmp output.final
# same-filesystem mv is a single atomic rename — readers never see a
# half-written "output.final"

# Move today's log file into an archive directory named by date
mkdir -p "logs/$(date +%Y-%m-%d)"
mv current.log "logs/$(date +%Y-%m-%d)/current.log"
```

---

## Real-World Recipes

```bash
# --- Atomic "Publish" Pattern for Generated Files ---
generate_report > report.html.tmp
mv report.html.tmp report.html
# guarantees anyone reading report.html always sees either the
# COMPLETE old version or the COMPLETE new version, never a partial write

# --- Organize Downloaded Files by Type ---
mkdir -p images docs archives
mv *.jpg *.png images/ 2>/dev/null
mv *.pdf *.docx docs/ 2>/dev/null
mv *.zip *.tar.gz archives/ 2>/dev/null

# --- Rotate a Log File Without Losing In-Flight Writes ---
mv current.log "current.log.$(date +%Y%m%d)"
touch current.log
# (the application should be signaled to reopen its log file handle
# after this, depending on how it manages its own file descriptor)

# --- Rename a Batch of Files to Match a New Numbering Scheme ---
i=1
for f in photo_*.jpg; do
  mv "$f" "$(printf 'image_%03d.jpg' "$i")"
  i=$((i + 1))
done

# --- Move a Deployment's Output Into Its Final Versioned Location ---
mv "build/output" "releases/v$(cat VERSION)"
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
