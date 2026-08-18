# ln — Practical Examples

> Real-world patterns for symlinks, hard links, and the common
> "current version" deployment pattern.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Creating Symbolic Links](#creating-symbolic-links)
- [Creating Hard Links](#creating-hard-links)
- [Updating an Existing Link](#updating-an-existing-link)
- [Inspecting Links](#inspecting-links)
- [Combining ln with Other Tools](#combining-ln-with-other-tools)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Symbolic link (the common, everyday case)
ln -s /path/to/original.txt shortcut.txt

# Hard link (the default, no -s)
ln original.txt hardlink.txt

# Verbose confirmation
ln -sv /path/to/original.txt shortcut.txt
# '/path/to/original.txt' -> 'shortcut.txt'
```

---

## Creating Symbolic Links

```bash
# Link to a directory
ln -s /mnt/large_disk/data ~/data

# Link using a relative path, computed automatically
ln -sr /opt/myapp/releases/v2.3.0 /opt/myapp/current

# Create a symlink inside a different directory, keeping the target's basename
ln -s /opt/configs/myapp.conf /etc/myapp/

# Symlink a script into a directory already on PATH
ln -s ~/scripts/backup.sh /usr/local/bin/backup
```

---

## Creating Hard Links

```bash
# Create a second name for the same file
ln important.db important_backup.db

# Confirm both names point to the same inode
ls -i important.db important_backup.db

# Hard-link several files into another directory at once
ln file1.txt file2.txt file3.txt other_dir/
```

---

## Updating an Existing Link

```bash
# Force-overwrite an existing symlink to point somewhere new
ln -sf /opt/myapp/releases/v2.4.0 /opt/myapp/current

# Prompt before overwriting an existing link
ln -si /opt/myapp/releases/v2.4.0 /opt/myapp/current

# Correctly replace a symlink that points to a directory (without -n,
# this can accidentally create the link INSIDE that directory instead)
ln -sfn /opt/myapp/releases/v2.4.0 /opt/myapp/current
```

---

## Inspecting Links

```bash
# See what a symlink points to
ls -l /opt/myapp/current

# Fully resolve a symlink (or chain of them) to its real final path
readlink -f /opt/myapp/current

# Check a file's hard link count
stat important.db

# Find all symlinks under a directory
find /etc -type l

# Find BROKEN symlinks specifically
find /etc -xtype l
```

---

## Combining ln with Other Tools

```bash
# Symlink the latest backup for easy access, alongside dated backups
BACKUP="backup-$(date +%F).tar.gz"
tar czf "$BACKUP" ./data/
ln -sf "$BACKUP" latest_backup.tar.gz

# Loop over a list of scripts, symlinking each into a bin directory
for script in ~/scripts/*.sh; do
  ln -sf "$script" "/usr/local/bin/$(basename "$script" .sh)"
done

# Verify a symlink isn't broken before relying on it in a script
if [ -e /opt/myapp/current ]; then
  echo "Link target exists"
else
  echo "Broken symlink or missing target!"
fi
```

---

## Real-World Recipes

```bash
# --- Blue/Green Style "Current Release" Symlink Deployment ---
mkdir -p /opt/myapp/releases/v2.4.0
# ... deploy new version files into that directory ...
ln -sfn /opt/myapp/releases/v2.4.0 /opt/myapp/current
sudo systemctl restart myapp

# --- Provide a Stable Config Path While Rotating Actual Config Files ---
cp new_config.yaml "/etc/myapp/config-$(date +%Y%m%d).yaml"
ln -sf "config-$(date +%Y%m%d).yaml" /etc/myapp/config.yaml

# --- Save Disk Space with Hard Links for Identical, Unchanging Files ---
ln original_dataset.csv snapshot_2026-08-11.csv
# both names reference the SAME data — no extra disk space used,
# unlike a full cp

# --- Make a Script Available Without Duplicating or Modifying PATH ---
ln -s ~/projects/mytool/mytool.py ~/.local/bin/mytool
chmod +x ~/projects/mytool/mytool.py

# --- Clean Up Broken Symlinks Left Behind After Removing Old Releases ---
find /opt/myapp/releases -xtype l -delete
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
