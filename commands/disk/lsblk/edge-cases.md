# lsblk — Edge Cases & Gotchas

> lsblk's output looks authoritative, but loop devices, stale mount info,
> permission requirements, and container environments all produce surprises.

---

## Table of Contents

- [Loop Devices Cluttering Output](#loop-devices-cluttering-output)
- [lsblk Doesn't Show Disk USAGE, Only Structure](#lsblk-doesnt-show-disk-usage-only-structure)
- [Running Without Root Hides Some Information](#running-without-root-hides-some-information)
- [MOUNTPOINTS Can Be Stale or Misleading After Manual umount -l](#mountpoints-can-be-stale-or-misleading-after-manual-umount--l)
- [Multiple Mountpoints for the Same Device](#multiple-mountpoints-for-the-same-device)
- [lsblk Inside a Container Shows the HOST's Devices](#lsblk-inside-a-container-shows-the-hosts-devices)
- [Size Rounding Hides Exact Byte Counts](#size-rounding-hides-exact-byte-counts)
- [Newly Attached Devices Not Appearing Immediately](#newly-attached-devices-not-appearing-immediately)
- [RAID/LVM/LUKS Naming Can Be Ambiguous](#raidlvmluks-naming-can-be-ambiguous)
- [-e and -I Filtering by Major Number Requires Knowing the Number](#-e-and--i-filtering-by-major-number-requires-knowing-the-number)
- [Confusing Device Name Persistence Across Reboots](#confusing-device-name-persistence-across-reboots)
- [JSON Output Structure Changes Between util-linux Versions](#json-output-structure-changes-between-util-linux-versions)

---

## Loop Devices Cluttering Output

### Systems using snap, Docker, or many mounted ISO/image files show dozens of loop entries
```bash
lsblk
# loop0    7:0    0  63.2M  1 loop /snap/core20/1234
# loop1    7:1    0  55.4M  1 loop /snap/snapd/5678
# loop2    7:2    0  91.7M  1 loop /snap/lxd/9012
# ... (potentially DOZENS more) ...
# sda      8:0    0   500G  0 disk
# └─sda1   8:1    0   500G  0 part /
# ⚠️ On a system with many snap packages installed, loop devices can
# genuinely dominate lsblk's output, burying the actual physical disks
# you probably care about far down the list.

# Fix: exclude loop devices explicitly (major number 7)
lsblk -e 7
# sda      8:0    0   500G  0 disk
# └─sda1   8:1    0   500G  0 part /
# ✅ Much cleaner — only real physical/virtual disks remain

# Or show only top-level disks entirely, skipping partition/loop detail
lsblk -d
```

---

## lsblk Doesn't Show Disk USAGE, Only Structure

### A common misconception: lsblk does NOT tell you how FULL a filesystem is
```bash
lsblk -o NAME,SIZE,MOUNTPOINT
# sda1  500G  /
# ⚠️ This shows the PARTITION's total SIZE (500G), NOT how much of
# that space is actually USED or FREE — lsblk has no concept of
# filesystem-level usage at all, since it operates at the BLOCK
# DEVICE level, one layer below where "used vs free space" is tracked.

# For actual usage information, use df instead:
df -h /
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        500G  320G  155G  68% /
# ✅ df reports the FILESYSTEM-level usage; lsblk and df answer
# genuinely DIFFERENT questions and are complementary, not interchangeable.
```

---

## Running Without Root Hides Some Information

### Certain columns silently show as empty/unknown for a non-root user
```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,OWNER
# NAME  SIZE MODEL          SERIAL       OWNER
# sda   500G Samsung SSD    S1234567     root
# ⚠️ As a non-privileged user, some fields (particularly detailed
# hardware info like SERIAL, or certain low-level topology details)
# may show as blank or "0" rather than the actual value, since reading
# them requires elevated access to specific sysfs/udev attributes.

sudo lsblk -o NAME,SIZE,MODEL,SERIAL,OWNER
# Running with sudo typically reveals the FULL set of details reliably
```

---

## MOUNTPOINTS Can Be Stale or Misleading After Manual umount -l

### A "lazy" unmount can leave lsblk showing an inconsistent state briefly
```bash
sudo umount -l /mnt/data
# "-l" (lazy unmount) detaches the mountpoint from the filesystem
# NAMESPACE immediately, but the underlying device may still be
# considered "busy" by the kernel until all references to it
# (open file handles, etc.) are actually released.

lsblk
# sdb1   32G   0 part
# ⚠️ Immediately after a lazy unmount, lsblk might show the device as
# having NO mountpoint (correct from the namespace's perspective) even
# though some process might still technically be using it under the
# hood — the MOUNTPOINTS column reflects the CURRENT mount table, not
# necessarily "is this device truly free to remove right now."

# Before physically removing a drive after unmounting, it's safer to
# also check for lingering open handles:
sudo lsof +D /mnt/data 2>/dev/null
# (should show nothing if it's genuinely safe to remove)
```

---

## Multiple Mountpoints for the Same Device

### Bind mounts can make a single device appear mounted in several places
```bash
sudo mount --bind /data /var/www/data
lsblk -o NAME,MOUNTPOINTS
# sda1  /data
#       /var/www/data
# ⚠️ The SAME underlying partition now shows MULTIPLE mountpoints,
# since bind mounts create additional entries in the mount table
# pointing at the same underlying device/inode — this is CORRECT
# behavior, but can confuse someone expecting a strict 1:1
# device-to-mountpoint mapping.

# lsblk's MOUNTPOINTS column (plural, note the "S") specifically
# supports showing a LIST for exactly this reason — older versions of
# lsblk (pre util-linux 2.37) only had a singular MOUNTPOINT column
# and would only show ONE of the multiple mountpoints, potentially
# hiding this detail entirely on older systems.
```

---

## lsblk Inside a Container Shows the HOST's Devices

### Containers typically share the host's kernel and block device view
```bash
docker run -it ubuntu lsblk
# Often shows the HOST machine's ENTIRE block device layout — every
# disk, partition, and volume on the underlying host — NOT a
# container-specific, isolated view, because containers share the
# host kernel's device model rather than having their own independent
# virtualized block device layer (unlike a full VM).

# This is expected behavior, not a security bug in lsblk itself, but
# worth being aware of: lsblk run inside a container can reveal
# information about the host's storage layout that the container's
# own filesystem view might otherwise seem to hide.
```

---

## Size Rounding Hides Exact Byte Counts

### Human-readable sizes can obscure meaningful precision differences
```bash
lsblk -o NAME,SIZE
# sda1   500G
# sdb1   500G
# ⚠️ Both show "500G," but the ACTUAL byte counts could differ
# slightly (e.g., 500.11 GB vs 499.98 GB) — human-readable rounding
# can make two genuinely DIFFERENT-sized devices look identical at a
# glance, which matters when verifying exact capacity for something
# like RAID array member matching or precise partition planning.

lsblk -b -o NAME,SIZE
# sda1   500107862016
# sdb1   499988279296
# ✅ -b (bytes) reveals the EXACT underlying size, useful whenever
# precise comparison actually matters
```

---

## Newly Attached Devices Not Appearing Immediately

### A device attached via certain interfaces might need a manual rescan
```bash
# After physically attaching a new drive, or after a SAN/iSCSI target
# is provisioned, the device might not appear in lsblk output right away
lsblk
# (new device missing)

# Trigger the kernel to rescan for changes
sudo partprobe
# or, for SCSI/iSCSI specifically:
echo "- - -" | sudo tee /sys/class/scsi_host/host0/scan

lsblk
# (new device now appears, assuming the rescan succeeded)

# udev/systemd typically handle this automatically for USB hotplug
# events, but SAN-attached or programmatically-provisioned storage
# sometimes requires this manual nudge.
```

---

## RAID/LVM/LUKS Naming Can Be Ambiguous

### Device-mapper names aren't always self-explanatory at a glance
```bash
lsblk
# sda1
# └─sda1_crypt         253:0
#   ├─vg0-root         253:1    /
#   └─vg0-swap         253:2    [SWAP]
# ⚠️ Names like "sda1_crypt" or "vg0-root" are conventions chosen by
# cryptsetup/LVM tooling at creation time, not something lsblk itself
# invents — on a system where these were named less descriptively
# (e.g., a generic "cryptvolume1" or "vg-lv0"), the tree structure
# alone might not make it obvious WHICH logical volume serves WHICH
# purpose without additional investigation:

sudo lvs -o lv_name,vg_name,lv_size,lv_attr
# Cross-referencing dedicated LVM tools often clarifies naming/purpose
# far better than lsblk's structural view alone can.
```

---

## -e and -I Filtering by Major Number Requires Knowing the Number

### Filtering by major device number isn't always intuitive
```bash
lsblk -e 7
# Excludes major number 7 (loop devices) — but WHY is loop devices
# "7" specifically? This isn't obvious without consulting a reference.

cat /proc/devices | grep -i loop
# 7 loop
# ✅ /proc/devices lists the major number assignments, confirming
# WHICH number corresponds to which device TYPE on this system —
# major numbers are relatively stable/standardized on Linux, but
# looking them up rather than memorizing them is the reliable approach.

# Common major numbers worth knowing:
# 7   = loop devices
# 8   = SCSI disk devices (sd*)
# 9   = metadisk / software RAID (md*)
# 259 = often used for NVMe devices on modern systems (varies)
```

---

## Confusing Device Name Persistence Across Reboots

### /dev/sdX names are NOT guaranteed to stay consistent between boots
```bash
# Before a reboot:
lsblk
# sda   500G   (the SSD)
# sdb    32G   (the USB drive)

# After a reboot (especially if hardware detection order changes,
# or a new device was added/removed):
lsblk
# sda    32G   (the USB drive — NOW sda!)
# sdb   500G   (the SSD — NOW sdb!)
# ⚠️ Device NAMES like sda/sdb are assigned by detection ORDER at boot
# time, NOT tied permanently to specific physical hardware — a script
# or /etc/fstab entry hardcoding "/dev/sdb1" can silently target the
# WRONG device after a reboot if detection order changes.

# Fix: always reference devices by their STABLE identifiers instead
# of the potentially-shifting sdX name, especially in /etc/fstab or
# any automation:
lsblk -f
# use the UUID or LABEL shown here instead of the device name directly
blkid /dev/sda1
# /dev/sda1: UUID="1234-5678-..." TYPE="ext4"
```

---

## JSON Output Structure Changes Between util-linux Versions

### Scripts parsing lsblk -J may break across different systems
```bash
lsblk -J -o NAME,SIZE
# Older util-linux versions may nest fields or name JSON keys slightly
# differently than newer ones (e.g., changes in how "size" is
# represented — as a formatted string vs including additional
# precision fields in some versions).

lsblk --version
# Always check the util-linux version when writing scripts intended
# to run across MULTIPLE different systems/distros, since JSON schema
# details have evolved across releases — testing the EXACT output
# format on each target system before relying on it in production
# automation is the safest approach.
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
