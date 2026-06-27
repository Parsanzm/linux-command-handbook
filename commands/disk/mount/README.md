# mount — The Complete Reference

> **Attach filesystems to the directory tree**
> Every file you read on Linux goes through the VFS layer.
> `mount` is the command that connects storage to that tree.

---

## Table of Contents

- [What is mount?](#what-is-mount)
- [Where does mount live?](#where-does-mount-live)
- [How mount works internally](#how-mount-works-internally)
- [The VFS Layer](#the-vfs-layer)
- [Syntax](#syntax)
- [Core Operations](#core-operations)
- [All Options (-o flags)](#all-options--o-flags)
- [Filesystem Types](#filesystem-types)
- [/etc/fstab — Persistent Mounts](#etcfstab--persistent-mounts)
- [/proc/mounts and /proc/self/mountinfo](#procmounts-and-procself-mountinfo)
- [Bind Mounts](#bind-mounts)
- [Namespace Mounts](#namespace-mounts)
- [mount vs udisksctl vs pmount](#mount-vs-udisksctl-vs-pmount)
- [Related Commands](#related-commands)

---

## What is mount?

`mount` attaches a filesystem (on a device, image, or network share) to a directory in the Linux directory tree. After mounting, accessing that directory accesses the filesystem — transparently to applications.

Every Linux system starts with the root filesystem (`/`) mounted by the kernel at boot. Everything else — `/home`, `/var`, `/proc`, `/sys`, USB drives, network shares — is mounted on top via `mount`.

**Three core uses:**
1. **Attach block devices** — disks, partitions, USB drives
2. **Attach virtual filesystems** — `/proc`, `/sys`, `tmpfs`
3. **Attach network filesystems** — NFS, CIFS/SMB, SSHFS

---

## Where does mount live?

```
/bin/mount          ← most Linux systems
/usr/bin/mount      ← some distros (symlink)
```

```bash
which mount
mount --version
# mount from util-linux 2.38.1
```

`mount` is part of **util-linux** — the essential Linux utilities package.

```bash
# Debian/Ubuntu:
dpkg -S $(which mount)      # util-linux: /bin/mount

# RHEL/Fedora:
rpm -qf $(which mount)      # util-linux-...
```

`mount` requires root privileges for most operations. Exceptions:
- Entries in `/etc/fstab` with `user` or `users` option
- FUSE filesystems (user-space filesystems)
- Systems with `udisks2` / `polkit` allowing desktop users to mount removable media

---

## How mount works internally

```
mount command → parse args → open device → detect/verify filesystem
→ mount(2) syscall → kernel VFS registers new mountpoint
→ update /proc/mounts
```

**Step by step:**

1. `mount` userspace tool parses arguments, resolves device/mountpoint
2. If no `-t` given: probes filesystem type via `libblkid` (reads magic bytes)
3. Calls the `mount(2)` system call:
   ```c
   mount(source, target, filesystemtype, mountflags, data);
   // source: "/dev/sda1"
   // target: "/mnt/data"
   // filesystemtype: "ext4"
   // mountflags: MS_RDONLY | MS_NOEXEC | ...
   // data: "errors=remount-ro,noatime"
   ```
4. Kernel loads the appropriate filesystem driver (module)
5. Driver reads superblock, verifies filesystem integrity
6. Kernel attaches the filesystem to the VFS tree at the mountpoint
7. `/proc/mounts` is updated (kernel-maintained)

**Kernel data structures:**
- `vfsmount` — represents a mounted filesystem instance
- `dentry` — directory entry (links filename to inode)
- `super_block` — filesystem-level metadata
- `inode` — file metadata (permissions, size, timestamps)

**Exit codes:**
| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Incorrect invocation or permissions |
| `2` | System error |
| `4` | Internal mount bug |
| `8` | User interrupt |
| `16` | Problems writing or locking `/etc/mtab` |
| `32` | Mount failure |
| `64` | Some mounts succeeded, others failed |

---

## The VFS Layer

The **Virtual Filesystem Switch (VFS)** is the kernel abstraction layer between system calls (`open()`, `read()`, `write()`) and actual filesystem drivers.

```
Application
    │
    │  open("/mnt/usb/file.txt")
    ▼
  VFS Layer  ──── path resolution ──── finds mountpoint /mnt/usb
    │                                   resolves to ext4 driver
    ▼
  ext4 driver ──── block I/O ──── disk blocks on /dev/sdb1
```

**What VFS provides:**
- Uniform interface: every filesystem looks the same to userspace
- Mount namespace support: different processes can see different trees
- Overlapping mounts: one directory can hide another
- Bind mounts: same files accessible at multiple paths

**The mount namespace:**
Each process has a mount namespace — a view of the mount tree. By default all processes share the same namespace. Containers (Docker, systemd-nspawn) create isolated namespaces.

```bash
# View your mount namespace ID:
ls -l /proc/self/ns/mnt
# lrwxrwxrwx ... /proc/self/ns/mnt -> mnt:[4026531840]

# Containers have different IDs:
docker exec container ls -l /proc/self/ns/mnt
# lrwxrwxrwx ... /proc/self/ns/mnt -> mnt:[4026532345]
```

---

## Syntax

```
mount [-l] [-t fstype]
mount [-o options] [-t fstype] device|UUID|LABEL mountpoint
mount --bind source target
mount --rbind source target
mount --move olddir newdir
mount --make-shared mountpoint
mount -a [-t fstype] [-O options]
```

---

## Core Operations

### Show current mounts
```bash
mount                          # all current mounts (from /proc/mounts)
mount -t ext4                  # only ext4 mounts
mount -t nfs,cifs              # multiple types
mount | column -t              # nicely aligned columns
findmnt                        # tree view (modern, preferred)
findmnt -t ext4                # filter by type
cat /proc/mounts               # raw kernel view
cat /etc/mtab                  # legacy (symlink to /proc/mounts on modern systems)
```

### Mount a device
```bash
# Basic
mount /dev/sdb1 /mnt/usb

# Specify filesystem type (usually auto-detected)
mount -t ext4 /dev/sda1 /mnt/data
mount -t vfat /dev/sdb1 /mnt/usb
mount -t ntfs /dev/sdc1 /mnt/windows

# By UUID (safer than /dev/sdX — persistent across reboots)
mount UUID="a1b2c3d4-e5f6-..." /mnt/data
# or:
mount -U a1b2c3d4-e5f6-... /mnt/data

# By LABEL
mount LABEL=MyDisk /mnt/data
# or:
mount -L MyDisk /mnt/data
```

### Mount with options
```bash
mount -o ro /dev/sda1 /mnt/data           # read-only
mount -o rw,noatime /dev/sda1 /mnt/data   # read-write, no access time
mount -o remount,ro /mnt/data              # remount existing as read-only
```

### Mount disk images
```bash
mount -o loop disk.img /mnt/img            # loop device (auto)
mount -t iso9660 -o loop cd.iso /mnt/cdrom
mount -o loop,offset=1048576 disk.img /mnt  # with partition offset
```

### Mount tmpfs (RAM filesystem)
```bash
mount -t tmpfs tmpfs /mnt/ram
mount -t tmpfs -o size=512m tmpfs /mnt/ram    # 512MB limit
mount -t tmpfs -o size=50% tmpfs /tmp         # 50% of RAM
```

### Mount network filesystems
```bash
# NFS
mount -t nfs server:/export/share /mnt/nfs
mount -t nfs4 server:/share /mnt/nfs -o rw,soft,timeo=30

# CIFS/SMB (Windows shares)
mount -t cifs //server/share /mnt/smb -o username=user,password=pass
mount -t cifs //192.168.1.10/share /mnt/smb -o credentials=/etc/samba/creds

# SSHFS (FUSE — user-space)
sshfs user@server:/remote/path /mnt/ssh
```

### Unmount
```bash
umount /mnt/data          # by mountpoint
umount /dev/sdb1          # by device
umount -l /mnt/data       # lazy: detach now, cleanup when last file closed
umount -f /mnt/nfs        # force (for hung NFS)
```

---

## All Options (-o flags)

Options are passed with `-o`, comma-separated.

### General Mount Options

| Option | Description |
|--------|-------------|
| `ro` | Read-only |
| `rw` | Read-write (default) |
| `remount` | Remount already-mounted filesystem |
| `bind` | Bind mount (see Bind Mounts section) |
| `rbind` | Recursive bind mount |
| `move` | Move mountpoint |
| `auto` | Can be mounted with `mount -a` |
| `noauto` | Not mounted with `mount -a` |
| `user` | Allow ordinary user to mount (implies `noexec,nosuid,nodev`) |
| `nouser` | Only root can mount (default) |
| `users` | Any user can mount and any user can unmount |
| `owner` | Owner of device can mount |
| `defaults` | `rw,suid,dev,exec,auto,nouser,async` |
| `_netdev` | Network device — wait for network before mounting |

### Performance Options

| Option | Description |
|--------|-------------|
| `atime` | Update access time on read (default) |
| `noatime` | Don't update access time (performance improvement) |
| `relatime` | Update atime only if older than mtime (default since Linux 2.6.30) |
| `strictatime` | Always update access time |
| `lazytime` | Delay time updates (inode kept in memory, flushed periodically) |
| `async` | All I/O done asynchronously (default) |
| `sync` | All I/O done synchronously (slower, safer) |
| `dirsync` | Directory changes done synchronously |

### Security Options

| Option | Description |
|--------|-------------|
| `exec` | Allow execution of binaries (default) |
| `noexec` | Don't allow execution of binaries |
| `suid` | Allow SUID/SGID bits to take effect (default) |
| `nosuid` | Ignore SUID/SGID bits |
| `dev` | Interpret character/block device files (default) |
| `nodev` | Don't interpret device files |
| `ro` | Read-only |

### ext4-Specific Options

| Option | Description |
|--------|-------------|
| `errors=remount-ro` | Remount read-only on error (default) |
| `errors=continue` | Keep going on error |
| `errors=panic` | Kernel panic on error |
| `data=journal` | Journal all data (safest, slowest) |
| `data=ordered` | Journal metadata, flush data first (default) |
| `data=writeback` | Journal metadata only (fastest, least safe) |
| `barrier=0` | Disable write barriers (faster, less safe) |
| `nodelalloc` | Disable delayed allocation |

### tmpfs-Specific Options

| Option | Description |
|--------|-------------|
| `size=N` | Maximum size (bytes, k, m, g, or %) |
| `nr_inodes=N` | Maximum number of inodes |
| `mode=MODE` | Permissions of root directory |
| `uid=N` | UID of root directory |
| `gid=N` | GID of root directory |

---

## Filesystem Types

| Type | `-t` flag | Description |
|------|-----------|-------------|
| ext4 | `ext4` | Default Linux filesystem |
| ext3 | `ext3` | Older Linux (journaling) |
| ext2 | `ext2` | Oldest Linux (no journaling) |
| xfs | `xfs` | High-performance, large files |
| btrfs | `btrfs` | Copy-on-write, snapshots |
| zfs | `zfs` | Advanced (via OpenZFS module) |
| vfat | `vfat` | FAT32 — USB drives, compatibility |
| exfat | `exfat` | Large FAT — modern USB, SD cards |
| ntfs | `ntfs` / `ntfs3` | Windows NTFS |
| iso9660 | `iso9660` | CD/DVD/ISO images |
| udf | `udf` | DVD, Blu-ray |
| tmpfs | `tmpfs` | RAM-based temporary filesystem |
| ramfs | `ramfs` | RAM filesystem (no size limit — careful!) |
| proc | `proc` | Kernel process info (`/proc`) |
| sysfs | `sysfs` | Kernel device/driver info (`/sys`) |
| devtmpfs | `devtmpfs` | Device files (`/dev`) |
| cgroup2 | `cgroup2` | Control groups v2 |
| overlay | `overlay` | Union filesystem (Docker layers) |
| nfs | `nfs` / `nfs4` | Network File System |
| cifs | `cifs` | SMB/CIFS (Windows shares) |
| fuse | `fuse` | Filesystem in Userspace |
| squashfs | `squashfs` | Read-only compressed (snap, live CDs) |

---

## /etc/fstab — Persistent Mounts

`/etc/fstab` defines filesystems to mount at boot (or on demand).

```bash
cat /etc/fstab
```

**Format:**
```
# device/UUID          mountpoint    type    options              dump  pass
UUID=abc123-...        /             ext4    errors=remount-ro    0     1
UUID=def456-...        /home         ext4    defaults             0     2
UUID=789abc-...        /boot/efi     vfat    umask=0077           0     1
//server/share         /mnt/smb      cifs    credentials=/etc/smb.cred,_netdev  0  0
server:/export         /mnt/nfs      nfs4    defaults,_netdev     0     0
tmpfs                  /tmp          tmpfs   size=2g,mode=1777    0     0
```

**Fields:**
| Field | Description |
|-------|-------------|
| device | Device file, UUID=..., LABEL=..., or network path |
| mountpoint | Directory where it's mounted |
| type | Filesystem type (`ext4`, `vfat`, `nfs`, `auto`) |
| options | Mount options (comma-separated) |
| dump | 0=don't backup, 1=backup (used by `dump` command, mostly legacy) |
| pass | fsck order: 0=skip, 1=first (root), 2=others |

```bash
# Mount all fstab entries not yet mounted
mount -a

# Mount specific fstab entry
mount /mnt/data      # if /mnt/data is in fstab, uses fstab options

# Validate fstab without actually mounting
mount --fake -av
```

**UUID vs /dev/sdX:**
```bash
# /dev/sdX names change depending on boot order, USB insertion order
# UUIDs are stable — always prefer UUID in fstab

# Find UUID of a device:
blkid /dev/sda1
lsblk -f
ls -l /dev/disk/by-uuid/
```

---

## /proc/mounts and /proc/self/mountinfo

```bash
# /proc/mounts — simple list (legacy compat)
cat /proc/mounts
# device mountpoint fstype options dump pass

# /proc/self/mountinfo — detailed (modern)
cat /proc/self/mountinfo
# mountID parentID major:minor root mountpoint mountoptions optional fields - fstype source superoptions

# findmnt reads /proc/self/mountinfo:
findmnt                      # tree view
findmnt /mnt/data            # info about specific mountpoint
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS   # custom columns
findmnt --verify             # verify fstab entries
```

---

## Bind Mounts

Bind mounts make a directory (or file) accessible at another path. No device, no new filesystem — just a second path to the same inodes.

```bash
# Bind mount a directory
mount --bind /source/dir /target/dir
# Now /target/dir shows the same content as /source/dir
# Changes in either path are reflected in the other

# Read-only bind mount
mount --bind /source/dir /target/dir
mount -o remount,ro,bind /target/dir

# Bind mount a single file
mount --bind /etc/hosts /etc/hosts.bak

# Recursive bind (includes submounts)
mount --rbind /source /target

# Common uses:
# Make /tmp available inside a chroot:
mount --bind /proc /chroot/proc
mount --bind /sys /chroot/sys
mount --bind /dev /chroot/dev

# Expose a subdirectory as root of another path:
mount --bind /var/www/html /srv/web
```

**Bind mounts in /etc/fstab:**
```
/source/dir    /target/dir    none    bind    0 0
/source/dir    /target/dir    none    bind,ro 0 0
```

---

## Namespace Mounts

Linux mount namespaces allow different processes to see different filesystem trees.

```bash
# Create a new mount namespace and run a shell in it
unshare --mount bash
# Changes to mounts here don't affect parent namespace

# Run command with private mount namespace
unshare -m command

# Containers use this: Docker, systemd-nspawn, LXC
# Each container has its own mount namespace

# See namespaces:
lsns -t mnt                # list all mount namespaces
ls -la /proc/*/ns/mnt      # one per process

# Enter another process's namespace:
nsenter --mount=/proc/PID/ns/mnt bash
```

**Propagation types (shared subtrees):**
```bash
mount --make-shared /mnt        # changes propagate to peer mounts
mount --make-private /mnt       # changes don't propagate
mount --make-slave /mnt         # receive changes but don't propagate
mount --make-unbindable /mnt    # can't be bind-mounted
```

---

## mount vs udisksctl vs pmount

| Tool | Use Case | Needs Root? |
|------|----------|------------|
| `mount` | Full control, all filesystems, scripting | Yes (usually) |
| `umount` | Unmount | Yes (usually) |
| `udisksctl` | Desktop/user mounting via D-Bus/polkit | No (for removable) |
| `pmount` | User-space mount for removable media | No |
| `sshfs` | FUSE: mount remote SSH filesystem | No |
| `bindfs` | FUSE: bind mount with permission mapping | No |

```bash
# udisksctl: mount without root (uses polkit)
udisksctl mount -b /dev/sdb1
udisksctl unmount -b /dev/sdb1
udisksctl power-off -b /dev/sdb   # safe remove

# pmount (older alternative)
pmount /dev/sdb1 usb_drive
pumount /dev/sdb1
```

---

## Related Commands

| Command | Relation |
|---------|---------|
| `umount` | Unmount a filesystem |
| `findmnt` | Find/list mounts (modern, tree view) |
| `lsblk` | List block devices and their mount points |
| `blkid` | Show block device UUIDs, labels, and types |
| `fdisk` / `parted` | Partition management |
| `mkfs` | Create a filesystem |
| `fsck` | Check and repair filesystem |
| `df` | Disk usage per filesystem |
| `mountpoint` | Check if a path is a mountpoint |
| `unshare` | Run command in new namespaces |
| `nsenter` | Enter existing namespace |
| `systemd-mount` | systemd-integrated mount management |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
