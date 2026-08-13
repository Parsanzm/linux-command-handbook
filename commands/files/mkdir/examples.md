# mkdir — Practical Examples

> Real-world patterns for creating directories in scripts, setup
> routines, and everyday shell use.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Creating Nested Directory Structures](#creating-nested-directory-structures)
- [Creating Multiple Directories at Once](#creating-multiple-directories-at-once)
- [Setting Permissions at Creation](#setting-permissions-at-creation)
- [Scripting with mkdir](#scripting-with-mkdir)
- [Combining mkdir with Other Tools](#combining-mkdir-with-other-tools)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Create a single directory
mkdir logs

# Create with verbose output confirming what happened
mkdir -v logs
# mkdir: created directory 'logs'

# Attempt to create a directory that already exists (errors by default)
mkdir logs
# mkdir: cannot create directory 'logs': File exists
```

---

## Creating Nested Directory Structures

```bash
# Fails — "config" doesn't exist yet as a parent
mkdir config/app/settings

# Succeeds — creates config/, then config/app/, then config/app/settings/
mkdir -p config/app/settings

# Safe to re-run — no error even if the structure already exists
mkdir -p config/app/settings
```

---

## Creating Multiple Directories at Once

```bash
# Several sibling directories in one command
mkdir src tests docs

# A common project scaffold in one line
mkdir -p src/{main,test}/{java,resources}
# Brace expansion (a shell feature, not mkdir itself) combined with -p
# creates: src/main/java, src/main/resources, src/test/java, src/test/resources

# Verbose output for each one created
mkdir -v src tests docs
```

---

## Setting Permissions at Creation

```bash
# Private directory, owner-only access
mkdir -m 700 secrets

# Group-shared directory
mkdir -m 750 team-reports

# World-readable directory (use deliberately, not by default)
mkdir -m 755 public-assets
```

---

## Scripting with mkdir

```bash
# Standard "ensure this directory exists" pattern at the top of a script
mkdir -p "$OUTPUT_DIR"

# Create a timestamped directory for a build/backup run
BACKUP_DIR="backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Fail the script clearly if directory creation itself fails
mkdir -p "$OUTPUT_DIR" || { echo "Failed to create $OUTPUT_DIR"; exit 1; }

# Create a directory only if it doesn't already exist, without erroring
[ -d "$DIR" ] || mkdir -p "$DIR"
```

---

## Combining mkdir with Other Tools

```bash
# Create a directory and immediately move into it
mkdir new_project && cd new_project

# Create a directory structure and populate it with template files
mkdir -p myapp/{src,tests,docs}
touch myapp/src/main.py myapp/tests/test_main.py myapp/docs/README.md

# Create a directory and set ownership together (requires appropriate privileges)
mkdir /opt/myapp && sudo chown appuser:appgroup /opt/myapp

# Create output directories for each item in a list
while read -r name; do
  mkdir -p "output/$name"
done < names.txt
```

---

## Real-World Recipes

```bash
# --- Standard Project Scaffold ---
mkdir -p myproject/{src,tests,docs,scripts}
cd myproject

# --- Ensure a Log Directory Exists Before an Application Starts ---
mkdir -p /var/log/myapp
chmod 750 /var/log/myapp

# --- Create a Dated Backup Directory Each Time a Script Runs ---
BACKUP_DIR="/backups/$(date +%F)"
mkdir -p "$BACKUP_DIR"
cp -r /data/* "$BACKUP_DIR/"

# --- Set Up a Private Directory for Sensitive Output ---
mkdir -pm 700 ~/.local/share/myapp/secrets

# --- Build a Full Directory Tree for a New Environment ---
for env in dev staging prod; do
  mkdir -p "deployments/$env"/{config,logs,data}
done

# --- Create a Directory Only If Missing, Skipping Otherwise, With a Message ---
if [ -d "$TARGET_DIR" ]; then
  echo "$TARGET_DIR already exists, skipping"
else
  mkdir -p "$TARGET_DIR"
  echo "Created $TARGET_DIR"
fi
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
