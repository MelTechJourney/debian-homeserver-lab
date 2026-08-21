# Storage Configuration

## Overview

This note documents the storage configuration of the `homeserver-lab` Debian server.

The objective was to separate the operating system from future application and service data by adding a dedicated virtual disk.

The final storage architecture is:

```text
/dev/sda
└── Debian operating system
    └── /

/dev/sdb
└── /dev/sdb1
    └── ext4
        └── /srv/data
```

The dedicated data disk provides a clean location for future persistent service data without mixing it directly with the operating system filesystem.

---

## 1. Initial Storage Layout

The Debian virtual machine was initially installed with a single 30 GB virtual disk.

The detected block devices can be inspected using:

```bash
lsblk
```

The main system disk appears as:

```text
/dev/sda
```

It contains the Debian installation, including the root filesystem and swap space.

A simplified representation is:

```text
/dev/sda
├── system partition
│   └── /
└── swap
    └── [SWAP]
```

The operating system disk is intentionally left dedicated to Debian.

---

## 2. Adding a Dedicated Data Disk

A second virtual disk was added to the virtual machine through Oracle VirtualBox.

The new disk has a capacity of:

```text
20 GB
```

The resulting virtual storage configuration therefore contains:

| Device | Size | Purpose |
|---|---:|---|
| `/dev/sda` | 30 GB | Debian operating system |
| `/dev/sdb` | 20 GB | Persistent application and service data |

The VirtualBox storage configuration is documented here:

![Dedicated virtual data disk](../screenshots/06-ajout-disque-virtuel-20go.png)

Separating data from the operating system provides several advantages:

- clearer storage organization;
- easier identification of service data;
- simpler future backup strategies;
- reduced coupling between system files and application data;
- practical experience managing additional Linux block devices.

---

## 3. Detecting the New Disk

After attaching the disk and starting the VM, the available block devices were checked:

```bash
lsblk
```

The new disk appeared as:

```text
sdb    20G    disk
```

At this point, the disk did not yet contain a usable partition.

The detection of the second disk is documented here:

![Second disk detection](../screenshots/07-detection-second-disque.png)

The important distinction is:

```text
/dev/sdb     physical or virtual block device
/dev/sdb1    partition created on that device
```

Before creating a filesystem, the disk first needed to be partitioned.

---

## 4. Partitioning the Disk

The second disk was partitioned using `fdisk`.

The command used was:

```bash
sudo fdisk /dev/sdb
```

`fdisk` provides an interactive interface for creating and modifying partition tables.

Because this disk was dedicated entirely to homelab data, a single partition using the available capacity was sufficient.

A new partition was created and the partition table was written to disk.

The resulting device became:

```text
/dev/sdb1
```

The partitioning process is documented here:

![Disk partitioning](../screenshots/08-partitionnement-disque-sdb.png)

After leaving `fdisk`, the result was checked with:

```bash
lsblk
```

The expected structure was:

```text
sdb
└── sdb1
```

This confirmed that the operating system recognized the newly created partition.

---

## 5. Important fdisk Precaution

Partitioning tools directly modify disk structures.

Before running:

```bash
sudo fdisk /dev/sdb
```

the target device should always be verified carefully.

Useful commands include:

```bash
lsblk
```

and:

```bash
lsblk -f
```

Selecting the wrong disk could modify or destroy an existing partition table.

In this lab:

```text
/dev/sda = operating system
/dev/sdb = new data disk
```

Therefore, all partitioning operations were intentionally performed on:

```text
/dev/sdb
```

and not `/dev/sda`.

---

## 6. Creating the ext4 Filesystem

After creating `/dev/sdb1`, the partition still required a filesystem.

The ext4 filesystem was selected.

The partition was formatted using:

```bash
sudo mkfs.ext4 /dev/sdb1
```

The operation created a new ext4 filesystem and assigned it a filesystem UUID.

The result was checked with:

```bash
lsblk -f
```

The filesystem creation is documented here:

![ext4 filesystem creation](../screenshots/09-formatage-ext4-sdb1.png)

The relevant result now showed:

```text
sdb
└── sdb1    ext4
```

At this stage, the partition contained a valid filesystem but was not yet mounted into the Linux directory tree.

---

## 7. Why ext4?

The filesystem selected for the data disk is:

```text
ext4
```

For this lab, ext4 provides a straightforward Linux-native filesystem suitable for general server storage.

It supports standard Linux filesystem features including:

- Unix ownership;
- user and group permissions;
- journaling;
- large files and filesystems;
- standard Linux administration tools.

It also works naturally with the permission model later configured on `/srv/data`.

---

## 8. Creating the Mount Point

Linux filesystems are accessed through the directory tree.

A dedicated mount point was chosen:

```text
/srv/data
```

The intended directory was created using:

```bash
sudo mkdir -p /srv/data
```

