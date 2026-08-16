# cp — Edge Cases & Gotchas

> `cp` looks like the simplest possible command, but trailing slashes,
> silent overwrites, and default metadata behavior routinely produce
> results that differ from what people expect.

---

## Table of Contents

- [cp Silently Overwrites the Destination by Default — No Warning at All](#cp-silently-overwrites-the-destination-by-default--no-warning-at-all)
- [A Trailing Slash on the SOURCE Changes Nothing in cp — Unlike rsync](#a-trailing-slash-on-the-source-changes-nothing-in-cp--unlike-rsync)
- [Whether the Destination Already Exists Changes the Entire Resulting Structure](#whether-the-destination-already-exists-changes-the-entire-resulting-structure)
- [Without -p, the Copy Gets NEW Timestamps and Permissions, Not the Original's](#without--p-the-copy-gets-new-timestamps-and-permissions-not-the-originals)
- [Copying Into a Directory You Don't Have Write Access To Fails, Even If You Own the Source File](#copying-into-a-directory-you-dont-have-write-access-to-fails-even-if-you-own-the-source-file)
- [Copying a Symlink Without -a/-d Follows It, Copying the TARGET's Content Instead of the Link](#copying-a-symlink-without--a-d-follows-it-copying-the-targets-content-instead-of-the-link)
- [cp -r on a Directory Copied Into Itself Can Recurse Forever](#cp--r-on-a-directory-copied-into-itself-can-recurse-forever)
- [-n and -i Together — the Last One Specified Wins](#-n-and--i-together--the-last-one-specified-wins)
- [A Partial/Interrupted Copy Leaves a Truncated File With No Automatic Warning](#a-partialinterrupted-copy-leaves-a-truncated-file-with-no-automatic-warning)
- [--reflink Copies Aren't Real Extra Disk Usage Until Something Changes — Easy to Misjudge Free Space](#--reflink-copies-arent-real-extra-disk-usage-until-something-changes--easy-to-misjudge-free-space)
- [Copying Many Small Files Is Much Slower Than Copying One Large Archive of the Same Size](#copying-many-small-files-is-much-slower-than-copying-one-large-archive-of-the-same-size)

---

## cp Silently Overwrites the Destination by Default — No Warning at All

### The single most consequential cp default to be aware of
```bash
cp draft_v2.txt final_report.txt
# ⚠️ If final_report.txt already existed with completely different,
# important content, it is now GONE — overwritten immediately and
# silently, with absolutely no confirmation prompt, no backup, and no
# way to undo it through cp itself.

# Always use -i when there's any doubt about whether the destination
# already contains something worth keeping:
cp -i draft_v2.txt final_report.txt
# cp: overwrite 'final_report.txt'? y

# Or -n to guarantee an existing file is NEVER touched, erring toward
# silently skipping rather than silently destroying:
cp -n draft_v2.txt final_report.txt
```

---

## A Trailing Slash on the SOURCE Changes Nothing in cp — Unlike rsync

### A dangerous assumption carried over from rsync habits
```bash
cp -r project project_backup
cp -r project/ project_backup
# ⚠️ These two produce the EXACT SAME result in cp — a trailing
# slash on the SOURCE has NO special meaning here, unlike rsync,
# where a trailing slash on the source changes whether the directory
# ITSELF or just its CONTENTS get copied. Someone bringing rsync
# habits/assumptions into a cp command may expect different behavior
# than what actually happens.

# To copy just the CONTENTS of a directory (not the directory itself)
# into an existing destination with cp, use an explicit "/." suffix instead:
cp -r project/. existing_backup_dir/
# NOW copies project's CONTENTS directly into existing_backup_dir/,
# rather than creating existing_backup_dir/project/
```

---

## Whether the Destination Already Exists Changes the Entire Resulting Structure

### The same command can produce two structurally different outcomes
```bash
cp -r project/ backup_target/
# CASE 1: backup_target/ does NOT already exist
# → backup_target/ is CREATED as a copy of project/ itself
# Result: backup_target/ contains project's files DIRECTLY

# CASE 2: backup_target/ ALREADY exists as a directory
# → project/ is copied INSIDE backup_target/
# Result: backup_target/project/ contains project's files
# ⚠️ The EXACT SAME command produces a DIFFERENT resulting directory
# structure depending purely on whether the destination happened to
# already exist beforehand — always verify the destination's
# existence/expected structure before running a recursive copy,
# especially in scripts where this can vary run to run.
```

---

## Without -p, the Copy Gets NEW Timestamps and Permissions, Not the Original's

### A common surprise when cp is used for backups specifically
```bash
touch -d "2020-01-01" original_report.txt
cp original_report.txt backup_report.txt
stat backup_report.txt
# Modify: 2026-08-11 14:32:00     ← ⚠️ NOT 2020-01-01! Without -p,
# the copy gets a BRAND NEW modification timestamp reflecting when
# the COPY operation itself happened, not the original file's actual
# history — someone relying on cp alone to "preserve" a file's
# original dates for archival/audit purposes will be surprised the
# copy doesn't actually carry that information forward at all.

cp -p original_report.txt backup_report_preserved.txt
stat backup_report_preserved.txt
# Modify: 2020-01-01 00:00:00     ← -p correctly carries the
# ORIGINAL timestamp forward onto the copy instead
```

---

## Copying Into a Directory You Don't Have Write Access To Fails, Even If You Own the Source File

### Ownership of the SOURCE is irrelevant to whether the copy can complete
```bash
cp my_file.txt /etc/some_protected_dir/
# cp: cannot create regular file '/etc/some_protected_dir/my_file.txt': Permission denied
# ⚠️ Owning my_file.txt (the SOURCE) has no bearing here whatsoever —
# what matters entirely is whether the current user has WRITE
# permission on the DESTINATION directory. This is easy to
# misdiagnose as "a problem with my file" when it's actually purely
# about destination directory permissions.

ls -ld /etc/some_protected_dir/
# Checking the destination directory's own permissions clarifies the
# actual cause
```

---

## Copying a Symlink Without -a/-d Follows It, Copying the TARGET's Content Instead of the Link

### A frequent surprise when backing up a directory containing symlinks
```bash
ln -s /var/data/real_file.txt mylink.txt
cp mylink.txt copy_of_link.txt
ls -l copy_of_link.txt
# -rw-r--r-- ... copy_of_link.txt     ← ⚠️ this is a REGULAR FILE
# containing a COPY of real_file.txt's CONTENT — NOT a symlink at
# all! Plain cp, by default, FOLLOWS symlinks and copies the TARGET's
# actual content, discarding the "this was a symlink" information
# entirely in the resulting copy.

cp -a mylink.txt copy_of_link_preserved.txt
ls -l copy_of_link_preserved.txt
# lrwxrwxrwx ... copy_of_link_preserved.txt -> /var/data/real_file.txt
# -a (or -d specifically for just this behavior) preserves the
# SYMLINK ITSELF as a symlink, rather than following and flattening it
```

---

## cp -r on a Directory Copied Into Itself Can Recurse Forever

### A classic infinite-loop trap when source and destination overlap
```bash
cd /data
cp -r ./project ./project/backup
# ⚠️ If the DESTINATION is a subdirectory OF the source being copied,
# cp can enter the newly-created destination directory AS PART OF the
# same ongoing recursive copy operation, attempting to copy files it
# JUST created, potentially resulting in extremely deep, runaway
# recursion, an error, or unpredictable behavior depending on the
# exact cp implementation and how it detects (or fails to detect)
# this overlap.

cp -r ./project ../project-backup
# Always copy to a destination OUTSIDE the source tree entirely to
# avoid this class of problem altogether
```

---

## -n and -i Together — the Last One Specified Wins

### Conflicting flags don't produce an error, just quiet flag-precedence behavior
```bash
cp -n -i file.txt existing.txt
# ⚠️ Since -i appears LAST on the command line, it takes effect
# (prompting before overwrite) — the earlier -n is effectively
# overridden, NOT combined or averaged with it in some intermediate
# way. GNU cp resolves conflicting overwrite-behavior flags by simply
# letting whichever one appears LAST on the command line win, with no
# warning about the earlier flag being superseded.

cp -i -n file.txt existing.txt
# ⚠️ Reversed order — NOW -n wins instead (never overwrite, no prompt)
# The exact same TWO flags, just reordered, produce genuinely
# DIFFERENT behavior — worth being deliberate about ordering when
# multiple overwrite-related flags appear together, or simply avoid
# combining them at all.
```

---

## A Partial/Interrupted Copy Leaves a Truncated File With No Automatic Warning

### An incomplete copy can look complete at a glance
```bash
cp largefile.iso /mnt/external_drive/
# ... drive is unplugged partway through, or disk fills up ...
# ⚠️ Depending on exactly how/when the interruption occurred, the
# destination can be left with a TRUNCATED, incomplete file that
# still appears to exist at the expected path — cp doesn't
# automatically verify the copy's completeness/integrity after the
# fact, and a truncated file sitting there can be mistaken for a
# successful copy without an explicit size/checksum check.

cp largefile.iso /mnt/external_drive/
echo $?
ls -la largefile.iso /mnt/external_drive/largefile.iso
# compare sizes explicitly, or use a checksum, rather than assuming
# presence alone means the copy fully completed
```

---

## --reflink Copies Aren't Real Extra Disk Usage Until Something Changes — Easy to Misjudge Free Space

### A copy-on-write reflink "copy" can look deceptively cheap or deceptively expensive
```bash
cp --reflink=auto huge_dataset.img huge_dataset_copy.img
du -sh huge_dataset.img huge_dataset_copy.img
# ⚠️ Both files may report their FULL logical size individually via
# `du`, even though they're currently sharing the SAME underlying
# physical data blocks on disk (on a filesystem supporting
# copy-on-write reflinks) — actual DISTINCT disk usage is much lower
# than "size of file A + size of file B" would suggest, UNTIL one of
# the two copies is actually modified, at which point ONLY the
# changed blocks get physically duplicated.

df -h .
# A more accurate picture of ACTUAL free space consumed comes from
# checking overall filesystem usage, not summing individual
# du-reported file sizes when reflink copies are involved
```

---

## Copying Many Small Files Is Much Slower Than Copying One Large Archive of the Same Size

### Per-file overhead adds up significantly at scale
```bash
cp -r ./node_modules/ /backup/
# ⚠️ A directory containing thousands of small files can take
# DRAMATICALLY longer to copy via plain cp -r than a single archive
# file containing the exact same total amount of data — each
# individual file copy carries its own per-file overhead (opening,
# creating, closing file descriptors, metadata operations), which
# accumulates significantly at scale, independent of total byte count.

# Archiving first, copying the single resulting file, then extracting
# at the destination is often meaningfully faster for large numbers
# of small files:
tar czf archive.tar.gz ./node_modules/
cp archive.tar.gz /backup/
tar xzf /backup/archive.tar.gz -C /backup/
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
