# lsblk — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Output & Columns](#output--columns)
- [Options](#options)
- [LVM, RAID, LUKS](#lvm-raid-luks)
- [lsblk vs Other Tools](#lsblk-vs-other-tools)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does lsblk do, and where does it get its information from?**
> `lsblk` (**list block devices**) displays a hierarchical view of all block devices — disks, partitions, LVM volumes, RAID arrays, LUKS containers — showing their sizes, types, and mount points. It reads this information primarily from **`/sys/block/`** (sysfs), a structured kernel interface, rather than parsing older, less structured sources like raw `/proc/partitions` text.

---

**Q2 🔥 What does the tree structure (indentation, ├─/└─ connectors) in lsblk's output actually represent?**
> It reflects the **real block-device dependency hierarchy** tracked by the kernel — a partition is shown as a child of its parent disk, an LVM logical volume as a child of the physical volume(s) beneath it, a LUKS-decrypted mapping as a child of the encrypted partition below it, and so on. It's not just a cosmetic indentation; it's derived from actual parent/child relationships (`/sys/block/*/slaves` and `/holders`).

---

**Q3. Is lsblk available on macOS?**
> No. lsblk relies on the Linux-specific `sysfs`/`/sys/block` interface, which doesn't exist on macOS or BSD systems. macOS users typically use `diskutil list` for equivalent functionality instead.

---

## Output & Columns

**Q4 🔥 What do the RM and RO columns in lsblk's default output mean?**
> `RM` indicates whether the device is **removable** media (1 for USB drives, SD cards, etc.; 0 for fixed disks). `RO` indicates whether the device is **read-only** (1 if read-only, 0 if writable).

---

**Q5. What's the difference between lsblk's default output and running it with the `-f` flag?**
> Default output focuses on device structure (name, size, type, mountpoint). `-f` (`--fs`) additionally shows **filesystem-level** information: filesystem type, filesystem version, label, and UUID for each device — details relevant to identifying and mounting filesystems specifically.

---

**Q6 🔥 Does lsblk tell you how FULL a filesystem is?**
> No. lsblk shows a partition or device's total **size**, not how much of that space is used or free at the filesystem level — that's a different question answered by `df -h`, which operates at the filesystem-usage layer rather than the block-device-structure layer lsblk focuses on.

---

## Options

**Q7 🔥 How would you show only top-level disks, without any partition breakdown?**
> ```bash
> lsblk -d
> ```
> `-d` (`--nodeps`) suppresses child devices (partitions, logical volumes, etc.), showing only the top-level block devices themselves.

---

**Q8. How do you exclude loop devices from lsblk's output, and why might you want to?**
> ```bash
> lsblk -e 7
> ```
> `-e` (`--exclude`) filters out devices by major device number; loop devices use major number 7. Systems with many snap packages, Docker images, or mounted ISO files can accumulate dozens of loop device entries that clutter the output and bury the physical disks you're actually interested in.

---

**Q9 🔥 How would you get lsblk's output in a format suitable for parsing by a script?**
> Two built-in options: `-J` (`--json`) for structured JSON output, ideal when a JSON parser like `jq` is available; or `-P` (`--pairs`) for `KEY="value"` formatted lines, useful for simpler shell parsing without a dedicated JSON tool.

---

**Q10. How do you display raw byte counts instead of human-readable sizes (like "500G")?**
> ```bash
> lsblk -b
> ```
> `-b` (`--bytes`) shows exact sizes in bytes rather than rounded, human-readable units — useful when precise size comparison matters (e.g., verifying two devices are genuinely identical in capacity for a RAID setup).

---

## LVM, RAID, LUKS

**Q11 🔥 How does lsblk represent a LUKS-encrypted partition with an LVM volume group on top of it?**
> As a nested tree: the raw partition (e.g., `sda1`) appears as the parent of its LUKS-decrypted mapping (typically shown with a type of `crypt`, often named something like `sda1_crypt`), which in turn is the parent of the LVM logical volumes (type `lvm`) built on top of it — accurately reflecting the full physical-partition → encryption → logical-volume storage stack in one view.

---

**Q12. How would a software RAID array appear in lsblk's output, and what would you check to confirm which physical disks back it?**
> The RAID array itself appears with a type like `raid1` (or `raid0`, `raid5`, etc., depending on the RAID level), and it would be shown as a **child** of each physical disk that's a member of the array — the same array device name would appropriately appear listed as a child under multiple different parent disks, reflecting that the array is built from all of them together. Cross-referencing `cat /proc/mdstat` provides additional detail specific to the RAID array's status and exact member composition.

---

## lsblk vs Other Tools

**Q13 🔥 What's the difference between lsblk and df, and when would you use each?**
> `lsblk` shows the **structural hierarchy** of block devices — what disks and partitions exist and how they relate — regardless of whether they're mounted or how full they are. `df` shows **usage information** for currently mounted filesystems — how much space is used/available. Use `lsblk` to understand "what storage exists and how is it organized"; use `df` to understand "how full is what's currently mounted."

---

**Q14. When would you reach for fdisk -l instead of lsblk?**
> When you need detailed **partition table** information — exact partition types, precise sector boundaries and offsets, or when you need to actually **edit** the partition table (fdisk supports interactive editing; lsblk is purely read-only/informational).

---

## Scenario-Based

**Q15 🔥 A server has dozens of loop device entries cluttering lsblk's output, making it hard to find the actual physical disks. What's the cause, and how do you fix the display?**
> This commonly happens on systems using **snap packages** heavily (each installed snap creates a loop-mounted squashfs image) or systems with many mounted container images/ISO files — each shows up as a separate `loop` type entry. The fix is excluding loop devices by their major number: `lsblk -e 7`, or using `lsblk -d` to see only top-level disks if partition-level loop detail isn't needed at all.

---

**Q16. Someone runs `lsblk -o NAME,SIZE,MOUNTPOINT` and concludes a nearly-full disk still has plenty of free space, based on the SIZE column showing "500G." What's wrong with this conclusion?**
> The `SIZE` column shows the partition's **total capacity**, not how much of it is actually used or free — lsblk operates at the block-device structural level and has no concept of filesystem-level usage at all. To determine actual free space, `df -h` should be used instead, which reports genuine used/available space for mounted filesystems.

---

**Q17 🔥 A script hardcodes `/dev/sdb1` in `/etc/fstab` to always mount a specific external drive, but after a server reboot, an entirely different device ends up mounted at that mountpoint. What went wrong, and what's the correct fix?**
> Device names like `/dev/sdb` are assigned based on detection **order** at boot time, not tied permanently to specific physical hardware — if the detection order changes (e.g., due to a new device being added, removed, or a change in how the BIOS/kernel enumerates controllers), what was `sdb` before might become `sda` or `sdc` after the next boot, silently pointing `/etc/fstab` at the wrong physical device. The fix is referencing the partition by its stable UUID or label instead: run `lsblk -f` (or `blkid`) to find the actual UUID, and use that in `/etc/fstab` rather than the potentially-shifting device name.

---

**Q18. A colleague runs `lsblk` inside a Docker container and is surprised to see the host machine's entire disk layout, including disks that have nothing to do with the containerized application. Is this expected, and why does it happen?**
> Yes, this is expected. Containers typically share the **host's kernel** and don't have their own independent virtualized block-device layer (unlike a full virtual machine) — `lsblk`, which reads from the kernel's `sysfs`, therefore reports the same block device information the host kernel tracks, regardless of which container is asking. This is a normal consequence of container architecture, not a bug in lsblk, though it's worth being aware of from a security/information-exposure perspective in multi-tenant container environments.

---

**Q19 🔥 Two drives both show as "500G" under lsblk's default output, but a RAID setup script fails, claiming the two devices are actually different sizes. How would you investigate, and what's the likely explanation?**
> Human-readable size formatting rounds values for display, so two devices with genuinely different exact byte counts (e.g., 500.11 GB vs 499.98 GB — common when drives from different manufacturers claim the "same" advertised capacity) can both display as "500G" despite not being byte-for-byte identical, which matters for certain RAID configurations expecting exactly matched member sizes. Running `lsblk -b -o NAME,SIZE` reveals the exact byte counts, confirming whether a genuine size mismatch exists.

---

**Q20. After physically attaching a new drive to a running server, it doesn't appear in `lsblk`'s output at all. What steps would you take to make it visible without rebooting?**
> Trigger the kernel to rescan for newly attached storage: running `sudo partprobe` often suffices for typical hot-plugged drives, and for SCSI/iSCSI-attached storage specifically, manually triggering a host rescan (`echo "- - -" | sudo tee /sys/class/scsi_host/hostN/scan`) may be needed. USB devices are usually detected automatically via udev, so a missing USB drive might instead indicate a hardware/cabling issue rather than a rescan problem — checking `dmesg` for related kernel messages helps distinguish between these cases.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