The `-p` option creates any missing parent directories and does not report an error if the directory already exists.

The directory can be checked with:

```bash
ls -ld /srv/data
```

Before the custom permissions were configured, it was owned by root.

---

## 9. Mounting the Filesystem

The new filesystem was manually mounted using:

```bash
sudo mount /dev/sdb1 /srv/data
```

The relationship becomes:

```text
/dev/sdb
    │
    └── /dev/sdb1
            │
            └── ext4
                 │
                 └── /srv/data
```

The mount was verified using:

```bash
lsblk
```

The output showed `/srv/data` as the mount point for `/dev/sdb1`.

Filesystem usage was also checked with:

```bash
df -h /srv/data
```

The successful mounting process is documented here:

![Data disk mounting](../screenshots/10-montage-disque-srv-data.png)

---

## 10. Troubleshooting: Missing Mount Point

An error was encountered during the first mount attempt.

The command:

```bash
sudo mount /dev/sdb1 /srv/data
```

returned an error indicating that the mount point did not exist.

The directory was checked and recreated:

```bash
sudo mkdir -p /srv/data
```

It was then verified with:

```bash
ls -ld /srv/data
```

The mount command was executed again:

```bash
sudo mount /dev/sdb1 /srv/data
```

This time the operation succeeded.

Verification:

```bash
lsblk
df -h /srv/data
```

The important troubleshooting sequence was therefore:

```text
Mount fails
    │
    ▼
Check mount point
    │
    ▼
Create /srv/data
    │
    ▼
Retry mount
    │
    ▼
Verify with lsblk and df
```

This illustrates an important distinction:

```text
/dev/sdb1 = filesystem device

/srv/data = directory where the filesystem is attached
```

Both must exist before the filesystem can be mounted successfully.

---

## 11. Manual Mount Limitation

At this stage, the filesystem could be mounted manually:

```bash
sudo mount /dev/sdb1 /srv/data
```

However, a manual mount alone is not sufficient for a permanent server configuration.

After a reboot, Linux needs configuration telling it that the filesystem should automatically be mounted again.

This is handled through:

```text
/etc/fstab
```

---

## 12. Retrieving the Filesystem UUID

Instead of relying directly on the device name `/dev/sdb1`, the filesystem was identified using its UUID.

The UUID was retrieved with:

```bash
sudo blkid /dev/sdb1
```

The output contains information similar to:

```text
/dev/sdb1: UUID="<UUID>" BLOCK_SIZE="4096" TYPE="ext4"
```

The actual UUID uniquely identifies the filesystem.

Using the UUID in `/etc/fstab` is preferable to relying only on a device path such as:

```text
/dev/sdb1
```

because Linux device names may depend on device detection order.

---

## 13. Configuring /etc/fstab

Persistent filesystem mounts are configured in:

```text
/etc/fstab
```

The data filesystem was added using the following structure:

```text
UUID=<FILESYSTEM-UUID> /srv/data ext4 defaults 0 2
```

The fields represent:

| Field | Purpose |
|---|---|
| `UUID=<...>` | Filesystem identifier |
| `/srv/data` | Mount point |
| `ext4` | Filesystem type |
| `defaults` | Standard mount options |
| `0` | Dump backup flag |
| `2` | Filesystem check order |

The root filesystem normally uses filesystem check priority `1`.

A secondary filesystem such as this data disk can use:

```text
2
```

---

## 14. systemd Reload Warning

After modifying `/etc/fstab`, the configuration was tested using:

```bash
sudo mount -a
```

A warning appeared indicating that `/etc/fstab` had been modified while systemd was still using the previous version.

The message recommended:

```bash
systemctl daemon-reload
```

The systemd configuration was therefore reloaded using:

```bash
sudo systemctl daemon-reload
```

The mount test was then executed again:

```bash
sudo mount -a
```

No mount error was returned.

This sequence is documented here:

![Persistent mount configuration](../screenshots/11-montage-persistant-fstab.png)

---

## 15. Testing /etc/fstab Safely

Testing `/etc/fstab` before rebooting is important.

The command used was:

```bash
sudo mount -a
```

This attempts to mount filesystems configured in `/etc/fstab` that are not already mounted as appropriate.

If the configuration contains an invalid UUID, filesystem type, option or mount point, the error can be detected while the current system is still running.

The verification sequence used in the lab was:

```bash
sudo systemctl daemon-reload
sudo mount -a
lsblk
```

The expected result is:

```text
sdb
└── sdb1    /srv/data
```

This provides confidence that the persistent mount configuration is valid before restarting the VM.

---

## 16. Reboot Verification

After validating `/etc/fstab`, the server was rebooted.

After logging back in, the block devices were checked again:

```bash
lsblk
```

The expected result remained:

```text
sdb
└── sdb1    /srv/data
```

This confirmed that the filesystem was automatically mounted during boot.

