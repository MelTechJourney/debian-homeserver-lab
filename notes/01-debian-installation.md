# Debian 12 Installation

## Overview

This note documents the initial installation and configuration of the Debian virtual machine used for the `debian-homeserver-lab` project.

The objective of this first stage is to establish a minimal Linux server that can later host additional services while remaining simple to administer and troubleshoot.

The server runs as a virtual machine in Oracle VirtualBox.

---

## 1. Virtual Machine

The server was created as a dedicated virtual machine using Oracle VirtualBox.

The VM is named:

```text
HOMESERVER-LAB
```

The guest operating system is:

```text
Debian GNU/Linux 12
```

The server uses two virtual disks:

| Disk | Size | Purpose |
|---|---:|---|
| `/dev/sda` | 30 GB | Debian operating system |
| `/dev/sdb` | 20 GB | Dedicated application and service data |

The second disk was added after the initial Debian installation and is documented separately in:

```text
notes/03-storage-configuration.md
```

### Why use a virtual machine?

Using a virtual machine provides an isolated environment in which Linux administration tasks can be practiced without directly modifying the host operating system.

It also makes it possible to:

- modify virtual hardware;
- add additional disks;
- change network modes;
- experiment with system configuration;
- rebuild the environment if necessary;
- isolate the server from the Windows host.

The VM configuration used for the lab is documented in the following screenshot:

![VirtualBox VM configuration](../screenshots/01-virtualbox-vm-configuration.png)

---

## 2. Debian Installation

Debian 12 was selected as the operating system for the server.

The installation was intentionally kept relatively minimal.

The objective is to use the machine primarily as a server rather than as a desktop workstation.

A graphical desktop environment is therefore unnecessary for the final server configuration.

This reduces the number of installed packages and keeps the environment focused on command-line administration.

---

## 3. Software Selection

During the Debian installation, the software selection stage determines which predefined groups of packages are installed.

For this lab, the goal is to retain the standard system utilities required for normal Debian administration without installing unnecessary desktop components.

The installation selection is documented here:

![Debian software selection](../screenshots/02-debian-software-selection.png)

The server is subsequently administered primarily through:

```text
TTY console
SSH
```

This is closer to the administration model commonly used for Linux servers.

---

## 4. Hostname

The server hostname is:

```text
homeserver-lab
```

The hostname can be checked using:

```bash
hostname
```

Expected output:

```text
homeserver-lab
```

A clear hostname is useful because it identifies the machine in:

- the shell prompt;
- system logs;
- SSH sessions;
- network configuration;
- monitoring systems;
- future infrastructure documentation.

Using a descriptive hostname also becomes increasingly important when multiple servers are introduced into a homelab.

---

## 5. Initial User

The primary administrative user created for the server is:

```text
mera
```

The current user can be verified using:

```bash
whoami
```

Expected output:

```text
mera
```

The server should normally be administered from this non-root account.

Administrative commands requiring elevated privileges are performed through `sudo`.

The installation and configuration of `sudo` are documented separately in:

```text
notes/02-linux-administration.md
```

---

## 6. Initial System Verification

After the installation completed and Debian booted successfully, several commands were used to verify the state of the system.

### Check the hostname

```bash
hostname
```

This confirms the identity of the server.

### Check the current user

```bash
whoami
```

This confirms which account is currently being used.

### Check network interfaces

```bash
ip -br a
```

The `-br` option displays a concise representation of the available network interfaces.

Typical output contains:

```text
lo
enp0s3
```

`lo` is the loopback interface.

`enp0s3` is the primary virtual network interface provided to the Debian VM.

### Check block devices

```bash
lsblk
```

This displays disks, partitions and their mount points.

Immediately after the initial installation, the main disk is:

```text
/dev/sda
```

The system partition is mounted on:

```text
/
```

Swap space is also configured on the system disk.

The initial verification is documented here:

![Initial system verification](../screenshots/03-verification-systeme-initial.png)

---

## 7. Understanding the Initial Storage Layout

The initial Debian installation resides on the first virtual disk:

```text
/dev/sda
```

The root filesystem is mounted at:

```text
/
```

The initial layout can be inspected with:

```bash
lsblk
```

A simplified representation is:

```text
/dev/sda
├── system partition
│   └── /
└── swap partition
    └── [SWAP]
```

The operating system disk is intentionally kept separate from the dedicated data disk added later.

The final design therefore becomes:

```text
/dev/sda
└── Operating system

/dev/sdb
└── Persistent service and application data
```

This separation makes the role of each disk easier to understand and provides a cleaner foundation for future services.

---

## 8. Command-Line Administration

The server does not depend on a graphical desktop for administration.

Most operations are performed through the shell.

Examples include:

```bash
lsblk
ip -br a
systemctl status ssh
df -h
groups
```

This approach provides practical experience with Linux administration tools and makes the environment suitable for remote management through SSH.

It also avoids depending on graphical configuration tools that may not be available on production-style Linux servers.

---

## 9. Root and Administrative Access

Debian distinguishes between the normal user account and the root account.

The root account has unrestricted administrative privileges.

It can be accessed when necessary using:

```bash
su -
```

However, routine administration should not be performed permanently as root.

The preferred workflow for this lab is:

```text
Normal user
    ↓
sudo
    ↓
Privileged command
```

This limits the amount of time spent in an unrestricted root shell and makes administrative actions more explicit.

During the initial setup, the minimal installation did not provide the `sudo` command.

It was therefore necessary to temporarily switch to root and install it.

This procedure is covered in:

```text
notes/02-linux-administration.md
```

---

## 10. Verification Commands

The following commands provide a quick baseline check of the server:

```bash
hostname
whoami
ip -br a
lsblk
df -h
```

For more detailed system information:

```bash
uname -a
```

To identify the Debian release:

```bash
cat /etc/os-release
```

These commands are useful when reconnecting to the server or troubleshooting because they quickly establish:

- which machine is being administered;
- which user is active;
- which operating system is running;
- which network interfaces exist;
- which disks are detected;
- which filesystems are mounted.

---

## 11. Result

At the end of this stage, the server has a functional Debian 12 base installation.

The resulting foundation consists of:

```text
Oracle VirtualBox
        │
        ▼
HOMESERVER-LAB
        │
        ├── Debian GNU/Linux 12
        │
        ├── Hostname: homeserver-lab
        │
        ├── User: mera
        │
        ├── Command-line administration
        │
        └── System disk: /dev/sda
```

The operating system is now ready for the next administrative tasks.

---

## 12. Next Steps

The next configuration stages are documented separately:

```text
02-linux-administration.md
03-storage-configuration.md
04-permissions.md
05-ssh-configuration.md
```

The next step is to configure administrative privileges and establish a proper `sudo` workflow.

---

## Commands Summary

```bash
# Display the hostname
hostname

# Display the current user
whoami

# Display network interfaces
ip -br a

# Display disks and partitions
lsblk

# Display mounted filesystem usage
df -h

# Display kernel and system information
uname -a

# Display Debian release information
cat /etc/os-release

# Switch to the root account
su -
```

---

## Key Takeaways

- Debian 12 provides the base operating system for the homelab server.
- The server is designed to be administered primarily from the command line.
- The hostname is `homeserver-lab`.
- The normal administrative user is `mera`.
- `/dev/sda` contains the operating system.
- A separate virtual disk will be used for persistent service data.
- Routine administration should use a normal user with `sudo` rather than a permanent root shell.
- Basic verification commands such as `hostname`, `whoami`, `ip`, `lsblk` and `df` provide a quick overview of the server state.
