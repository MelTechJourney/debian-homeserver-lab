# SSH Configuration and Key-Based Authentication

## Overview

This note documents the SSH configuration of the Debian 12 `homeserver-lab`.

The objective of this stage was to establish secure remote administration from the Windows host without relying on the VirtualBox console.

The configuration process included:

- verifying the OpenSSH service;
- configuring VirtualBox networking;
- identifying the VM on the local network;
- testing network connectivity;
- establishing the first SSH connection;
- verifying the SSH host fingerprint;
- troubleshooting a password authentication failure;
- generating an Ed25519 SSH key pair;
- installing the public key on Debian;
- validating key-based authentication;
- checking SSH file permissions and server settings.

The current SSH architecture is:

```text
Windows Host
     │
     │ SSH
     ▼
Local Network
     │
     ▼
homeserver-lab
     │
     └── OpenSSH Server
             │
             └── User: mera
```

Sensitive or unnecessary network information is intentionally not included in the public screenshots of this repository.

---

## 1. Why SSH?

Until this stage, the Debian server could be administered directly through the VirtualBox console.

This works, but it is not the preferred administration method for a server.

SSH provides remote command-line access:

```text
Administrator workstation
          │
          │ encrypted SSH connection
          ▼
       Server
```

Once SSH is operational, most server administration can be performed directly from the host terminal.

This provides several advantages:

- easier copy and paste;
- better terminal usability;
- remote administration;
- encrypted communication;
- SSH key authentication;
- reduced dependency on the VirtualBox console.

The VirtualBox console remains useful as a recovery method if remote access fails.

---

## 2. OpenSSH Server

OpenSSH Server was installed on the Debian machine.

The SSH service can be inspected using:

```bash
systemctl status ssh
```

The expected state is:

```text
active (running)
```

This confirms that the SSH daemon is running.

The service is responsible for accepting incoming SSH connections.

---

## 3. Checking the SSH Listening Port

The listening network sockets can be inspected using:

```bash
ss -tulpn
```

To filter specifically for SSH:

```bash
ss -tulpn | grep :22
```

SSH normally listens on:

```text
TCP port 22
```

This verification helps distinguish between two different situations:

```text
SSH service not running
        │
        └── No SSH listener

SSH service running
        │
        └── TCP/22 listening
```

A listening port confirms that the SSH server is ready to accept network connections.

---

## 4. VirtualBox Networking

The VM initially used VirtualBox NAT networking.

For the lab, the network configuration was later changed to:

```text
Bridged Adapter
```

With bridged networking, the virtual machine behaves more like another machine connected to the local network.

Conceptually:

```text
                     Local Network
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
       Windows Host                Debian VM
                               homeserver-lab
```

This makes direct communication between the host and VM easier to understand and closer to the networking model of a physical homeserver.

---

## 5. Checking the VM Network Address

The Debian network interfaces can be inspected with:

```bash
ip -br a
```

The main VirtualBox network interface is:

```text
enp0s3
```

The command displays the address currently assigned to that interface.

The routing table can also be inspected with:

```bash
ip route
```

This provides information about:

- the default gateway;
- the active interface;
- the local network route.

Actual addresses are intentionally omitted from this documentation because they are not required to reproduce the configuration.

---

## 6. Testing Network Connectivity

Before testing SSH authentication, basic network connectivity was verified from the Windows host.

From PowerShell:

```powershell
ping <SERVER-IP>
```

A successful response confirms that the host can reach the VM at the IP layer.

This separates network troubleshooting from SSH troubleshooting.

The diagnostic logic becomes:

```text
Can the host reach the server?
        │
        ├── No
        │   └── Investigate network configuration
        │
        └── Yes
            └── Continue with SSH testing
```

In this lab, the ping test completed successfully with no packet loss.

---

## 7. First SSH Connection

The first SSH connection was initiated from Windows PowerShell using:

```powershell
ssh mera@<SERVER-IP>
```

The syntax is:

```text
ssh USER@HOST
```

where:

```text
USER = Linux account
HOST = server address or hostname
```

On the first connection, the SSH client displayed a warning indicating that the authenticity of the host could not yet be established.

This is expected when connecting to a new SSH server for the first time.

