# mount — Practical Examples

> Real-world patterns from sysadmin, DevOps, containers, and security.

---

## Table of Contents

- [Viewing Current Mounts](#viewing-current-mounts)
- [Block Devices & Partitions](#block-devices--partitions)
- [USB & Removable Media](#usb--removable-media)
- [Disk Images & ISO Files](#disk-images--iso-files)
- [Network Filesystems](#network-filesystems)
- [tmpfs: RAM Filesystems](#tmpfs-ram-filesystems)
- [Bind Mounts](#bind-mounts)
- [Read-Only & Remount](#read-only--remount)
- [/etc/fstab Recipes](#etcfstab-recipes)
- [Loop Devices](#loop-devices)
- [Overlay Filesystem](#overlay-filesystem)
- [Chroot & Container Setup](#chroot--container-setup)
- [FUSE Filesystems](#fuse-filesystems)
- [Scripting with mount](#scripting-with-mount)

---

## Viewing Current Mounts

```bash
# All mounts (simple)
mount
mount | grep -v "^cgroup\|^tmpfs\|^devtmpfs"   # hide virtual FS

# Tree view (modern — preferred)
findmnt
findmnt --real                 # only real filesystems (no virtual)
findmnt -t ext4,xfs,btrfs     # specific types
findmnt -o TARGET,SOURCE,FSTYPE,SIZE,AVAIL,USE%   # custom columns

# Block devices with mount points
lsblk
lsblk -f                      # include UUID, FSTYPE, LABEL
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,UUID

# Disk usage per filesystem
df -h
df -hT                         # include filesystem type
df -h --output=source,fstype,size,used,avail,pcent,target

# Raw kernel view
cat /proc/mounts
cat /proc/self/mountinfo       # detailed (includes propagation type)
```

---

## Block Devices & Partitions

```bash
# Identify available devices
lsblk
fdisk -l          # root only
blkid             # show UUIDs, labels, types

# Mount by device name (fragile — name may change)
mount /dev/sdb1 /mnt/data

# Mount by UUID (stable — preferred)
blkid /dev/sdb1   # find UUID first
mount UUID="550e8400-e29b-41d4-a716-446655440000" /mnt/data

# Mount by LABEL
e2label /dev/sdb1                  # show current label
e2label /dev/sdb1 MyData          # set label
mount LABEL=MyData /mnt/data      # mount by label

# Specify filesystem type explicitly
mount -t ext4 /dev/sda1 /mnt/data
mount -t xfs /dev/sdb1 /mnt/data
mount -t btrfs /dev/sdc1 /mnt/data
mount -t vfat /dev/sdd1 /mnt/usb     # FAT32
mount -t exfat /dev/sde1 /mnt/usb    # exFAT

# Mount with performance options
mount -o noatime,nodiratime /dev/sda1 /mnt/data   # skip access time
mount -o rw,noatime,data=writeback /dev/sda1 /mnt  # max performance ext4

# Mount NTFS (Windows)
mount -t ntfs-3g /dev/sdb1 /mnt/windows       # NTFS-3G (read-write)
mount -t ntfs /dev/sdb1 /mnt/windows -o ro    # kernel ntfs (read-only)
mount -t ntfs3 /dev/sdb1 /mnt/windows         # new kernel ntfs3 driver

# Unmount
umount /mnt/data
umount /dev/sdb1          # by device
umount -l /mnt/data       # lazy: detach now, cleanup when unused
```

---

## USB & Removable Media

```bash
# Identify inserted USB
dmesg | tail -20              # see kernel messages after plugging in
lsblk                         # find device name (e.g., /dev/sdb)
blkid /dev/sdb1               # check filesystem type

# Mount USB (auto-detect filesystem)
mkdir -p /mnt/usb
mount /dev/sdb1 /mnt/usb

# Mount FAT32/exFAT USB with correct permissions
mount -t vfat /dev/sdb1 /mnt/usb -o uid=1000,gid=1000,dmask=022,fmask=133

# Mount exFAT
mount -t exfat /dev/sdb1 /mnt/usb

# Safe removal sequence
sync                          # flush all pending writes
umount /mnt/usb               # unmount
# Now physically remove

# If "device is busy":
lsof /mnt/usb                 # find processes using it
fuser -m /mnt/usb             # find PIDs
fuser -km /mnt/usb            # kill processes (careful!)
umount -l /mnt/usb            # lazy unmount as last resort

# User-space mounting (no root needed)
udisksctl mount -b /dev/sdb1
udisksctl unmount -b /dev/sdb1
udisksctl power-off -b /dev/sdb    # safe eject
```

---

## Disk Images & ISO Files

```bash
# Mount ISO file
mkdir -p /mnt/iso
mount -t iso9660 -o loop,ro ubuntu.iso /mnt/iso
mount -o loop ubuntu.iso /mnt/iso          # auto-detect iso9660

# Mount raw disk image
mount -o loop disk.img /mnt/img

# Mount disk image with partition offset
# First: find offset
fdisk -l disk.img
# Partition 1 starts at sector 2048 (sector = 512 bytes)
# offset = 2048 * 512 = 1048576 bytes

mount -o loop,offset=1048576 disk.img /mnt/part1

# Better: use kpartx or losetup
losetup -f --show disk.img           # attach to loop device
# → /dev/loop0
kpartx -av disk.img                  # create partition devices
# → /dev/mapper/loop0p1

mount /dev/mapper/loop0p1 /mnt/part1
# Cleanup:
kpartx -dv disk.img
losetup -d /dev/loop0

# Mount qcow2 VM image (requires qemu-nbd)
modprobe nbd max_part=8
qemu-nbd --connect=/dev/nbd0 disk.qcow2
mount /dev/nbd0p1 /mnt/vm
# Cleanup:
umount /mnt/vm
qemu-nbd --disconnect /dev/nbd0

# Create and mount a disk image from scratch
dd if=/dev/zero of=newdisk.img bs=1M count=1024   # 1GB image
mkfs.ext4 newdisk.img
mount -o loop newdisk.img /mnt/newdisk
```

---

## Network Filesystems

```bash
# --- NFS ---
# Install client
apt install nfs-common          # Debian/Ubuntu
dnf install nfs-utils           # RHEL/Fedora

# Mount NFS share
mount -t nfs server:/export/share /mnt/nfs
mount -t nfs4 server:/share /mnt/nfs            # NFSv4
mount -t nfs 192.168.1.10:/data /mnt/nfs \
  -o rw,soft,timeo=30,retrans=3,nfsvers=4       # with options

# NFS options explained:
# soft: return error if server unreachable (vs hard: wait forever)
# timeo=30: timeout in 0.1s units (30 = 3 seconds)
# retrans=3: retry 3 times before error
# nfsvers=4: use NFSv4
# rsize=1048576,wsize=1048576: read/write buffer size (1MB)

# In /etc/fstab:
# server:/export  /mnt/nfs  nfs4  defaults,_netdev,rw,soft  0 0

# --- CIFS/SMB (Windows shares) ---
apt install cifs-utils

# Mount with credentials inline (insecure — visible in /proc/mounts)
mount -t cifs //server/share /mnt/smb \
  -o username=alice,password=secret,domain=CORP

# Mount with credentials file (secure)
cat > /etc/samba/credentials << EOF
username=alice
password=secret
domain=CORP
EOF
chmod 600 /etc/samba/credentials

mount -t cifs //server/share /mnt/smb \
  -o credentials=/etc/samba/credentials,uid=1000,gid=1000

# Specify SMB version
mount -t cifs //server/share /mnt/smb \
  -o credentials=/etc/creds,vers=3.0

# In /etc/fstab:
# //server/share  /mnt/smb  cifs  credentials=/etc/samba/creds,_netdev,uid=1000  0 0

# --- SSHFS (FUSE — no root needed) ---
apt install sshfs
sshfs user@server:/remote/path /mnt/ssh
sshfs -o allow_other,default_permissions user@server:/path /mnt/ssh
fusermount -u /mnt/ssh          # unmount SSHFS

# In /etc/fstab (requires allow_other in /etc/fuse.conf):
# user@server:/path  /mnt/ssh  fuse.sshfs  defaults,_netdev,user,allow_other  0 0
```

---

## tmpfs: RAM Filesystems

```bash
# Basic tmpfs (RAM-backed, swappable)
mount -t tmpfs tmpfs /mnt/ram

# With size limit (IMPORTANT — without limit, can fill all RAM)
mount -t tmpfs -o size=512m tmpfs /mnt/ram
mount -t tmpfs -o size=2g tmpfs /mnt/ram
mount -t tmpfs -o size=50% tmpfs /mnt/ram     # 50% of total RAM

# With permissions (useful for shared dirs)
mount -t tmpfs -o size=1g,mode=1777 tmpfs /tmp    # sticky bit like /tmp
mount -t tmpfs -o size=256m,uid=1000,gid=1000 tmpfs /mnt/userram

# tmpfs use cases:
# Fast /tmp
mount -t tmpfs -o size=2g,mode=1777,nosuid,nodev tmpfs /tmp

# Build directory (speed up compilation)
mount -t tmpfs -o size=4g tmpfs /var/cache/ccache

# Browser cache in RAM
mkdir -p /mnt/browser_cache
mount -t tmpfs -o size=512m tmpfs /mnt/browser_cache

# ramfs (no swap, no size limit — careful!)
mount -t ramfs ramfs /mnt/ramfs   # ⚠️ grows without limit!

# In /etc/fstab:
# tmpfs  /tmp  tmpfs  size=2g,mode=1777,nosuid,nodev  0 0
# tmpfs  /dev/shm  tmpfs  defaults,size=1g  0 0
```

---

## Bind Mounts

```bash
# Basic bind mount
mount --bind /source/directory /target/directory
# Same content visible at two paths

# Read-only bind mount
mount --bind /etc/ssl/certs /chroot/etc/ssl/certs
mount -o remount,ro,bind /chroot/etc/ssl/certs

# Bind mount single file
mount --bind /etc/resolv.conf /chroot/etc/resolv.conf

# Recursive bind (includes all submounts)
mount --rbind /dev /chroot/dev

# Bind mount in /etc/fstab:
# /source  /target  none  bind  0 0
# /source  /target  none  bind,ro  0 0

# Use cases:

# Expose a subdirectory as web root:
mount --bind /home/alice/public_html /var/www/html

# Share /proc /sys /dev into chroot
for dir in proc sys dev dev/pts; do
  mount --bind /$dir /chroot/$dir
done

# Make a directory available inside container without full mount:
mount --bind /data/secrets /container/secrets
mount -o remount,ro,bind /container/secrets   # read-only for container

# Overlapping bind — hide content:
mkdir /tmp/empty
mount --bind /tmp/empty /sensitive/dir   # /sensitive/dir now appears empty
```

---

## Read-Only & Remount

```bash
# Mount as read-only initially
mount -o ro /dev/sda1 /mnt/data

# Remount existing mount as read-only (without unmounting)
mount -o remount,ro /mnt/data
mount -o remount,ro /           # remount root as read-only (emergency)

# Remount as read-write
mount -o remount,rw /mnt/data
mount -o remount,rw /           # make root writable again

# Add noexec to existing mount (remount with new options)
mount -o remount,noexec,nosuid,nodev /tmp

# Remount with changed options (keep others)
mount -o remount,relatime /dev/sda1   # change only atime policy

# Force read-only on filesystem with errors:
# In /etc/fstab: errors=remount-ro
# If filesystem has errors, kernel remounts read-only automatically
```

---

## /etc/fstab Recipes

```bash
# Full featured fstab examples:

# Root filesystem (ext4)
# UUID=...  /  ext4  errors=remount-ro,relatime  0  1

# Home on separate partition
# UUID=...  /home  ext4  defaults,noatime  0  2

# EFI boot partition
# UUID=...  /boot/efi  vfat  umask=0077,shortname=winnt  0  1

# Swap partition
# UUID=...  none  swap  sw  0  0

# tmpfs for /tmp (RAM-backed)
# tmpfs  /tmp  tmpfs  defaults,size=2g,mode=1777,nosuid,nodev  0  0

# tmpfs for /dev/shm (shared memory)
# tmpfs  /dev/shm  tmpfs  defaults,size=1g  0  0

# NFS share
# server:/export  /mnt/nfs  nfs4  defaults,_netdev,rw,soft,timeo=30  0  0

# CIFS share
# //server/share  /mnt/smb  cifs  credentials=/etc/samba/creds,_netdev,uid=1000,gid=1000  0  0

# USB drive (by UUID, user can mount)
# UUID=...  /mnt/usb  vfat  defaults,user,noauto,umask=0000  0  0

# Loop-mounted image
# /path/to/disk.img  /mnt/img  ext4  loop  0  0

# Bind mount
# /source/dir  /target/dir  none  bind  0  0

# Read-only bind
# /source/dir  /target/dir  none  bind,ro  0  0

# Test fstab without mounting:
mount --fake -av -T /etc/fstab.test    # test alternate fstab file
findmnt --verify                        # verify current fstab
```

---

## Loop Devices

```bash
# List loop devices
losetup -l
losetup -a              # all attached loop devices

# Create loop device from image
losetup /dev/loop0 disk.img
losetup -f disk.img     # find first free loop device
losetup -f --show disk.img   # attach and print device name

# Mount via loop device
losetup -f --show disk.img   # → /dev/loop0
mount /dev/loop0 /mnt/img

# Shortcut (auto-creates loop device)
mount -o loop disk.img /mnt/img

# Mount partition inside image
losetup -f --show --partscan disk.img   # --partscan creates loop0p1, loop0p2...
mount /dev/loop0p1 /mnt/part1

# Detach loop device
umount /mnt/img
losetup -d /dev/loop0
losetup -D              # detach ALL loop devices
```

---

## Overlay Filesystem

Used by Docker, container runtimes, and live systems.

```bash
# Basic overlay mount
mkdir -p /overlay/{lower,upper,work,merged}
echo "base file" > /overlay/lower/base.txt

mount -t overlay overlay \
  -o lowerdir=/overlay/lower,upperdir=/overlay/upper,workdir=/overlay/work \
  /overlay/merged

# Now: /overlay/merged shows lower + upper combined
# Reads: from lower (or upper if file exists there)
# Writes: go to upper only (lower is unchanged)
# Deletes: recorded as "whiteout" in upper

ls /overlay/merged           # base.txt (from lower)
echo "new" > /overlay/merged/new.txt   # goes to upper/
cat /overlay/upper/new.txt   # ✅ exists here
cat /overlay/lower/new.txt   # ❌ doesn't exist here

# Multiple lower layers (Docker-style)
mount -t overlay overlay \
  -o lowerdir=/layer3:/layer2:/layer1,upperdir=/upper,workdir=/work \
  /merged
# Layers resolved right to left: layer1 → layer2 → layer3 → upper

umount /overlay/merged
```

---

## Chroot & Container Setup

```bash
# Full chroot setup (Debian/Ubuntu example)
CHROOT=/mnt/chroot

# Mount essential virtual filesystems
mount --bind /proc $CHROOT/proc
mount --bind /sys $CHROOT/sys
mount --bind /dev $CHROOT/dev
mount --bind /dev/pts $CHROOT/dev/pts
mount -t tmpfs tmpfs $CHROOT/tmp

# Enter chroot
chroot $CHROOT /bin/bash

# Cleanup after chroot
umount $CHROOT/dev/pts
umount $CHROOT/dev
umount $CHROOT/sys
umount $CHROOT/proc
umount $CHROOT/tmp

# Arch Linux chroot (arch-chroot does this automatically)
arch-chroot /mnt /bin/bash

# systemd-nspawn (more complete container)
systemd-nspawn -D /mnt /bin/bash
systemd-nspawn -D /mnt --boot    # boot full system
```

---

## FUSE Filesystems

```bash
# SSHFS — mount remote SSH
sshfs user@host:/path /mnt/remote
sshfs -o reconnect,ServerAliveInterval=15 user@host:/path /mnt/remote
fusermount -u /mnt/remote    # unmount

# bindfs — bind mount with permission mapping
bindfs --map=root/alice /source /target    # root files appear as alice's
bindfs --perms=a-w /source /target         # make all files appear read-only
fusermount -u /target

# s3fs — mount S3 bucket
echo "ACCESS_KEY:SECRET_KEY" > ~/.passwd-s3fs
chmod 600 ~/.passwd-s3fs
s3fs mybucket /mnt/s3 -o passwd_file=~/.passwd-s3fs
fusermount -u /mnt/s3

# rclone — mount cloud storage
rclone mount remote:bucket /mnt/cloud --daemon
fusermount -u /mnt/cloud

# Check FUSE mounts
mount | grep fuse
findmnt -t fuse,fuse.sshfs,fuse.s3fs
```

---

## Scripting with mount

```bash
# Check if a path is a mountpoint
mountpoint /mnt/data && echo "is mounted" || echo "not mounted"
mountpoint -q /mnt/data    # quiet: just exit code

# Check if device is mounted
grep -qs "/dev/sdb1" /proc/mounts && echo "mounted"

# Mount only if not already mounted
if ! mountpoint -q /mnt/data; then
  mount /dev/sdb1 /mnt/data
fi

# Unmount only if mounted
umount_if_mounted() {
  mountpoint -q "$1" && umount "$1"
}
umount_if_mounted /mnt/data

# Wait for device to appear (USB insertion)
wait_for_device() {
  local dev="$1"
  local timeout=30
  local count=0
  while [ ! -b "$dev" ] && [ $count -lt $timeout ]; do
    sleep 1
    ((count++))
  done
  [ -b "$dev" ]
}

wait_for_device /dev/sdb1 && mount /dev/sdb1 /mnt/usb

# Auto-mount script using udev/systemd
# /etc/systemd/system/mnt-data.mount:
# [Unit]
# Description=Data disk
# [Mount]
# What=/dev/disk/by-uuid/...
# Where=/mnt/data
# Type=ext4
# Options=noatime
# [Install]
# WantedBy=multi-user.target

# Get filesystem info after mounting
df -h /mnt/data
stat -f /mnt/data    # filesystem stats
tune2fs -l /dev/sda1 | grep -E "Block size|Inode count|Mount count"
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
