# SSH Commands

## Check the SSH service

Display the current OpenSSH server status:

```bash
systemctl status ssh
```

Expected state:

```text
active (running)
```

If the service is not running, start it:

```bash
sudo systemctl start ssh
```

Restart the service:

```bash
sudo systemctl restart ssh
```

Enable SSH automatically at boot:

```bash
sudo systemctl enable ssh
```

---

## Check the SSH listening port

Display processes listening on TCP port 22:

```bash
ss -tulpn | grep :22
```

SSH normally listens on:

```text
TCP/22
```

This command helps confirm that the SSH server is actually accepting network connections.

---

## Check the server network configuration

Display network interfaces:

```bash
ip -br a
```

Display the routing table:

```bash
ip route
```

The primary VirtualBox interface used in this lab is:

```text
enp0s3
```

---

## Test connectivity from Windows

From Windows PowerShell:

```powershell
ping <SERVER-IP>
```

This verifies basic network connectivity before troubleshooting SSH itself.

A successful ping does not guarantee that SSH authentication works, but it confirms that the server is reachable at the IP layer.

---

## Connect from Windows

Connect to the Debian server:

```powershell
ssh mera@<SERVER-IP>
```

General syntax:

```text
ssh USER@HOST
```

For this lab:

```text
USER    mera
HOST    <SERVER-IP>
```

---

## Verify the SSH host fingerprint

During the first SSH connection, the client displays the server's host-key fingerprint.

Verify the Ed25519 fingerprint directly on Debian:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Compare the SHA-256 fingerprint displayed on Debian with the fingerprint displayed by the Windows SSH client.

If they match, the host can be accepted.

At the SSH prompt, enter:

```text
yes
```

The accepted host key is then stored by the client in its `known_hosts` file.

---

## Check SSH logs

Display recent SSH service logs:

```bash
sudo journalctl -u ssh -n 50 --no-pager
```

This is useful for diagnosing errors such as:

```text
Failed password
Authentication failure
Connection errors
```

Display more SSH logs:

```bash
sudo journalctl -u ssh --no-pager
```

Follow SSH logs in real time:

```bash
sudo journalctl -u ssh -f
```

Press:

```text
Ctrl+C
```

to stop following the logs.

---

## Check the Linux account

Verify that the `mera` account exists:

```bash
getent passwd mera
```

Display identity and group information:

```bash
id mera
```

---

## Change the Linux account password

Reset or change the password for `mera`:

```bash
sudo passwd mera
```

This changes the Linux account password.

It does not change the passphrase protecting an SSH private key.

---

## Password vs SSH key passphrase

A prompt similar to:

```text
mera@<SERVER-IP>'s password:
```

requests the Linux account password.

A prompt similar to:

```text
Enter passphrase for key ...
```

requests the passphrase protecting the private SSH key stored on the client.

These are two separate authentication elements.

---

## Inspect the effective SSH configuration

Display the effective OpenSSH server configuration:

```bash
sudo sshd -T
```

Filter the most relevant authentication settings:

```bash
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin|pubkeyauthentication'
```

Current lab state before SSH hardening:

```text
pubkeyauthentication yes
passwordauthentication yes
permitrootlogin without-password
```

This means:

```text
Public-key authentication    Enabled
Password authentication      Enabled
Root password authentication Disabled
Root key authentication      Potentially allowed
```

This is not yet the final hardened configuration.

---

## Generate an Ed25519 SSH key

From Windows PowerShell:

```powershell
ssh-keygen -t ed25519 -C "homeserver-lab"
```

The generated key pair normally consists of:

```text
id_ed25519
id_ed25519.pub
```

The files have different roles:

```text
id_ed25519
└── Private key

id_ed25519.pub
└── Public key
```

Never share, upload or commit the private key:

```text
id_ed25519
```

---

## List SSH files on Windows

From PowerShell:

```powershell
Get-ChildItem $env:USERPROFILE\.ssh
```

This displays the contents of the current Windows user's SSH directory.

---

## Display the public key

From PowerShell:

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

The `.pub` file contains the public key and can be installed on servers where authentication should be authorized.

---

## Install the public key on Debian

From Windows PowerShell:

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh mera@<SERVER-IP> "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

This command:

```text
1. Reads the public key on Windows
2. Opens an SSH connection to Debian
3. Creates ~/.ssh if necessary
4. Appends the public key to authorized_keys
```

Only the public key is transferred.

The private key remains on the Windows client.

---

## Check authorized SSH keys

On Debian:

```bash
ls -l ~/.ssh/authorized_keys
```

To inspect the file when necessary:

```bash
cat ~/.ssh/authorized_keys
```

The contents are public keys, not private keys.

Even though public keys are not secret credentials, they do not need to be published in the repository.

---

## Check SSH directory permissions

Inspect the `.ssh` directory:

```bash
ls -ld ~/.ssh
```

Expected mode:

```text
700
```

Equivalent symbolic permissions:

