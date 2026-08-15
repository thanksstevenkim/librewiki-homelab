# 01. VM Setup

This document describes how the first virtual machine for the LibreWiki Homelab was created.

The initial goal was deliberately simple:

> Create a clean Ubuntu Server VM on Apple Silicon, make sure networking works, and prepare it for later MediaWiki installation.

The VM was created on a MacBook Air using UTM.

## Host Environment

Host machine:

- MacBook Air 13"
- Apple M5
- macOS
- Apple Silicon / ARM64

Virtualization software:

- UTM

Because the host uses Apple Silicon, the guest operating system was also installed as **ARM64** rather than x86_64.

Using an ARM64 guest avoids unnecessary CPU emulation and allows the VM to run using hardware virtualization.

## Guest Operating System

The selected guest OS was:

- Ubuntu Server 26.04 LTS
- ARM64 architecture

Ubuntu Desktop was intentionally not used.

This homelab is intended to simulate a server environment, so a graphical desktop environment is unnecessary. Ubuntu Server provides a smaller and simpler environment that can be administered through the console and SSH.

## Creating the VM

In UTM:

1. Create a new virtual machine.
2. Select **Virtualize**.
3. Select **Linux**.
4. Choose the Ubuntu Server ARM64 ISO.
5. Configure the virtual hardware.
6. Start the VM and run the Ubuntu Server installer.

### Boot Image

The VM should boot from an actual Ubuntu `.iso` image.

A `.torrent` file is not a bootable installer. It is only metadata used by a BitTorrent client to download the ISO.

The installation image should have a filename similar to:

```text
ubuntu-26.04-live-server-arm64.iso
```

## VM Resources

The initial VM configuration was:

```text
Memory: 4096 MiB
Disk:   64 GiB
CPU:    UTM default
```

### Memory

4 GiB was selected because the first stage of the project runs several services on the same VM:

- Nginx
- PHP-FPM
- MediaWiki
- MariaDB

This is more than enough for a small experimental wiki while leaving room for additional VMs later.

### CPU

The CPU core count was left at the UTM default.

There was no reason to allocate a large number of cores for the initial MediaWiki environment.

### Disk

The default 64 GiB virtual disk was kept.

The initial installation does not require this much space, but additional capacity is useful for:

- MediaWiki files
- database growth
- logs
- temporary XML imports
- backups
- packages installed during experimentation

The virtual disk can grow as data is written, so the configured maximum size does not necessarily correspond to immediately consumed host storage.

## Display Settings

Display output was left enabled during installation.

Hardware OpenGL acceleration was left disabled because the guest is a server without a graphical desktop environment.

After installation, most administration is performed over SSH, so graphical acceleration is unnecessary.

## Virtualization Engine

UTM's default virtualization configuration was kept.

There was no project-specific need to enable experimental virtualization options.

## Shared Directory

No macOS shared directory was configured during the initial setup.

Files can instead be transferred using normal server administration tools such as:

```bash
scp
```

This also provides more realistic practice for managing a remote Linux server.

A UTM shared directory can still be added later if needed.

## VM Name

The VM was named:

```text
librewiki-ubuntu
```

The name reflects both the project and the guest operating system.

Future VMs can follow the same role-oriented naming convention, for example:

```text
librewiki-ubuntu
librewiki-db-rocky
librewiki-monitor
```

The exact naming scheme may change as the infrastructure becomes more structured.

## Ubuntu Installation

Most Ubuntu Server installer options were left at their defaults.

This included:

- DHCP network configuration
- default package mirror
- default storage layout
- no proxy
- no additional Featured Server Snaps

The goal at this stage was to install a minimal and predictable Ubuntu Server environment.

### OpenSSH Server

During installation:

```text
Install OpenSSH server
```

was enabled.

This allows the VM to be administered from the macOS terminal after installation.

### Import SSH Key

The installer option to automatically import an SSH key was skipped.

SSH public-key authentication was configured manually after installation instead.

This made it possible to practice the normal SSH key setup process directly.

### Featured Server Snaps

