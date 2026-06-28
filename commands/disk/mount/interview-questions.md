# mount — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [fstab & Persistence](#fstab--persistence)
- [Filesystem Types & Options](#filesystem-types--options)
- [Bind Mounts & Overlay](#bind-mounts--overlay)
- [Network Filesystems](#network-filesystems)
- [Namespaces & Containers](#namespaces--containers)
- [Security](#security)
- [Scenario-Based](#scenario-based)
- [Advanced & Internals](#advanced--internals)

---

## Conceptual

**Q1 🔥 What does `mount` do at the kernel level?**
> `mount` calls the `mount(2)` system call which:
> 1. Loads the appropriate filesystem driver (kernel module) if not already loaded
> 2. Reads and verifies the filesystem's superblock
> 3. Creates a `vfsmount` structure linking the filesystem to the VFS tree
> 4. Attaches this structure at the target directory (mountpoint)
> 5. Updates `/proc/mounts`
>
> After this, any path lookup that crosses the mountpoint is redirected to the new filesystem's inode tree — transparently to all userspace applications.

---

**Q2 🔥 What is the VFS (Virtual Filesystem Switch) and why does it matter?**
> The VFS is a kernel abstraction layer that provides a **uniform interface** between system calls (`open()`, `read()`, `write()`, `stat()`) and actual filesystem drivers (ext4, xfs, nfs, tmpfs...).
>
> Because of VFS:
> - Applications don't need to know what filesystem they're reading from
> - Different filesystems can be mixed freely in the directory tree
> - New filesystems can be added without changing application code
> - Network filesystems look identical to local ones

---

**Q3. What is a mountpoint?**
> A mountpoint is an existing **directory** where a filesystem is attached. After mounting, the directory's original contents are hidden and replaced by the root of the newly mounted filesystem. The original contents reappear after unmounting.

---

**Q4 🔥 What is `/proc/mounts` and how is it different from `/etc/fstab`?**
> - `/proc/mounts` — **live kernel view** of all currently mounted filesystems. Maintained by the kernel. Always accurate. Read-only (you can't edit it).
> - `/etc/fstab` — **configuration file** defining filesystems to mount at boot or on demand. Written by the administrator. Not automatically applied — just read by `mount -a` at boot.
>
> `/etc/mtab` on modern systems is a symlink to `/proc/mounts`.

---

**Q5. What is a mount namespace?**
> A mount namespace is a per-process view of the filesystem mount tree. All processes in the same namespace share the same mount tree. Processes can be placed in a **new namespace** (via `unshare --mount`) where they have their own isolated mount tree — changes there don't affect other namespaces.
>
> This is the foundation for container isolation: Docker, podman, and systemd-nspawn each give containers their own mount namespace.

---

## Basic Usage

**Q6 🔥 What is the difference between `mount /dev/sdb1 /mnt` and `mount UUID=... /mnt`?**
> `/dev/sdb1` — device name assigned by the kernel at boot. **Not stable** — can change if disks are added/removed or boot order changes.
>
> `UUID=...` — universally unique identifier stored in the filesystem itself. **Stable** — survives reboots, disk reordering, and cable changes.
>
> Always use UUID (or LABEL) in `/etc/fstab`. Use `/dev/sdX` only for one-time mounts where persistence doesn't matter.

---

**Q7. How do you see all currently mounted filesystems?**
> ```bash
> mount                    # classic (reads /proc/mounts)
> findmnt                  # modern tree view (preferred)
> cat /proc/mounts         # raw kernel source
> df -hT                   # with usage stats and type
> lsblk -f                 # block devices with mountpoints
> ```

---

**Q8 🔥 What does `mount -o remount,ro /` do and when would you use it?**
> Remounts the root filesystem as **read-only** without unmounting it. Used in:
> - **Emergency recovery**: before running fsck on a filesystem that can't be unmounted
> - **System shutdown**: `init` remounts root read-only before halt to prevent corruption
> - **Security hardening**: force read-only after initial setup
>
> ```bash
> mount -o remount,ro /         # make root read-only
> mount -o remount,rw /         # make root read-write again
> ```

---

**Q9. How do you mount a filesystem read-only?**
> ```bash
> mount -o ro /dev/sda1 /mnt/data      # at mount time
> mount -o remount,ro /mnt/data         # already mounted → remount
> ```

---

**Q10. How do you unmount a filesystem that reports "device is busy"?**
> ```bash
> # Find what's using it:
> lsof /mnt/data            # open files
> fuser -m /mnt/data        # PIDs
>
> # Solutions:
> cd /tmp                   # move shell out of mountpoint
> fuser -km /mnt/data       # kill processes (last resort)
> umount -l /mnt/data       # lazy: detach now, cleanup when processes exit
> umount -f /mnt/nfs        # force (for hung NFS)
> ```

---

**Q11. What is `mount -a`?**
> Mounts all filesystems listed in `/etc/fstab` that have the `auto` option (default) and are not yet mounted. Run at boot by the init system. Can also be run manually after editing fstab to apply changes without rebooting.

---

## fstab & Persistence

**Q12 🔥 Explain the six fields of `/etc/fstab`.**
> ```
> UUID=abc123  /home  ext4  defaults,noatime  0  2
> │            │      │     │                 │  │
> │            │      │     │                 │  └─ pass: fsck order (0=skip, 1=root, 2=others)
> │            │      │     │                 └──── dump: backup (0=no, 1=yes — legacy)
> │            │      │     └────────────────────── options: comma-separated mount options
> │            │      └──────────────────────────── type: filesystem type
> │            └─────────────────────────────────── mountpoint: where to attach
> └──────────────────────────────────────────────── device: UUID, LABEL, or /dev/...
> ```

---

**Q13 🔥 What happens if `/etc/fstab` has an error?**
> At boot, the system tries to mount all fstab entries. If a critical entry (like `/home`) fails:
> - The system may drop to an **emergency shell** ("Give root password for maintenance")
> - Or enter **recovery mode** depending on the init system
>
> Prevention:
> ```bash
> mount --fake -av      # dry-run: test fstab without actually mounting
> findmnt --verify      # verify fstab entries against current system
> ```

---

**Q14. What is `_netdev` option and why is it critical for network filesystems?**
> `_netdev` tells the init system (systemd) to **wait for the network** before mounting this filesystem. Without it, the system tries to mount NFS/CIFS entries before the network interface is up — causing:
> - Long boot delays (30-90 second timeout per entry)
> - Boot failure
> - Emergency shell drop
>
> ```
> server:/share  /mnt/nfs  nfs4  defaults,_netdev  0  0
> ```

---

**Q15. Why is the `pass` field important and what are valid values?**
> Controls `fsck` (filesystem check) order at boot:
> - `0` — skip fsck (swap, virtual filesystems, network mounts)
> - `1` — check first (root filesystem `/` only)
> - `2` — check after root (other local filesystems)
>
> Root must be `1`. Other local filesystems should be `2`. Multiple filesystems with `2` are checked in parallel. Never set `pass=1` for non-root.

---

## Filesystem Types & Options

**Q16 🔥 What is the difference between `noatime`, `relatime`, and `strictatime`?**
> Every file read updates its **access time (atime)** on disk — causing a write for every read.
>
> - `strictatime` — always update atime (POSIX default, performance killer)
> - `relatime` — update atime only if it's older than mtime or ctime (Linux default since 2.6.30) — good balance
> - `noatime` — never update atime — maximum performance, breaks some programs that rely on atime (e.g., `mutt`, some backup tools)
>
> Recommendation: `noatime` for SSDs and performance-critical mounts, `relatime` (default) for general use.

---

**Q17. What does `nosuid,noexec,nodev` do and when should you use it?**
> Security mount options:
> - `noexec` — prevent executing binaries from this filesystem
> - `nosuid` — ignore SUID/SGID bits (prevent privilege escalation)
> - `nodev` — don't interpret device files (prevent creating fake `/dev/mem` etc.)
>
> Use on: `/tmp`, `/home`, `/var`, removable media, network shares — any filesystem where users can write files and you don't want them to escalate privileges.

---

**Q18. What is `sync` vs `async` mount option?**
> - `async` (default) — writes are buffered in kernel page cache and flushed later. Faster but data at risk if power loss before flush.
> - `sync` — every write goes directly to disk before returning to the caller. Safe but very slow (10-100x slower for write-heavy workloads).
>
> Use `sync` for: USB drives that users pull without unmounting, critical database files, write-once logging. Use `async` + `barriers` for normal use.

---

**Q19. What filesystem type should you use for `/tmp` and why?**
> **`tmpfs`** — RAM-backed filesystem:
> ```
> tmpfs  /tmp  tmpfs  size=2g,mode=1777,nosuid,nodev  0  0
> ```
> Benefits:
> - Very fast (in RAM)
> - Automatically cleared on reboot (no stale temp files)
> - Files never written to disk
>
> Caveats:
> - Uses RAM/swap (set a size limit!)
> - Lost on reboot (shouldn't matter for temp files)

---

**Q20. What is `tmpfs` vs `ramfs`?**
> - `tmpfs` — has a configurable size limit, uses swap when RAM is tight, contents visible in `/proc/meminfo` as `Shmem`
> - `ramfs` — **no size limit** (can consume all RAM), **no swap** (all in physical RAM), can't be unmounted if it consumes all memory
>
> Always use `tmpfs` with a `size=` option in production. `ramfs` exists for embedded systems and special cases.

---

## Bind Mounts & Overlay

**Q21 🔥 What is a bind mount and how does it differ from a symlink?**
> A **bind mount** makes a directory (or file) accessible at a second path — at the kernel level. A **symlink** is a filesystem entry that redirects path resolution.
>
> Key differences:
> | | Bind Mount | Symlink |
> |---|---|---|
> | Kernel level | Yes (VFS) | No (userspace path) |
> | Works in chroot | Yes | No (symlink target may not exist in chroot) |
> | Follows across namespaces | No | Yes |
> | Can be read-only | Yes | No |
> | `chroot` sees it as real dir | Yes | Depends on target |
>
> ```bash
> mount --bind /source /target    # bind mount
> ln -s /source /target           # symlink
> ```

---

**Q22. How do you create a read-only bind mount?**
> A two-step process — bind first, then remount read-only:
> ```bash
> mount --bind /source /target
> mount -o remount,ro,bind /target
> ```
> In `/etc/fstab`:
> ```
> /source  /target  none  bind     0  0
> /target  /target  none  remount,ro,bind  0  0
> ```

---

**Q23 🔥 What is an overlay filesystem and what is it used for?**
> Overlay (OverlayFS) combines multiple directory layers into one unified view:
> - **lower** — read-only base layer(s)
> - **upper** — read-write layer (receives all changes)
> - **work** — scratch space (same filesystem as upper)
> - **merged** — the unified view presented to users
>
> Reads come from upper (if file exists there) or lower. Writes go to upper only. Lower is never modified.
>
> Used by: **Docker** (each image layer is a lower layer), **live CDs**, **snap packages**, **systemd-nspawn**.
>
> ```bash
> mount -t overlay overlay \
>   -o lowerdir=/lower,upperdir=/upper,workdir=/work \
>   /merged
> ```

---

**Q24. What is `--rbind` vs `--bind`?**
> - `--bind` — bind mounts the directory but **not** its submounts. Submounted filesystems inside source appear empty at target.
> - `--rbind` — **recursive** bind: includes all submounts. The entire mount tree under source is replicated at target.
>
> ```bash
> mount --bind /source /target    # submounts not included
> mount --rbind /source /target   # all submounts included
> ```

---

## Network Filesystems

**Q25 🔥 What is the difference between `hard` and `soft` NFS mounts?**
> - **hard** (default) — if the NFS server becomes unreachable, operations **block indefinitely** until the server returns. Processes accessing the mount are stuck (unkillable with SIGTERM, only SIGKILL works, and even that sometimes fails).
> - **soft** — if the server is unreachable, operations **return an error** after the timeout (`timeo`) × retries (`retrans`). The application gets an I/O error and can handle it.
>
> Rule: Use `soft` for non-critical data. Use `hard,intr` for critical data (allows signal interruption with Ctrl+C).
> ```bash
> mount -t nfs server:/share /mnt -o soft,timeo=30,retrans=3
> ```

---

**Q26. Why should you never put CIFS credentials inline in `/etc/fstab`?**
> `/etc/fstab` is world-readable (`-rw-r--r--`). Inline credentials like `username=alice,password=secret` appear in `/proc/mounts` and `/etc/mtab` — readable by all users.
>
> Correct approach:
> ```bash
> # /etc/samba/credentials (chmod 600, owned by root):
> username=alice
> password=secret
>
> # /etc/fstab:
> //server/share  /mnt  cifs  credentials=/etc/samba/credentials,_netdev  0  0
> ```

---

## Namespaces & Containers

**Q27 🔥 How do Docker containers use mount namespaces?**
> When Docker starts a container, it calls `unshare(CLONE_NEWNS)` to create a new mount namespace for the container process. Inside this namespace, Docker:
> 1. Sets up an overlay filesystem (image layers as lower, container layer as upper)
> 2. Bind mounts `/proc`, `/sys`, `/dev` into the container
> 3. Bind mounts any user-defined volumes (`-v` flag)
>
> The container sees an isolated filesystem tree. Mounts inside the container don't affect the host (private propagation by default).

---

**Q28. What is `unshare --mount` and what does it do?**
> Creates a new mount namespace for the current process and its children:
> ```bash
> unshare --mount bash
> # Now in a new mount namespace
> mount /dev/sdb1 /mnt/test    # only visible in this namespace
> exit
> # Host sees nothing changed
> ```
> The new namespace starts as a copy of the parent's namespace. Changes (mounts/unmounts) are private unless the mount points are configured as `shared`.

---

**Q29. What are mount propagation types?**
> Controls whether mount/unmount events propagate between namespaces:
> - `shared` — events propagate to peers (default on most systems for `/`)
> - `private` — events don't propagate (isolated)
> - `slave` — receives events from master but doesn't propagate back
> - `unbindable` — like private, plus can't be bind-mounted
>
> ```bash
> mount --make-private /mnt     # make private
> mount --make-shared /mnt      # make shared
> cat /proc/self/mountinfo | grep "shared:"  # check propagation
> ```

---

## Security

**Q30 🔥 What are `nosuid`, `noexec`, `nodev` and which filesystems should have them?**
> - `nosuid` — SUID/SGID bits are ignored. Prevents privilege escalation via SUID binaries placed on the filesystem.
> - `noexec` — binaries cannot be executed from this filesystem. Note: scripts run by interpreters (`python script.py`) are NOT blocked — only direct execution.
> - `nodev` — device files are not interpreted. Prevents creating fake `/dev/mem`, `/dev/sda` etc. that could access raw hardware.
>
> Should be set on: `/tmp`, `/var/tmp`, `/home`, `/var`, removable media, network mounts — anywhere users can write but shouldn't have execution/privilege rights.

---

**Q31. What security risk does mounting with `suid` on a user-writable filesystem create?**
> If `/tmp` or `/home` is mounted with the `suid` option (default), a user can:
> 1. Copy a SUID-root binary to `/tmp/`
> 2. If they can somehow make it SUID root (or find one that's already SUID)
> 3. Execute it to gain root
>
> Always mount user-writable filesystems with `nosuid,nodev`:
> ```
> tmpfs  /tmp  tmpfs  size=2g,mode=1777,nosuid,nodev,noexec  0  0
> ```

---

## Scenario-Based

**Q32 🔥 After adding an NFS entry to `/etc/fstab` and rebooting, the system boots very slowly. Why and how do you fix it?**
> The NFS entry is missing `_netdev`. The boot system tries to mount NFS before the network is initialized, waits for the timeout (typically 30-90 seconds per entry), then continues.
>
> Fix: add `_netdev` to the fstab options:
> ```
> server:/share  /mnt/nfs  nfs4  defaults,_netdev,soft,timeo=30  0  0
> ```
> Then test: `mount --fake -av` to validate without rebooting.

---

**Q33 🔥 A server's root filesystem has been remounted read-only by the kernel. What happened and how do you recover?**
> The kernel remounts root read-only when it detects filesystem errors (from `dmesg`). This is the `errors=remount-ro` option at work (default for ext4).
>
> ```bash
> dmesg | grep -E "error|remount|EXT4"   # find the error
>
> # Recovery options:
> # Option 1: remount read-write (if error is non-critical)
> mount -o remount,rw /
>
> # Option 2: fsck (requires read-only or unmounted FS)
> # On next reboot, or from live USB:
> fsck -y /dev/sda1
>
> # Option 3: from emergency shell at boot:
> mount -o remount,rw /
> fsck /dev/sda1   # run fsck while booting from same disk (risky)
> ```

---

**Q34. How do you inspect the contents of a `.img` disk image that has multiple partitions?**
> ```bash
> # Show partition table:
> fdisk -l disk.img
> # or:
> parted disk.img print
>
> # Method 1: kpartx (creates device mapper entries)
> kpartx -av disk.img
> # → /dev/mapper/loop0p1, /dev/mapper/loop0p2, ...
> mount /dev/mapper/loop0p1 /mnt/part1
> # Cleanup:
> kpartx -dv disk.img
>
> # Method 2: losetup with --partscan
> losetup -f --show --partscan disk.img
> # → /dev/loop0, creates /dev/loop0p1, /dev/loop0p2 ...
> mount /dev/loop0p1 /mnt/part1
> # Cleanup:
> umount /mnt/part1
> losetup -d /dev/loop0
>
> # Method 3: manual offset calculation
> fdisk -l disk.img   # note start sector of partition
> # Partition 1: starts at sector 2048, sector size = 512
> # offset = 2048 * 512 = 1048576
> mount -o loop,offset=1048576 disk.img /mnt/part1
> ```

---

**Q35 🔥 You need to set up a chroot environment for system recovery. What mounts do you need?**
> ```bash
> CHROOT=/mnt/system
>
> # Mount the root filesystem first (already done for recovery)
> mount /dev/sda1 $CHROOT
>
> # Bind essential virtual filesystems:
> mount --bind /proc $CHROOT/proc
> mount --bind /sys $CHROOT/sys
> mount --bind /dev $CHROOT/dev
> mount --bind /dev/pts $CHROOT/dev/pts   # terminal support
>
> # Optional: bind resolv.conf for network DNS
> mount --bind /etc/resolv.conf $CHROOT/etc/resolv.conf
>
> # Enter chroot:
> chroot $CHROOT /bin/bash
>
> # Cleanup after exit:
> umount $CHROOT/dev/pts
> umount $CHROOT/dev
> umount $CHROOT/sys
> umount $CHROOT/proc
> umount $CHROOT
> ```
> Without these bind mounts: `apt`, `grub`, `systemctl`, and most tools won't work properly inside the chroot.

---

**Q36. How does Docker volume mounting work under the hood?**
> When you run `docker run -v /host/path:/container/path`, Docker:
> 1. Creates a bind mount from `/host/path` to `/container/path` inside the container's mount namespace
> 2. With **shared** propagation, so changes on either side are visible to both
>
> The container sees the host directory as part of its own filesystem, but it's actually the host directory accessed via VFS bind mount. This is why volume data persists after the container is removed — it's stored on the host filesystem, not in the container's overlay layers.

---

**Q37 🔥 What is the difference between `umount` and `umount -l`? When would you use lazy unmount?**
> - `umount` — immediately detaches the filesystem. Fails with "device is busy" if any process has open files or cwd there.
> - `umount -l` (lazy) — immediately **detaches from the directory tree** (path lookups stop), but the filesystem stays alive until the last file descriptor is closed and the last process leaves.
>
> Use lazy unmount when:
> - NFS server is down and processes are stuck
> - You need to unmount immediately but can't kill all processes
> - Cleaning up after a test/temporary mount with background processes
>
> Risks: data may not be flushed, the mount can't be remounted at the same path until fully cleaned up, and processes may continue accessing stale data.

---

**Q38 🔥 A disk was used on another Linux system with different UIDs. Files are owned by unknown UIDs. How do you fix this after mounting?**
> The filesystem stores numeric UIDs/GIDs. After mounting on a different system where those numbers don't correspond to the same users, files appear with wrong ownership.
>
> Solutions:
> ```bash
> # Option 1: chown recursively (if you know the mapping)
> # Old system: alice=1001, New system: alice=1000
> find /mnt/disk -uid 1001 -exec chown 1000 {} +
> find /mnt/disk -gid 1001 -exec chgrp 1000 {} +
>
> # Option 2: mount with uid/gid override (FAT/NTFS/CIFS only)
> mount -t vfat /dev/sdb1 /mnt/usb -o uid=1000,gid=1000
>
> # Option 3: use --numeric-owner with tar to transfer with uid mapping:
> tar -cf - /source | ssh dest "tar -xf - --numeric-owner"
>
> # Option 4: bindfs FUSE with uid mapping
> bindfs --map=1001/1000 /mnt/disk /mnt/remapped
> ```

---

## Advanced & Internals

**Q39. What is the difference between `mount(2)` syscall flags `MS_RDONLY`, `MS_NOSUID`, `MS_NOEXEC`?**
> These are kernel-level bit flags passed to the `mount(2)` syscall corresponding to mount options:
> - `MS_RDONLY` (1) → `ro`
> - `MS_NOSUID` (2) → `nosuid`
> - `MS_NODEV` (4) → `nodev`
> - `MS_NOEXEC` (8) → `noexec`
> - `MS_SYNCHRONOUS` (16) → `sync`
> - `MS_REMOUNT` (32) → `remount`
> - `MS_MANDLOCK` (64) → mandatory locking
> - `MS_BIND` (4096) → `bind`
> - `MS_MOVE` (8192) → `move`
> - `MS_REC` (16384) → `rbind`
>
> The userspace `mount` command translates option strings to these flags before calling `mount(2)`.

---

**Q40. How do you check which filesystem type a block device has without mounting it?**
> ```bash
> blkid /dev/sdb1
> # /dev/sdb1: UUID="..." TYPE="ext4" PARTUUID="..."
>
> file -s /dev/sdb1
> # /dev/sdb1: Linux rev 1.0 ext4 filesystem data ...
>
> lsblk -f /dev/sdb1
> # NAME   FSTYPE FSVER LABEL UUID   FSAVAIL FSUSE% MOUNTPOINT
> # sdb1   ext4   1.0         abc... 
>
> # Read superblock directly:
> tune2fs -l /dev/sdb1    # ext2/3/4
> xfs_info /dev/sdb1      # XFS (must be mounted or use -r)
> btrfs inspect-internal dump-super /dev/sdb1   # btrfs
> ```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
