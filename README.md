# 🏠 Debian HomeServer Lab

A hands-on homelab built with **Debian 12** and **Oracle VirtualBox**, focused on learning and practicing Linux system administration.

The goal of this project is to progressively build a clean and functional Linux server environment while documenting each major configuration step.

The lab currently covers:

- Debian server installation
- Linux user and privilege management
- Dedicated data storage
- Filesystem configuration
- Persistent mounts
- Linux groups and permissions
- SSH remote administration

> This repository documents a learning environment. It will evolve progressively as new services, security mechanisms and infrastructure components are added.

---

## 📌 Project Status

### Phase 1 — Server Foundation ✅

- [x] Create the virtual machine
- [x] Install Debian 12
- [x] Configure the administrative user
- [x] Install and configure `sudo`
- [x] Add a dedicated virtual data disk
- [x] Partition the data disk
- [x] Format the partition with ext4
- [x] Mount the disk on `/srv/data`
- [x] Configure persistent mounting with `/etc/fstab`
- [x] Create the `homelab` group
- [x] Configure storage permissions
- [x] Validate non-root write access
- [x] Enable and test SSH
- [x] Configure SSH key authentication

---

## 🖥️ Environment

| Component | Configuration |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Operating System | Debian GNU/Linux 12 |
| Hostname | `homeserver-lab` |
| System Disk | 30 GB |
| Data Disk | 20 GB |
| Data Filesystem | ext4 |
| Data Mount Point | `/srv/data` |
| Remote Administration | OpenSSH |
| SSH Authentication | Ed25519 key |

---

## 1. Debian Installation

The project starts with a dedicated Debian 12 virtual machine running inside Oracle VirtualBox.

The VM is intentionally kept relatively lightweight because the server is primarily designed to be administered from the command line.

During installation, only the required Debian components and standard system utilities were selected.

![Debian installation](docs/screenshots/01-debian-installation.png)

![Software selection](docs/screenshots/02-software-selection.png)

---

## 2. Initial System Verification

After the first boot, the base system was checked before making additional changes.

```bash
hostname
whoami
ip -br a
lsblk
```

These commands verify:

- the configured hostname;
- the current user;
- available network interfaces;
- detected block devices and partitions.

![Initial system verification](docs/screenshots/03-verification-systeme-initial.png)

At this point, the Debian installation is operational and the main system disk is correctly detected.

---

## 3. Administrative Privileges

The minimal installation did not initially include `sudo`.

The root account was therefore used temporarily to update the package index and install it.

```bash
su -
apt update
apt install sudo
```

The administrative user was then added to the `sudo` group:

```bash
usermod -aG sudo mera
groups mera
```

![Sudo installation and configuration](docs/screenshots/04-installation-configuration-sudo.png)

After reconnecting the user session, privilege escalation was tested:

```bash
sudo whoami
```

Expected result:

```text
root
```

![Sudo verification](docs/screenshots/05-verification-acces-root-sudo.png)

The server can now be administered without using the root account for normal operations.

---

## 4. Dedicated Data Disk

To separate operating system files from future application and service data, a second **20 GB virtual disk** was attached to the VM.

This provides the following layout:

```text
System disk
└── Debian operating system

Data disk
└── Application and service data
```

![VirtualBox data disk](docs/screenshots/06-ajout-disque-virtuel-20go.png)

After booting Debian, the new disk was detected with:

```bash
lsblk
```

The additional disk appears as:

```text
/dev/sdb
```

![Second disk detection](docs/screenshots/07-detection-second-disque.png)

---

## 5. Disk Partitioning

The new disk initially contained no partition table.

It was partitioned using `fdisk`:

```bash
sudo fdisk /dev/sdb
```

A new Linux partition was created using the available disk space.

The resulting partition is:

```text
/dev/sdb1
```

![Disk partitioning](docs/screenshots/08-partitionnement-disque-sdb.png)

The screenshot also documents an incorrect input made during the interactive `fdisk` process before the partition was successfully created.

Keeping this step documented reflects the actual troubleshooting process rather than presenting an artificially perfect installation.

---

## 6. ext4 Filesystem

