# 02. SSH Setup

This document describes how SSH access was configured for the LibreWiki Homelab Ubuntu Server VM.

The goal of this stage was to move from local console administration inside UTM to normal remote administration from the macOS terminal.

The final target was:

```text
MacBook
   │
   │ SSH
   ▼
Ubuntu Server VM
```

with public-key authentication enabled.

## Environment

SSH client:

- macOS built-in OpenSSH client

SSH server:

- Ubuntu Server 26.04 LTS
- OpenSSH Server installed during the Ubuntu installation process

VM:

```text
librewiki-ubuntu
```

## Confirming the SSH Server

The Ubuntu installer was configured with:

```text
Install OpenSSH server
```

enabled.

After installation, the SSH service can be checked with:

```bash
systemctl status ssh
```

A healthy service should report:

```text
active (running)
```

The service can also be checked more briefly with:

```bash
systemctl is-active ssh
```

Expected result:

```text
active
```

## Finding the Server IP Address

The VM's IPv4 address can be displayed with:

```bash
hostname -I
```

or inspected in detail with:

```bash
ip addr
```

The address used for SSH is the non-loopback IPv4 address associated with the VM's main network interface.

For example:

```text
192.168.64.x
```

The following address should not be used for remote SSH access:

```text
127.0.0.1
```

because it refers to the server itself.

IPv6 addresses were also assigned, but IPv4 was used for the initial homelab configuration because it is simpler to inspect and troubleshoot.

## First Password-Based Login

Before configuring key authentication, the server was accessed using the Ubuntu account password.

From the macOS terminal:

```bash
ssh steven@192.168.64.x
```

The actual VM address should be substituted.

On the first connection, OpenSSH may display a host authenticity warning similar to:

```text
The authenticity of host '192.168.64.x' can't be established.
```

After confirming the host fingerprint, the server is added to:

```text
~/.ssh/known_hosts
```

on the Mac.

Password-based login was retained temporarily while public-key authentication was being configured.

## SSH Key Authentication

The preferred authentication method for this homelab is public-key authentication.

The model is:

```text
Mac
├── private key
└── public key
        │
        ▼
Ubuntu Server
└── ~/.ssh/authorized_keys
```

The private key remains only on the Mac.

The public key is copied to the Ubuntu server.

## Checking Existing Keys on macOS

Before generating a new key, existing SSH keys were checked:

```bash
ls -la ~/.ssh
```

Typical Ed25519 key files are:

```text
id_ed25519
id_ed25519.pub
```

In this environment, an existing SSH key was already available on the Mac, so a new one was not required.

If no suitable key exists, one can be generated with:

```bash
ssh-keygen -t ed25519
```

The default location is:

```text
~/.ssh/id_ed25519
```

and the corresponding public key is:

```text
~/.ssh/id_ed25519.pub
```

## A Mistake During Initial Setup

An SSH key was initially generated inside the Ubuntu VM by mistake.

This does not damage the configuration.

It simply creates a key pair belonging to the Ubuntu server rather than the Mac.

That key can potentially be useful later if the Ubuntu application server needs to connect to another machine, such as a future Rocky Linux database server.

For example:

```text
Mac
 │
 │ Mac SSH key
 ▼
Ubuntu Server
 │
 │ Ubuntu SSH key
 ▼
Rocky Linux
```

The mistakenly generated server-side key therefore did not need to be removed.

## Copying the Mac Public Key to Ubuntu

The existing Mac public key was installed on the Ubuntu server using:

```bash
ssh-copy-id steven@192.168.64.x
```

This command authenticates using the existing account password and then adds the selected public key to:

```text
~/.ssh/authorized_keys
```

on the Ubuntu server.

The expected server-side structure becomes:

```text
/home/steven/
└── .ssh/
    └── authorized_keys
```

## Testing Public-Key Authentication

After installing the public key, a new SSH connection was opened:

```bash
ssh steven@192.168.64.x
```

Successful key authentication means the Ubuntu account password is no longer required for the login.

If the private key itself has a passphrase, macOS may request the key passphrase instead.

The connection can be verified with:

```bash
whoami
```

Expected result:

```text
steven
```

The hostname can also be checked:

```bash
hostname
```

Expected result:

```text
librewiki-ubuntu
```

## Inspecting `authorized_keys`

The installed public keys can be inspected on the server with:

```bash
cat ~/.ssh/authorized_keys
```

A typical Ed25519 key starts with:

```text
ssh-ed25519
```

The private key must never be copied to the server or stored in this repository.

## File Permissions

OpenSSH expects appropriate permissions on the user's SSH directory and key files.

