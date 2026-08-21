# Linux Permissions and Shared Storage

## Overview

This note documents the permission model configured for the dedicated data directory:

```text
/srv/data
```

After the data disk had been partitioned, formatted and mounted successfully, the filesystem was technically operational.

However, the regular user could not initially create files inside `/srv/data`.

The objective of this stage was to provide controlled write access without using overly permissive permissions such as:

```text
777
```

A dedicated Linux group named `homelab` was therefore introduced.

The final permission model is:

```text
/srv/data
│
├── Owner: root
├── Group: homelab
├── Permissions: 2775
│
└── Members of homelab
    └── Read / Write / Execute
```

---

## 1. Initial Situation

The dedicated ext4 filesystem had already been mounted on:

```text
/srv/data
```

The mount itself was working correctly.

It could be verified using:

```bash
lsblk
```

and:

```bash
df -h /srv/data
```

However, mounting a filesystem does not automatically grant write access to regular users.

Storage configuration and filesystem permissions are separate concerns.

---

## 2. Initial Write Test

A first write test was performed as the regular user:

```bash
touch /srv/data/test.txt
```

The operation failed with a permission error.

This demonstrated that:

```text
Filesystem mounted correctly ≠ User allowed to write
```

The problem was therefore no longer related to the disk, filesystem or mount configuration.

It was a Linux ownership and permission issue.

---

## 3. Inspecting Directory Permissions

The permissions of the mount point can be inspected using:

```bash
ls -ld /srv/data
```

The `-d` option is important because it displays information about the directory itself rather than listing its contents.

A Linux file or directory has three main permission categories:

```text
Owner
Group
Others
```

Conceptually:

```text
        owner     group     others
          │         │          │
          ▼         ▼          ▼
        rwx       rwx        rwx
```

Each category can independently receive:

```text
r = read
w = write
x = execute
```

For a directory, these permissions have specific meanings.

| Permission | Directory meaning |
|---|---|
| `r` | List directory contents |
| `w` | Create, delete or rename entries |
| `x` | Enter/traverse the directory |

Write access to a directory generally requires both:

```text
w + x
```

---

## 4. Avoiding chmod 777

A simple way to make a directory writable by everyone would be:

```bash
sudo chmod 777 /srv/data
```

This was intentionally not used.

`777` would grant:

```text
Owner  → rwx
Group  → rwx
Others → rwx
```

That means every local user would have full access to the directory.

Instead, the lab uses a dedicated group to control who receives write access.

The objective is:

```text
root
  │
  └── owns /srv/data

homelab group
  │
  └── authorized users/services

everyone else
  │
  └── no write permission
```

---

## 5. Creating the homelab Group

A dedicated group was created:

```bash
sudo groupadd homelab
```

The group exists specifically to manage access to homelab data.

This separates two different responsibilities:

```text
sudo
└── administrative privileges

homelab
└── access to shared service data
```

A user does not need full administrative privileges simply because it needs access to application data.

---

## 6. Adding the User to the Group

The regular administrative user was added to the new group:

```bash
sudo usermod -aG homelab mera
```

The options mean:

```text
-a    append
-G    supplementary groups
```

Using both options is important.

The command adds `homelab` to the user's existing supplementary groups instead of replacing them.

The configured membership can be checked with:

```bash
groups mera
```

or:

```bash
id mera
```

---

## 7. Changing Directory Ownership

The ownership of `/srv/data` was configured using:

```bash
sudo chown root:homelab /srv/data
```

The syntax is:

```text
chown OWNER:GROUP PATH
```

Therefore:

```text
root:homelab
```

means:

```text
Owner → root
Group → homelab
```

The directory remains controlled by root while members of the `homelab` group can receive the permissions assigned to the group.

The result can be inspected with:

```bash
ls -ld /srv/data
```

---

## 8. Configuring Permissions

The permissions were configured using:

```bash
sudo chmod 2775 /srv/data
```

The resulting mode can be separated into:

```text
2 7 7 5
│ │ │ │
│ │ │ └── Others
│ │ └──── Group
│ └────── Owner
└──────── Special permission: setgid
```

The three standard permission digits are:

```text
775
```

while the leading:

```text
2
```

enables the setgid bit.

The configuration process is documented here:

![Storage permissions](../screenshots/12-configuration-permissions-srv-data.png)

---

## 9. Understanding Numeric Permissions

Linux permissions can be represented numerically.

Each permission has a value:

```text
read    = 4
write   = 2
execute = 1
```

