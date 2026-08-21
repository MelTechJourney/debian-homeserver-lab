# Linux Storage Commands

## Disk and partition overview

```bash
lsblk
```

Displays block devices, partitions and their mount points.

```bash
lsblk -f
```

Displays additional filesystem information, including:

- filesystem type;
- UUID;
- filesystem label;
- mount point.

```bash
df -h
```

Displays disk usage for mounted filesystems in a human-readable format.

```bash
df -h /srv/data
```

Displays disk usage specifically for the filesystem mounted on `/srv/data`.

---

## Identify a filesystem

```bash
sudo blkid /dev/sdb1
```

Displays filesystem information for `/dev/sdb1`, including its UUID and filesystem type.

Example structure:

```text
/dev/sdb1: UUID="<UUID>" BLOCK_SIZE="4096" TYPE="ext4"
```

The UUID can be used in `/etc/fstab` to identify the filesystem persistently.

---

## Partition a disk

Before modifying a disk, always verify the available block devices:

```bash
lsblk
```

In this lab:

```text
/dev/sda    Debian system disk
/dev/sdb    Dedicated data disk
```

Open the data disk with `fdisk`:

```bash
sudo fdisk /dev/sdb
```

Useful commands inside `fdisk`:

```text
p
```

Displays the current partition table.

```text
n
```

Creates a new partition.

```text
w
```

Writes the changes to disk and exits.

```text
q
```

Exits without saving changes.

> Always verify the target device before modifying a partition table. Running partitioning or formatting commands on the wrong disk can destroy existing data.

---

## Create an ext4 filesystem

After creating the partition, format it with ext4:

```bash
sudo mkfs.ext4 /dev/sdb1
```

Verify the result:

```bash
lsblk -f
```

Expected structure:

```text
sdb
└── sdb1    ext4
```

> `mkfs.ext4` creates a new filesystem. Existing data on the target partition will be lost.

---

## Create a mount point

Create the directory that will receive the filesystem:

```bash
sudo mkdir -p /srv/data
```

Verify that it exists:

```bash
ls -ld /srv/data
```

In this lab, `/srv/data` is the main persistent storage location for future services.

---

## Mount a filesystem manually

Mount the data partition:

```bash
sudo mount /dev/sdb1 /srv/data
```

Verify the mount:

```bash
lsblk
```

and:

```bash
df -h /srv/data
```

Expected relationship:

```text
/dev/sdb
└── /dev/sdb1
    └── ext4
        └── /srv/data
```

---

## Unmount a filesystem

Unmount the filesystem using its mount point:

```bash
sudo umount /srv/data
```

The filesystem should not be actively used by a process when attempting to unmount it.

Verify the result with:

```bash
lsblk
```

---

## Persistent mounting with fstab

Persistent filesystem mounts are configured in:

```text
/etc/fstab
```

Retrieve the filesystem UUID:

```bash
sudo blkid /dev/sdb1
```

The entry used for the data disk follows this structure:

```text
UUID=<FILESYSTEM-UUID> /srv/data ext4 defaults 0 2
```

The fields represent:

```text
UUID=<...>    Filesystem identifier
/srv/data     Mount point
ext4          Filesystem type
defaults      Standard mount options
0             Dump flag
2             Filesystem check order
```

---

## Reload systemd after modifying fstab

After changing `/etc/fstab`, reload the systemd configuration:

```bash
sudo systemctl daemon-reload
```

Then test the mount configuration:

```bash
sudo mount -a
```

If no error is returned, verify the result:

```bash
lsblk
```

and:

```bash
df -h /srv/data
```

> Always test `/etc/fstab` with `mount -a` before rebooting. An invalid mount configuration can cause problems during system startup.

---

## Check filesystem information

Display disks and filesystems:

```bash
lsblk -f
```

Display the filesystem UUID:

```bash
sudo blkid /dev/sdb1
```

Display mounted filesystem usage:

```bash
df -h
```

Display only the dedicated data filesystem:

```bash
df -h /srv/data
```

---

## Check a mount point

Verify that the directory exists:

```bash
ls -ld /srv/data
```

Check whether the filesystem is mounted:

```bash
lsblk
```

Check its disk usage:

```bash
df -h /srv/data
```

---

## Storage troubleshooting

If `/srv/data` is not working as expected, use the following sequence.

### 1. Check whether the disk exists

```bash
lsblk
```

Expected data disk:

```text
/dev/sdb
```

### 2. Check whether the partition exists

```bash
lsblk
```

Expected partition:

```text
/dev/sdb1
```

### 3. Check the filesystem

```bash
lsblk -f
```

Expected filesystem:

```text
ext4
```

### 4. Check the mount point

```bash
ls -ld /srv/data
```

If it does not exist:

```bash
sudo mkdir -p /srv/data
```

### 5. Check the filesystem UUID

```bash
sudo blkid /dev/sdb1
```

Compare the result with the entry in:

```text
/etc/fstab
```

### 6. Reload systemd if fstab was modified

```bash
sudo systemctl daemon-reload
```

### 7. Test fstab

```bash
sudo mount -a
```

### 8. Verify the final state

```bash
lsblk
df -h /srv/data
```

---

## Current Lab Storage Layout

```text
/dev/sda
└── Debian operating system
    └── /

/dev/sdb
└── /dev/sdb1
    └── ext4
        └── /srv/data
```

---

## Quick Reference

```bash
# List disks and partitions
lsblk

# List filesystems and UUIDs
lsblk -f

# Display mounted filesystem usage
df -h

# Identify the data filesystem
sudo blkid /dev/sdb1

# Partition the data disk
sudo fdisk /dev/sdb

# Create an ext4 filesystem
sudo mkfs.ext4 /dev/sdb1

# Create the mount point
sudo mkdir -p /srv/data

# Mount the filesystem
sudo mount /dev/sdb1 /srv/data

# Unmount the filesystem
sudo umount /srv/data

# Reload systemd after modifying fstab
sudo systemctl daemon-reload

# Test fstab
sudo mount -a

# Verify the data filesystem
lsblk
df -h /srv/data
```