---

## 8. SSH Host Fingerprint

The first SSH connection displayed an Ed25519 host fingerprint.

Instead of accepting it immediately, the fingerprint was independently checked directly on the Debian server.

The server fingerprint was displayed using:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

This produces an SHA-256 fingerprint corresponding to the server's Ed25519 host key.

The value displayed on Debian was compared with the fingerprint displayed by the Windows SSH client.

The two fingerprints matched.

Only after confirming the match was the host accepted.

---

## 9. Why Verify the Host Fingerprint?

The SSH host key identifies the server.

During a first connection, the client does not yet know whether the remote machine presenting itself is the expected server.

Fingerprint verification provides an independent way to confirm:

```text
SSH client fingerprint
          │
          │ compare
          ▼
Server fingerprint
```

If the values match, the host key presented over the network corresponds to the key stored on the server being administered.

After accepting the host, the client stores information about it in:

```text
known_hosts
```

Future connections can then detect unexpected host-key changes.

---

## 10. Accepting the Host

After verifying the fingerprint, the connection prompt could safely be accepted with:

```text
yes
```

The full word was required.

Entering only:

```text
y
```

did not confirm the prompt.

Once accepted, the host key was added to the Windows SSH client's known hosts.

This step normally occurs only on the first connection unless the server host key later changes.

---

## 11. Password Authentication Failure

After establishing network connectivity and reaching the SSH server, authentication initially failed.

The important distinction was that SSH itself was already working.

The server logs contained messages indicating:

```text
authentication failure
```

and:

```text
Failed password for mera
```

This meant:

```text
Network             OK
SSH server          OK
TCP/22              OK
User recognized     OK
Password auth       FAILED
```

Therefore, continuing to troubleshoot the network would not have addressed the actual problem.

---

## 12. Inspecting SSH Logs

The SSH service logs were inspected directly on Debian using:

```bash
sudo journalctl -u ssh -n 50 --no-pager
```

This displayed recent events generated by the SSH service.

The logs showed failed password authentication attempts for the expected user.

This confirmed that the connection was reaching the server correctly.

`journalctl` therefore provided the information needed to move the troubleshooting process from:

```text
"SSH does not work"
```

to the much more precise:

```text
"SSH works, but password authentication fails."
```

---

## 13. Authentication Problem Root Cause

The password authentication issue was caused by a keyboard input difference between the Debian console and Windows.

The password contained numeric characters.

During the initial Debian configuration, those characters were not entered as expected because the console keyboard behavior differed from the Windows environment.

As a result, the stored password did not match the password that was later entered from the Windows SSH client.

The account password was reset on Debian:

```bash
sudo passwd mera
```

After setting a known password and retrying:

```powershell
ssh mera@<SERVER-IP>
```

authentication succeeded.

---

## 14. Troubleshooting Lesson

This incident demonstrates the importance of separating connection problems from authentication problems.

### Network or service problem

Symptoms may include:

```text
Connection timed out
```

or:

```text
Connection refused
```

Possible areas to investigate:

```text
IP addressing
routing
firewall
SSH service
listening port
```

### Authentication problem

Symptoms may include:

```text
Permission denied
```

or server logs such as:

```text
Failed password
```

Possible areas to investigate:

```text
username
password
SSH authentication policy
account state
SSH keys
```

The error message and server logs should determine the troubleshooting path.

---

## 15. Moving to SSH Key Authentication

Password authentication was sufficient to establish the initial connection, but the next objective was to configure public-key authentication.

SSH key authentication uses a pair of cryptographic keys:

```text
Private key
    │
    │ remains on client
    │
    └──────────────┐
                   │ authentication
                   ▼
              SSH Server
                   │
                   └── Public key
                       authorized on server
```

The private key must remain private.

The public key is designed to be copied to servers where access should be authorized.

---

## 16. Generating an Ed25519 Key Pair

The SSH key pair was generated on the Windows host.

First, the existing SSH directory was inspected:

```powershell
Get-ChildItem $env:USERPROFILE\.ssh
```

If no suitable key already exists, an Ed25519 key pair can be created using:

```powershell
ssh-keygen -t ed25519 -C "homeserver-lab"
```

