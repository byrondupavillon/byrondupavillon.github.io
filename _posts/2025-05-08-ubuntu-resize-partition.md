---
title: "How to Safely Resize Linux Partitions Without Losing Data"
date: 2025-05-08
categories:
  - linux
tags:
  - disk management
  - partitioning
  - resizing
  - ext4
  - disaster recovery
---

This guide explains how to **extend an existing partition** on a Linux system. In the example, we’ll extend the disk `/dev/vdb` from 128.8 GB to 268.4 GB.

> ⚠️ **IMPORTANT:** Always back up your data before performing disk operations. While these steps are generally safe, there is still a risk of data loss.

---

## 🧰 Prerequisites

- Root access to the server
- A backup of your data
- Familiarity with the Linux shell
- Ensure the file system on the disk is **ext4**
- Tools: `fdisk`, `e2fsck`, `resize2fs`, and optionally `tmux` if working remotely

---

## 1. 🔍 Check Current Disk Layout

Run this command to identify the current disk and partition structure:

```bash
sudo fdisk -l
```

Example output:

```bash
Disk /dev/vdb: 128.8 GB, 128849018880 bytes
16 heads, 15 sectors/track, 1048576 cylinders, total 251658240 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0xda3b18b2

   Device Boot      Start         End      Blocks   Id  System
/dev/vdb1            2048   251658239   125828096   83  Linux
```

---

## 2. 🧱 Extend Disk from Hosting Side

Resize the virtual disk in your cloud provider's panel or ask your hosting support team to add space.

After resizing, **reboot the server** so the kernel detects the new size:

```bash
sudo reboot
```

---

## 3. 🔁 Verify Disk Resize

Run again:

```bash
sudo fdisk -l
```

Expected result (showing increased size):

```bash
Disk /dev/vdb: 268.4 GB, 268435456000 bytes
16 heads, 15 sectors/track, 2184533 cylinders, total 524288000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0xda3b18b2

   Device Boot      Start         End      Blocks   Id  System
/dev/vdb1            2048   251658239   125828096   83  Linux
```

> 📝 Note: The partition has not grown yet — only the disk. You must now resize the partition and file system.

---

## 4. ⛔️ Unmount the Partition

Before resizing, unmount the partition:

> ⚠️ **IMPORTANT:** Make sure no services (e.g. MySQL) are using the volume..

```bash
sudo umount /mnt/data
```

Replace `/mnt/data` with the actual mount point.

---

## 5. 🧱 Resize the Partition Using `fdisk`

We'll delete and recreate the partition to use the new disk space.

### Launch `fdisk`:

```bash
sudo fdisk /dev/vdb
```

### Inside `fdisk`, run these commands:

- `p` — print partition table
- `d` — delete partition 1
- `n` — create new primary partition  
    - Accept all default values (it will use full disk)
- `w` — write and save changes

Example interaction:

```bash
Command (m for help): p
...

Command (m for help): d
Selected partition 1

Command (m for help): n
Partition type: p
Partition number: 1
First sector: 2048
Last sector: [accept default]

Command (m for help): w
```

If prompted to remove ext4 signature: **say No**.

---

## 6. 🔍 Run Filesystem Check

Use `e2fsck` to verify the integrity of the partition:

> 📝 Note: You may want to run the below in a tmux session as it takes awhile.

```bash
sudo e2fsck -f -C 0 /dev/vdb1
```

- `-f` forces a check even if the file system seems clean.
- `-C 0` writes progress information to the specified file descriptor (0 = standard input).

Expected result

```bash
e2fsck 1.42.9 (4-Feb-2014)
Pass 1: Checking inodes, blocks, and sizes # This step may take awhile
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/data: 950/7864320 files (38.8% non-contiguous), 27506699/31457024 blocks
```

If you see errors like:

```
ext2fs_open2: Bad magic number in super-block
e2fsck: Superblock invalid, trying backup blocks...
/dev/xvdg1: recovering journal
e2fsck: unable to set superblock flags on /dev/xvdg1
/dev/xvdg1: ***** FILE SYSTEM WAS MODIFIED *****
/dev/xvdg1: ********** WARNING: Filesystem still has errors **********
```

Run the command again or verify you're using the correct device.

---

## 7. 📏 Resize the Filesystem

Now resize the file system to fill the new partition:

```bash
sudo resize2fs /dev/vdb1
```

Output:

```
resize2fs 1.42.9 (4-Feb-2014)
Resizing the filesystem on /dev/vdb1 to 65273856 (4k) blocks.
The filesystem on /dev/vdb1 is now 65273856 blocks long.
```

---

## 8. 🔁 Remount the Partition

Remount the partition and verify it works:

```bash
sudo mount /dev/vdb1 /mnt/data
```

Confirm available space:

```bash
df -h
```

You should now see the increased size reflected.

---

## ✅ Summary

| Step | Description |
|------|-------------|
| 1    | Confirm disk layout with `fdisk -l` |
| 2    | Resize the disk at the provider level |
| 3    | Reboot and verify disk is larger |
| 4    | Unmount the partition |
| 5    | Recreate partition using `fdisk` |
| 6    | Run `e2fsck` to check the filesystem |
| 7    | Resize the filesystem using `resize2fs` |
| 8    | Remount and verify space |

---

## 🧯 Troubleshooting Tips

- Always verify you're modifying the correct disk and partition.
- If `e2fsck` fails, do not proceed with `resize2fs`.
- Use `lsblk` or `blkid` to cross-check device names.
