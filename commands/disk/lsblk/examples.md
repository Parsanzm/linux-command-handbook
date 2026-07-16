# lsblk — Practical Examples

> Real-world patterns for identifying disks, diagnosing storage layouts, and scripting around lsblk.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Finding a Newly Inserted USB Drive](#finding-a-newly-inserted-usb-drive)
- [Filesystem Details](#filesystem-details)
- [Choosing Custom Columns](#choosing-custom-columns)
- [Scripting with JSON and Pairs Output](#scripting-with-json-and-pairs-output)
- [Investigating LVM, RAID, and LUKS Layers](#investigating-lvm-raid-and-luks-layers)
- [Finding Unmounted Partitions](#finding-unmounted-partitions)
- [Combining lsblk with Other Tools](#combining-lsblk-with-other-tools)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# See everything, default tree view
lsblk

# See just one specific disk
lsblk /dev/sda

# See only top-level disks, no partition breakdown
lsblk -d

# Show device paths in full (/dev/sda1) instead of bare names (sda1)
lsblk -p

# Include devices that would otherwise be hidden (e.g., empty/zero-size)
lsblk -a
```

---

## Finding a Newly Inserted USB Drive

```bash
# Before inserting the USB drive
lsblk > /tmp/before.txt

# ... insert USB drive ...

lsblk > /tmp/after.txt
diff /tmp/before.txt /tmp/after.txt
# Shows exactly which new device appeared

# Faster one-liner: look at the RM (removable) column directly
lsblk -o NAME,SIZE,RM,TYPE,MOUNTPOINTS
# Any TYPE=disk row with RM=1 is a removable device (USB, SD card, etc.)

# Watch for a device to appear live
watch -n 1 lsblk
# insert the drive, watch it appear in the tree within a second
```

---

## Filesystem Details

```bash
# Show filesystem type, label, and UUID alongside the tree
lsblk -f
# NAME   FSTYPE FSVER LABEL   UUID                                 MOUNTPOINTS
# sda
# ├─sda1 vfat   FAT32 EFI     1234-ABCD                             /boot/efi
# ├─sda2 swap         swap    a1b2c3d4-...                          [SWAP]
# └─sda3 ext4         root    e5f6g7h8-...                          /

# Just the UUID for a specific partition (useful for /etc/fstab entries)
lsblk -no UUID /dev/sda3

# Just the filesystem type
lsblk -no FSTYPE /dev/sda3
```

---

## Choosing Custom Columns

```bash
# A concise summary: name, size, type, mountpoint
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT

# Include filesystem AND ownership info together
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,OWNER,GROUP,MODE

# Show model and serial number (useful for physically identifying a
# specific drive in a multi-disk server)
lsblk -o NAME,SIZE,MODEL,SERIAL

# Show absolutely every available column, for exploration
lsblk -O | less -S
```

---

## Scripting with JSON and Pairs Output

```bash
# Get a list of all disk names using JSON + jq
lsblk -J | jq -r '.blockdevices[] | select(.type=="disk") | .name'
# sda
# sdb
# nvme0n1

# Get the size of each disk
lsblk -J -o NAME,SIZE,TYPE | jq -r '.blockdevices[] | select(.type=="disk") | "\(.name): \(.size)"'

# Find all mounted filesystems and their mountpoints via JSON
lsblk -J -o NAME,MOUNTPOINTS,FSTYPE | jq -r '.blockdevices[] | select(.mountpoints[0] != null) | "\(.name) -> \(.mountpoints[0])"'

# PAIRS format for simple shell parsing without a JSON tool
lsblk -P -o NAME,SIZE,TYPE,MOUNTPOINT | while read -r line; do
  eval "$line"
  echo "Device: $NAME, Size: $SIZE, Type: $TYPE"
done
```

---

## Investigating LVM, RAID, and LUKS Layers

```bash
# See the full stack: physical disk -> LUKS -> LVM -> filesystem
lsblk
# sda
# └─sda1
#   └─sda1_crypt
#     ├─vg0-root  /
#     └─vg0-swap  [SWAP]

# Show only LVM logical volumes
lsblk -o NAME,TYPE,SIZE,MOUNTPOINT | grep lvm

# Show only encrypted (LUKS) devices
lsblk -o NAME,TYPE,SIZE | grep crypt

# See a RAID array and which physical disks back it
lsblk -o NAME,TYPE,SIZE,MOUNTPOINT | grep -A2 raid

# Cross-reference with dedicated LVM/RAID tools for MORE detail once
# lsblk has shown you WHICH device to investigate further
pvs   # LVM physical volumes
vgs   # LVM volume groups
lvs   # LVM logical volumes
cat /proc/mdstat   # software RAID array status
```

---

## Finding Unmounted Partitions

```bash
# Show devices with an EMPTY mountpoint column (candidates for mounting)
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
# Any "part" type row with a blank MOUNTPOINT is currently unmounted

# Script-friendly version using JSON
lsblk -J -o NAME,TYPE,MOUNTPOINTS | jq -r '
  .blockdevices[] | select(.type=="part" and .mountpoints[0]==null) | .name
'

# Common next step: mount one of the found unmounted partitions
sudo mkdir -p /mnt/data
sudo mount /dev/sdb1 /mnt/data
lsblk /dev/sdb   # confirm it's now mounted
```

---

## Combining lsblk with Other Tools

```bash
# Cross-check lsblk's view against actual disk usage
lsblk -o NAME,SIZE,MOUNTPOINT
df -h

# Combine with blkid for full filesystem identification detail
lsblk -f
sudo blkid

# Combine with fdisk for detailed partition table info on a specific disk
lsblk /dev/sda
sudo fdisk -l /dev/sda

# Find which physical disk a specific mountpoint actually lives on
findmnt /                     # shows the SOURCE device for a mountpoint
lsblk $(findmnt -no SOURCE /)  # then inspect that device's full context in lsblk
```

---

## Real-World Recipes

```bash
# --- Preparing a New Disk for Use ---

lsblk                          # identify the new, unpartitioned disk (e.g., sdb)
sudo fdisk /dev/sdb             # create a partition
lsblk /dev/sdb                  # confirm the new partition (sdb1) appeared
sudo mkfs.ext4 /dev/sdb1
sudo mkdir -p /mnt/newdisk
sudo mount /dev/sdb1 /mnt/newdisk
lsblk -f /dev/sdb                # confirm filesystem type and mountpoint

# --- Diagnosing "No Space Left" Across a Complex Storage Stack ---

lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
df -h
# Cross-reference which lsblk device backs the full mountpoint reported by df

# --- Safely Identifying a USB Drive Before Wiping It ---

lsblk -o NAME,SIZE,MODEL,SERIAL,MOUNTPOINT
# ALWAYS verify size/model/serial match the INTENDED drive before
# running anything destructive like dd or mkfs — lsblk is the safest
# first step to avoid targeting the wrong device by mistake

# --- Auditing All Mounted Filesystems on a Server ---

lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT | grep -v "^$"

# --- Quick Health Check Script ---

#!/bin/bash
echo "=== Block Devices ==="
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
echo ""
echo "=== Unmounted Partitions ==="
lsblk -J -o NAME,TYPE,MOUNTPOINTS | jq -r '
  .blockdevices[].children[]? | select(.type=="part" and .mountpoints[0]==null) | .name
'
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