The default location is typically:

```text
C:\Users\<USER>\.ssh\
```

The generated pair consists of:

```text
id_ed25519
id_ed25519.pub
```

Their roles are:

```text
id_ed25519
└── PRIVATE KEY
    └── Never copy to the server
    └── Never commit to GitHub

id_ed25519.pub
└── PUBLIC KEY
    └── Can be installed on the server
```

---

## 17. Protecting the Private Key

A passphrase was configured for the private key.

This means that possession of the private key file alone is not sufficient to immediately use it.

During authentication, the SSH client can request:

```text
Enter passphrase for key ...
```

This is different from:

```text
mera@<SERVER-IP>'s password:
```

The distinction is important.

### Account password

Authenticates against the Linux user account on the server.

### Key passphrase

Protects the private key stored on the client.

The passphrase itself is not transmitted to the SSH server.

---

## 18. Installing the Public Key

Copying and pasting the long public key directly into the VirtualBox console was impractical.

Because password-based SSH access was already working, the public key was transferred directly over SSH from Windows.

The public key can first be displayed using:

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

Instead of manually retyping it, it was sent directly to the server:

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh mera@<SERVER-IP> "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

This command:

1. reads the public key on Windows;
2. establishes an SSH connection to Debian;
3. creates `~/.ssh` if necessary;
4. appends the public key to `authorized_keys`.

The private key never leaves the Windows host.

---

## 19. authorized_keys

Authorized public keys for the user are stored in:

```text
~/.ssh/authorized_keys
```

For the `mera` account, this corresponds conceptually to:

```text
/home/mera/
└── .ssh/
    └── authorized_keys
```

When a client presents a key during authentication, OpenSSH can check whether the corresponding public key is authorized for the account.

---

## 20. SSH Directory Permissions

The SSH directory permissions were verified using:

```bash
ls -ld ~/.ssh
```

and:

```bash
ls -l ~/.ssh/authorized_keys
```

The configured permissions were:

```text
~/.ssh
└── 700

~/.ssh/authorized_keys
└── 600
```

Symbolically:

```text
~/.ssh
drwx------

authorized_keys
-rw-------
```

This means:

### `.ssh` — 700

```text
Owner  → rwx
Group  → ---
Others → ---
```

### `authorized_keys` — 600

```text
Owner  → rw-
Group  → ---
Others → ---
```

These restrictive permissions prevent other local users from modifying the SSH authorization files.

---

## 21. Testing Key Authentication

After installing the public key, a new SSH connection was opened from Windows:

```powershell
ssh mera@<SERVER-IP>
```

Instead of requesting the Linux account password, the client requested the passphrase protecting the private key.

The successful authentication sequence became:

```text
Windows
   │
   │ id_ed25519
   │ + key passphrase
   ▼
SSH authentication
   │
   ▼
homeserver-lab
   │
   ▼
mera@homeserver-lab
```

This confirmed that public-key authentication was operational.

---

## 22. Verifying the Remote Session

Once connected, the session can be verified with:

```bash
whoami
hostname
```

Expected results:

```text
mera
homeserver-lab
```

This confirms both:

```text
Authenticated user
        +
Remote machine identity
```

---

## 23. Inspecting Effective SSH Configuration

The effective OpenSSH server configuration was inspected using:

```bash
sudo sshd -T
```

To display only the settings relevant to authentication:

```bash
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin|pubkeyauthentication'
```

At the current stage of the lab, the relevant configuration is:

```text
pubkeyauthentication yes
passwordauthentication yes
permitrootlogin without-password
```

This means:

### Public-key authentication

```text
pubkeyauthentication yes
```

SSH key authentication is enabled.

### Password authentication

```text
passwordauthentication yes
```

Password authentication is still enabled.

### Root SSH login

```text
permitrootlogin without-password
```

Root cannot authenticate using a password, but key-based root authentication is not yet completely disabled.

This is therefore **not the final hardened SSH configuration**.

---

## 24. Current Authentication Model

The current state is:

```text
SSH Server
│
├── Public-key authentication
│   └── ENABLED
│
├── Password authentication
│   └── ENABLED
│
└── Root SSH access
    ├── Password → blocked
    └── Key      → potentially allowed
```

