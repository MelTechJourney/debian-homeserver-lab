# Linux Users, Groups and Permissions

## User identity

Display the currently authenticated user:

```bash
whoami
```

Display the current user's UID, GID and group memberships:

```bash
id
```

Display identity information for a specific user:

```bash
id mera
```

Display active groups for the current session:

```bash
groups
```

Display the configured groups for `mera`:

```bash
groups mera
```

---

## Root access

Open a root login shell:

```bash
su -
```

Verify the current identity:

```bash
whoami
```

Expected result:

```text
root
```

Leave the root shell:

```bash
exit
```

For routine administration, prefer `sudo` instead of remaining in a root shell.

---

## Verify sudo access

Test whether the current user can execute commands with elevated privileges:

```bash
sudo whoami
```

Expected result:

```text
root
```

This executes only the requested command with root privileges.

---

## Create a group

Create the `homelab` group:

```bash
sudo groupadd homelab
```

Verify that the group exists:

```bash
getent group homelab
```

---

## Add a user to a group

Add `mera` to the `homelab` supplementary group:

```bash
sudo usermod -aG homelab mera
```

Verify the configured membership:

```bash
groups mera
```

or:

```bash
id mera
```

The options mean:

```text
-a    Append to existing supplementary groups
-G    Specify supplementary groups
```

> Always use `-aG` when adding a supplementary group with `usermod`. Using `-G` without `-a` can replace the user's existing supplementary group memberships.

---

## Refresh group membership

A group added with `usermod` may not become active in an existing login session.

Check the groups active in the current session:

```bash
groups
```

If `homelab` is missing, log out and reconnect.

Then check again:

```bash
groups
```

Expected result should include:

```text
homelab
```

---

## Change ownership

The general syntax is:

```bash
sudo chown <OWNER>:<GROUP> <PATH>
```

For the homelab data directory:

```bash
sudo chown root:homelab /srv/data
```

This configures:

```text
Owner    root
Group    homelab
```

Verify:

```bash
ls -ld /srv/data
```

---

## Change permissions

The general syntax is:

```bash
sudo chmod <MODE> <PATH>
```

For `/srv/data`:

```bash
sudo chmod 2775 /srv/data
```

This configures:

```text
Owner     rwx
Group     rwx
Others    r-x
Setgid    enabled
```

---

## Numeric permissions

Linux permissions use the following numeric values:

```text
4 = read
2 = write
1 = execute
```

The values are combined.

```text
7 = 4 + 2 + 1 = rwx
6 = 4 + 2     = rw-
5 = 4 + 1     = r-x
4 = 4         = r--
```

Therefore:

```text
775
```

means:

```text
Owner     rwx
Group     rwx
Others    r-x
```

---

## Special permission bits

The first digit in:

```text
2775
```

represents a special permission bit.

The value:

```text
2
```

enables setgid.

For a directory, setgid causes new files and subdirectories to inherit the directory's group.

For this lab:

```text
/srv/data
└── group = homelab
```

With setgid enabled:

```text
/srv/data
├── file-a       → group: homelab
├── file-b       → group: homelab
└── directory-a  → group: homelab
```

This makes shared storage easier to manage.

---

## Inspect directory permissions

Display ownership and permissions for `/srv/data`:

```bash
ls -ld /srv/data
```

A correctly configured result should contain a permission structure similar to:

```text
drwxrwsr-x
```

The important section is:

```text
rws
```

The `s` in the group execute position indicates that setgid is enabled.

---

## Inspect file permissions

Display files and their ownership:

```bash
ls -l /srv/data
```

This shows information such as:

```text
permissions
owner
group
size
modification date
filename
```

For files created by `mera` inside the configured data directory, the expected ownership model is:

```text
Owner    mera
Group    homelab
```

---

## Test write access

Test whether the current user can create a file:

```bash
touch /srv/data/test.txt
```

Verify the result:

```bash
ls -l /srv/data
```

If the permission model is working, the command succeeds without `sudo`.

The created file should inherit the `homelab` group.

---

## Remove a test file

After validation, the test file can be removed:

```bash
rm /srv/data/test.txt
```

Verify:

```bash
ls -l /srv/data
```

---

## Permission troubleshooting

If a command returns:

```text
Permission denied
```

or:

```text
Permission non accordée
```

first check the current user:

```bash
whoami
```

Check active groups:

```bash
groups
```

Display detailed identity information:

```bash
id
```

Inspect the target directory:

```bash
ls -ld /srv/data
```

Check the configured groups for the user:

```bash
groups mera
```

The expected configuration for this lab is:

```text
User       mera
Group      homelab
Directory  /srv/data
Owner      root
Group      homelab
Mode       2775
```

If `groups mera` contains `homelab` but the current:

```bash
groups
```

does not, reconnect the user session.

---

## chown vs chmod

`chown` changes ownership:

```bash
sudo chown root:homelab /srv/data
```

It controls:

```text
Owner
Group
```

`chmod` changes permission bits:

```bash
sudo chmod 2775 /srv/data
```

It controls:

```text
Read
Write
Execute
Special permission bits
```

Both must be configured correctly for the desired access model.

---

## Avoid chmod 777

Avoid using:

```bash
sudo chmod 777 /srv/data
```

as a default solution to permission problems.

`777` means:

```text
Owner     rwx
Group     rwx
Others    rwx
```

This grants every local user full access to the directory.

The homelab instead uses:

```bash
sudo chown root:homelab /srv/data
sudo chmod 2775 /srv/data
```

This grants write access through the dedicated `homelab` group.

---

## Current Lab Permission Model

```text
/srv/data
│
├── Owner
│   └── root
│
├── Group
│   └── homelab
│
├── Mode
│   └── 2775
│
├── Setgid
│   └── enabled
│
└── Authorized user
    └── mera
        └── member of homelab
```

Administrative privileges and data permissions remain separate:

```text
sudo
└── Administrative privileges

homelab
└── Shared data access
```

---

## Quick Reference

```bash
# Current user
whoami

# Current identity and groups
id

# Active groups
groups

# Check groups configured for mera
groups mera

# Switch to root
su -

# Leave root shell
exit

# Test sudo
sudo whoami

# Create the homelab group
sudo groupadd homelab

# Check the group
getent group homelab

# Add mera to homelab
sudo usermod -aG homelab mera

# Configure /srv/data ownership
sudo chown root:homelab /srv/data

# Configure permissions and setgid
sudo chmod 2775 /srv/data

# Inspect directory permissions
ls -ld /srv/data

# Test write access
touch /srv/data/test.txt

# Inspect resulting file ownership
ls -l /srv/data
```