The newly created partition was formatted using the **ext4** filesystem:

```bash
sudo mkfs.ext4 /dev/sdb1
```

The resulting filesystem was verified with:

```bash
lsblk -f
```

![ext4 filesystem creation](docs/screenshots/09-formatage-ext4-sdb1.png)

The dedicated data partition is now ready to be mounted.

---

## 7. Data Mount Point

A dedicated mount point was created:

```bash
sudo mkdir -p /srv/data
```

The new filesystem was then mounted:

```bash
sudo mount /dev/sdb1 /srv/data
```

The result was verified using:

```bash
lsblk
df -h /srv/data
```

![Data disk mounting](docs/screenshots/10-montage-disque-srv-data.png)

The screenshot also shows a failed initial mount attempt caused by the absence of the expected mount point.

The directory was recreated and the mount completed successfully.

This provides a useful example of basic Linux storage troubleshooting.

---

## 8. Persistent Mount Configuration

A manual mount does not automatically survive a reboot.

The data filesystem was therefore configured in `/etc/fstab`.

First, its UUID was retrieved:

```bash
sudo blkid /dev/sdb1
```

The configuration follows this structure:

```text
UUID=<FILESYSTEM-UUID> /srv/data ext4 defaults 0 2
```

Using a filesystem UUID instead of `/dev/sdb1` avoids depending on a device name that could theoretically change.

After modifying `/etc/fstab`, systemd was reloaded:

```bash
sudo systemctl daemon-reload
```

The configuration was then tested:

```bash
sudo mount -a
lsblk
```

![Persistent mount configuration](docs/screenshots/11-montage-persistant-fstab.png)

The disk is now automatically mounted on:

```text
/srv/data
```

---

## 9. Storage Permissions

Initially, `/srv/data` was owned by `root`, preventing the regular user from writing to the directory.

A dedicated Linux group was created:

```bash
sudo groupadd homelab
```

The administrative user was added to it:

```bash
sudo usermod -aG homelab mera
```

The storage directory ownership was then changed:

```bash
sudo chown root:homelab /srv/data
```

Permissions were configured with:

```bash
sudo chmod 2775 /srv/data
```

![Storage permissions](docs/screenshots/12-configuration-permissions-srv-data.png)

The leading `2` enables the **setgid bit** on the directory.

As a result, files and directories created inside `/srv/data` inherit the `homelab` group, making the directory suitable for future services that need shared access to the data volume.

---

## 10. Write Access Validation

After reconnecting the user session so that the new group membership became active, write access was tested without `sudo`.

```bash
touch /srv/data/test.txt
```

The result was checked with:

```bash
ls -l /srv/data
```

![Write access validation](docs/screenshots/13-test-ecriture-srv-data.png)

The file was successfully created by the regular user and inherited the `homelab` group.

This confirms that the permission model works as intended.

---

## 11. SSH Remote Administration

OpenSSH is installed and running on the server.

The service can be checked with:

```bash
systemctl status ssh
```

The listening SSH socket can also be verified with:

```bash
ss -tulpn | grep :22
```

Remote connectivity from the host machine was successfully tested.

SSH key authentication was then configured using an **Ed25519 key pair**.

The public key is stored on the server in:

```text
~/.ssh/authorized_keys
```

The associated private key remains exclusively on the client machine and is **never stored in this repository**.

Screenshots containing unnecessary network addressing or SSH connection information are intentionally excluded from the public documentation.

---

## 🔐 Security Principles

This lab is not intended to represent a fully hardened production environment yet.

However, several basic security principles are already applied:

- daily administration from a non-root account;
- privilege escalation through `sudo`;
- SSH key authentication;
- separation between system and data storage;
- dedicated group-based storage permissions;
- no passwords stored in the repository;
- no SSH private keys stored in the repository;
- unnecessary network information excluded from public screenshots.

Further hardening will be implemented as the lab evolves.

---

## 💾 Current Storage Architecture

```text
homeserver-lab
│
├── /dev/sda
│   └── Debian 12
│       └── /
│
└── /dev/sdb
    └── /dev/sdb1
        └── ext4
            └── /srv/data
```