This configuration was intentionally left unchanged after validating key authentication.

The next security stage will harden it.

---

## 25. Why Password Authentication Was Not Disabled Immediately

SSH configuration changes can lock an administrator out of the server if applied incorrectly.

For that reason, password authentication should not be disabled until key authentication has been independently tested.

The safe sequence is:

```text
Configure public key
        │
        ▼
Open a NEW SSH session
        │
        ▼
Verify key authentication
        │
        ▼
Keep existing session available
        │
        ▼
Modify SSH configuration
        │
        ▼
Validate sshd configuration
        │
        ▼
Reload SSH
        │
        ▼
Test another new connection
```

At the current stage, key authentication has been successfully tested.

The hardening itself remains a future task.

---

## 26. Planned SSH Hardening

The target configuration for the next stage is:

```text
PubkeyAuthentication yes
PasswordAuthentication no
PermitRootLogin no
```

The intended authentication architecture will therefore become:

```text
Windows Client
      │
      │ Ed25519 key
      ▼
OpenSSH Server
      │
      ├── Key authentication → allowed
      ├── Password login     → disabled
      └── Root SSH login     → disabled
```

This configuration has **not yet been applied**.

It remains part of the project roadmap.

---

## 27. Important Lockout Precaution

When SSH hardening is performed later, the currently working SSH session should remain open.

A second terminal should be used to test the new configuration.

Conceptually:

```text
Terminal 1
└── Existing working SSH session
    └── Keep open as recovery access

Terminal 2
└── Test new SSH connection
```

Only after the second connection succeeds should the original session be closed.

The VirtualBox console also remains available as a local recovery method for this VM.

---

## 28. SSH Diagnostic Commands

### Check the SSH service

```bash
systemctl status ssh
```

### Check whether TCP/22 is listening

```bash
ss -tulpn | grep :22
```

### Inspect recent SSH logs

```bash
sudo journalctl -u ssh -n 50 --no-pager
```

### Inspect effective SSH configuration

```bash
sudo sshd -T
```

### Filter authentication settings

```bash
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin|pubkeyauthentication'
```

### Check SSH directory permissions

```bash
ls -ld ~/.ssh
```

### Check authorized_keys permissions

```bash
ls -l ~/.ssh/authorized_keys
```

These commands provide a useful baseline for SSH troubleshooting.

---

## 29. SSH Troubleshooting Workflow

A structured SSH troubleshooting process can be divided into layers.

### Layer 1 — Network

From the client:

```powershell
ping <SERVER-IP>
```

If the server cannot be reached, investigate the network before SSH authentication.

### Layer 2 — SSH Service

On Debian:

```bash
systemctl status ssh
```

Confirm:

```text
active (running)
```

### Layer 3 — Listening Port

```bash
ss -tulpn | grep :22
```

Confirm that SSH is listening.

### Layer 4 — Server Logs

```bash
sudo journalctl -u ssh -n 50 --no-pager
```

Look for authentication or connection errors.

### Layer 5 — Account

Verify the user:

```bash
getent passwd mera
```

### Layer 6 — SSH Authentication Configuration

```bash
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin|pubkeyauthentication'
```

### Layer 7 — Public-Key Files

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

This creates a troubleshooting sequence from the lowest-level connectivity issue to the authentication configuration.

---

## 30. Public vs Private SSH Keys

One of the most important security distinctions in SSH is between public and private keys.

### Public key

Example file:

```text
id_ed25519.pub
```

Purpose:

```text
Installed on servers where access is authorized
```

It is not considered a secret in the same way as the private key.

### Private key

Example file:

```text
id_ed25519
```

Purpose:

```text
Proves the client's identity
```

It must remain private.

The private key should never be:

- uploaded to GitHub;
- sent through chat;
- copied to the server unnecessarily;
- included in screenshots;
- committed to configuration repositories.

---

## 31. Repository Safety

SSH configuration documentation can unintentionally expose unnecessary information.

This repository therefore avoids publishing:

```text
private SSH keys
passwords
authentication secrets
public network addresses
unnecessary local network details
```

Local private IP addresses are not equivalent to public Internet addresses, but they are still omitted when they provide no technical value to the documentation.

