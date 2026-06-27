# mount — Edge Cases & Gotchas

> mount failures can leave systems unbootable, data inaccessible, or processes hung.
> These are the traps that catch even senior sysadmins.

---

## Table of Contents

- [/etc/fstab Errors That Break Boot](#etcfstab-errors-that-break-boot)
- [Device Name Instability](#device-name-instability)
- [Mount Point Traps](#mount-point-traps)
- [Busy Device / Unmount Failures](#busy-device--unmount-failures)
- [NFS Hangs](#nfs-hangs)
- [Filesystem Corruption on Mount](#filesystem-corruption-on-mount)
- [Bind Mount Surprises](#bind-mount-surprises)
- [Loop Device Exhaustion](#loop-device-exhaustion)
- [tmpfs: No Size Limit Danger](#tmpfs-no-size-limit-danger)
- [Permission & Ownership Surprises](#permission--ownership-surprises)
- [Overlay Filesystem Pitfalls](#overlay-filesystem-pitfalls)
- [Namespace & Container Traps](#namespace--container-traps)
- [Security Issues](#security-issues)

---

## /etc/fstab Errors That Break Boot

### Wrong UUID or typo = system won't boot
```bash
# A wrong UUID in fstab for a non-root partition:
# At boot: "Give root password for maintenance" or emergency shell

# Prevention: always test fstab before rebooting
mount --fake -av          # dry run: verify all fstab entries
findmnt --verify          # verify fstab against current mounts

# If already broken — recovery:
# 1. Boot from live USB
# 2. Mount root filesystem
# 3. Fix /etc/fstab
# 4. Reboot

# Or in emergency shell:
mount -o remount,rw /    # make root writable
vi /etc/fstab            # fix the error
```

### Missing `_netdev` for network filesystems
```bash
# If NFS/CIFS entry in fstab lacks _netdev:
# System tries to mount BEFORE network is up → hangs at boot
# ⚠️ Can cause 90-second timeout per entry, or total hang

# WRONG:
# server:/share  /mnt/nfs  nfs4  defaults  0  0

# CORRECT:
# server:/share  /mnt/nfs  nfs4  defaults,_netdev  0  0
# _netdev tells systemd: wait for network before mounting
```

### `pass` field errors
```bash
# /etc/fstab 6th field (pass) controls fsck order:
# 0 = don't check
# 1 = check first (ROOT only)
# 2 = check after root

# If two filesystems both have pass=1:
# fsck runs them in parallel — may cause issues on same disk

# If root has pass=0:
# Root filesystem is never fsck'd — dangerous

# Correct pattern:
# /       ext4  defaults  0  1   ← root: checked first
# /home   ext4  defaults  0  2   ← others: checked after
# swap    swap  sw        0  0   ← swap: never
# tmpfs   tmpfs defaults  0  0   ← virtual: never
```

### Typo in options field
```bash
# Common mistakes:
# defaults,noatim    → typo: noatim (should be noatime) → mount fails
# ro rw              → space instead of comma → mount fails
# uid=alice          → name instead of number for some filesystems → may fail

# Test before applying:
mount -o noatim /dev/sda1 /mnt/test    # will fail with "bad option"
mount -o noatime /dev/sda1 /mnt/test   # correct
```

---

## Device Name Instability

### /dev/sdX names change between boots
```bash
# /dev/sda, /dev/sdb change based on:
# - Boot order
# - USB insertion order
# - Kernel detection order
# - Adding/removing disks

# ⚠️ NEVER use /dev/sdX in /etc/fstab for non-root partitions
# ✅ ALWAYS use UUID or LABEL

# Find UUID:
blkid /dev/sdb1
lsblk -f
ls -l /dev/disk/by-uuid/

# Find LABEL:
lsblk -f
blkid -s LABEL /dev/sdb1

# /dev/disk/by-id/ also works (uses hardware serial):
ls -l /dev/disk/by-id/
# /dev/disk/by-id/ata-Samsung_SSD_860_EVO_S3YJNX0M123456-part1
```

### /dev/loop names can change
```bash
# Loop device numbers (loop0, loop1...) are assigned dynamically
# Use losetup -f to find the next free one:
LOOP=$(losetup -f)
losetup $LOOP disk.img
mount $LOOP /mnt/img
```

---

## Mount Point Traps

### Mounting hides existing directory contents
```bash
# If /mnt/data has files:
ls /mnt/data    # file1.txt  file2.txt

# After mounting:
mount /dev/sdb1 /mnt/data

ls /mnt/data    # shows filesystem on /dev/sdb1 — original files HIDDEN!
# The original files still exist on disk but are inaccessible while mounted

# After unmounting:
umount /mnt/data
ls /mnt/data    # original files are back!
```

### Mount point must exist
```bash
mount /dev/sdb1 /mnt/newdir
# mount: /mnt/newdir: mount point does not exist

# Always create first:
mkdir -p /mnt/newdir
mount /dev/sdb1 /mnt/newdir
```

### Mount point is on same filesystem as what's being mounted
```bash
# Circular-ish situation — works but confusing:
# If /mnt is on the same partition as /dev/sda1, and you mount sda1 to /mnt:
mount /dev/sda1 /mnt    # OK — creates overlay, /mnt now shows sda1

# But this can hide /mnt/other_stuff
# Be careful with what's in mount points
```

### Stale mount points after failed unmount
```bash
# Mount point appears busy, unmount fails:
umount /mnt/nfs    # device is busy

# But NFS server is down — mount is stale
# ls /mnt/nfs hangs forever!

# Force unmount:
umount -f /mnt/nfs    # force (NFS)
umount -l /mnt/nfs    # lazy: detach from tree now, cleanup later

# Still hung? Check what's using it:
lsof /mnt/nfs
fuser -m /mnt/nfs

# Nuclear option (data loss risk):
fuser -km /mnt/nfs    # kill all processes using it
umount /mnt/nfs
```

---

## Busy Device / Unmount Failures

### "Target is busy" — find the culprit
```bash
umount /mnt/data
# umount: /mnt/data: target is busy.

# Method 1: lsof
lsof /mnt/data                # list open files on mountpoint
lsof +D /mnt/data             # recursive (slower but thorough)

# Method 2: fuser
fuser -m /mnt/data            # PIDs using the mountpoint
fuser -vm /mnt/data           # verbose: user + PID + access type

# Method 3: find the shell
ls -la /proc/*/cwd | grep /mnt/data   # processes with cwd there

# Common causes:
# - Shell with cwd inside the mount
# - Background process running from mount
# - Open file handles (even deleted files)

# Fix:
cd /tmp                    # move shell out of mount
fuser -km /mnt/data        # kill processes (last resort)
umount -l /mnt/data        # lazy unmount if processes can't be killed
```

### Lazy unmount consequences
```bash
umount -l /mnt/data    # detaches from directory tree immediately
# ⚠️ Processes with open files can still access the filesystem
# ⚠️ Filesystem stays "mounted" internally until last file closed
# ⚠️ You can't re-mount to the same path until it's fully cleaned up
# ⚠️ Data may not be fully written

# Check if lazy unmount is complete:
cat /proc/mounts | grep /dev/sdb1   # still appears? not done yet
lsof | grep /dev/sdb1               # still open files?
```

---

## NFS Hangs

### Hard mount hangs forever on server failure
```bash
# Default NFS mount is "hard" — retries forever
# If server goes down: all operations on mount HANG, unkillable

mount -t nfs server:/share /mnt/nfs    # hard mount (default)
# Server dies → ls /mnt/nfs hangs, can't Ctrl+C, process unkillable!

# ✅ Use soft mount for non-critical data:
mount -t nfs server:/share /mnt/nfs -o soft,timeo=30,retrans=3
# soft: returns error after timeout (instead of hanging forever)
# timeo=30: timeout = 3 seconds (30 × 0.1s)
# retrans=3: retry 3 times before giving up

# For critical data: hard mount + intr (allow signal interruption)
mount -t nfs server:/share /mnt/nfs -o hard,intr,timeo=30
```

### NFS mount succeeds but files hang on access
```bash
# Server firewall blocks NFS ports:
# mount succeeds (RPC portmapper answered)
# but file access hangs (data port blocked)

# Check NFS ports:
rpcinfo -p server        # list RPC services
showmount -e server      # list exports

# NFSv4 uses only port 2049 (easier firewall):
mount -t nfs4 server:/share /mnt/nfs   # only needs port 2049
```

---

## Filesystem Corruption on Mount

### Mounting dirty filesystem (improper shutdown)
```bash
# After a crash, ext4 journal may need replay
mount /dev/sda1 /mnt/data
# May see: EXT4-fs: recovery complete.
# Or: EXT4-fs error: ...

# If mount fails due to serious corruption:
fsck /dev/sda1            # repair (unmounted!)
fsck -y /dev/sda1         # auto-answer yes to all questions

# NEVER run fsck on a mounted filesystem!
fsck /dev/sda1            # ❌ if sda1 is mounted
umount /dev/sda1 && fsck /dev/sda1   # ✅ unmount first

# Forced fsck on next boot:
touch /forcefsck           # triggers fsck at next boot (older distros)
tune2fs -C 0 /dev/sda1    # reset mount count to trigger fsck
shutdown -rF now           # reboot with forced fsck (some distros)
```

### Mounting read-only due to errors
```bash
# Kernel may remount filesystem read-only after detecting errors:
dmesg | grep "remounting read-only"
dmesg | grep "EXT4-fs error"

# Filesystem is now in emergency read-only mode
mount | grep ro    # check: (ro) instead of (rw)

# Recovery:
# 1. Identify the error from dmesg
# 2. Unmount and run fsck
umount /dev/sda1
fsck -y /dev/sda1
mount /dev/sda1 /mnt/data   # should be rw now
```

---

## Bind Mount Surprises

### Bind mount doesn't inherit original's mount options
```bash
# Original mount:
mount -o ro /dev/sda1 /mnt/source   # read-only

# Bind mount:
mount --bind /mnt/source /mnt/bind   # ⚠️ may be READ-WRITE!

# The bind mount doesn't inherit ro — it needs its own:
mount --bind /mnt/source /mnt/bind
mount -o remount,ro,bind /mnt/bind   # explicitly make it read-only
```

### Bind mount doesn't show submounts
```bash
# /mnt/source has submounts (e.g., /mnt/source/usb is a separate mount)
mount --bind /mnt/source /mnt/bind
ls /mnt/bind/usb   # ❌ empty! submount not included

# Use --rbind for recursive (includes submounts):
mount --rbind /mnt/source /mnt/bind
ls /mnt/bind/usb   # ✅ shows submount contents
```

### Bind mount in fstab with "ro" doesn't work directly
```bash
# /etc/fstab:
# /source  /target  none  bind,ro  0 0

# This doesn't work as expected — the ro is ignored for bind mounts in fstab
# Linux applies bind first (rw), then the ro is silently ignored

# Workaround: use two lines
# /source  /target  none  bind     0 0
# /target  /target  none  remount,ro,bind  0 0
```

---

## Loop Device Exhaustion

### Running out of loop devices
```bash
# Default: 8 loop devices (loop0-loop7)
# With many mounted images, you run out

ls /dev/loop*    # see available
losetup -l       # see used

# Increase at runtime:
modprobe loop max_loop=64    # add more loop devices

# Or create one manually:
mknod /dev/loop8 b 7 8
chmod 660 /dev/loop8

# Persistent (in /etc/modprobe.d/loop.conf):
echo "options loop max_loop=64" > /etc/modprobe.d/loop.conf

# Docker and snap use loop devices heavily:
losetup -l | grep snap   # snap packages each use a loop device
```

---

## tmpfs: No Size Limit Danger

### tmpfs without size limit fills all RAM + swap
```bash
# ⚠️ DANGEROUS:
mount -t tmpfs tmpfs /mnt/ram   # no size limit!
# If a process writes 100GB to this mount:
# RAM fills → swap fills → OOM killer → system crash/lockup

# ✅ ALWAYS set a size limit:
mount -t tmpfs -o size=512m tmpfs /mnt/ram

# ramfs is even more dangerous — no swap, no limit, can't be freed:
mount -t ramfs ramfs /mnt/ram   # ❌ avoid in production

# Check current tmpfs usage:
df -h | grep tmpfs
cat /proc/meminfo | grep -i shmem   # shared memory (includes tmpfs)
```

### /tmp as tmpfs: processes lose temp files on reboot
```bash
# If /tmp is tmpfs, it's cleared on reboot — correct behavior
# But some programs expect /tmp to persist across reboots (wrong assumption)
# Also: tmpfs /tmp means temp files use RAM, not disk

# If /tmp fills: programs crash
df -h /tmp   # check usage
ls -la /tmp  # find large files
```

---

## Permission & Ownership Surprises

### FAT/NTFS don't have Unix permissions
```bash
# FAT32, exFAT, NTFS don't store Unix permissions
# All files appear with same permissions after mount

# Control permissions via mount options:
mount -t vfat /dev/sdb1 /mnt/usb \
  -o uid=1000,gid=1000,fmask=133,dmask=022
# uid/gid: owner of all files
# fmask: permission mask for files (133 = ~133 = 644)
# dmask: permission mask for dirs (022 = ~022 = 755)

# For NTFS:
mount -t ntfs-3g /dev/sdb1 /mnt/win \
  -o uid=1000,gid=1000,umask=022
```

### Mount as root, files owned by root
```bash
# tmpfs mounted by root:
mount -t tmpfs tmpfs /mnt/ram
ls -la /mnt/ram
# drwxr-xr-x root root ...   ← root owns it!

# Regular users can't write:
touch /mnt/ram/file   # Permission denied

# Fix: set uid/gid or mode:
mount -t tmpfs -o size=1g,uid=1000,gid=1000,mode=755 tmpfs /mnt/ram
# or world-writable (like /tmp):
mount -t tmpfs -o size=1g,mode=1777 tmpfs /mnt/ram
```

---

## Overlay Filesystem Pitfalls

### workdir must be on same filesystem as upperdir
```bash
mount -t overlay overlay \
  -o lowerdir=/lower,upperdir=/upper,workdir=/work \
  /merged
# ❌ Error if /upper and /work are on different filesystems!
# workdir MUST be on the same filesystem as upperdir

# ✅ Correct: put both on same partition
mkdir -p /data/overlay/{upper,work}
mount -t overlay overlay \
  -o lowerdir=/lower,upperdir=/data/overlay/upper,workdir=/data/overlay/work \
  /merged
```

### Overlay doesn't work on NFS/FUSE lower layers (older kernels)
```bash
# Older kernels don't support overlay on NFS or FUSE lower directories
# Solution: use local filesystem for lower layers
# Or upgrade kernel (4.x+ has better overlay support)
```

### Deleted files leave whiteout entries in upper
```bash
# Deleting a file from overlay creates a "whiteout" in upper layer:
rm /merged/file_from_lower.txt

ls /upper/
# c---------  file_from_lower.txt   ← character device with 0:0 = whiteout!

# The lower layer is unchanged:
ls /lower/
# file_from_lower.txt   ← still here

# After umount and new mount: the whiteout hides the lower file again
```

---

## Namespace & Container Traps

### Mount inside container doesn't affect host
```bash
# Docker containers have their own mount namespace
docker exec container mount /dev/sdb1 /mnt/data
# ❌ This may fail (/dev/sdb1 not visible in container)
# ✅ Or succeed but only inside the container — host doesn't see it

# To mount inside container: run privileged or use volumes
docker run --privileged ...
docker run -v /host/path:/container/path ...
```

### Propagation: shared vs private mounts
```bash
# By default: mounts inside a bind mount of / are SHARED (propagate to parent namespace)
# In containers: this can be surprising

# Make mount private (don't propagate):
mount --make-private /mnt

# Make mount shared:
mount --make-shared /mnt

# Check propagation:
cat /proc/self/mountinfo | awk '{print $5, $7, $9}'
# shared:N = shared with peers
# private = no propagation
```

---

## Security Issues

### SUID on noexec filesystems
```bash
# noexec prevents executing binaries, BUT:
# If /tmp is mounted with noexec, users can still:
# - Copy a script to /tmp and run it from elsewhere
# - Use interpreters: python /tmp/script.py (noexec doesn't block this!)

# nosuid prevents SUID binaries from working:
mount -o nosuid,noexec,nodev /dev/sdb1 /mnt/untrusted
# ✅ Prevents SUID escalation from this filesystem
```

### Credentials in /proc/mounts
```bash
# CIFS mount with inline credentials:
mount -t cifs //server/share /mnt -o username=alice,password=secret123
cat /proc/mounts
# //server/share /mnt cifs username=alice,password=secret123 ...
# ⚠️ Password visible in /proc/mounts — readable by ALL users!

# ✅ Always use credentials file:
mount -t cifs //server/share /mnt -o credentials=/etc/samba/creds
cat /proc/mounts
# //server/share /mnt cifs credentials=/etc/samba/creds ...
# Only the file path is shown, not the contents
```

### World-readable /etc/fstab with credentials path
```bash
ls -l /etc/fstab
# -rw-r--r-- root root    ← world-readable!
# If fstab contains credentials file path, users can read the path

# The credentials file itself must be root-only:
chmod 600 /etc/samba/credentials
chown root:root /etc/samba/credentials
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
