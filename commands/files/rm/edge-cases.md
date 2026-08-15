# rm — Edge Cases & Gotchas

> `rm` is unforgiving by design — no undo, no confirmation, no recycle
> bin. Most of the damage caused by this command comes from a small,
> well-known set of patterns. Knowing them is the actual safety net.

---

## Table of Contents

- [A Space in the Wrong Place Turns "Remove This Subfolder" into "Remove Everything"](#a-space-in-the-wrong-place-turns-remove-this-subfolder-into-remove-everything)
- [Wildcards Are Expanded by the SHELL Before rm Ever Sees Them](#wildcards-are-expanded-by-the-shell-before-rm-ever-sees-them)
- [rm -rf Ignores Errors on Missing Files — Which Can Hide a Real Problem](#rm--rf-ignores-errors-on-missing-files--which-can-hide-a-real-problem)
- [Removing an Open File Doesn't Free Disk Space Until the Program Closes It](#removing-an-open-file-doesnt-free-disk-space-until-the-program-closes-it)
- [A File Still Has Other Hard Links Isn't Actually "Deleted" at All](#a-file-still-has-other-hard-links-isnt-actually-deleted-at-all)
- [alias rm='rm -i' Gives a False Sense of Security](#alias-rmrm--i-gives-a-false-sense-of-security)
- [rm Doesn't Ask "Are You Sure" for a Directory Tree Containing Thousands of Files](#rm-doesnt-ask-are-you-sure-for-a-directory-tree-containing-thousands-of-files)
- [--no-preserve-root Removes the ONE Safety Net Built Directly Into rm](#--no-preserve-root-removes-the-one-safety-net-built-directly-into-rm)
- [shred Doesn't Reliably Work on SSDs or Modern Filesystems](#shred-doesnt-reliably-work-on-ssds-or-modern-filesystems)
- [A Trailing Slash on a Symlink to a Directory Changes What Gets Removed](#a-trailing-slash-on-a-symlink-to-a-directory-changes-what-gets-removed)
- [Deleting Files Doesn't Immediately Reclaim Disk Space Reported by df](#deleting-files-doesnt-immediately-reclaim-disk-space-reported-by-df)

---

## A Space in the Wrong Place Turns "Remove This Subfolder" into "Remove Everything"

### The single most infamous rm disaster pattern
```bash
rm -rf /home/alice /old_backup
# Intent: remove /old_backup, a subfolder of /home/alice
# ⚠️ A single EXTRA SPACE turns this into TWO separate arguments:
# /home/alice (the ENTIRE home directory) AND /old_backup — both get
# recursively, forcefully removed, when only one was ever intended.

rm -rf /home/alice/old_backup
# The CORRECT, intended command — no space, a single unambiguous path

# This exact class of mistake (a stray space before a leading slash)
# is one of the most commonly cited real-world rm disasters —
# always double-check spacing in any rm -rf command, especially
# ones built from a variable or copy-pasted from elsewhere
rm -rf "$BASE_DIR" /old_backup
# ⚠️ if $BASE_DIR happens to be EMPTY/unset, this becomes:
# rm -rf  /old_backup → effectively rm -rf /old_backup, but if
# other similar variable-based commands are built carelessly, an
# unset variable elsewhere in a longer path can produce catastrophic,
# unintended results
```

---

## Wildcards Are Expanded by the SHELL Before rm Ever Sees Them

### rm never actually "sees" the wildcard — only the fully expanded list
```bash
cd /var/www/html
rm -rf ./*
# ⚠️ This is GENERALLY safe (removes contents of the CURRENT
# directory only) — but if `cd` had silently FAILED just before this
# line (a typo in the path, a permissions issue, the directory not
# existing), the shell simply stays in whatever directory it was
# ALREADY in, and `rm -rf ./*` then executes there instead — a classic,
# devastating pattern when combined in a script:

cd /var/www/html || exit 1
rm -rf ./*
# ⚠️ WITHOUT the "|| exit 1" safeguard, a failed cd doesn't stop the
# script — it just silently continues in the WRONG directory, and the
# subsequent rm -rf ./* then wipes out whatever unrelated directory
# the shell actually happened to be in at that moment. ALWAYS check
# that a cd preceding a destructive rm actually succeeded.
```

---

## rm -rf Ignores Errors on Missing Files — Which Can Hide a Real Problem

### -f suppresses more than just confirmation prompts
```bash
rm -f "$CONFIG_FILE"
echo $?
# 0    ← ⚠️ -f makes rm return SUCCESS even if $CONFIG_FILE never
# existed in the first place, or the removal failed for some OTHER
# reason entirely (a permissions issue, a busy mount) — -f
# specifically suppresses the errors that would normally surface
# these problems, which is exactly the point for "don't error if
# it's already gone" scripting, but can also mask a genuine, different
# failure that a script author might have wanted to know about.

# If distinguishing "already gone" from "failed for another reason"
# matters, check more explicitly rather than relying purely on -f's
# blanket error suppression:
if [ -e "$CONFIG_FILE" ]; then
  rm "$CONFIG_FILE" || echo "Failed to remove existing $CONFIG_FILE"
fi
```

---

## Removing an Open File Doesn't Free Disk Space Until the Program Closes It

### A very common source of "df shows the disk is still full" confusion
```bash
# A long-running process has a large log file open
rm /var/log/myapp/huge.log
df -h /var/log
# ⚠️ The disk usage figure DOESN'T immediately drop, even though the
# file was successfully removed — unlink() only removes the
# DIRECTORY ENTRY (the filename); the actual DATA BLOCKS remain
# allocated as long as the still-running process holds an open file
# descriptor referencing that (now nameless) data. Only once the
# PROCESS closes that file descriptor (or exits/restarts) does the
# space actually get freed.

# The standard fix for a runaway log file scenario like this:
lsof +L1 /var/log/myapp
# lists files with a link count of 0 but still open — genuinely
# "deleted but still consuming space" files

# Rather than removing the file, TRUNCATE it in place instead,
# leaving the process's existing file descriptor pointing at the same
# (now empty) file:
: > /var/log/myapp/huge.log
# or, if the app supports it, sending a signal to make it reopen its
# log file after a proper rm + recreation
```

---

## A File Still Has Other Hard Links Isn't Actually "Deleted" at All

### rm only removes ONE directory entry, not necessarily the underlying data
```bash
ln original.txt hardlink_copy.txt
# original.txt and hardlink_copy.txt now both point to the SAME
# underlying data (same inode)

rm original.txt
cat hardlink_copy.txt
# ⚠️ The content is STILL FULLY ACCESSIBLE through hardlink_copy.txt
# — removing original.txt only removed ONE of the two directory
# entries pointing at that shared data; the underlying data isn't
# actually freed until EVERY hard link to it has been removed.

ls -l hardlink_copy.txt
# the link count column reveals how many directory entries still
# reference this same underlying data
```

---

## alias rm='rm -i' Gives a False Sense of Security

### A commonly recommended "safety habit" with real, easily overlooked gaps
```bash
alias rm='rm -i'
rm important_file.txt
# rm: remove regular file 'important_file.txt'? y
# Seems safe — a confirmation prompt appears

# ⚠️ BUT this alias only applies in INTERACTIVE shell sessions where
# the alias is actually defined and loaded — it does NOT apply:
#   - inside scripts (which typically don't load interactive aliases
#     at all, by design)
#   - when rm is invoked via its FULL PATH (/usr/bin/rm bypasses
#     shell aliases entirely)
#   - on a DIFFERENT machine, a different user account, or after a
#     fresh shell config that hasn't set up the same alias
#   - when rm -f is used, since an explicit -f OVERRIDES any earlier
#     -i on the same command line anyway

/usr/bin/rm important_file.txt
# ⚠️ This SILENTLY BYPASSES the alias entirely — no prompt at all,
# despite the alias being active in this very shell

# The alias is a reasonable everyday habit for interactive use, but
# should never be relied upon as an actual safety GUARANTEE
```

---

## rm Doesn't Ask "Are You Sure" for a Directory Tree Containing Thousands of Files

### Scale alone triggers no additional caution by default
```bash
rm -rf massive_directory_with_50000_files/
# ⚠️ Regardless of how much data or how many files are about to be
# permanently destroyed, plain rm -rf proceeds IMMEDIATELY with zero
# confirmation and zero indication of the scale involved — there's no
# built-in "this will delete 50,000 files, are you sure?" warning at
# any size threshold whatsoever.

# -I (capital i) provides a single, size-aware confirmation for
# exactly this situation, without the tedium of -i's per-file prompting:
rm -I -r massive_directory_with_50000_files/
# rm: remove 50000 arguments recursively? y
# Asks ONCE, but specifically calls out the scale involved, giving a
# meaningful moment to reconsider before proceeding
```

---

## --no-preserve-root Removes the ONE Safety Net Built Directly Into rm

### Modern coreutils protects against the most catastrophic possible mistake by default
```bash
rm -rf /
# rm: it is dangerous to operate recursively on '/'
# rm: use --no-preserve-root to override this failsafe
# ⚠️ Modern GNU coreutils REFUSES this by default — --preserve-root
# is now the DEFAULT behavior specifically to prevent the single most
# catastrophic possible rm mistake (wiping the entire root filesystem).

rm -rf --no-preserve-root /
# ⚠️ This explicitly REMOVES that protection and proceeds — there is
# essentially NEVER a legitimate reason to actually need this flag;
# its presence in any command, script, or tutorial should be treated
# as an extremely serious red flag demanding scrutiny before running.
```

---

## shred Doesn't Reliably Work on SSDs or Modern Filesystems

### The "secure delete" tool has real, commonly misunderstood limitations
```bash
shred -u sensitive_file.txt
# ⚠️ shred's overwrite-before-delete approach was designed around
# traditional magnetic HARD DRIVES, where overwriting a specific
# location on disk reliably overwrites that exact physical data.
# Modern SSDs (with wear-leveling, which transparently relocates
# data across physical cells) and copy-on-write/journaling
# filesystems (which may write NEW data to a DIFFERENT physical
# location rather than overwriting in place) can mean the ORIGINAL
# data remains recoverable on the underlying physical media even
# after shred reports success, since shred has no reliable way to
# target the actual physical location the data now occupies.

# For SSDs specifically, full-disk encryption (set up BEFORE data is
# ever written) combined with securely destroying the encryption key
# is a far more reliable approach to genuine data destruction than
# after-the-fact file shredding:
cryptsetup luksErase /dev/sdX
```

---

## A Trailing Slash on a Symlink to a Directory Changes What Gets Removed

### A subtle interaction between symlinks and rm's directory handling
```bash
ln -s /var/data/real_folder mylink
rm -r mylink
# ⚠️ Removes the SYMLINK ITSELF — the actual target directory
# (/var/data/real_folder) and its contents remain completely intact.
# This is the expected, safe behavior for the common case.

rm -r mylink/
# ⚠️ WITH a trailing slash, some shells/rm implementations may
# instead attempt to operate on the CONTENTS the symlink points to,
# rather than the symlink itself — behavior here can vary by exact
# system/shell, making this a genuinely risky ambiguity to leave
# untested. Always verify the actual target with `ls -l mylink` before
# running an rm command against a symlink you're not 100% certain about.
```

---

## Deleting Files Doesn't Immediately Reclaim Disk Space Reported by df

### Beyond just the open-file case — filesystem-level caching/journaling can also delay this
```bash
rm large_file.iso
df -h .
# ⚠️ Even outside the "still-open-file" scenario covered above, some
# filesystems and storage layers (journaling filesystems, certain
# network/clustered filesystems, some container/overlay storage
# drivers) can show a brief delay between a successful rm and the
# freed space actually being reflected in df's reported figures, due
# to underlying metadata journaling/caching behavior.

sync
df -h .
# Forcing a sync can sometimes help make freed space visible sooner,
# though the exact timing still depends on the specific filesystem
# and storage stack involved
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