The values are added together.

### Read + Write + Execute

```text
4 + 2 + 1 = 7
```

Therefore:

```text
7 = rwx
```

### Read + Execute

```text
4 + 1 = 5
```

Therefore:

```text
5 = r-x
```

The standard portion of:

```text
2775
```

is therefore:

```text
775
```

which represents:

```text
Owner  → 7 → rwx
Group  → 7 → rwx
Others → 5 → r-x
```

So `/srv/data` allows:

| Category | Permissions |
|---|---|
| Owner (`root`) | Read, write, execute |
| Group (`homelab`) | Read, write, execute |
| Others | Read and execute |

Users outside the `homelab` group cannot create or modify files in the directory.

---

## 10. Understanding the setgid Bit

The leading `2` in:

```text
2775
```

enables the **setgid** bit on the directory.

Without setgid, newly created files normally receive the creator's primary group, depending on the environment and creation context.

For a shared service directory, this can create inconsistent group ownership.

With setgid enabled on `/srv/data`, new files and subdirectories inherit the group of the parent directory.

Because `/srv/data` belongs to:

```text
root:homelab
```

new entries created inside it inherit:

```text
homelab
```

The resulting behavior is:

```text
/srv/data
   │
   │ group = homelab
   │ setgid enabled
   │
   ├── file-a
   │   └── group = homelab
   │
   ├── file-b
   │   └── group = homelab
   │
   └── directory
       └── group = homelab
```

This is useful for directories shared by multiple users or services.

---

## 11. Recognizing setgid in ls Output

The directory can be inspected with:

```bash
ls -ld /srv/data
```

With setgid enabled, the group execute position contains:

```text
s
```

instead of:

```text
x
```

For example:

```text
drwxrwsr-x
```

Breaking this down:

```text
d          directory

rwx        owner permissions

rws        group permissions
  │
  └── setgid enabled

r-x        permissions for others
```

The `s` therefore confirms that setgid is active.

---

## 12. Group Membership and Active Sessions

After running:

```bash
sudo usermod -aG homelab mera
```

the account configuration is updated immediately.

However, an already active shell session may still use the old group list.

This means:

```bash
groups mera
```

can show that the user belongs to `homelab` while:

```bash
groups
```

in the current session may not yet include it.

The session was therefore closed and reopened.

After reconnecting:

```bash
groups
```

confirmed that the `homelab` group was active.

This distinction is important when troubleshooting Linux permissions.

---

## 13. Final Write Test

After reconnecting, write access was tested again:

```bash
touch /srv/data/test.txt
```

This time the command succeeded.

The directory contents were inspected:

```bash
ls -l /srv/data
```

The successful test is documented here:

![Write access validation](../screenshots/13-test-ecriture-srv-data.png)

The file ownership confirmed the expected behavior:

```text
user  → mera
group → homelab
```

Conceptually:

```text
mera
 │
 ├── member of homelab
 │
 ▼
/srv/data
 │
 ├── group = homelab
 ├── group permissions = rwx
 └── setgid enabled
        │
        ▼
     test.txt
        │
        ├── owner = mera
        └── group = homelab
```

This validated the complete permission model.

---

## 14. Why the First Test Failed

The first write attempt failed because the normal user did not initially have sufficient permissions on `/srv/data`.

The failure was useful because it confirmed that the filesystem was not unintentionally world-writable.

The problem was solved by changing the access model rather than bypassing it.

Instead of:

```bash
sudo chmod 777 /srv/data
```

the configuration became:

```bash
sudo groupadd homelab
sudo usermod -aG homelab mera
sudo chown root:homelab /srv/data
sudo chmod 2775 /srv/data
```

followed by a new user session.

This provides controlled shared access while retaining clear ownership.

---

## 15. Permission Troubleshooting Workflow

When a user receives:

```text
Permission denied
```

or:

```text
Permission non accordée
```

the following sequence can be used.

### Step 1 — Check the active user

```bash
whoami
```

### Step 2 — Check active groups

```bash
groups
```

or:

```bash
id
```

### Step 3 — Check configured group membership

```bash
groups mera
```

### Step 4 — Inspect the target directory

```bash
ls -ld /srv/data
```

Check:

```text
owner
group
permissions
```

### Step 5 — Verify the expected group

For this lab:

```text
homelab
```

should appear in the user's active groups.

### Step 6 — Refresh the session if necessary

```text
logout
↓
login again
```

### Step 7 — Retry the operation

```bash
touch /srv/data/test.txt
```