No Featured Server Snaps were selected.

Services such as Docker or other server applications will be installed manually later when they are actually needed.

This avoids adding software to the VM without understanding why it is present.

## Installation Media

After the Ubuntu installer finishes, it displays a message similar to:

```text
Remove the installation medium, then press Enter
```

In a physical computer, this means removing the installation USB drive.

In UTM, the equivalent action is to detach the Ubuntu installation ISO from the virtual CD/DVD drive.

If the ISO remains attached and the VM boots from it again, the Ubuntu installer may start again.

The correct procedure is:

1. Shut down the VM if necessary.
2. Open the VM configuration in UTM.
3. Locate the virtual CD/DVD drive.
4. Eject or detach the Ubuntu ISO.
5. Boot the VM again.

The VM should then boot from the installed virtual disk.

## First Boot

A successful installation eventually reaches a console login prompt similar to:

```text
librewiki-ubuntu login:
```

This screen is the Linux console or TTY.

Ubuntu Server does not install a desktop GUI by default, so a text-based login screen is expected.

After logging in, the shell prompt appears:

```text
steven@librewiki-ubuntu:~$
```

At this point the VM installation itself is complete.

## Initial Package Update

After the first login, the installed packages were updated:

```bash
sudo apt update
sudo apt upgrade
```

This ensures that the VM starts with the latest packages available for the installed Ubuntu release.

## Checking the Hostname

The server hostname can be checked with:

```bash
hostname
```

More detailed information is available through:

```bash
hostnamectl
```

The expected hostname for this VM is:

```text
librewiki-ubuntu
```

## Checking Network Configuration

Network interfaces can be inspected with:

```bash
ip addr
```

or the shorter form:

```bash
ip a
```

The IPv4 address appears after `inet` on the main network interface.

For example:

```text
inet 192.168.64.x/24
```

The loopback address:

```text
127.0.0.1
```

should not be used when connecting from macOS.

A simpler command for displaying assigned IP addresses is:

```bash
hostname -I
```

### Routing

The routing table can be checked with:

```bash
ip route
```

A typical default route looks similar to:

```text
default via 192.168.64.1 dev enp0s1
```

### Internet Connectivity

Basic network connectivity can be tested with:

```bash
ping -c 4 1.1.1.1
```

DNS resolution can then be tested separately:

```bash
ping -c 4 ubuntu.com
```

If both succeed, the VM has:

- an assigned IP address
- working routing
- external connectivity
- working DNS resolution

## Connecting from macOS

Once the Ubuntu VM has an IPv4 address, it can be accessed from the Mac terminal.

Example:

```bash
ssh steven@192.168.64.x
```

The actual VM address should be substituted.

Successful SSH access confirms that the following path works:

```text
macOS
  │
  │ SSH / TCP 22
  ▼
UTM virtual network
  │
  ▼
Ubuntu Server
```

At this point, most further configuration can be performed through the macOS terminal rather than the UTM console.

## Notes on IP Addresses

The initial VM uses DHCP.

This means the address assigned to the VM is not guaranteed to remain unchanged forever.

A static address is not necessary during the first stage of the project.

Stable addressing will become more important after additional machines are added, especially when MediaWiki and MariaDB are separated onto different VMs.

At that point, options such as the following can be evaluated:

- static guest addresses
- DHCP reservations
- separate virtual networks
- application and database subnets

## Initial Architecture

At the end of this stage, the environment looks like:

```text
MacBook Air M5
│
├── macOS
│
└── UTM
    │
    └── librewiki-ubuntu
        └── Ubuntu Server 26.04 LTS ARM64
```

Only one VM exists at this point.

The deliberate approach is to begin with the smallest useful architecture and add complexity only when the project provides a reason to do so.

## Result

The VM setup stage is considered complete when all of the following are true:

- Ubuntu Server boots from the virtual disk
- the installation ISO is detached
- login through the local console works
- the VM receives an IPv4 address
- internet and DNS connectivity work
- SSH access from macOS works

The next stage is configuring SSH public-key authentication and basic remote administration.
