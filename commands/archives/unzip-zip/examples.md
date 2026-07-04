# zip / unzip — Practical Examples

> Real-world patterns for backups, deployment, sharing, and everyday file handling.

---

## Table of Contents

- [Basic Archive Creation](#basic-archive-creation)
- [Basic Extraction](#basic-extraction)
- [Listing & Inspecting Without Extracting](#listing--inspecting-without-extracting)
- [Updating & Modifying Existing Archives](#updating--modifying-existing-archives)
- [Excluding Files](#excluding-files)
- [Password-Protected Archives](#password-protected-archives)
- [Compression Level Tuning](#compression-level-tuning)
- [Selective Extraction](#selective-extraction)
- [Piping & Streaming](#piping--streaming)
- [Cross-Platform Considerations](#cross-platform-considerations)
- [Scripting with zip/unzip](#scripting-with-zipunzip)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Archive Creation

```bash
# Zip a few specific files
zip archive.zip file1.txt file2.txt report.pdf

# Zip an entire directory (recursive is REQUIRED for directories)
zip -r archive.zip myfolder/

# Zip the current directory's contents into an archive one level up
zip -r ../backup.zip .

# Zip multiple directories at once
zip -r archive.zip folder1/ folder2/ folder3/

# Verbose creation — see every file as it's added
zip -rv archive.zip myfolder/
```

---

## Basic Extraction

```bash
# Extract into the current directory
unzip archive.zip

# Extract into a specific destination (creates the dir if it doesn't exist)
unzip archive.zip -d /path/to/destination

# Extract, automatically overwriting anything that already exists
unzip -o archive.zip

# Extract, but skip any file that already exists (don't touch it)
unzip -n archive.zip

# Quiet extraction (no per-file listing spam — useful in scripts)
unzip -q archive.zip -d /opt/app
```

---

## Listing & Inspecting Without Extracting

```bash
# Quick listing: filenames, sizes, dates
unzip -l archive.zip

# Verbose listing: adds compression ratio, method, CRC per file
unzip -v archive.zip

# Detailed zipinfo-style output (even more detail than -v)
zipinfo archive.zip
zipinfo -l archive.zip

# Check whether a file is even a valid zip archive
file archive.zip
unzip -t archive.zip     # test integrity — reports OK or errors per entry

# Just check if a SPECIFIC file exists inside the archive
unzip -l archive.zip | grep "config.yaml"
```

---

## Updating & Modifying Existing Archives

```bash
# Add a new file to an existing archive
zip archive.zip newfile.txt

# Update: replace entries only if the source file is NEWER, add anything new
zip -u archive.zip *.txt

# Freshen: like -u, but ONLY updates existing entries, never adds new files
zip -f archive.zip

# Delete a specific file from an archive (without re-extracting/re-zipping everything)
zip -d archive.zip unwanted_file.txt

# Delete an entire directory's worth of entries from within the archive
zip -d archive.zip "old_folder/*"

# Rename effectively by delete + re-add
zip -d archive.zip oldname.txt
zip archive.zip newname.txt
```

---

## Excluding Files

```bash
# Exclude a specific pattern while creating
zip -r archive.zip myproject/ -x "*.log"

# Exclude multiple patterns
zip -r archive.zip myproject/ -x "*.log" -x "*.tmp" -x "*/.git/*"

# Exclude common development artifacts (typical project archive)
zip -r archive.zip myproject/ \
  -x "*/.git/*" \
  -x "*/node_modules/*" \
  -x "*/__pycache__/*" \
  -x "*.pyc"

# Include only specific file types (inverse of exclude)
zip -r archive.zip myproject/ -i "*.py" "*.md"
```

---

## Password-Protected Archives

```bash
# Create with an interactive password prompt (recommended over -P)
zip -e archive.zip sensitive_file.txt
# Enter password:
# Verify password:

# Non-interactive (⚠️ password ends up in shell history / process list — see edge-cases.md)
zip -P "TempPass123" archive.zip sensitive_file.txt

# Extract, prompting interactively for the password
unzip archive.zip

# Extract non-interactively with a known password
unzip -P "TempPass123" archive.zip

# For genuinely sensitive data, layer GPG on top instead of relying on ZIP's own weak encryption
zip -r archive.zip sensitive_folder/
gpg -c archive.zip                       # produces archive.zip.gpg, prompts for passphrase
rm archive.zip                           # remove the unencrypted intermediate
# Later, to decrypt:
gpg -d archive.zip.gpg > archive.zip
```

---

## Compression Level Tuning

```bash
# Maximum compression (slowest, smallest file) — good for archival storage
zip -9 -r archive.zip myfolder/

# No compression, just bundling (fastest) — good for already-compressed media
zip -0 -r archive.zip photos/

# Compare sizes/speed at different levels
time zip -1 -r fast.zip myfolder/
time zip -9 -r small.zip myfolder/
ls -lh fast.zip small.zip

# Practical tip: skip compression entirely for folders full of jpg/mp4/zip files —
# they're already compressed and -0 saves CPU time for negligible size difference
zip -0 -r media_bundle.zip videos/
```

---

## Selective Extraction

```bash
# Extract only files matching a pattern
unzip archive.zip "*.pdf"

# Extract only a specific subfolder from within the archive
unzip archive.zip "docs/*"

# Extract just ONE specific file
unzip archive.zip "config/settings.json"

# Extract everything EXCEPT certain patterns
unzip archive.zip -x "*.log" "*.tmp"

# Extract, flattening directory structure (discard the folder hierarchy)
unzip -j archive.zip -d flat_output/
```

---

## Piping & Streaming

```bash
# View a single file's content WITHOUT extracting it to disk
unzip -p archive.zip path/inside/archive.txt

# Pipe a file's content directly into another command
unzip -p archive.zip data.csv | head -20
unzip -p archive.zip config.json | jq '.version'

# Create a zip archive and stream it directly (e.g., piping to another host)
zip -r - myfolder/ | ssh user@remote "cat > backup.zip"

# Count lines in a compressed CSV without ever extracting it to disk
unzip -p archive.zip data.csv | wc -l
```

---

## Cross-Platform Considerations

```bash
# Convert line endings when extracting text files for the current platform
unzip -a archive.zip

# Preserve symlinks as symlinks rather than copying their target's content
zip -ry archive.zip myfolder/

# Extract with lowercase filenames (useful for archives created on
# case-insensitive filesystems, like older macOS/Windows exports)
unzip -L archive.zip

# Case-insensitive matching when extracting specific files
unzip -C archive.zip "readme.txt"
```

---

## Scripting with zip/unzip

```bash
# Check if a zip archive is valid before trusting it in a pipeline
if unzip -t archive.zip > /dev/null 2>&1; then
  echo "Archive is valid"
else
  echo "Archive is corrupt or not a zip file"
  exit 1
fi

# Extract only if the archive contains a specific expected file
if unzip -l archive.zip | grep -q "manifest.json"; then
  unzip -o archive.zip -d /opt/app
else
  echo "Missing expected manifest.json — aborting"
  exit 1
fi

# Safely extract into a temp directory first, then move into place
TMPDIR=$(mktemp -d)
unzip -q archive.zip -d "$TMPDIR"
mv "$TMPDIR"/* /opt/app/
rmdir "$TMPDIR"

# Get just the list of filenames (no headers/footers) for further processing
unzip -Z1 archive.zip
# Equivalent to zipinfo -1, gives a clean, script-friendly file list
```

---

## Real-World Recipes

```bash
# --- Sharing a Project with a Non-Technical Recipient ---

zip -r project_share.zip project/ \
  -x "*/.git/*" -x "*/node_modules/*" -x "*.env"

# --- Website Deployment Package ---

cd /var/www/html
zip -r ../deploy_$(date +%Y%m%d).zip . -x "*.log"
scp ../deploy_$(date +%Y%m%d).zip user@server:/tmp/
ssh user@server "cd /var/www/html && unzip -o /tmp/deploy_*.zip"

# --- Quick Backup Before a Risky Change ---

zip -r backup_before_change_$(date +%s).zip config/ data/

# --- Extracting a Downloaded Archive Safely ---

unzip -t downloaded.zip && unzip downloaded.zip -d ~/Downloads/extracted/

# --- Splitting Work: Compress Logs, Keep Originals Deletable ---

zip -9r logs_archive.zip /var/log/myapp/*.log
zip -d logs_archive.zip "*.tmp.log"       # remove accidental temp entries after the fact

# --- Bundling Build Artifacts for CI/CD ---

cd dist/
zip -r ../release-v1.2.3.zip . -x "*.map"
cd ..
# upload release-v1.2.3.zip to release page / artifact storage

# --- Emailing a Password-Protected Archive of Sensitive Documents ---

zip -e -r confidential.zip documents/
# share the password through a SEPARATE channel (never in the same email)
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