The SSH screenshots containing local network information were therefore intentionally excluded from the repository.

---

## 32. Current SSH Architecture

The current architecture can be summarized as:

```text
┌─────────────────────────────┐
│        Windows Host         │
│                             │
│ OpenSSH Client              │
│                             │
│ Private key:                │
│ ~/.ssh/id_ed25519           │
└──────────────┬──────────────┘
               │
               │ SSH / TCP
               │
               ▼
┌─────────────────────────────┐
│       homeserver-lab        │
│          Debian 12          │
│                             │
│ OpenSSH Server              │
│                             │
│ User: mera                  │
│                             │
│ ~/.ssh/authorized_keys      │
│        ▲                    │
│        │                    │
│   Public key                │
└─────────────────────────────┘
```

The private key remains on the client.

Only the public key is installed on the server.

---

## 33. Current Security State

### Completed

```text
[✓] OpenSSH server operational
[✓] Network connectivity validated
[✓] SSH host fingerprint verified
[✓] Remote login validated
[✓] Ed25519 key pair created
[✓] Private key protected with a passphrase
[✓] Public key installed on Debian
[✓] Key authentication tested successfully
[✓] ~/.ssh permissions verified
[✓] authorized_keys permissions verified
```

### Pending

```text
[ ] Disable SSH password authentication
[ ] Completely disable root SSH login
[ ] Validate hardened sshd configuration
[ ] Review firewall rules
[ ] Review SSH exposure when remote-access services are introduced
```

---

## 34. Result

At the end of this stage:

```text
Windows
│
├── OpenSSH Client
│
├── Ed25519 private key
│   └── protected by passphrase
│
└──── SSH ─────────────────────┐
                              │
                              ▼
                       homeserver-lab
                              │
                       OpenSSH Server
                              │
                              ├── mera
                              │
                              └── authorized_keys
                                  └── Ed25519 public key
```

Remote administration no longer depends on the VirtualBox console for normal operation.

SSH key authentication is functional and ready for the next hardening stage.

---

## Commands Summary

### Debian

```bash
# Display network interfaces
ip -br a

# Display routing information
ip route

# Check SSH service status
systemctl status ssh

# Check whether SSH is listening on TCP/22
ss -tulpn | grep :22

# Display the server Ed25519 host fingerprint
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub

# Inspect recent SSH logs
sudo journalctl -u ssh -n 50 --no-pager

# Reset the user password if required
sudo passwd mera

# Check the user account
getent passwd mera

# Check SSH directory permissions
ls -ld ~/.ssh

# Check authorized_keys permissions
ls -l ~/.ssh/authorized_keys

# Inspect effective SSH authentication settings
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin|pubkeyauthentication'
```

### Windows PowerShell

```powershell
# Test network connectivity
ping <SERVER-IP>

# Connect through SSH
ssh mera@<SERVER-IP>

# Inspect existing SSH keys
Get-ChildItem $env:USERPROFILE\.ssh

# Generate an Ed25519 key pair
ssh-keygen -t ed25519 -C "homeserver-lab"

# Display the public key
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub

# Install the public key on Debian
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh mera@<SERVER-IP> "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"

# Test key authentication
ssh mera@<SERVER-IP>
```

---

## Key Takeaways

- SSH provides encrypted remote administration of the Debian server.
- Network connectivity and SSH authentication are separate troubleshooting layers.
- `systemctl status ssh` verifies the SSH service.
- `ss -tulpn` can confirm that TCP/22 is listening.
- The server host fingerprint should be verified before accepting a first SSH connection.
- `journalctl -u ssh` is useful for diagnosing authentication failures.
- The password failure encountered in this lab was caused by a keyboard input difference rather than a network or SSH service problem.
- Ed25519 public-key authentication is now functional.
- The private SSH key remains exclusively on the Windows client.
- The private key is protected with a passphrase.
- `authorized_keys` contains the public keys permitted to authenticate as the user.
- `.ssh` uses restrictive permissions.
- Password authentication is still enabled and must not be documented as hardened yet.
- Root SSH login has not yet been completely disabled.
- SSH hardening should only be performed after a working key-based connection has been independently tested.
