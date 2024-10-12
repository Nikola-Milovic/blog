---
date: 2024-10-09T17:43:38+02:00
draft: false
params:
  author: Nikola Milovic
title: How Linux works
---

#book #linux 

Linux is divided in
1. Hardware, actual devices, CPU, mouse, display
2. Kernel, manages the hardware and acts as the middleman between the user and the hardware
3. User space, programs that run outside of the privilege of the kernel, shells, browsers, games...

## Device files

device files are sometimes called device nodes, they are located in `/dev/*`.

`ls -la /dev`

If this character is b, c, p, or s, the file is a device. These letters stand for block, character, pipe, and socket

- **Block device**, programs access data from a block device in fixed chunks.
- **Character device**, character devices work with data streams. You can only read characters from or write characters to character devices. eg printers
- **Pipe device**, named pipes are like character devices, with another process at the other end of the I/O stream instead of a kernel driver.
- **Socket device**, sockets are special-purpose interfaces that are frequently used for interprocess communication. They’re often found outside of the /dev directory. Socket files represent Unix domain sockets

## Disks & FS

**Disks and Filesystems in Linux: A Simple Breakdown**

---

### **1. Disks in Linux**

- **Physical Storage Devices**:
  - Hard drives, SSDs, USB drives, and virtual disks.
  - Referred to as "disks" in Linux systems.

- **Device Representation**:
  - Linux represents disks as device files in `/dev/`.
    - **SATA/SCSI disks**: `/dev/sda`, `/dev/sdb`, etc.
    - **NVMe disks**: `/dev/nvme0n1`, `/dev/nvme1n1`, etc.
  - Partitions are numbered: `/dev/sda1`, `/dev/sda2`, etc.

- **Identifying Disks**:
  - Use commands like `lsblk`, `fdisk -l`, or `parted -l` to list disks and partitions.

---

### **2. Partitioning**

- **What is a Partition?**
  - A way to divide a disk into separate sections.
  - Each partition can function as an independent disk.

- **Types of Partitions**:
  - **Primary Partitions**:
    - Limited to four per disk in the MBR scheme.
    - Can be used to boot operating systems.
  - **Extended Partitions**:
    - A special primary partition acting as a container.
    - Holds multiple logical partitions.
  - **Logical Partitions**:
    - Created within an extended partition.
    - Allows more than four partitions on a disk.

- **Partition Tables**:
  - **MBR (Master Boot Record)**:
    - Traditional scheme.
    - Supports disks up to 2TB.
    - Allows up to four primary partitions.
  - **GPT (GUID Partition Table)**:
    - Modern scheme.
    - Supports large disks (over 2TB).
    - Allows numerous partitions.

---

### **3. Filesystems**

- **Definition**:
  - A filesystem determines how data is stored and retrieved.
  - It organizes files on a partition.

- **Common Linux Filesystems**:
  - **ext4**: Default for many Linux distributions; reliable and widely used.
  - **XFS**: High-performance, suitable for large files.
  - **Btrfs**: Supports advanced features like snapshots and pooling.
  - **FAT32/exFAT**: For compatibility with Windows and macOS.

- **Creating a Filesystem**:
  - Use `mkfs` commands:
    - `mkfs.ext4 /dev/sda1` to format a partition with ext4.
    - `mkfs.xfs /dev/sda2` for XFS.

---

### **4. Mounting Filesystems**

- **Mount Points**:
  - Directories where partitions are attached to the filesystem hierarchy.
  - Common mount points include `/`, `/home`, `/var`, `/mnt`, `/media`.

- **Mounting Process**:
  - **Manual Mounting**:
    - Create a mount point: `mkdir /mnt/mydisk`.
    - Mount the partition: `mount /dev/sda1 /mnt/mydisk`.
  - **Automatic Mounting**:
    - Edit `/etc/fstab` to include the partition for mounting at boot.

- **Unmounting**:
  - Use `umount /dev/sda1` to safely remove a mounted filesystem.

---

### **5. The Linux Filesystem Hierarchy**

- **Root Directory (`/`)**:
  - The top-level directory in Linux.
  - All other directories branch from here.

- **Standard Directories**:
  - **`/bin`**: Essential binaries (executables).
  - **`/etc`**: Configuration files.
  - **`/home`**: User home directories.
  - **`/var`**: Variable data like logs and databases.
  - **`/usr`**: User applications and utilities.
  - **`/tmp`**: Temporary files.

---

### **6. Key Concepts**

- **Everything is a File**:
  - Devices, directories, and even hardware are accessed like files.

- **Device Files**:
  - Found in `/dev/`.
  - Represent hardware devices (e.g., `/dev/sda` for a disk).

- **Swap Space**:
  - Acts as virtual memory when RAM is full.
  - Can be a dedicated partition or a swap file.

- **Permissions**:
  - Access to files and directories is controlled via permissions.
  - Important for security and system integrity.

---

### **7. Practical Steps**

- **Checking Disk Space**:
  - `df -h`: Shows disk space usage in human-readable form.
  - `du -h`: Displays disk usage of files and directories.

- **Partitioning a Disk**:
  - Use `fdisk /dev/sda` for MBR disks.
  - Use `gdisk /dev/sda` for GPT disks.
  - Use `parted` for a more user-friendly interface.

- **Formatting a Partition**:
  - Choose a filesystem based on needs.
  - Example: `mkfs.ext4 /dev/sda1`.

- **Mounting a Partition**:
  - `mount /dev/sda1 /mnt/data`.

- **Updating `/etc/fstab`**:
  - Ensure persistent mounting across reboots.
  - Example entry:
    ```
    /dev/sda1   /mnt/data   ext4   defaults   0 2
    ```

---

### **8. Summary**

- **Disks** are physical storage devices, represented as files in `/dev/`.
- **Partitions** divide disks into manageable sections.
- **Filesystems** determine how data is stored on partitions.
- **Mounting** integrates partitions into the system's directory tree.
- **Swap Space** extends RAM capacity using disk space.
- **Understanding the hierarchy** and management of disks and filesystems is crucial for system administration.

### BTRFS 

[[btrfs]]
