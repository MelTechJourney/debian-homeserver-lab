# Linux Administration and sudo Configuration

## Overview

This note documents the initial administrative configuration of the Debian 12 server.

After the base installation, the normal user account did not initially have access to the `sudo` command.

The objective of this stage was therefore to establish a proper administration model based on:

- a normal user account for daily operations;
- temporary privilege escalation with `sudo`;
- limited direct use of the root account;
- verification of group membership and administrative access.

The primary administrative user used in this lab is:

```text
mera
```

The server hostname is:

```text
homeserver-lab
```

---

## 1. Initial Situation

After the first Debian login, basic system information was checked using:

```bash
hostname
whoami
ip -br a
lsblk
```

The current user was:

```text
mera
```

At this point, an attempt to use `sudo` failed because the command was not installed.

For example:

```bash
sudo whoami
```

returned an error indicating that `sudo` could not be found.

This meant that administrative configuration initially had to be performed directly from the root account.

---

## 2. Switching to the Root Account

The root account was accessed using:

```bash
su -
```

The `-` option starts a login shell for root and loads the root environment.

After entering the root password, the shell prompt changed from a normal user prompt:

```text
mera@homeserver-lab:~$
```

to a root prompt:

```text
root@homeserver-lab:~#
```

The current identity can also be verified with:

```bash
whoami
```

Expected result:

```text
root
```

Direct root access was used only to perform the initial administrative configuration.

---

## 3. Updating the Package Index

Before installing `sudo`, the Debian package index was updated:

```bash
apt update
```

This retrieves the current package metadata from the configured Debian repositories.

The command does not upgrade the installed packages by itself.

Its purpose is to refresh the local information used by APT when searching for and installing packages.

---

## 4. Installing sudo

The `sudo` package was installed while operating as root:

```bash
apt install sudo
```

APT downloaded and installed the package along with the required configuration.

The installation process is documented here:

![sudo installation and configuration](../screenshots/04-installation-configuration-sudo.png)

After installation, the `sudo` command became available on the system.

However, installing the package alone does not automatically grant administrative privileges to a normal user.

The user must also be authorized to use it.

---

## 5. Adding the User to the sudo Group

On Debian, membership in the `sudo` group is used to grant normal users access to administrative commands through `sudo`.

The user `mera` was added to the group using:

```bash
usermod -aG sudo mera
```

The options have specific meanings:

```text
-a    append the user to additional groups
-G    specify supplementary groups
```

The `-a` option is important.

Using `usermod -G` without `-a` can replace the user's existing supplementary group memberships instead of simply adding a new one.

---

## 6. Verifying Group Membership

After modifying the account, its group memberships were checked:

```bash
groups mera
```

The output confirmed that `mera` belonged to the `sudo` group.

A simplified example is:

```text
mera : mera ... sudo ...
```

This confirms that the account configuration has been modified correctly.

The same screenshot also documents the installation and group configuration:

![sudo installation and configuration](../screenshots/04-installation-configuration-sudo.png)

---

## 7. Session Refresh

Linux group membership changes do not necessarily affect an already active login session immediately.

After adding `mera` to the `sudo` group, the user session therefore needs to be refreshed.

The cleanest method is to:

```text
log out
↓
log back in
```

After reconnecting, the new session loads the updated group memberships.

The active groups can then be checked with:

```bash
groups
```

or:

```bash
id
```

This distinction is important when troubleshooting permissions.

A user may have been correctly added to a group in the system configuration while an older shell session still operates with the previous group list.

---

## 8. Testing sudo

After reconnecting as `mera`, administrative access was tested with:

```bash
sudo whoami
```

The user's password was requested.

Expected result:

```text
root
```

This does not mean that the user permanently became root.

Instead, only the requested command was executed with elevated privileges.

The successful verification is documented here:

![sudo access verification](../screenshots/05-verification-acces-root-sudo.png)

At this point, the normal user can perform administrative operations without maintaining a permanent root shell.

---

## 9. Administration Model

The resulting administration model is:

```text
mera
 │
 │ normal shell
 │
 ├── Standard commands
 │
 └── sudo <command>
          │
          ▼
     Root privileges
     for that command
```

For example, a normal command can be executed directly:

```bash
lsblk
```

An operation requiring administrative privileges uses:

```bash
sudo mount /dev/sdb1 /srv/data
```

A system configuration command can use:

```bash
sudo systemctl daemon-reload
```

This makes privileged operations explicit.

---

## 10. sudo vs su -

Both `sudo` and `su -` can provide administrative access, but they are used differently.

### `sudo`

Example:

```bash
sudo systemctl status ssh
```

Only the specified command is executed with elevated privileges.

After the command completes, the shell remains the normal user's shell.

### `su -`

Example:

```bash
su -
```

This opens a root login shell.

Subsequent commands are executed as root until the shell is closed:

```bash
exit
```

