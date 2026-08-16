# cp — Practical Examples

> Real-world patterns for copying files, directories, and backups
> safely and predictably.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Copying Directories](#copying-directories)
- [Preserving Attributes](#preserving-attributes)
- [Safe, Overwrite-Aware Copying](#safe-overwrite-aware-copying)
- [Copying Multiple Files](#copying-multiple-files)
- [Scripting with cp](#scripting-with-cp)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Copy a single file, giving the copy a new name
cp original.txt copy.txt

# Copy a file into an existing directory, keeping its name
cp report.txt archive/

# Copy with verbose confirmation
cp -v report.txt archive/
# 'report.txt' -> 'archive/report.txt'
```

---

## Copying Directories

```bash
# Recursively copy an entire directory
cp -r project/ project_backup/

# Copy a directory's CONTENTS into an already-existing destination
cp -r project/. existing_backup_dir/

# Full archive copy — recursive, all attributes and symlinks preserved
cp -a project/ project_backup/
```

---

## Preserving Attributes

```bash
# Preserve timestamps, ownership, and permissions
cp -p config.yaml config.yaml.bak

# Preserve everything, including symlink structure, for a directory tree
cp -a /etc/myapp/ /backups/myapp-config/

# Compare timestamps before and after to see the difference -p makes
touch -d "2020-01-01" original.txt
cp original.txt without_preserve.txt
cp -p original.txt with_preserve.txt
stat original.txt without_preserve.txt with_preserve.txt
```

---

## Safe, Overwrite-Aware Copying

```bash
# Prompt before overwriting an existing file
cp -i new_config.yaml /etc/myapp/config.yaml

# Never overwrite — silently skip if the destination already exists
cp -n new_config.yaml /etc/myapp/config.yaml

# Only copy if the source is newer than an existing destination
cp -u source_data.csv /backups/source_data.csv
```

---

## Copying Multiple Files

```bash
# Copy several files into one destination directory
cp file1.txt file2.txt file3.txt destination/

# Copy all .txt files in the current directory into another
cp *.txt archive/

# Copy files matching a pattern, recursively, preserving structure
find . -name "*.log" -exec cp {} /var/log/collected/ \;
```

---

## Scripting with cp

```bash
# Back up a config file before modifying it, with a timestamp in the name
cp config.yaml "config.yaml.bak.$(date +%Y%m%d-%H%M%S)"

# Copy a file only if it doesn't already exist at the destination
[ -f dest/file.txt ] || cp source/file.txt dest/file.txt

# Fail the script clearly if a copy operation itself fails
cp important_file.txt /backup/ || { echo "Backup failed"; exit 1; }

# Copy a whole directory tree into a fresh, uniquely named location
DEST="releases/release-$(date +%Y%m%d)"
mkdir -p "$DEST"
cp -a build/. "$DEST/"
```

---

## Real-World Recipes

```bash
# --- Back Up a File Before Editing It ---
cp important_config.conf important_config.conf.bak
vim important_config.conf

# --- Clone a Project Directory for a New Feature Branch's Local Testing ---
cp -a ./myproject/ ./myproject-testing/

# --- Copy Only Newer Files During a Manual, Simple Sync ---
cp -u -r ./source_dir/. ./dest_dir/

# --- Deploy a Build Artifact into a Versioned Directory ---
VERSION=$(cat VERSION)
mkdir -p "/opt/myapp/releases/$VERSION"
cp -a ./dist/. "/opt/myapp/releases/$VERSION/"
ln -sfn "/opt/myapp/releases/$VERSION" /opt/myapp/current

# --- Copy a Template File for Each Entry in a List ---
while read -r name; do
  cp template.conf "configs/${name}.conf"
done < names.txt

# --- Space-Efficient Copy-on-Write Duplicate (Btrfs/XFS/APFS) ---
cp --reflink=auto large_dataset.img large_dataset_copy.img
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
