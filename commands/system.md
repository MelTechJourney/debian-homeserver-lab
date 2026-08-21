# Linux System Commands

## Identity and system information

```bash
hostname
```

Displays the hostname of the current machine.

```bash
whoami
```

Displays the currently authenticated user.

```bash
id
```

Displays the current user's UID, primary group and supplementary groups.

```bash
id mera
```

Displays identity and group information for the user `mera`.

```bash
uname -a
```

Displays kernel and system information.

```bash
cat /etc/os-release
```

Displays Debian release information.

---

## Network information

```bash
ip -br a
```

Displays a concise overview of network interfaces and IP addresses.

```bash
ip route
```

Displays the routing table and default gateway.

```bash
ping <IP>
```

Tests network connectivity to another host.

Example:

```bash
ping 192.168.1.10
```

---

## Service management

```bash
systemctl status <service>
```

Displays the state of a systemd service.

Example:

```bash
systemctl status ssh
```

```bash
sudo systemctl start <service>
```

Starts a service.

```bash
sudo systemctl stop <service>
```

Stops a service.

```bash
sudo systemctl restart <service>
```

Restarts a service.

```bash
sudo systemctl enable <service>
```

Enables a service at boot.

```bash
sudo systemctl disable <service>
```

Disables automatic startup.

---

## Logs

```bash
journalctl
```

Displays systemd logs.

```bash
sudo journalctl -u <service>
```

Displays logs for a specific service.

Example:

```bash
sudo journalctl -u ssh
```

```bash
sudo journalctl -u ssh -n 50 --no-pager
```

Displays the 50 most recent SSH log entries without opening the pager.

---

## Package management

```bash
sudo apt update
```

Refreshes package metadata.

```bash
sudo apt upgrade
```

Upgrades installed packages.

```bash
sudo apt install <package>
```

Installs a package.

```bash
sudo apt remove <package>
```

Removes a package.

```bash
apt search <package>
```

Searches available packages.

```bash
apt show <package>
```

Displays package information.

---

## Power management

```bash
sudo reboot
```

Restarts the server.

```bash
sudo poweroff
```

Shuts down the server.