```text
drwx------
```

Inspect `authorized_keys`:

```bash
ls -l ~/.ssh/authorized_keys
```

Expected mode:

```text
600
```

Equivalent symbolic permissions:

```text
-rw-------
```

---

## Correct SSH file permissions

If necessary, configure the `.ssh` directory:

```bash
chmod 700 ~/.ssh
```

Configure `authorized_keys`:

```bash
chmod 600 ~/.ssh/authorized_keys
```

Verify:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

---

## Test key-based authentication

Open a new Windows PowerShell terminal.

Connect again:

```powershell
ssh mera@<SERVER-IP>
```

If the private key is protected with a passphrase, the client may request:

```text
Enter passphrase for key ...
```

If authentication succeeds without requesting the Linux account password, public-key authentication is operational.

---

## Verify the remote session

After connecting:

```bash
whoami
```

Expected:

```text
mera
```

Verify the server:

```bash
hostname
```

Expected:

```text
homeserver-lab
```

A quick combined verification can be performed with:

```bash
whoami
hostname
```

---

## SSH troubleshooting workflow

When SSH does not work, troubleshoot each layer separately.

### 1. Check network connectivity

From Windows:

```powershell
ping <SERVER-IP>
```

### 2. Check the SSH service

On Debian:

```bash
systemctl status ssh
```

### 3. Check the listening port

```bash
ss -tulpn | grep :22
```

### 4. Check SSH logs

```bash
sudo journalctl -u ssh -n 50 --no-pager
```

### 5. Check the user account

```bash
getent passwd mera
id mera
```

### 6. Check the SSH configuration

```bash
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin|pubkeyauthentication'
```

### 7. Check public-key files

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

The troubleshooting order is therefore:

```text
Network
   ↓
SSH service
   ↓
Listening port
   ↓
Server logs
   ↓
User account
   ↓
Authentication configuration
   ↓
SSH keys and permissions
```

---

## Diagnose password authentication failures

If the client returns:

```text
Permission denied
```

check the server logs:

```bash
sudo journalctl -u ssh -n 50 --no-pager
```

If the logs contain:

```text
Failed password for mera
```

the connection successfully reached the SSH server.

The problem is then related to authentication rather than basic network connectivity.

Possible causes include:

```text
Incorrect password
Incorrect username
Keyboard layout/input difference
Account configuration
SSH authentication policy
```

---

## Current SSH Authentication State

The current lab configuration is:

```text
Public-key authentication
└── Enabled and tested

Password authentication
└── Still enabled

Root password SSH login
└── Disabled

Root key SSH login
└── Not yet completely disabled
```

---

## Future SSH Hardening

The target configuration for the next hardening stage is:

```text
PubkeyAuthentication yes
PasswordAuthentication no
PermitRootLogin no
```

These changes have not yet been applied.

Before disabling password authentication, always verify that public-key authentication works in a separate SSH session.

---

## Safe SSH Hardening Workflow

When modifying SSH authentication settings later:

```text
Terminal 1
└── Keep the existing working SSH session open

Terminal 2
└── Test a new SSH connection
```

The intended workflow is:

```text
Working SSH session
        ↓
Modify configuration
        ↓
Validate configuration
        ↓
Reload SSH
        ↓
Open a second SSH session
        ↓
Verify authentication
        ↓
Only then close the original session
```

The VirtualBox console can also be used as recovery access if the SSH configuration becomes invalid.

---

## Security Notes

Never commit the private SSH key:

```text
id_ed25519
```

Never commit:

```text
passwords
private keys
authentication tokens
secrets
```

The public key:

```text
id_ed25519.pub
```

is not a secret like the private key, but publishing it is unnecessary for this repository.

Local IP addresses should also be replaced in reusable documentation with:

```text
<SERVER-IP>
```

when the actual address is not required.

---

## Quick Reference

### Debian

```bash
# Check SSH service
systemctl status ssh

# Check TCP/22
ss -tulpn | grep :22

# Check network interfaces
ip -br a

# Check routing
ip route

# Verify server host fingerprint
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub

# Inspect recent SSH logs
sudo journalctl -u ssh -n 50 --no-pager

# Follow SSH logs
sudo journalctl -u ssh -f

# Check the user
getent passwd mera
id mera

# Change the user password
sudo passwd mera

# Check effective SSH authentication settings
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin|pubkeyauthentication'

# Check SSH permissions
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys

# Correct SSH permissions if necessary
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### Windows PowerShell

```powershell
# Test connectivity
ping <SERVER-IP>

# Connect to the server
ssh mera@<SERVER-IP>

# List existing SSH files
Get-ChildItem $env:USERPROFILE\.ssh

# Generate an Ed25519 key pair
ssh-keygen -t ed25519 -C "homeserver-lab"

# Display the public key
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub

# Install the public key on Debian
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh mera@<SERVER-IP> "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"

# Test key-based authentication
ssh mera@<SERVER-IP>
```
