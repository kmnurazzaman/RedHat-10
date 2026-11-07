# SSH Hardening Guide – RHEL 10

**Author:** K M Nurazzaman  
**Date:** 10 November 2026  
**Platform:** Red Hat Enterprise Linux 10

## Overview

This guide documents essential SSH hardening practices for securing remote access to a RHEL 10 server using OpenSSH.

The configuration focuses on:

- Disabling direct root SSH login
- Enforcing public-key authentication
- Limiting authentication attempts
- Restricting SSH access
- Controlling session timeouts
- Disabling unnecessary forwarding features
- Improving SSH logging

## 1. Prerequisites

Before applying the configuration, ensure that:

- You have root or sudo privileges.
- OpenSSH server is installed.
- The SSH service is running.
- You have an active administrative SSH session.
- Public-key authentication has been configured and tested.
- Firewall rules allow SSH connections.

Check the SSH service:

```bash
sudo systemctl status sshd
```

## 2. Back Up the Existing Configuration

Before making changes, create a backup of the original SSH configuration.

```bash
sudo cp -a /etc/ssh/sshd_config \
/etc/ssh/sshd_config.backup
```

This allows you to restore the original configuration if an issue occurs.

## 3. Configure SSH Public-Key Authentication

Public-key authentication is recommended instead of password-based SSH authentication.

### Generate an ED25519 key pair

Run this command on your client machine:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

The command generates:

- Private key: `~/.ssh/id_ed25519`
- Public key: `~/.ssh/id_ed25519.pub`

**Important:** Never share your private key.

### Copy the public key to the RHEL server

```bash
ssh-copy-id username@server_ip
```

Alternatively, manually add the public key to:

```bash
~/.ssh/authorized_keys
```

Ensure the permissions are correct:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Test key-based authentication in a separate terminal before disabling password authentication.

## 4. Harden the SSH Configuration

Edit the SSH daemon configuration:

```bash
sudo vim /etc/ssh/sshd_config
```

The following is a baseline hardening configuration:

### Complete configuration

```sshconfig
# ==========================================================
# SSH HARDENING CONFIGURATION
# RHEL 10
# Author: K M Nurazzaman
# ==========================================================

# ----- Network -----

Port 22
AddressFamily inet

# ----- Authentication -----

PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
UsePAM yes

MaxAuthTries 3
MaxStartups 10:30:60
LoginGraceTime 60

PermitEmptyPasswords no

# ----- Access Control -----

# Uncomment and specify authorized users if required.
# AllowUsers alice bob carol

# Alternatively, restrict access to a specific group.
# AllowGroups sshadmins

# ----- Session Management -----

ClientAliveInterval 300
ClientAliveCountMax 2

# ----- Logging -----

LogLevel VERBOSE

# ----- Forwarding and Miscellaneous -----

X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no

# ==========================================================
```

### Configuration explanation

| Directive | Purpose |
|---|---|
| `Port 22` | Defines the SSH listening port. |
| `AddressFamily inet` | Restricts SSH to IPv4. |
| `PermitRootLogin no` | Prevents direct root SSH login. |
| `PubkeyAuthentication yes` | Enables public-key authentication. |
| `PasswordAuthentication no` | Disables password-based SSH authentication. |
| `KbdInteractiveAuthentication no` | Disables keyboard-interactive authentication. |
| `UsePAM yes` | Enables PAM integration. |
| `MaxAuthTries 3` | Limits authentication attempts per connection. |
| `MaxStartups 10:30:60` | Controls unauthenticated concurrent connections. |
| `LoginGraceTime 60` | Sets the authentication time limit in seconds. |
| `PermitEmptyPasswords no` | Rejects empty-password accounts. |
| `AllowUsers` | Restricts SSH access to specified users. |
| `AllowGroups` | Restricts SSH access to specified groups. |
| `ClientAliveInterval 300` | Sends a client-alive message after 300 seconds of inactivity. |
| `ClientAliveCountMax 2` | Disconnects after two unanswered client-alive messages. |
| `LogLevel VERBOSE` | Provides more detailed SSH authentication logging. |
| `X11Forwarding no` | Disables X11 forwarding. |
| `AllowAgentForwarding no` | Disables SSH agent forwarding. |
| `AllowTcpForwarding no` | Disables TCP port forwarding. |

