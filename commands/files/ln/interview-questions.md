# ln — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Hard Links vs Symbolic Links](#hard-links-vs-symbolic-links)
- [Internals](#internals)
- [Practical Usage](#practical-usage)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does ln do?**
> Creates a link — an additional way to reference a file — without duplicating its actual data. It supports two mechanisms: hard links (another name for the same underlying data) and symbolic links (a separate file that points to another path by name).

---

**Q2 🔥 Does ln create a hard link or a symbolic link by default, with no flags?**
> A hard link — this surprises many people, since symbolic links are far more commonly used and discussed in everyday practice, but `-s` must be explicitly specified to get symlink behavior instead.

---

**Q3. What's the fundamental difference between what a hard link and a symlink actually are?**
> A hard link is simply another directory entry pointing to the exact same inode (the same underlying data) as an existing file — they're indistinguishable and equally "real." A symlink is a genuinely separate, small file whose content is just a text path string pointing at another location.

---

## Hard Links vs Symbolic Links

**Q4 🔥 Can a hard link cross filesystem boundaries? Can a symlink?**
> A hard link cannot — inode numbers are only meaningful within a single filesystem, so there's no way to hard-link across separate filesystems/devices. A symlink can, since it just stores a path string with no dependency on inode numbers at all.

---

**Q5. Can you create a hard link to a directory?**
> No, not for regular users — this is deliberately disallowed, since arbitrary directory hard links could create cycles in the filesystem's directory tree, breaking assumptions many tools and the kernel itself make about tree traversal. A symlink to a directory is the standard supported alternative.

---

**Q6 🔥 What happens if you delete the original file that a hard link points to? What about a symlink?**
> For a hard link, nothing happens to the remaining link — the underlying data persists as long as at least one hard link (directory entry) to it still exists; there's no concept of an "original" being specially privileged. For a symlink, the symlink file itself still exists afterward but becomes "broken" or "dangling," pointing at a path that no longer resolves to anything.

---

**Q7. Is a symlink allowed to point to something that doesn't exist?**
> Yes — ln performs no existence check when creating a symlink, so a "broken" symlink is completely valid to create, with no warning at creation time. The failure only surfaces later, when something actually tries to use (dereference) the link.

---

## Internals

**Q8 🔥 What system calls does ln use for each type of link?**
> `link(2)` for a hard link — adding a new directory entry pointing to an existing inode. `symlink(2)` for a symbolic link — creating a genuinely new inode whose content is a path string.

---

**Q9. How can you tell a hard link apart from the original file it references, after creation?**
> You can't, in any meaningful sense — they're both equally valid, equally "real" directory entries pointing to the same inode, with no way to determine which was created first or which is the "original." `ls -i` (showing inode numbers) reveals they're the same underlying file, but neither has special status over the other.

---

**Q10 🔥 What does the "size" shown by `ls -l` for a symlink actually represent?**
> The length of the target path string stored in the symlink itself — NOT the size of the file the symlink points to. A symlink pointing to a multi-gigabyte file can show a "size" of just a few bytes, since that figure reflects the stored path text, not the target's actual content size.

---

**Q11. What does the "link count" shown in `ls -l`'s permission column represent for a regular file?**
> The number of hard links (directory entries) currently pointing to that file's underlying inode. A value of 1 means no additional hard links exist beyond the one being viewed; a higher number indicates the same data is referenced by multiple filenames.

---

## Practical Usage

**Q12 🔥 What's the purpose of the -n flag when updating a symlink?**
> When the destination is itself an existing symlink pointing to a directory, `-n` (`--no-dereference`) tells ln to treat that destination as the literal link to replace, rather than following it and creating the new link INSIDE that target directory — essential for correctly updating a "current version" style pointer symlink.

---

**Q13. What does `readlink -f` do, and when would you use it?**
> Fully resolves a symlink (or an entire chain of symlinks) to its final, real underlying path — useful for confirming exactly what a link (or chain of links) ultimately points to, especially when troubleshooting a broken or confusingly nested symlink.

---

## Scenario-Based

**Q14 🔥 A deployment script runs `ln -sf /opt/myapp/releases/v2.4.0 /opt/myapp/current` to update a "current release" pointer, but afterward the application is still running the old version. What likely went wrong?**
> `/opt/myapp/current` was very likely already an existing symlink pointing to a directory (the previous release) — without `-n`, `ln -sf` follows that existing symlink and creates the new link INSIDE the target directory instead of replacing the pointer itself. Adding `-n` (`ln -sfn ...`) fixes this by treating `/opt/myapp/current` as the literal link to overwrite.

---

**Q15. Someone creates `ln -s data/file.txt shortcuts/link.txt` while standing in their home directory, expecting it to resolve relative to their home directory — but the link turns out broken. Why?**
> A relative symlink's stored path is resolved relative to the SYMLINK's own location when later followed, not relative to the directory the `ln` command happened to be run from. Since the link lives in `shortcuts/`, it's actually looking for `shortcuts/data/file.txt`, not `data/file.txt` relative to the original working directory. Using an absolute path, or `ln -sr` to have ln compute the correct relative path automatically, avoids this.

---

**Q16 🔥 A teammate creates `ln important.txt important_backup.txt` believing they've made a safety backup, then edits important.txt and is alarmed to find important_backup.txt shows the same changes. What's the misconception?**
> A hard link is not an independent copy — both names reference the exact same underlying data (same inode), so editing content through either name modifies the same actual data. There is no independent "backup" here in the sense most people mean by that word; `cp` is required to create a genuinely separate, independent copy that won't reflect later changes to the original.

---

**Q17. During a security review, someone notices `ls -l` shows wide-open permissions (rwxrwxrwx) on a symlink pointing to a sensitive file. Is this actually a security exposure?**
> Generally no by itself — symlink permission bits are essentially cosmetic/unused on Linux; what actually governs access when the link is dereferenced is the TARGET file's own permissions, not the symlink's displayed bits. The real security question is what permissions the target file itself has, not what's shown on the symlink.

---

**Q18 🔥 A script tries to create a hard link between a file on the local disk and a path on a mounted USB drive, and it fails with "Invalid cross-device link." Why, and what's the fix?**
> Hard links reference an inode number, which is only meaningful within a single filesystem — there's no mechanism for referencing an inode across separate filesystem instances, so this is a fundamental limitation, not a fixable configuration issue. A symbolic link (`ln -s`) has no such restriction, since it just stores a path string, and works as the correct alternative here.

---

**Q19. A directory audit finds several symlinks that appear to point at valid-looking paths, but attempting to open any of them fails. What's the most likely explanation, and how would you find all such cases at once?**
> These are very likely broken (dangling) symlinks — their target existed at some point but has since been removed, renamed, or moved, while the symlink object itself persists unaffected on disk. `find <directory> -xtype l` locates all broken symlinks under a given path at once, rather than checking each one individually.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