The persistent storage configuration was therefore operational.

---

## 17. Checking Mounted Filesystems

Several commands can be used to inspect storage.

### Block device overview

```bash
lsblk
```

Useful for identifying:

- disks;
- partitions;
- mount points.

### Filesystem information

```bash
lsblk -f
```

Useful for identifying:

- filesystem types;
- UUIDs;
- mount points.

### Disk usage

```bash
df -h
```

The `-h` option displays sizes in a human-readable format.

To inspect only the dedicated data filesystem:

```bash
df -h /srv/data
```

### Filesystem UUID

```bash
sudo blkid /dev/sdb1
```

These commands provide different views of the same storage configuration.

---

## 18. Current Storage Architecture

The final storage layout at this stage is:

```text
HOMESERVER-LAB
│
├── /dev/sda — 30 GB
│   │
│   └── Debian 12
│       └── /
│
└── /dev/sdb — 20 GB
    │
    └── /dev/sdb1
        │
        └── ext4
            │
            └── /srv/data
```

The operating system and future service data are now logically separated.

---

## 19. Purpose of /srv/data

The directory:

```text
/srv/data
```

is intended to become the persistent data location for services deployed later in the project.

The objective is to avoid scattering service data throughout the root filesystem.

Future services can use subdirectories such as:

```text
/srv/data/
├── service-a/
├── service-b/
└── backups/
```

The exact structure will be introduced as services are deployed.

At this stage, `/srv/data` only establishes the storage foundation.

---

## 20. Permissions After Mounting

After the filesystem was mounted, the root of the new filesystem was initially controlled by the root account.

A normal user therefore could not automatically write to:

```text
/srv/data
```

This is expected.

Mounting a filesystem and configuring access permissions are two separate operations.

The storage layer answers:

```text
Where is the data stored?
```

The permission layer answers:

```text
Who can access and modify it?
```

The permission configuration is documented separately in:

```text
notes/04-permissions.md
```

---

## 21. Troubleshooting Workflow

When a filesystem does not appear where expected, the following sequence provides a useful diagnostic workflow.

### Step 1 — Check the disk

```bash
lsblk
```

Confirm that the expected disk exists.

### Step 2 — Check the filesystem

```bash
lsblk -f
```

Confirm that the partition has the expected filesystem.

### Step 3 — Check the mount point

```bash
ls -ld /srv/data
```

Confirm that the target directory exists.

### Step 4 — Check the UUID

```bash
sudo blkid /dev/sdb1
```

Compare it with the UUID configured in:

```text
/etc/fstab
```

### Step 5 — Reload systemd if fstab changed

```bash
sudo systemctl daemon-reload
```

### Step 6 — Test the configuration

```bash
sudo mount -a
```

### Step 7 — Verify the result

```bash
lsblk
df -h /srv/data
```

This provides a structured way to troubleshoot most basic mount configuration problems.

---

## 22. Result

At the end of this stage:

```text
20 GB VirtualBox disk
        │
        ▼
     /dev/sdb
        │
        ▼
     /dev/sdb1
        │
        ▼
       ext4
        │
        ▼
    /srv/data
        │
        ▼
Persistent through /etc/fstab
```

The server now has a dedicated and persistent storage location independent of the operating system disk.

---

## Commands Summary

```bash
# Display block devices
lsblk

# Display block devices and filesystem information
lsblk -f

# Partition the second disk
sudo fdisk /dev/sdb

# Create an ext4 filesystem
sudo mkfs.ext4 /dev/sdb1

# Create the mount point
sudo mkdir -p /srv/data

# Inspect the mount point
ls -ld /srv/data

# Mount the filesystem manually
sudo mount /dev/sdb1 /srv/data

# Check filesystem usage
df -h /srv/data

# Retrieve filesystem UUID
sudo blkid /dev/sdb1

# Reload systemd configuration after modifying fstab
sudo systemctl daemon-reload

# Test filesystems configured in fstab
sudo mount -a

# Verify the final mount
lsblk
df -h /srv/data
```

---

## Key Takeaways

- `/dev/sda` contains the Debian operating system.
- A separate 20 GB virtual disk was added as `/dev/sdb`.
- `/dev/sdb1` was created as the data partition.
- The partition was formatted with ext4.
- `/srv/data` was created as the dedicated mount point.
- A filesystem cannot be mounted on a nonexistent mount-point directory.
- `lsblk`, `lsblk -f`, `blkid` and `df` provide complementary storage information.
- The filesystem UUID is used in `/etc/fstab` for persistent mounting.
- `/etc/fstab` should be tested with `mount -a` before rebooting.
- systemd may require `systemctl daemon-reload` after `/etc/fstab` is modified.
- A successful reboot confirmed that `/srv/data` mounts automatically.
- Mount configuration and filesystem permissions are separate concerns.
