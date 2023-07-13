# Raspberry Pi 4B Encrypted SSH (AArch64)
This is a guide for installing
[Debian 12 (Bookworm)](https://wiki.debian.org/DebianBookworm)
on a
[Raspberry Pi 4B](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/).
The root partition will be encrypted with
[LUKS](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup)
and you will be able to remotely decrypt the Pi on boot via
[SSH](https://en.wikipedia.org/wiki/Secure_Shell).

In this guide, the
[AArch64 (ARM64)](https://en.wikipedia.org/wiki/AArch64)
version of Debian will be used.

> **Note**
> - Any command that starts with "$" can be run as an unprivileged user.
> - Any command that starts with "#" must be run as root.
> - Any command that starts with "(chroot) #" must be run within
> the [chroot](#chrooting).

# Contents
- [Prerequisites](#prerequisites)
  - [Essential](#essential)
  - [Different architectures](#different-architectures)
- [Getting started](#getting-started)
  - [Partitioning](#partitioning)
  - [Environment variables](#environment-variables)
  - [Encrypting root](#encrypting-root)
  - [Opening root](#opening-root)
  - [Formatting partitions](#formatting-partitions)
- [Bootstrapping](#bootstrapping)
  - [Mounting partitions](#mounting-partitions)
  - [Debootstrap](#debootstrap)
    - [First half](#first-half)
    - [QEMU static binary](#qemu-static-binary)
    - [Second half](#second-half)
- [Chrooting](#chrooting)
  - [Extra packages](#extra-packages)
  - [Configuration](#configuration)
    - [locale.gen](#localegen)
    - [fstab](#fstab)
    - [crypttab](#crypttab)
    - [raspi-firmware](#raspi-firmware)
    - [hostname](#hostname)
    - [hosts](#hosts)
    - [raspi-extra-cmdline](#raspi-extra-cmdline)
    - [dropbear.conf](#dropbearconf)
    - [resolved.conf](#resolvedconf)
    - [50-dhcp.network](#50-dhcpnetwork)
  - [Users](#users)
- [SSH](#ssh)
  - [Generating keys](#generating-keys)
  - [Copying keys](#copying-keys)
  - [SSH config](#ssh-config)
- [Cleaning up](#cleaning-up)
- [Booting](#booting)

# Prerequisites
## Essential
The following packages are necessary
for this guide:
- [coreutils](https://pkgs.org/download/coreutils)
- [cryptsetup](https://pkgs.org/download/cryptsetup)
- [debootstrap](https://pkgs.org/download/debootstrap)
- [dosfstools](https://pkgs.org/download/dosfstools)
- [e2fsprogs](https://pkgs.org/download/e2fsprogs)
- [openssh](https://pkgs.org/download/openssh)
- [udisks2](https://pkgs.org/download/udisks2)
- [util-linux](https://pkgs.org/download/util-linux)

## Different architectures
If you will be installing the Pi's system while on
a device of a different architecture, e.g. you are using
an x86_64 device, then you will also need the following packages:
- [binfmt-support](https://pkgs.org/download/binfmt-support)
- [qemu-user-static](https://pkgs.org/download/qemu-user-static)

> **Note**
> Please refer to your distribution's wiki on how these packages
should be configured to support
[chrooting](https://en.wikipedia.org/wiki/Chroot)
into a system with a different architecture.

As an example, on
[Void Linux](https://voidlinux.org/)
you would run the following commands:
```
# xbps-install binfmt-support
# ln -s /etc/sv/binfmt-support /var/service/
# xbps-install qemu-user-static
```

# Getting started
Throughout this guide, /dev/mmcblk0 will be
the drive which will be installed to.
Make sure you check which drive you are meant
to be installing Debian to and use that instead!

For networking, this guide uses a router and DNS server which
has the IPv4 address of 192.168.1.1, the netmask is 255.255.255.0,
and the Raspberry Pi has the IPv4 address of 192.168.1.123.

If you want to find out your own networking information,
you can boot the Pi before starting this guide and
check through the `ip` command.

If you wish to configure static IP addresses for the Pi,
this is **not** currently covered in this guide.

## Partitioning
Using
[fdisk](https://www.man7.org/linux/man-pages/man8/fdisk.8.html),
[cfdisk](https://www.man7.org/linux/man-pages/man8/cfdisk.8.html),
or another similar program of your choosing,
create the following partition layout:

> **Note**
> Make sure you use a new empty DOS partition table.

| Number | Size         | Type            |
| ------ | ------------ | --------------- |
| 1      | 512M         | W95 FAT32 (LBA) |
| 2      | Rest of disk | Linux           |

If you wish not to experiment, you can use
this layout. If you know what you are doing and wish
to deviate, go ahead!

## Environment variables
Set the following environment variables to assist
with the installation:
```sh
# First partition (boot)
export DEV_BOOT="/dev/mmcblk0p1"

# Second partition (encrypted root)
export DEV_LUKS="/dev/mmcblk0p2"

# The name of the decrypted root partition (any name of your choosing)
export NAME_LUKS="crypted"

# The decrypted root partition
export DEV_ROOT="/dev/mapper/${NAME_LUKS}"
```

## Encrypting root
To encrypt the root partition, you can use the following command:
```
# cryptsetup luksFormat \
    -c aes-xts-plain64 \
    -h sha512 \
    -i 5000 \
    -s 512 \
    --pbkdf argon2id \
    --type luks2 \
    --use-urandom \
    --verify-passphrase \
    "$DEV_LUKS"
```

If you do not understand what these options mean, they are
probably fine left as they are. If you do understand and would
rather choose others, that is also fine!

## Opening root
To decrypt the root partition, use the following command:
```
# cryptsetup open "$DEV_LUKS" "$NAME_LUKS"
```

## Formatting partitions
To format the partitions, use the following commands:
```
# mkfs.fat -F32 "$DEV_BOOT"
# mkfs.ext4 "$DEV_ROOT"
```

# Bootstrapping
This section describes the process of bootstrapping
a Debian system.

## Mounting partitions
Mount the root and boot partitions.
```
# mount "$DEV_ROOT" /mnt
# mkdir -p /mnt/boot/firmware
# mount "$DEV_BOOT" /mnt/boot/firmware
```

## Debootstrap
### First half
To begin the bootstrapping process, run the following command:
```
# debootstrap \
    --arch=arm64 \
    --components=main,non-free-firmware \
    --force-check-gpg \
    --foreign \
    --include=console-setup,linux-image-arm64,raspi-firmware,systemd-sysv \
    --variant=minbase \
    bookworm \
    /mnt \
    https://deb.debian.org/debian
```

### QEMU static binary
If you are using a system with an architecture that differs
from the Pi, you need to copy the appropriate QEMU static binary
to the new system.

As this guide is for AArch64/Arm64, the command will be as follows:
```
# cp /usr/bin/qemu-aarch64-static /mnt/usr/bin/
```

### Second half
To finalize the bootstrapping process, run the following command:
```
# chroot /mnt /debootstrap/debootstrap --second-stage
```

# Chrooting
To prepare the system for chrooting, run the following commands:
```
# mount --rbind /dev /mnt/dev
# mount --rbind /proc /mnt/proc
# mount --rbind /sys /mnt/sys
```

If you are using a different architecture to the Pi, run
the following command to enter the chroot:
```
# chroot /mnt qemu-aarch64-static /bin/bash
```

If you are using the same architecture, simply run this
command instead:
```
# chroot /mnt /bin/bash
```

## Extra packages
The following packages are essential for this guide:
```
(chroot) # apt install \
    cryptsetup-initramfs \
    dropbear-initramfs \
    locales \
    zstd
```

The following packages are optional, but may be useful:
```
(chroot) # apt install \
    iproute2 \
    less \
    man-db \
    sudo
```

You may also want to install a text editor of your choice,
such as:
- [emacs](https://pkgs.org/download/emacs)
- [nano](https://pkgs.org/download/nano)
- [neovim](https://pkgs.org/download/neovim)
- [vi](https://pkgs.org/download/vi)
- [vim](https://pkgs.org/download/vim)

## Configuration
This section describes the configuration required to
achieve the goal of this guide.

### locale.gen
Uncomment the locales you wish to generate
in `/etc/locale.gen`.

For example, the following content should be fine.
```
en_US.UTF-8 UTF-8
```

Once you have chosen which locales you wish to generate, run the
following command:
```
(chroot) # locale-gen
```

### fstab
The /etc/fstab file contains volume configuration
so the system knows which partitions to mount where,
and how they should be mounted.

To generate a configuration which should be fine,
you can use the following commands:
```
# echo "UUID=$(lsblk -ndo UUID ${DEV_ROOT}) / ext4 defaults,relatime 0 1" \
    >> /mnt/etc/fstab

# echo "UUID=$(lsblk -ndo UUID ${DEV_BOOT}) /boot/firmware vfat defaults,relatime 0 2" \
    >> /mnt/etc/fstab
```

### crypttab
The /etc/crypttab file contains information about encrypted volumes.
This will be used to determine how the root partition will be decrypted
at boot.

Run the following command:
```
# echo "${NAME_LUKS} UUID=$(lsblk -ndo UUID ${DEV_LUKS}) none luks,initramfs,tries=0" \
    >> /mnt/etc/crypttab
```

### raspi-firmware
The /etc/default/raspi-firmware file contains some of the configuration
which is used when the Pi is booting.

You can either edit the following values in a text editor
(ensuring they are uncommented), or just run the following commands:
```
# echo "ROOTPART=UUID=$(lsblk -ndo UUID ${DEV_ROOT})" \
    >> /mnt/etc/default/raspi-firmware

# echo 'CONSOLES="tty0"' >> /mnt/etc/default/raspi-firmware
```

### hostname
The /etc/hostname file determine the hostname to give to your Pi.
You can use any name you wish, as an example this guide will use "mypi"
```
# echo "mypi" >> /mnt/etc/hostname
```

### hosts
The /etc/hosts file determines some default hostname/IP address 
associations.

For this guide, you can enter the following:
```
# echo "127.0.0.1 localhost" >> /mnt/etc/hosts
# echo "::1 localhost" >> /mnt/etc/hosts
# echo "127.0.1.1 mypi mypi.local" >> /mnt/etc/hosts
```

### raspi-extra-cmdline
The /etc/default/raspi-extra-cmdline file contains extra parameters
you wish to pass to the kernel at boot.

The below options are the minimum required for this guide.

> **Warning**
> Make sure you adjust the `ip` kernel parameter appropriately
```
# echo "cryptdevice=UUID=$(lsblk -ndo UUID ${DEV_LUKS}):${NAME_LUKS} ip=192.168.1.123::192.168.1.1:255.255.255.0:mypi:eth0:none" \
    >> /mnt/etc/default/raspi-extra-cmdline
```

### dropbear.conf
The /etc/dropbear/initramfs/dropbear.conf file contains configuration
for dropbear. This determines behavior of the SSH server used for
connecting to, and decrypting, the Pi at boot time.

The following options should be fine:
```
# echo 'DROPBEAR_OPTIONS="-s -j -k -I 60 -c /bin/cryptroot-unlock"' \
    >> /mnt/etc/dropbear/initramfs/dropbear.conf
```

### resolved.conf
The /etc/systemd/resolved.conf file allows configuring
the systemd-resolved resolver.

The following configuration should be fine for this guide:

/mnt/etc/systemd/resolved.conf
```
[Resolve]
DNS=192.168.1.1
LLMNR=no
```

### 50-dhcp.network
This file will configure the DHCP client used by the Pi
after booting into the system.

The following configuration should be fine for this guide:

/mnt/etc/systemd/50-dhcp.network
```
[Match]
Name=e*

[Network]
DHCP=ipv4

[DHCP]
UseDNS=no
```

## Users
Make sure you set the root password and create your own
user account.

You can set the root password using the following:
```
(chroot) # passwd
```

You can create a user account using the following:
```
(chroot) # useradd -m -g sudo <username>
(chroot) # passwd <username>
```

If you wish not to use sudo, use the following commands
to create a user account instead:
```
(chroot) # useradd -m <username>
(chroot) # passwd <username>
```

# SSH
## Generating keys
Next, generate the SSH keys to be used when
decrypting and logging into the Pi.

You can use separate keys if you wish to, for simplicity
this guide uses one keypair.

During this process, you will be prompted to enter a passphrase.
You can choose to enter one, or simply press
enter/return to skip protecting your key with a passphrase.

The following commands should be fine, but of course you can
modify the options to `ssh-keygen` if you wish to.
```
$ mkdir -p ~/.ssh
$ ssh-keygen -t ed25519 -a 64 -f ~/.ssh/raspi
```

This will generate an
[ED25519](https://www.cryptopp.com/wiki/Ed25519)
key with 64
[KDF](https://en.wikipedia.org/wiki/Key_derivation_function)
rounds,
placing the private key at `~/.ssh/raspi` and
the public key at `~/.ssh/raspi.pub`

If you choose to leave the passphrase blank, the KDF
rounds will not be important as there is no passphrase
to derive from.

## Copying keys
Append the previously generated public key (the one ending in .pub)
to the relevant authorized_keys files for the Pi.

```
# cat <path-to-pub-key> >> /mnt/etc/dropbear/initramfs/authorized_keys

# mkdir /mnt/home/<username>/.ssh/authorized_keys
# cat <path-to-pub-key> >> /mnt/home/<username>/.ssh/authorized_keys

(chroot) # chown -R <username> /home/<username>/

# chmod 0600 /mnt/etc/dropbear/initramfs/authorized_keys \
    /mnt/home/<username>/.ssh/authorized_keys
```

## SSH config
To simplify connecting to the Pi, you can make an SSH
config at `~/.ssh/config` (on your system, not in chroot),
for example:

~/.ssh/config
```
Host pi-boot
    HostName 192.168.1.123
    User root
    Port 22
    IdentityFile ~/.ssh/raspi

Host pi
    HostName 192.168.1.123
    User <username>
    Port 22
    IdentityFile ~/.ssh/raspi
```

# Final commands
The following commands are necessary to have a working network:
```
(chroot) # apt install systemd-resolved
(chroot) # ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
(chroot) # update-initramfs -u -k all
```

# Cleaning up
To clean up the Pi, you can run the following commands
(make sure you power off the right drive/SD card):
```
(chroot) # exit
# rm /mnt/usr/bin/qemu-aarch64-static
# sync
# umount -R /mnt
# cryptsetup close "$DEV_ROOT"
# udisksctl power-off -b /dev/mmcblk0
```

# Booting
Insert the SD card, or attach the drive to the Pi.
Plug in and turn it on.

> **Note**
> You may need to wait a minute or two when trying to
> SSH into the Pi (it can take a while to load).

On your system, enter the following command to remotely decrypt it:
```
$ ssh pi-boot
```

You should be prompted to enter a passphrase to decrypt the
root partition.

After the decryption has finished, you should be closed out
of the SSH session, and soon you should be able to SSH into
your new Debian install using:
```
$ ssh pi
```

# Final thoughts
By now you should hopefully have a working Debian 12 install
on your Raspberry Pi which can be decrypted remotely via SSH.

If you have any problems, please do open
an issue or pull request.

I hope this guide has been as much a learning experience for you as
it was for me!