### Step 8 — Inspect the result

```bash
ls -l /srv/data
```

This workflow helps identify whether the problem comes from:

- incorrect ownership;
- incorrect permissions;
- missing group membership;
- group membership not yet active in the current session.

---

## 16. chown vs chmod

Two commands were required because they modify different properties.

### chown

```bash
sudo chown root:homelab /srv/data
```

Changes:

```text
owner
group
```

It answers:

```text
Who owns this filesystem object?
```

### chmod

```bash
sudo chmod 2775 /srv/data
```

Changes:

```text
permission bits
special mode bits
```

It answers:

```text
What can the owner, group and others do?
```

Both concepts work together.

Correct permissions assigned to the wrong group would not solve the access problem.

Likewise, correct ownership with insufficient permission bits would still prevent access.

---

## 17. User Ownership vs Group Ownership

The test file created inside `/srv/data` belongs to the user who created it:

```text
mera
```

while the setgid directory causes it to inherit:

```text
homelab
```

as its group.

This produces:

```text
mera:homelab
```

The two values serve different purposes.

The user ownership identifies who created or owns the file.

The group ownership allows controlled sharing between authorized accounts.

This will become particularly useful when services are later introduced into the homelab.

---

## 18. Future Service Accounts

The `homelab` group is designed to support more than one interactive user.

Future service accounts could potentially be granted access to shared storage by adding them to the group:

```bash
sudo usermod -aG homelab <service-user>
```

This should only be done when the service genuinely requires access to the shared directory.

The objective is not to place every service into one unrestricted group automatically.

As the homelab grows, more specific directories and groups may be introduced when stronger isolation is required.

For example:

```text
/srv/data/
├── application-a/
├── application-b/
└── backups/
```

Each directory can later receive permissions appropriate to the service using it.

---

## 19. Security Considerations

The current configuration is appropriate for the learning objectives of this stage, but permissions should always be adapted to the actual service requirements.

Important principles include:

- avoid `chmod 777` as a default solution;
- grant write access only where necessary;
- use groups to manage shared access;
- separate administrative privileges from data permissions;
- inspect ownership before changing permissions;
- avoid running applications as root only to bypass filesystem permission problems.

A permission error should normally be investigated rather than immediately bypassed with broader privileges.

---

## 20. Current Permission Architecture

The current model can be summarized as:

```text
                    Debian
                      │
          ┌───────────┴───────────┐
          │                       │
        sudo                   homelab
          │                       │
          ▼                       ▼
Administrative access      Shared data access
                                  │
                                  ▼
                              /srv/data
                                  │
                       owner = root
                       group = homelab
                       mode  = 2775
                                  │
                                  ▼
                        Authorized members
                        can create content
```

The two groups deliberately solve different problems.

---

## 21. Result

At the end of this stage:

```text
/srv/data
│
├── Owner
│   └── root
│
├── Group
│   └── homelab
│
├── Permissions
│   └── 2775
│
├── setgid
│   └── enabled
│
└── mera
    └── member of homelab
        │
        └── write access validated
```

The dedicated storage can now be used by authorized non-root accounts without making the directory universally writable.

---

## Commands Summary

```bash
# Inspect the data directory
ls -ld /srv/data

# Test write access
touch /srv/data/test.txt

# Create the homelab group
sudo groupadd homelab

# Add the user to the group
sudo usermod -aG homelab mera

# Check configured user groups
groups mera

# Display detailed user/group information
id mera

# Change directory ownership
sudo chown root:homelab /srv/data

# Configure rwx/rwx/r-x permissions and setgid
sudo chmod 2775 /srv/data

# Check active session groups
groups

# Inspect the resulting directory permissions
ls -ld /srv/data

# Inspect files and their ownership
ls -l /srv/data
```

---

## Key Takeaways

- A successfully mounted filesystem is not necessarily writable by a regular user.
- Linux filesystem access depends on ownership and permission bits.
- `chown` controls ownership while `chmod` controls permissions.
- The `homelab` group provides controlled shared access to `/srv/data`.
- `2775` means setgid plus `rwxrwxr-x`.
- The setgid bit causes new entries to inherit the `homelab` group.
- Supplementary group changes generally require a refreshed login session.
- `groups`, `id` and `ls -ld` are useful permission troubleshooting tools.
- `chmod 777` was deliberately avoided.
- Administrative privileges and shared data access should remain separate concepts.
- The final write test confirmed that `mera` could write to `/srv/data` without using `sudo`.
