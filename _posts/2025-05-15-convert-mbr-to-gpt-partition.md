---
title: "💽 How to convert MBR to GPT and Resize Linux Partitions"
date: 2025-05-15
description: "Convert a disk from MBR to GPT without data loss and grow the partition to use full disk space—safely and efficiently on Linux."
categories:
  - linux
tags:
  - gpt
  - mbr
  - partitioning
  - disks
  - gdisk
  - filesystem
  - sysadmin
---

When dealing with disks over 2TB in size, MBR partitioning isn't enough. Here's a step-by-step guide to convert an MBR-partitioned disk to GPT **without data loss**, and grow the partition to use the full disk space on a live Linux system.

## 🧾 1. Check Current Disk Layout

Use `fdisk -l` to get an overview:

```bash
sudo fdisk -l
```

Sample output shows `/dev/xvdd` as a 2.15 TiB disk with MBR and a single partition:

```
Disk /dev/xvdd: 2.15 TiB, 2362232012800 bytes, 4613734400 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x724f42e8

Device     Boot Start        End    Sectors Size Id Type
/dev/xvdd1       2048 4194303966 4194301919   2T 83 Linux
```
`Disklabel type: dos` indicates MBR.

## 🛠️ 2. Inspect the Partition Table with `gdisk`

```bash
sudo gdisk -l /dev/xvdd
```

Expected output:

```
GPT fdisk (gdisk) version 1.0.8

Partition table scan:
  MBR: MBR only
  BSD: not present
  APM: not present
  GPT: not present


***************************************************************
Found invalid GPT and valid MBR; converting MBR to GPT format
in memory.
***************************************************************

Disk /dev/xvdd: 4613734400 sectors, 2.1 TiB
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): 9F68DE6D-CACD-4BF5-A1BD-72395ADD1A5F
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 4613734366
Partitions will be aligned on 2048-sector boundaries
Total free space is 419432414 sectors (200.0 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048      4194303966   2.0 TiB     8300  Linux filesystem
```

At this point, no changes have been written to disk yet—only in memory.

## 💾 3. Write the New GPT Table

To commit the conversion:

```bash
sudo gdisk /dev/xvdd
```

You'll see the same warning about potential destructiveness. To proceed:

- Enter `w` to write changes
- Confirm with `Y`

```bash
GPT fdisk (gdisk) version 1.0.8

Partition table scan:
  MBR: MBR only
  BSD: not present
  APM: not present
  GPT: not present


***************************************************************
Found invalid GPT and valid MBR; converting MBR to GPT format
in memory. THIS OPERATION IS POTENTIALLY DESTRUCTIVE! Exit by
typing 'q' if you don't want to convert your MBR partitions
to GPT format!
***************************************************************


Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): Y
OK; writing new GUID partition table (GPT) to /dev/xvdd.
Warning: The kernel is still using the old partition table.
The new table will be used at the next reboot or after you
run partprobe(8) or kpartx(8)
The operation has completed successfully.
```

You'll be notified to run `partprobe` or reboot.

## 🔄 4. Inform the Kernel of the Changes

```bash
sudo partprobe -s /dev/xvdd
```

Now the new GPT layout is in use:

```bash
/dev/xvdd: gpt partitions 1
```

## 📈 5. Grow the Partition and Filesystem

### Unmount the disk:
```bash
sudo umount /mnt/backups
```

### Extend the partition:
```bash
sudo growpart /dev/xvdd 1
```

Output:
```
CHANGED: partition=1 start=2048 old: size=4194301919 end=4194303967 new: size=4613732319 end=4613734367
```

### Check the filesystem:
```bash
sudo e2fsck -f /dev/xvdd1
```

Output:
```
e2fsck 1.45.5 (07-Jan-2020)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/xvdd1: 3573/131072000 files (2.8% non-contiguous), 492321372/524287739 blocks
```

### Resize the filesystem:
```bash
sudo resize2fs /dev/xvdd1
```

Output:
```
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/xvdd1 is mounted on /mnt/backups; on-line resizing required
old_desc_blocks = 250, new_desc_blocks = 275
The filesystem on /dev/xvdd1 is now 576716539 (4k) blocks long.
```

### Mount the disk:
```bash
sudo mount /dev/xvdd1 /mnt/backups
```

## ✅ Done!

You've now:
- Converted the disk from MBR to GPT
- Extended the partition
- Resized the filesystem

The disk is now using the full 2.15 TiB space, with a clean GPT layout. Perfect for modern storage requirements.