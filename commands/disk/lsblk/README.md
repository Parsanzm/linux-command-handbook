# lsblk — The Complete Reference

> **List Block devices: see every disk, partition, and volume, in a clear tree structure**
> Part of util-linux, written by Milan Broz, first released around 2010.
> The modern, readable replacement for parsing raw `/proc/partitions` or `/sys/block` by hand.

---

## Table of Contents

- [What is lsblk?](#what-is-lsblk)
- [Where does lsblk live?](#where-does-lsblk-live)
- [How lsblk works internally](#how-lsblk-works-internally)
- [Syntax](#syntax)
- [Understanding the Default Output](#understanding-the-default-output)
- [Block Device Types (TYPE column)](#block-device-types-type-column)
- [All Key Options](#all-key-options)
- [Choosing Which Columns to Show](#choosing-which-columns-to-show)
- [Output Formats (Tree, List, JSON, Pairs)](#output-formats-tree-list-json-pairs)
- [Filtering and Specific Devices](#filtering-and-specific-devices)
- [lsblk and LVM / RAID / LUKS](#lsblk-and-lvm--raid--luks)
- [lsblk vs Other Disk Tools](#lsblk-vs-other-disk-tools)
- [Related Commands](#related-commands)

---

## What is lsblk?

`lsblk` (**list block devices**) displays information about all available or specified **block devices** — physical disks, partitions, optical drives, LVM volumes, RAID arrays, and LUKS-encrypted containers — in a clean, hierarchical tree that shows exactly how they relate to each other (which partitions belong to which disk, which logical volumes sit on top of which physical volumes, and so on).

```bash
lsblk
# NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
# sda      8:0    0  500G  0 disk
# ├─sda1   8:1    0  512M  0 part /boot/efi
# ├─sda2   8:2    0   32G  0 part [SWAP]
# └─sda3   8:3    0  467G  0 part /
```

**Why lsblk replaced older methods:** before lsblk, seeing this same information required combining `fdisk -l`, `cat /proc/partitions`, `mount`, and manual reading of `/sys/block/*/`. lsblk pulls all of it together — device hierarchy, size, type, and mount point — into one readable, tree-formatted view, reading directly from the kernel's `sysfs` rather than parsing older, less structured interfaces.

---

## Where does lsblk live?

```
/usr/bin/lsblk
/bin/lsblk
```

```bash
which lsblk
lsblk --version
# lsblk from util-linux 2.39.3
```

Part of **util-linux**, the standard package bundling core low-level Linux utilities (`fdisk`, `mount`, `dmesg`, `lsblk`, and many others). Present by default on virtually every mainstream Linux distribution; **not** available on macOS/BSD, which lack the same `sysfs` block-device model entirely (macOS users typically reach for `diskutil list` instead).

---

## How lsblk works internally

### Reading from sysfs, not parsing /proc

Unlike some older tools that scrape loosely-structured text files, `lsblk` gets its information primarily from **`/sys/block/`** and related `sysfs` entries — a structured, kernel-exposed filesystem interface describing every registered block device, its size, its relationships to other devices, and more.

```bash
ls /sys/block/
# sda  sdb  nvme0n1  loop0 ...

cat /sys/block/sda/size
# 976773168     ← size in 512-byte SECTORS, not bytes directly

cat /sys/block/sda/sda1/start
# 2048          ← starting sector of partition sda1 relative to sda
```

`lsblk` reads and interprets these structured values (converting sectors to human-readable sizes, resolving parent/child relationships via `/sys/block/*/slaves` and `/sys/block/*/holders`, and cross-referencing `/proc/mounts` for mount points) so you don't have to.

### Why the tree structure is more than cosmetic

The indentation and `├─`/`└─` connectors in lsblk's default output directly reflect the **actual block-device dependency hierarchy** the kernel tracks — a partition is a "child" of its disk, an LVM logical volume is a "child" of its volume group's physical volumes, a LUKS-decrypted mapping is a "child" of the encrypted partition beneath it, and so on. This hierarchy is genuinely useful for understanding complex storage stacks (RAID + LVM + LUKS combinations, for example), not just a display convenience.

---

## Syntax

```bash
lsblk [OPTIONS] [DEVICE...]
```

```bash
lsblk                          # show ALL block devices, default columns
lsblk /dev/sda                 # show only this device (and its children)
lsblk -f                        # include filesystem type and UUID
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT   # choose SPECIFIC columns to display
```

---

## Understanding the Default Output

```bash
lsblk
```

```
NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda       8:0    0   500G  0 disk
├─sda1    8:1    0   512M  0 part /boot/efi
├─sda2    8:2    0    32G  0 part [SWAP]
└─sda3    8:3    0   467G  0 part /
sdb       8:16   1    32G  0 disk
└─sdb1    8:17   1    32G  0 part /media/usb
nvme0n1 259:0    0     1T  0 disk
└─nvme0n1p1 259:1 0    1T  0 part /data
```

| Column | Meaning |
|--------|---------|
| **NAME** | Device name, with tree indentation showing parent/child relationships |
| **MAJ:MIN** | Major:minor device number — the kernel's internal identifier pair for this device |
| **RM** | Removable flag: `1` if removable media (USB drive, SD card), `0` if not |
| **SIZE** | Human-readable size of the device or partition |
| **RO** | Read-only flag: `1` if the device is read-only, `0` if writable |
| **TYPE** | Kind of block device: `disk`, `part`, `rom`, `lvm`, `crypt`, `raid1`, etc. |
| **MOUNTPOINTS** | Where the device is currently mounted, if anywhere (blank if unmounted; `[SWAP]` for swap partitions) |

---

## Block Device Types (TYPE column)

| Type | Meaning |
|------|---------|
| `disk` | A whole physical (or virtual) disk |
| `part` | A partition on a disk |
| `rom` | Read-only optical media (CD/DVD drive) |
| `loop` | A loop device (a regular file mounted as if it were a block device) |
| `lvm` | An LVM logical volume |
| `crypt` | A LUKS-decrypted (dm-crypt) mapped device |
| `raid0`/`raid1`/`raid5`/etc. | A software RAID array (via mdadm) |
| `mpath` | A multipath device (multiple physical paths to the same storage, common in SANs) |

```bash
lsblk -o NAME,TYPE,SIZE
# Filtering or focusing on TYPE is especially useful when a system
# has many layered devices (LVM on top of RAID on top of LUKS,
# for example) and you need to understand which layer is which
```

---

## All Key Options

| Option | Long | Description |
|--------|------|--------------|
| `-a` | `--all` | Include empty devices (hidden by default) |
| `-f` | `--fs` | Show filesystem type, label, UUID, and mount point |
| `-o COLS` | `--output=COLS` | Choose specific columns to display (comma-separated) |
| `-O` | `--output-all` | Show EVERY available column |
| `-l` | `--list` | List format instead of tree (flat, no indentation) |
| `-p` | `--paths` | Show full device PATHS (`/dev/sda1`) instead of bare names (`sda1`) |
| `-d` | `--nodeps` | Don't show partitions/children — only the top-level devices |
| `-J` | `--json` | Output as JSON, ideal for scripting/parsing |
| `-P` | `--pairs` | Output as `KEY="value"` pairs, one line per device — also script-friendly |
| `-b` | `--bytes` | Show sizes in raw BYTES instead of human-readable units |
| `-m` | `--perms` | Show device file owner, group, and mode |
| `-t` | `--topology` | Show low-level I/O topology details (block sizes, alignment) |
| `-e LIST` | `--exclude=LIST` | Exclude devices by major number (e.g., exclude loop devices) |
| `-I LIST` | `--include=LIST` | Include ONLY devices with these major numbers |
| `-s` | `--inverse` | Reverse the dependency tree (show a device's PARENTS instead of children) |

```bash
lsblk -a                        # include empty/zero-size devices too
lsblk -d                        # just disks, no partition details
lsblk -p                        # full /dev/ paths in output
lsblk -e 7                       # exclude loop devices (major number 7)
```

---

## Choosing Which Columns to Show

```bash
# See EVERY available column name
lsblk --help
# (lists all valid column names for -o at the bottom of help output)

# Show only what you care about
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT

# Include filesystem details
lsblk -o NAME,FSTYPE,LABEL,UUID,MOUNTPOINT

# Include ownership/permission info
lsblk -o NAME,SIZE,OWNER,GROUP,MODE

# See absolutely everything available at once
lsblk -O
```

---

## Output Formats (Tree, List, JSON, Pairs)

```bash
# Default: TREE format, showing parent/child hierarchy visually
lsblk

# LIST format: flat, no tree indentation — easier for simple line-by-line parsing
lsblk -l

# JSON format — ideal for scripts/programs that need STRUCTURED data
lsblk -J
# {
#    "blockdevices": [
#       {"name": "sda", "maj:min": "8:0", "rm": false, "size": "500G", "ro": false, "type": "disk", "mountpoints": [null],
#          "children": [
#             {"name": "sda1", "maj:min": "8:1", ... "mountpoints": ["/boot/efi"]}
#          ]
#       }
#    ]
# }

# Parse JSON output with jq
lsblk -J | jq '.blockdevices[] | select(.type=="disk") | .name'

# PAIRS format — KEY="value" per line, easy for shell script parsing
# without needing a JSON parser
lsblk -P
# NAME="sda" MAJ:MIN="8:0" RM="0" SIZE="500G" RO="0" TYPE="disk" MOUNTPOINTS=""
lsblk -P -o NAME,SIZE,TYPE | grep 'TYPE="disk"'
```

---

## Filtering and Specific Devices

```bash
# Show info for just ONE specific device
lsblk /dev/sda

# Show only top-level disks, no partition breakdown
lsblk -d
# NAME MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
# sda    8:0    0  500G  0 disk
# sdb    8:16   1   32G  0 disk

# Exclude specific device TYPES by major number (loop devices are
# major number 7, and commonly cluttered with many entries on
# systems using snap/flatpak/docker heavily)
lsblk -e 7

# Show only REMOVABLE devices (useful for finding a just-inserted USB drive)
lsblk -o NAME,SIZE,RM,MOUNTPOINT | awk '$3 == 1'

# Show devices WITHOUT a mountpoint (unmounted partitions — potential
# candidates for mounting, or leftover unused partitions)
lsblk -o NAME,SIZE,MOUNTPOINT | awk '$NF == ""'
```

---

## lsblk and LVM / RAID / LUKS

### Seeing through multiple storage layers at once

```bash
lsblk
# NAME               MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
# sda                  8:0    0   500G  0 disk
# └─sda1               8:1    0   500G  0 part
#   └─sda1_crypt      253:0    0   500G  0 crypt
#     └─vg0-root      253:1    0   450G  0 lvm   /
#     └─vg0-swap      253:2    0    50G  0 lvm   [SWAP]
# This single view shows the ENTIRE stack: a raw partition (sda1),
# LUKS-encrypted on top of it (sda1_crypt), with an LVM volume group
# (vg0) built on top of THAT, providing the actual root and swap
# logical volumes — untangling this by hand from /proc or /sys alone
# would require cross-referencing several different interfaces manually.
```

### Software RAID example

```bash
lsblk
# NAME     MAJ:MIN RM   SIZE RO TYPE   MOUNTPOINTS
# sda        8:0    0    2T  0 disk
# └─md0      9:0    0    4T  0 raid1
#   └─...                            (further LVM/filesystem layers, if any)
# sdb        8:16   0    2T  0 disk
# └─md0      9:0    0    4T  0 raid1  (SAME md0 shown again, as sdb's child too)
# Both physical disks correctly show md0 as a shared child, reflecting
# that the RAID1 array is built from BOTH of them together.
```

---

## lsblk vs Other Disk Tools

| Tool | Best for | Key difference from lsblk |
|------|----------|-------------------------------|
| `lsblk` | Quick hierarchical overview of ALL block devices and their relationships | Read-only, purely informational |
| `fdisk -l` | Detailed PARTITION TABLE information (partition types, exact sector boundaries) per disk | Lower-level, disk-by-disk detail; can also EDIT partition tables interactively |
| `df -h` | Disk USAGE (how full a MOUNTED filesystem is) | Reports usage of mounted filesystems, not the underlying block device structure itself |
| `blkid` | Filesystem UUIDs and types specifically | Narrower focus, purely on filesystem identification metadata |
| `parted -l` | Detailed partition table info, GPT/MBR aware, with editing capability | Similar detail level to fdisk, different interface/conventions |
| `mount` | What's CURRENTLY mounted and where, with mount OPTIONS | Shows active mounts specifically, not the full device hierarchy whether mounted or not |

```bash
lsblk                    # "what devices exist, and how do they relate?"
fdisk -l /dev/sda         # "what does sda's partition table actually look like?"
df -h                     # "how full is each mounted filesystem?"
blkid                     # "what filesystem/UUID does each partition have?"
```

---

## Related Commands

| Command | Relation |
|---------|----------|
| `fdisk` | Lower-level partition table viewing AND editing |
| `parted` | Similar to fdisk, with GPT support and scriptable editing |
| `df` | Shows disk USAGE of mounted filesystems, complementary to lsblk's structural view |
| `mount` | Shows/manages active mounts; lsblk shows the MOUNTPOINTS column reflecting mount's current state |
| `blkid` | Shows filesystem type/UUID/label for block devices |
| `du` | Shows disk usage of files/directories WITHIN a filesystem, unrelated to block-device structure itself |
| `mdadm` | Manages Linux software RAID arrays that appear as `raid*` type entries in lsblk |
| `cryptsetup` | Manages LUKS encrypted volumes that appear as `crypt` type entries in lsblk |
| `lvm` (`pvs`/`vgs`/`lvs`) | Manages LVM physical/volume groups/logical volumes shown as `lvm` type entries |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
