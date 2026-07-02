# chmod — Edge Cases & Gotchas

> Permission mistakes are quiet — no crash, no error, just a system that mysteriously
> "doesn't work" or, worse, one that's silently insecure.

---

## Table of Contents

- [Recursive chmod on Files vs Directories](#recursive-chmod-on-files-vs-directories)
- [Directory Execute vs Read Confusion](#directory-execute-vs-read-confusion)
- [Owner Can Be Locked Out of Their Own File](#owner-can-be-locked-out-of-their-own-file)
- [setuid/setgid Silently Dropped](#setuidsetgid-silently-dropped)
- [Symlinks and chmod](#symlinks-and-chmod)
- [The Sticky Bit Misconception](#the-sticky-bit-misconception)
- [777 Doesn't Mean "Fully Open" Forever](#777-doesnt-mean-fully-open-forever)
- [Uppercase S / T in ls -l Output](#uppercase-s--t-in-ls--l-output)
- [chmod vs ACLs Conflict](#chmod-vs-acls-conflict)
- [Root Bypasses Everything](#root-bypasses-everything)
- [Windows-Mounted Filesystems](#windows-mounted-filesystems)
- [chmod on Files You Don't Own](#chmod-on-files-you-dont-own)
- [--preserve-root Isn't a Full Safety Net](#--preserve-root-isnt-a-full-safety-net)

---

## Recursive chmod on Files vs Directories

### `chmod -R 755` makes every file executable — usually wrong
```bash
chmod -R 755 project/
# ⚠️ Every single file (including .txt, .conf, .jpg) becomes executable!
# This is almost never what you want for a general project tree.

# Correct: different permissions for dirs vs files
find project/ -type d -exec chmod 755 {} \;
find project/ -type f -exec chmod 644 {} \;

# Or: use capital X (conditional execute)
chmod -R a+rX project/
# Directories: always get +x (need it to be traversable)
# Files: only get +x if they already had it somewhere (owner/group/other)
```

---

## Directory Execute vs Read Confusion

### Read without execute on a directory is nearly useless
```bash
chmod 644 somedir     # r--, r--, r-- — NOT rwx!
ls somedir            # ✅ works — you can list names (read bit)
cd somedir            # ❌ Permission denied — no execute bit
cat somedir/file.txt  # ❌ Permission denied — can't traverse into it

# Directories almost always need at least the execute bit for anyone who
# needs to access anything inside them, even just by exact filename.
chmod 755 somedir     # rwx for owner, r-x for group/other — usable
```

### Execute without read on a directory — "blind traversal"
```bash
chmod 711 somedir     # rwx, --x, --x
ls somedir            # ❌ Permission denied — can't list contents
cat somedir/knownfile.txt   # ✅ works — if you already know the filename!
# Useful for "hidden" directories where you don't want browsing,
# but specific known paths should still work (e.g., some /home setups).
```

---

## Owner Can Be Locked Out of Their Own File

### Permission checks are strictly by class — owner isn't automatically privileged
```bash
chmod 077 myfile.txt
# owner: ---, group: rwx, other: rwx
cat myfile.txt         # ❌ Permission denied, even though you own it!

# The kernel checks: "are you the owner?" → yes → use OWNER bits only
# It does NOT fall through to group/other bits just because owner bits are empty.

# Fix: as the owner, you can still chmod it back (chmod itself only requires
# ownership of the file, not read/write access to its contents)
chmod 644 myfile.txt   # works — chmod doesn't need read/write, just ownership
```

---

## setuid/setgid Silently Dropped

### setuid on scripts is ignored by the kernel
```bash
chmod u+s myscript.sh
./myscript.sh
# On Linux, the kernel ignores setuid on scripts with a #! shebang
# (security measure against a well-known class of race-condition exploits)
# It DOES work on setuid for a compiled binary.

# If you need "run this script as another user," use sudo + a controlled
# sudoers entry instead of relying on setuid scripts.
```

### Writing to a setuid/setgid file drops the bit automatically
```bash
ls -l myprogram
# -rwsr-xr-x ... myprogram   (setuid set)

vim myprogram          # any write to the file...
ls -l myprogram
# -rwxr-xr-x ... myprogram   ⚠️ setuid silently cleared!

# This is intentional: the kernel clears setuid/setgid on write to prevent
# a scenario where modifying a file's contents keeps elevated privileges.
# You must re-apply chmod u+s after any edit/recompile.
```

### chown also clears setuid/setgid on many systems
```bash
chmod u+s /usr/local/bin/tool
chown newowner /usr/local/bin/tool
ls -l /usr/local/bin/tool
# setuid bit often cleared as a side effect of chown — verify after!
```

---

## Symlinks and chmod

### chmod on a symlink changes the TARGET, not the link itself
```bash
ln -s realfile.txt link.txt
chmod 600 link.txt
ls -l realfile.txt
# realfile.txt permissions changed, NOT link.txt
# Symlinks themselves conventionally show as lrwxrwxrwx and their
# "permissions" are ignored by the kernel entirely — only the target matters.

ls -l link.txt
# lrwxrwxrwx ... link.txt -> realfile.txt   ← always looks like 777
```

### Broken symlinks
```bash
ln -s /nonexistent/file broken_link
chmod 644 broken_link
# chmod: cannot operate on dangling symlink 'broken_link'
# There's no target to apply the permission change to.
```

---

## The Sticky Bit Misconception

### Sticky bit is about DELETION, not general write protection
```bash
chmod 1777 /shared/dropbox
# Anyone can still WRITE new files into the directory (777 allows it)
# Anyone can still CREATE files
# But a user can only DELETE or RENAME files THEY personally own
# (or if they own the directory, or are root)

# It does NOT prevent:
# - reading other users' files (that's controlled by the FILE's own perms)
# - overwriting the CONTENTS of a file you have write access to
# It ONLY restricts unlink/rename operations within the directory.
```

---

## 777 Doesn't Mean "Fully Open" Forever

### Individual file permissions still apply underneath
```bash
mkdir shared_dir && chmod 777 shared_dir
touch shared_dir/secret.txt
chmod 600 shared_dir/secret.txt    # owner-only

# Other users:
ls shared_dir/                     # ✅ can see the filename (dir is 777)
cat shared_dir/secret.txt          # ❌ Permission denied (file itself is 600)

# 777 on the directory only controls directory OPERATIONS
# (listing, creating, deleting entries) — not access to file contents.
```

---

## Uppercase S / T in ls -l Output

### Uppercase means the special bit is set but execute is NOT
```bash
chmod 4644 file.txt
ls -l file.txt
# -rwSr--r--  ← uppercase S: setuid set, but owner execute bit is off
# This combination is almost always a misconfiguration — setuid is
# meaningless on a non-executable file.

chmod 1644 dir_without_x
ls -ld dir_without_x
# drwSr--r-T  ← sticky set but "other execute" is off — unusual/confusing

# A properly configured setuid binary shows lowercase 's':
chmod 4755 program
ls -l program
# -rwsr-xr-x  ← lowercase s: setuid AND executable
```

---

## chmod vs ACLs Conflict

### Modifying the "group" bits with chmod can silently change ACL permissions
```bash
setfacl -m u:bob:rwx file.txt      # bob gets explicit rwx via ACL
chmod g-w file.txt                  # intending to just tighten the group bit

getfacl file.txt
# The chmod on the "group" class actually updates the ACL "mask" entry,
# which can silently REDUCE bob's effective permissions too!
# mask::r-x   ← now caps bob's rwx down to r-x effectively

# Once ACLs are in play, prefer setfacl/getfacl over chmod for fine control,
# and always re-check with getfacl after any chmod.
```

---

## Root Bypasses Everything

### chmod 000 doesn't stop root
```bash
chmod 000 secret.txt
sudo cat secret.txt     # ✅ still works — root ignores permission bits entirely
cat secret.txt          # ❌ Permission denied (as non-root)

# Never rely on file permissions alone to protect data from a
# process/user that has root access. Use encryption for real secrecy.
```

---

## Windows-Mounted Filesystems

### chmod on FAT32/exFAT/NTFS mounts often has no real effect
```bash
mount | grep /mnt/usb
# /dev/sdb1 on /mnt/usb type vfat (rw,...)

chmod 600 /mnt/usb/file.txt
ls -l /mnt/usb/file.txt
# Permissions may appear unchanged, or apply uniformly to ALL files —
# FAT-family filesystems don't natively support Unix permission bits.
# The umask/permission mount options (e.g., umask=022) control this instead,
# not per-file chmod.

# Check mount options:
mount | grep vfat
# Reformat mount with explicit uid/gid/umask options if control is needed
```

---

## chmod on Files You Don't Own

### You need ownership (or root), not just write access
```bash
chmod 777 someone_elses_file.txt
# chmod: changing permissions of 'someone_elses_file.txt': Operation not permitted

# Even if the file is group-writable and you're in that group, you can
# still NOT chmod it unless you own it or are root.
# chmod authority is tied to file ownership specifically, separate from
# read/write access to the file's contents.

sudo chmod 777 someone_elses_file.txt   # works, as root
```

---

## --preserve-root Isn't a Full Safety Net

### It only protects the literal root directory, not everything under it
```bash
chmod -R 777 /                       # ⚠️ blocked by default (preserve-root)
# chmod: it is dangerous to operate recursively on '/'
# chmod: use --no-preserve-root to override this failsafe

chmod -R 777 /*                      # ⚠️ NOT blocked! Glob expands to
                                      # /bin /etc /home ... individually,
                                      # so --preserve-root never triggers.
# This is a classic scripting trap: wildcard expansion bypasses the guard
# meant to save you from destroying the filesystem's permission structure.
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
