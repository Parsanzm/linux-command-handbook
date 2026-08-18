# ln — Edge Cases & Gotchas

> `ln` looks simple, but its default behavior (hard link, not symlink),
> directory-target ambiguity, and broken-link handling routinely
> surprise people expecting the more familiar symlink behavior.

---

## Table of Contents

- [ln With No Flag Creates a HARD Link, Not a Symlink — the Opposite of Common Assumption](#ln-with-no-flag-creates-a-hard-link-not-a-symlink--the-opposite-of-common-assumption)
- [A Symlink to a Directory, Linked Again Without -n, Creates the New Link INSIDE It](#a-symlink-to-a-directory-linked-again-without--n-creates-the-new-link-inside-it)
- [Hard Links Cannot Cross Filesystem Boundaries — Symlinks Can](#hard-links-cannot-cross-filesystem-boundaries--symlinks-can)
- [A Broken Symlink Is Completely Valid to Create, and ln Won't Warn You](#a-broken-symlink-is-completely-valid-to-create-and-ln-wont-warn-you)
- [Relative Symlinks Are Resolved Relative to the LINK's Location, Not Your Current Directory](#relative-symlinks-are-resolved-relative-to-the-links-location-not-your-current-directory)
- [Hard Links to Directories Are Effectively Impossible on Purpose](#hard-links-to-directories-are-effectively-impossible-on-purpose)
- [Deleting a Symlink's Target Doesn't Delete the Symlink Itself — It Just Breaks It](#deleting-a-symlinks-target-doesnt-delete-the-symlink-itself--it-just-breaks-it)
- [Editing a File Through a Hard Link Changes It Everywhere — There's No "Original" Anymore](#editing-a-file-through-a-hard-link-changes-it-everywhere--theres-no-original-anymore)
- [ls -l's Size Column for a Symlink Shows the PATH STRING's Length, Not the Target's Size](#ls--ls-size-column-for-a-symlink-shows-the-path-strings-length-not-the-targets-size)
- [A Symlink Chain (Link to a Link) Can Get Confusing and Even Loop](#a-symlink-chain-link-to-a-link-can-get-confusing-and-even-loop)
- [Permissions Shown on a Symlink Are Usually Meaningless — the TARGET's Permissions Actually Apply](#permissions-shown-on-a-symlink-are-usually-meaningless--the-targets-permissions-actually-apply)

---

## ln With No Flag Creates a HARD Link, Not a Symlink — the Opposite of Common Assumption

### The single most common ln mistake
```bash
ln original.txt shortcut.txt
# ⚠️ Many people, especially those newer to Unix, ASSUME this creates
# a symlink (since symlinks are far more commonly used and discussed
# in everyday practice) — but ln's DEFAULT, with no flags at all, is
# actually a HARD link. This can be genuinely surprising when
# discovered later, especially since a hard link looks completely
# ordinary in `ls -l` output, giving no visual clue it's "special" at all.

ls -l shortcut.txt
# -rw-r--r-- 2 alice alice 1024 ... shortcut.txt
# ⚠️ No arrow, no special indicator — nothing distinguishes this from
# any other regular file at a glance

ln -s original.txt shortcut.txt
# -s is REQUIRED explicitly for the (far more commonly intended) symlink behavior
```

---

## A Symlink to a Directory, Linked Again Without -n, Creates the New Link INSIDE It

### A classic trap when updating a "current version" style symlink
```bash
ln -s /opt/myapp/releases/v2.3.0 /opt/myapp/current
# First run: creates /opt/myapp/current -> v2.3.0, as intended

ln -sf /opt/myapp/releases/v2.4.0 /opt/myapp/current
# ⚠️ Since /opt/myapp/current ALREADY EXISTS as a symlink POINTING TO
# A DIRECTORY, this command follows it and creates the NEW link
# INSIDE that target directory instead — resulting in
# /opt/myapp/releases/v2.3.0/current -> v2.4.0, NOT updating
# /opt/myapp/current to point to v2.4.0 at all, which is almost
# certainly not what was intended for a "current release" pointer pattern.

ln -sfn /opt/myapp/releases/v2.4.0 /opt/myapp/current
# -n (--no-dereference) is ESSENTIAL here — it tells ln to treat
# /opt/myapp/current as the LINK TO REPLACE, not as a directory to
# link INTO, correctly updating the pointer as intended
```

---

## Hard Links Cannot Cross Filesystem Boundaries — Symlinks Can

### A fundamental limitation, not a bug or configuration issue
```bash
ln /home/alice/file.txt /mnt/external_drive/file_link.txt
# ln: failed to create hard link '/mnt/external_drive/file_link.txt' =>
# '/home/alice/file.txt': Invalid cross-device link
# ⚠️ Hard links reference an INODE NUMBER, which is only meaningful
# WITHIN a single filesystem — there's no way to reference an inode
# on a completely different filesystem/device, so hard links simply
# CANNOT span across them, by fundamental design, not as a fixable limitation.

ln -s /home/alice/file.txt /mnt/external_drive/file_link.txt
# Symlinks have NO such restriction — since a symlink just stores a
# PATH STRING (not an inode reference), it can point ANYWHERE,
# including entirely different filesystems, network mounts, or even
# paths that don't exist yet
```

---

## A Broken Symlink Is Completely Valid to Create, and ln Won't Warn You

### No existence check happens when creating a symlink
```bash
ln -s /this/path/does/not/exist.txt mylink.txt
# (no error at all — this succeeds unconditionally)

ls -l mylink.txt
# lrwxrwxrwx ... mylink.txt -> /this/path/does/not/exist.txt

cat mylink.txt
# cat: mylink.txt: No such file or directory
# ⚠️ ln happily creates a symlink pointing to something that doesn't
# exist, with NO warning at creation time — the failure only surfaces
# later, when something actually tries to USE (dereference) the link.
# This is intentional and sometimes useful (linking to something that
# will exist LATER), but can also silently leave broken links lying
# around unnoticed.

find . -xtype l
# Periodically checking for broken symlinks specifically is the way
# to catch these proactively, since ln itself gives no warning
```

---

## Relative Symlinks Are Resolved Relative to the LINK's Location, Not Your Current Directory

### A very common source of "why is my symlink broken" confusion
```bash
cd /home/alice
ln -s data/file.txt /home/alice/projects/shortcut.txt
# ⚠️ This creates a symlink at /home/alice/projects/shortcut.txt whose
# STORED TARGET PATH is the literal relative string "data/file.txt" —
# when later RESOLVED, that relative path is interpreted relative to
# the SYMLINK'S OWN LOCATION (/home/alice/projects/), NOT relative to
# wherever you happened to be standing (/home/alice) when you RAN
# the ln command that created it.

ls -l /home/alice/projects/shortcut.txt
cat /home/alice/projects/shortcut.txt
# cat: /home/alice/projects/shortcut.txt: No such file or directory
# ⚠️ BROKEN — it's actually looking for
# /home/alice/projects/data/file.txt, which doesn't exist, NOT
# /home/alice/data/file.txt as might have been intended

# Use -r to have ln compute the correct RELATIVE path automatically,
# or specify an ABSOLUTE target path to sidestep the ambiguity entirely:
ln -sr /home/alice/data/file.txt /home/alice/projects/shortcut.txt
```

---

## Hard Links to Directories Are Effectively Impossible on Purpose

### A deliberate filesystem-level restriction, not an oversight
```bash
ln somedir/ hardlink_to_dir/
# ln: somedir/: hard link not allowed for directory
# ⚠️ Regular users cannot create hard links to directories at all —
# this is DELIBERATE, since allowing arbitrary directory hard links
# could create cycles in the filesystem's directory tree structure
# (a directory that is, in some sense, its own ancestor), which would
# break countless assumptions tools and the kernel itself make about
# tree traversal (like avoiding infinite loops when walking the
# filesystem). Symlinks to directories are the standard, supported
# alternative for this purpose instead.

ln -s somedir/ symlink_to_dir/
# The correct approach for "linking" a directory
```

---

## Deleting a Symlink's Target Doesn't Delete the Symlink Itself — It Just Breaks It

### The symlink object persists independently of what it points to
```bash
ln -s original.txt mylink.txt
rm original.txt
ls -l mylink.txt
# lrwxrwxrwx ... mylink.txt -> original.txt
# ⚠️ The symlink FILE ITSELF still exists on disk, completely
# unaffected by the target's removal — it's now simply "broken" or
# "dangling," pointing at a path that no longer resolves to anything.
# Attempting to actually USE it (cat, open, cd into it if it pointed
# to a directory) fails, but the symlink object itself remains
# present until explicitly removed on its own.

rm mylink.txt
# Removing the SYMLINK requires its own separate rm — it doesn't
# disappear automatically just because its target is gone
```

---

## Editing a File Through a Hard Link Changes It Everywhere — There's No "Original" Anymore

### A frequent point of confusion given how backups are colloquially discussed
```bash
ln important.txt important_backup.txt
echo "new content" >> important.txt
cat important_backup.txt
# new content     ← ⚠️ the "backup" ALSO shows the new content!
# Because both names reference the EXACT SAME underlying data
# (same inode), editing the content through EITHER name modifies the
# SAME actual data — there is no independent "backup" here at all in
# the sense most people mean by that word. A hard link is NOT a
# genuine backup/snapshot mechanism; it's just an additional name for
# the SAME live, mutable data.

# For a genuine, independent point-in-time backup, use cp instead:
cp important.txt important_backup_real.txt
# NOW editing important.txt leaves important_backup_real.txt
# completely unaffected, since it's truly separate data
```

---

## ls -l's Size Column for a Symlink Shows the PATH STRING's Length, Not the Target's Size

### An easy figure to misread when scanning directory listings
```bash
ln -s /very/long/path/to/some/large/file.iso shortcut.txt
ls -l shortcut.txt
# lrwxrwxrwx 1 alice alice 34 Aug 11 14:32 shortcut.txt -> /very/long/path/to/some/large/file.iso
# ⚠️ That "34" is the LENGTH OF THE TARGET PATH STRING (34
# characters) — NOT the size of file.iso itself, which might be
# several gigabytes. Someone scanning a directory listing for large
# files could easily misinterpret a symlink's tiny "size" figure,
# either wrongly dismissing it as unimportant or (less commonly)
# being confused about why a "large file" shows such a small number.

# To see the size of what a symlink actually points to, resolve and
# check the TARGET directly:
ls -lL shortcut.txt
# -L makes ls follow the symlink and report the TARGET's actual size
# instead of the link's own metadata
```

---

## A Symlink Chain (Link to a Link) Can Get Confusing and Even Loop

### Multiple hops are legal, but readability and correctness both suffer
```bash
ln -s real_file.txt link_a.txt
ln -s link_a.txt link_b.txt
ln -s link_b.txt link_c.txt
cat link_c.txt
# Works fine — the kernel transparently follows the ENTIRE chain
# (link_c -> link_b -> link_a -> real_file.txt) until it reaches an
# actual file, but a LONG or TANGLED chain like this becomes
# genuinely hard for a HUMAN to trace and reason about at a glance.

ln -s link_c.txt real_file.txt
# ⚠️ A CIRCULAR chain (a loop) is also technically creatable — the
# kernel detects this and refuses to resolve it beyond a certain
# depth, returning an error ("Too many levels of symbolic links")
# rather than looping forever, but the broken circular symlink itself
# still exists on disk until manually cleaned up.

readlink -f link_c.txt
# Following the FULL chain to its ultimate real target directly is
# the most reliable way to understand what a chain of symlinks
# actually resolves to, rather than manually tracing each hop by hand
```

---

## Permissions Shown on a Symlink Are Usually Meaningless — the TARGET's Permissions Actually Apply

### ls -l shows a symlink's OWN permission bits, which are essentially never what's enforced
```bash
ln -s /etc/shadow mylink
ls -l mylink
# lrwxrwxrwx 1 alice alice 11 Aug 11 14:32 mylink -> /etc/shadow
# ⚠️ Note "rwxrwxrwx" — WIDE OPEN permissions shown on the symlink
# itself. This does NOT mean anyone can actually read /etc/shadow
# through this link! Symlink permission bits are essentially
# meaningless/unused on Linux — what ACTUALLY governs access when the
# link is dereferenced is the TARGET FILE's own permissions
# (/etc/shadow's real, restrictive permissions in this example), not
# the symlink's own displayed (and largely cosmetic) permission bits.

cat mylink
# cat: mylink: Permission denied
# Confirms the TARGET's actual restrictive permissions are what's
# enforced, regardless of the wide-open bits shown on the symlink itself
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