Typical permissions are:

```text
~/.ssh                 700
~/.ssh/authorized_keys 600
```

They can be enforced with:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Ownership can be checked with:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

The files should belong to the intended Linux user.

## Why Password Login Was Not Disabled Immediately

Password authentication was deliberately left enabled during the initial setup.

Disabling it too early creates a risk of locking the administrator out of the VM if:

- the wrong public key was installed
- permissions are incorrect
- SSH configuration contains an error
- the key becomes unavailable

The safer sequence is:

```text
Password login works
        ↓
Public key installed
        ↓
Key login tested
        ↓
Key login tested again
        ↓
Only then consider disabling passwords
```

Because this is a local UTM homelab, password login is not currently a major exposure.

Hardening can be introduced later as part of the security exercises.

## SSH Configuration

The main OpenSSH server configuration file is:

```text
/etc/ssh/sshd_config
```

Additional distribution-managed configuration may also exist under:

```text
/etc/ssh/sshd_config.d/
```

Before modifying SSH settings, the active configuration can be inspected.

For example:

```bash
sudo sshd -T
```

This displays the effective configuration after all configuration files have been processed.

Useful values include:

```text
passwordauthentication
pubkeyauthentication
permitrootlogin
```

## Testing Configuration Changes Safely

If SSH configuration is modified later, the configuration syntax should be checked before restarting the service:

```bash
sudo sshd -t
```

If the command returns no output, the syntax check passed.

Then the service can be reloaded:

```bash
sudo systemctl reload ssh
```

A second SSH session should remain open while testing major authentication changes.

This avoids losing all remote access if the new configuration is incorrect.

## Root Login

Administrative access is performed through the normal user account:

```text
steven
```

and privilege escalation uses:

```bash
sudo
```

Direct SSH login as `root` is unnecessary for this homelab.

This provides a cleaner separation between:

```text
normal login
```

and:

```text
privileged administration
```

## Useful SSH Commands

Connect to the VM:

```bash
ssh steven@192.168.64.x
```

Display the current user:

```bash
whoami
```

Display the server hostname:

```bash
hostname
```

Display remote connection information:

```bash
who
```

Inspect SSH service status:

```bash
systemctl status ssh
```

Inspect SSH logs:

```bash
journalctl -u ssh
```

Follow SSH logs in real time:

```bash
sudo journalctl -u ssh -f
```

## File Transfer with SCP

Once SSH is working, files can also be copied between macOS and the server using SCP.

For example:

```bash
scp example.txt steven@192.168.64.x:/tmp/
```

This became useful later when transferring the generated MediaWiki:

```text
LocalSettings.php
```

from the Mac browser download directory to the Ubuntu VM.

The reverse direction is also possible:

```bash
scp steven@192.168.64.x:/path/to/file .
```

## Why SSH Matters for the Homelab

Once SSH was configured, the UTM console stopped being the primary administration interface.

The normal workflow became:

```text
Mac terminal
    │
    │ SSH
    ▼
Ubuntu Server
    │
    ├── package management
    ├── configuration editing
    ├── service management
    ├── log inspection
    └── troubleshooting
```

This is closer to the way Linux servers are commonly managed in real environments than continuously using a virtual machine's local console.

## Current Security Model

At this stage:

- SSH public-key authentication works
- the account password still exists
- direct root SSH login is unnecessary
- the VM is reachable only through the local virtualized network
- private SSH keys remain on the Mac
- no private keys are committed to Git

The current goal is not maximum hardening.

The goal is to establish a working and understandable remote administration environment that can be hardened incrementally later.

## Future Improvements

Possible SSH-related exercises include:

- disable password authentication
- explicitly disable root login
- configure `AllowUsers`
- test SSH connection limits
- configure fail2ban
- create a separate administrative account
- use an SSH agent
- create per-server SSH keys
- configure `~/.ssh/config` aliases
- test SSH access between Ubuntu and Rocky Linux
- restrict SSH access using firewall rules

A future macOS SSH client configuration might look like:

```sshconfig
Host librewiki
    HostName 192.168.64.x
    User steven
    IdentityFile ~/.ssh/id_ed25519
```

This would allow the server to be accessed simply with:

```bash
ssh librewiki
```

## Result

The SSH stage is complete when:

- OpenSSH Server is running
- the VM can be reached from macOS
- password-based SSH login works as a fallback
- the Mac public key is present in `authorized_keys`
- public-key authentication works successfully
- the private key remains only on the Mac
- normal server administration can be performed without using the UTM console

The next stage is installing and configuring Nginx and PHP-FPM for the MediaWiki application.