The system and application data are therefore separated across two virtual disks.

`/srv/data` will serve as the main storage location for future self-hosted services.

---

## 🌐 Current Architecture

```text
┌─────────────────────────────┐
│         Host Machine        │
│                             │
│      Oracle VirtualBox      │
└──────────────┬──────────────┘
               │
               │ Virtual Network
               │
┌──────────────▼──────────────┐
│       homeserver-lab        │
│          Debian 12          │
│                             │
│  ┌───────────────────────┐  │
│  │     OpenSSH Server    │  │
│  └───────────────────────┘  │
│                             │
│  System: /dev/sda           │
│  Data:   /dev/sdb1          │
│          ↓                  │
│       /srv/data             │
└─────────────────────────────┘
```

This architecture is deliberately simple for the first stage of the project.

The server foundation can now be used to deploy additional infrastructure and services.

---

## 🛠️ Troubleshooting Encountered

The lab was built manually rather than from a preconfigured image, which exposed several useful problems during setup.

### `sudo` unavailable

The initial Debian installation did not provide `sudo`.

Solution:

```bash
su -
apt update
apt install sudo
usermod -aG sudo mera
```

### Data mount point missing

An initial attempt to mount the data partition failed because `/srv/data` did not exist.

Solution:

```bash
sudo mkdir -p /srv/data
sudo mount /dev/sdb1 /srv/data
```

### Permission denied on `/srv/data`

The regular user initially could not create files in the data directory.

A dedicated group and setgid permissions were configured:

```bash
sudo groupadd homelab
sudo usermod -aG homelab mera
sudo chown root:homelab /srv/data
sudo chmod 2775 /srv/data
```

After reconnecting the user session, write access worked correctly.

These issues are intentionally documented because troubleshooting is part of the purpose of the homelab.

---

## 🗺️ Roadmap

### Phase 1 — Server Foundation

- [x] Debian installation
- [x] Administrative privileges
- [x] Dedicated storage
- [x] Persistent filesystem
- [x] Group permissions
- [x] SSH access
- [x] SSH key authentication

### Phase 2 — Server Hardening

- [ ] Harden SSH configuration
- [ ] Disable unnecessary authentication methods
- [ ] Configure a firewall
- [ ] Review exposed services
- [ ] Configure automatic security updates

### Phase 3 — Services

- [ ] Install a container runtime
- [ ] Deploy initial self-hosted services
- [ ] Organize persistent application data
- [ ] Configure service networking

### Phase 4 — Operations

- [ ] Implement backups
- [ ] Add monitoring
- [ ] Centralize logs
- [ ] Document recovery procedures
- [ ] Introduce configuration automation

---

## 📁 Repository Structure

```text
debian-homeserver-lab/
│
├── README.md
│
└── docs/
    └── screenshots/
        ├── 01-debian-installation.png
        ├── 02-software-selection.png
        ├── 03-verification-systeme-initial.png
        ├── 04-installation-configuration-sudo.png
        ├── 05-verification-acces-root-sudo.png
        ├── 06-ajout-disque-virtuel-20go.png
        ├── 07-detection-second-disque.png
        ├── 08-partitionnement-disque-sdb.png
        ├── 09-formatage-ext4-sdb1.png
        ├── 10-montage-disque-srv-data.png
        ├── 11-montage-persistant-fstab.png
        ├── 12-configuration-permissions-srv-data.png
        └── 13-test-ecriture-srv-data.png
```

---

## ⚠️ Disclaimer

This project is a **personal learning homelab**, not a production deployment guide.

Configurations are documented as they are implemented and may evolve as the environment becomes more secure and complex.

Passwords, private SSH keys and other authentication secrets must never be committed to this repository.

Some network information is also intentionally excluded from screenshots when it is not necessary to demonstrate the technical configuration.

---

## 🎯 Project Goal

The objective is not simply to run self-hosted applications.

The project is intended to provide practical experience with:

- Linux administration;
- storage and filesystems;
- users, groups and permissions;
- networking;
- remote administration;
- service management;
- system security;
- troubleshooting;
- automation;
- infrastructure documentation.

Each new component will be added progressively and documented in this repository.