For routine administration in this lab, `sudo` is preferred.

A root shell remains useful for specific recovery or initial configuration situations but should not be the default working environment.

---

## 11. Checking the Current Identity

Before executing sensitive commands, it can be useful to verify which account is currently active.

Use:

```bash
whoami
```

Normal session:

```text
mera
```

Root shell:

```text
root
```

More detailed account information can be obtained with:

```bash
id
```

This displays:

- user ID;
- primary group;
- supplementary groups.

For example:

```bash
id mera
```

can be used to confirm the groups assigned to the administrative account.

---

## 12. Understanding Linux Groups

Linux permissions are based primarily on three access categories:

```text
user
group
other
```

Groups allow multiple users to share access to specific resources without giving those permissions to every account on the system.

During this stage, the important administrative group is:

```text
sudo
```

Later in the project, another dedicated group is introduced:

```text
homelab
```

The two groups serve different purposes:

| Group | Purpose |
|---|---|
| `sudo` | Administrative privilege escalation |
| `homelab` | Shared access to homelab data |

The `homelab` group is documented in:

```text
notes/04-permissions.md
```

Separating these responsibilities avoids using administrative privileges as a substitute for normal filesystem permissions.

---

## 13. Privilege Principle

A useful Linux administration principle is to perform normal operations without elevated privileges and use administrative privileges only when required.

For example:

```bash
ls -l /srv/data
```

does not require root privileges.

Changing ownership generally does:

```bash
sudo chown root:homelab /srv/data
```

Installing software also requires administrative privileges:

```bash
sudo apt install <package>
```

This distinction helps reduce accidental system modifications.

---

## 14. Package Management Workflow

After `sudo` was configured, package administration can be performed directly from the normal account.

Update the package index:

```bash
sudo apt update
```

Install a package:

```bash
sudo apt install <package>
```

Remove a package:

```bash
sudo apt remove <package>
```

Search for a package:

```bash
apt search <package>
```

Display package information:

```bash
apt show <package>
```

A typical installation workflow is therefore:

```bash
sudo apt update
sudo apt install <package>
```

---

## 15. Troubleshooting: sudo Command Not Found

### Symptom

Running:

```bash
sudo whoami
```

returns an error similar to:

```text
sudo: command not found
```

### Cause

The `sudo` package is not installed.

### Resolution

Switch to root:

```bash
su -
```

Update APT:

```bash
apt update
```

Install `sudo`:

```bash
apt install sudo
```

Add the user to the appropriate group:

```bash
usermod -aG sudo mera
```

Verify:

```bash
groups mera
```

Then log out and reconnect as `mera`.

Finally test:

```bash
sudo whoami
```

Expected result:

```text
root
```

---

## 16. Troubleshooting: User Added to Group but sudo Still Fails

If the user has been added to the `sudo` group but the current session still does not recognize the new privileges, first verify the configured groups:

```bash
groups mera
```

Then verify the groups active in the current session:

```bash
groups
```

If necessary:

```text
log out
↓
log back in
```

Then test again:

```bash
sudo whoami
```

This is caused by the fact that supplementary group membership is established when the user session starts.

---

## 17. Result

At the end of this configuration stage:

```text
Debian 12
│
├── root
│   └── Initial privileged configuration
│
└── mera
    │
    ├── Normal user account
    │
    └── Member of sudo
            │
            └── Administrative commands
                through sudo
```

The server can now be administered from the normal `mera` account while elevated privileges remain available when required.

---

## 18. Security Considerations

The current configuration establishes the basic administrative model but does not represent the final hardening state of the server.

Current principles include:

- use a non-root account for routine administration;
- use `sudo` for privileged operations;
- avoid remaining logged in as root unnecessarily;
- keep administrative and data-access groups separate;
- verify the active identity before sensitive operations.

Additional security configuration will be introduced later in the project, particularly around SSH access and network exposure.

---

## Commands Summary

```bash
# Switch to root
su -

# Check the current user
whoami

# Update the APT package index
apt update

# Install sudo as root
apt install sudo

# Add the user to the sudo group
usermod -aG sudo mera

# Check the configured groups for the user
groups mera

# Leave the root shell
exit

# Check active groups
groups

# Display detailed identity information
id

# Verify sudo access
sudo whoami

# Update packages as the normal administrative user
sudo apt update
```

---

## Key Takeaways

- The minimal Debian installation did not initially provide the `sudo` command.
- The root account was temporarily used to install and configure `sudo`.
- The `mera` account was added to the `sudo` group.
- `usermod -aG` appends a supplementary group without replacing existing group memberships.
- Group membership changes may require a new login session.
- `sudo whoami` provides a simple test of privilege escalation.
- Routine administration should be performed from the normal user account.
- `sudo` should be used only for commands that require elevated privileges.
- Administrative access and shared filesystem access are handled through separate Linux groups.