**Note:** `AddressFamily inet` disables IPv6 SSH connections. Remove it if your environment requires IPv6.

`AllowUsers` and `AllowGroups` are optional access restrictions. Configure them carefully to avoid locking out administrative users.

## 5. Validate the SSH Configuration

Always validate the configuration before restarting the SSH service.

```bash
sudo sshd -t
```

If the command returns no output, the configuration has passed the syntax check.

To inspect the effective configuration:

```bash
sudo sshd -T
```

You can also check individual settings:

```bash
sudo sshd -T | grep -E \
'permitrootlogin|passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication|maxauthtries'
```

**Important:** RHEL may load additional configuration from `/etc/ssh/sshd_config.d/`. Check the effective configuration with `sshd -T` rather than assuming the main configuration file contains the final values.

## 6. Restart and Verify SSH

After successful validation, restart the SSH service:

```bash
sudo systemctl restart sshd
```

Verify that the service is running:

```bash
sudo systemctl status sshd
```

Check the listening port:

```bash
sudo ss -lntp | grep sshd
```

## 7. Test SSH Access

Open a new terminal from your client machine.

Test public-key authentication:

```bash
ssh username@server_ip
```

Verify that root SSH login is disabled:

```bash
ssh root@server_ip
```

Verify that password authentication is disabled by testing a connection without an available private key.

**Security precaution:** Do not close your existing administrative SSH session until you have successfully tested a new connection.

## 8. Firewall Configuration

If you retain the default SSH port, verify that the SSH service is permitted through the firewall.

```bash
sudo firewall-cmd --list-services
```

If SSH is not listed:

```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

Verify the result:

```bash
sudo firewall-cmd --list-all
```

## 9. SSH Authentication Logs

Review SSH authentication events to investigate successful and failed login attempts.

```bash
sudo journalctl -u sshd
```

View recent SSH service logs:

```bash
sudo journalctl -u sshd -n 50 --no-pager
```

Follow logs in real time:

```bash
sudo journalctl -u sshd -f
```

## 10. Rollback Procedure

If the SSH service becomes inaccessible after a configuration change, restore the backup using console access or another administrative session.

```bash
sudo cp -a /etc/ssh/sshd_config.backup \
/etc/ssh/sshd_config
```

Validate the restored configuration:

```bash
sudo sshd -t
```

Restart SSH:

```bash
sudo systemctl restart sshd
```

Remember to check configuration drop-in files if they also contain settings that need to be restored.

## 11. Final SSH Hardening Checklist

- [ ] Back up the original SSH configuration.
- [ ] Generate an ED25519 key pair.
- [ ] Configure public-key authentication.
- [ ] Test key-based login.
- [ ] Disable direct root SSH login.
- [ ] Disable password authentication.
- [ ] Disable keyboard-interactive authentication if not required.
- [ ] Limit authentication attempts.
- [ ] Configure appropriate SSH access restrictions.
- [ ] Disable unnecessary forwarding features.
- [ ] Validate the configuration with `sshd -t`.
- [ ] Verify effective settings with `sshd -T`.
- [ ] Restart and verify the SSH service.
- [ ] Test a new SSH connection before closing the existing session.

## Conclusion

SSH hardening is an important part of securing Linux servers. By enforcing key-based authentication, restricting privileged access, controlling authentication attempts, and disabling unnecessary SSH features, administrators can reduce the attack surface of RHEL 10 systems.

These configurations provide a practical baseline for Linux server security in enterprise and production environments.
