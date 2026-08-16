# mv — Edge Cases & Gotchas

> `mv` looks like a simple rename/relocate tool, but cross-filesystem
> behavior, silent overwrites, and trailing-slash ambiguity routinely
> produce results that differ from what people expect.

---

## Table of Contents

- [mv Silently Overwrites the Destination by Default — No Warning at All](#mv-silently-overwrites-the-destination-by-default--no-warning-at-all)
- [Moving Across Filesystems Is NOT Atomic — And Can Fail Midway](#moving-across-filesystems-is-not-atomic--and-can-fail-midway)
- [Whether the Destination Is an Existing Directory Changes Everything](#whether-the-destination-is-an-existing-directory-changes-everything)
- [Same-Filesystem mv Is Genuinely Atomic — a Reader Never Sees a Half-Moved File](#same-filesystem-mv-is-genuinely-atomic--a-reader-never-sees-a-half-moved-file)
- [Moving a Directory Onto Itself, or Into Its Own Subdirectory, Fails or Misbehaves](#moving-a-directory-onto-itself-or-into-its-own-subdirectory-fails-or-misbehaves)
- [A Cross-Filesystem Move That Fails Partway Can Leave BOTH Copies Existing](#a-cross-filesystem-move-that-fails-partway-can-leave-both-copies-existing)
- [-u Compares Timestamps, Which Can Be Wrong If Clocks or Timezones Differ](#-u-compares-timestamps-which-can-be-wrong-if-clocks-or-timezones-differ)
- [mv Doesn't Follow Symlinks the Way cp Sometimes Does — It Moves the Link Itself](#mv-doesnt-follow-symlinks-the-way-cp-sometimes-does--it-moves-the-link-itself)
- [Renaming a File Someone Else Has Open Doesn't Break Their Access](#renaming-a-file-someone-else-has-open-doesnt-break-their-access)
- [-T Forces DEST to Be Treated as a File, Not a Directory — Changes Behavior Significantly](#-t-forces-dest-to-be-treated-as-a-file-not-a-directory--changes-behavior-significantly)
- [Moving Many Small Files Across Filesystems Is Much Slower Than One Archive](#moving-many-small-files-across-filesystems-is-much-slower-than-one-archive)

---

## mv Silently Overwrites the Destination by Default — No Warning at All

### The single most consequential mv default to know
```bash
mv new_draft.txt final_report.txt
# ⚠️ If final_report.txt already existed with different, important
# content, it's now GONE — overwritten immediately with no
# confirmation, no backup, and no built-in undo through mv itself.

mv -i new_draft.txt final_report.txt
# mv: overwrite 'final_report.txt'? y
# -i prompts first, giving a chance to back out

mv -n new_draft.txt final_report.txt
# Silently skips the move entirely if final_report.txt already
# exists — never overwrites, no prompt needed
```

---

## Moving Across Filesystems Is NOT Atomic — And Can Fail Midway

### The instant, all-or-nothing behavior only applies within a single filesystem
```bash
mv huge_file.iso /mnt/different_disk/huge_file.iso
# ⚠️ Since this crosses filesystem boundaries, mv transparently falls
# back to copying the data first, then deleting the original — this
# is NOT an atomic operation the way a same-filesystem rename is. If
# the process is interrupted (power loss, disk full, a killed
# process) PARTWAY through, the destination can end up with a
# TRUNCATED, incomplete copy, while the ORIGINAL file may or may not
# still be intact depending on exactly when the interruption occurred
# relative to the delete step.

df . /mnt/different_disk
# Checking whether source and destination are actually on the SAME
# filesystem beforehand clarifies whether atomic same-filesystem
# behavior can be expected, or whether copy+delete semantics apply instead
```

---

## Whether the Destination Is an Existing Directory Changes Everything

### The exact same command means something different depending on what's already there
```bash
mv myfile.txt sometarget
# CASE 1: sometarget does NOT exist, or exists as a FILE
# → myfile.txt is RENAMED to "sometarget"

# CASE 2: sometarget ALREADY exists as a DIRECTORY
# → myfile.txt is MOVED INSIDE it, becoming "sometarget/myfile.txt"
# ⚠️ The EXACT SAME command produces a COMPLETELY different result
# depending purely on whether "sometarget" happens to already exist
# as a directory — a script relying on this without verifying the
# destination's actual state beforehand can produce very different
# outcomes across different runs/environments.

# -T forces the "treat as a plain rename target, never a directory"
# interpretation explicitly, removing this ambiguity:
mv -T myfile.txt sometarget
```

---

## Same-Filesystem mv Is Genuinely Atomic — a Reader Never Sees a Half-Moved File

### A property worth knowing and deliberately relying on
```bash
generate_large_report > report.html.tmp
mv report.html.tmp report.html
# ⚠️ On the SAME filesystem, this rename() is a single, atomic
# operation — any OTHER process reading report.html at any point in
# time sees EITHER the complete previous version OR the complete new
# version, NEVER a partially-written intermediate state. This is a
# genuinely useful, deliberate pattern ("write to a temp file, then
# atomically rename into place") for safely publishing generated
# content without a reader ever catching it mid-write.

# ⚠️ This guarantee does NOT hold if source and destination are on
# DIFFERENT filesystems — in that case mv falls back to copy+delete,
# and a reader COULD observe a partially-copied file during the
# window before the copy completes
```

---

## Moving a Directory Onto Itself, or Into Its Own Subdirectory, Fails or Misbehaves

### A structural impossibility that mv can't cleanly resolve
```bash
mv project/ project/backup/
# mv: cannot move 'project/' to a subdirectory of itself, 'project/backup/project'
# ⚠️ mv correctly DETECTS and REFUSES this specific case (moving a
# directory into its own subdirectory), since the destination would
# need to exist "inside" the very thing being moved — a structural
# impossibility mv recognizes and blocks rather than attempting.

mv project/ project/
# mv: 'project/' and 'project/' are the same file
# ⚠️ Moving something onto itself is similarly detected and refused
```

---

## A Cross-Filesystem Move That Fails Partway Can Leave BOTH Copies Existing

### The lack of atomicity across filesystems has a specific, concrete failure mode
```bash
mv huge_dataset.tar /mnt/backup_drive/
# ⚠️ If this is interrupted AFTER the copy to /mnt/backup_drive/
# completes but BEFORE the original is deleted (a crash, a killed
# process, in that exact narrow window), BOTH the original file AND
# the new copy can end up existing simultaneously afterward — mv's
# fallback copy+delete sequence isn't wrapped in any transactional
# guarantee tying the two steps together.

# For anything where this partial-failure risk genuinely matters,
# verify explicitly after a cross-filesystem move rather than
# assuming the source was definitely cleaned up:
ls -la huge_dataset.tar /mnt/backup_drive/huge_dataset.tar 2>/dev/null
```

---

## -u Compares Timestamps, Which Can Be Wrong If Clocks or Timezones Differ

### A subtly risky flag when source and destination timestamps aren't trustworthy
```bash
mv -u data.csv /mnt/network_share/data.csv
# ⚠️ -u decides whether to actually perform the move by comparing
# MODIFICATION TIMESTAMPS between source and destination — if the
# local machine's clock, the remote filesystem's clock, or timezone
# handling across a network mount are meaningfully out of sync, -u
# can make the WRONG decision (skipping a move that should have
# happened, or performing one that shouldn't have) based on
# inaccurate timestamp comparisons that have nothing to do with the
# files' actual content differing.

date; ssh remote_host date
# Worth verifying clock consistency between systems before relying
# heavily on -u's timestamp-based logic across a network boundary
```

---

## mv Doesn't Follow Symlinks the Way cp Sometimes Does — It Moves the Link Itself

### Symlink handling actually works in mv's favor here, unlike a common cp pitfall
```bash
ln -s /var/data/real_file.txt mylink.txt
mv mylink.txt newlocation/mylink.txt
ls -l newlocation/mylink.txt
# lrwxrwxrwx ... newlocation/mylink.txt -> /var/data/real_file.txt
# mv moves the SYMLINK ITSELF (updating its location), rather than
# following it and moving the TARGET's content — this is actually the
# expected, sensible default behavior, in contrast to cp, where
# plain cp (without -a) instead FOLLOWS a symlink and copies the
# target's content, discarding the symlink structure entirely. Worth
# knowing this asymmetry exists between the two commands' defaults.
```

---

## Renaming a File Someone Else Has Open Doesn't Break Their Access

### A file descriptor stays valid even after the filename it was opened through changes
```bash
tail -f growing_log.txt &
mv growing_log.txt archived_log.txt
# ⚠️ The already-running `tail -f` process CONTINUES following the
# SAME underlying file/inode without interruption, even though its
# FILENAME has now changed out from under it — an open file
# descriptor references the underlying inode, not the path string
# used to open it, so renaming doesn't invalidate or redirect
# existing open handles at all.

# A NEW process looking for growing_log.txt by its OLD name, however,
# would fail to find it — only processes that ALREADY had it open
# continue working transparently through the rename
```

---

## -T Forces DEST to Be Treated as a File, Not a Directory — Changes Behavior Significantly

### Useful specifically to avoid the "existing directory" ambiguity covered above
```bash
mv -T source_dir/ target_dir/
# ⚠️ WITHOUT -T, if target_dir/ already exists, source_dir/ gets
# moved INSIDE it (target_dir/source_dir/) — but WITH -T, mv instead
# treats target_dir as a literal rename target, REPLACING it entirely
# if it already exists as an empty directory, or erroring if it's
# non-empty — a meaningfully different, more predictable outcome for
# scripts that need the "rename to exactly this name, no
# move-inside-existing-directory ambiguity" behavior specifically.
```

---

## Moving Many Small Files Across Filesystems Is Much Slower Than One Archive

### Per-file overhead compounds significantly once rename() atomicity no longer applies
```bash
mv ./thousands_of_small_files/ /mnt/other_disk/
# ⚠️ Once source and destination are on DIFFERENT filesystems, EVERY
# individual file requires its own full copy+delete cycle (since
# rename() can't be used across filesystem boundaries at all) — a
# directory with thousands of small files can take dramatically
# longer to move this way than a single archive of the same total
# byte size, since same-filesystem mv's near-instant metadata-only
# behavior doesn't apply here.

# Archiving first, moving the single resulting file, then extracting
# at the destination is often meaningfully faster across filesystems:
tar czf archive.tar.gz ./thousands_of_small_files/
mv archive.tar.gz /mnt/other_disk/
tar xzf /mnt/other_disk/archive.tar.gz -C /mnt/other_disk/
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
