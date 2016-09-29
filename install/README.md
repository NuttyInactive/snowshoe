# Setting up `snowshoe`

This markdown file walks through how to setup an instance of `snowshoe`. It is
meant as a quick reference for me in the event that I ever decide to perform a
clean installation of my box, but it may also be used to help others install
or configure their own boxes.

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Install Arch Linux on VirtualBox](#install-arch-linux-on-virtualbox)
  - [Partition the Virtual Disk](#partition-the-virtual-disk)
  - [Mount the File Systems](#mount-the-file-systems)
  - [Select the `pacman` Mirrors](#select-the-pacman-mirrors)
  - [Install Base Packages](#install-base-packages)
  - [Generate the `fstab` File](#generate-the-fstab-file)
  - [Change Root](#change-root)
  - [Install VirtualBox Guest Additions](#install-virtualbox-guest-additions)
  - [Create the `swapfile`](#create-the-swapfile)
  - [Create the `initramfs`](#create-the-initramfs)
  - [Create a New User](#create-a-new-user)
  - [Configure the Time Zone](#configure-the-time-zone)
  - [Configure the Locale](#configure-the-locale)
  - [Configure the Network](#configure-the-network)
  - [Configure the Hostname](#configure-the-hostname)
  - [Configure the Bootloader](#configure-the-bootloader)
  - [Reboot the Virtual Machine](#reboot-the-virtual-machine)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Install Arch Linux on VirtualBox

This part assumes that you have already installed [VirtualBox][] on your host
machine, created a virtual machine, and downloaded the latest [Arch Linux][]
disk image. If not, go do that.

First, enable EFI boot in the VirtualBox settings for your virtual machine.
Mount the Arch Linux ISO and boot the virtual machine.

[VirtualBox]: https://www.virtualbox.org/
[Arch Linux]: https://www.archlinux.org/

### Partition the Virtual Disk

Create a GPT partition table. I will use a simple single root partitioning
scheme (one root partition and one EFI system partition). For my use case, I
do not see a need for discrete partitioning.

```sh
parted /dev/sda
(parted) mklabel gpt
(parted) mkpart ESP fat32 1MiB 513MiB
(parted) set 1 boot on
(parted) mkpart primary ext4 513MiB 100%
(parted) quit
```

Format the UEFI and root partitions.

```sh
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
```

### Mount the File Systems

```sh
mount /dev/sda2 /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

### Select the `pacman` Mirrors

Use `rankmirrors` to make a list of mirrors sorted by their speed.

```sh
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
rankmirrors -n 10 /etc/pacman.d/mirrorlist.bak > /etc/pacman.d/mirrorlist
```

### Install Base Packages

This will install the packages that belong to the [base][] and [base-devel][]
groups.

```sh
pacstrap /mnt base base-devel
```

[base]: https://www.archlinux.org/groups/x86_64/base/
[base-devel]: https://www.archlinux.org/groups/x86_64/base-devel/

### Generate the `fstab` File

The `/etc/fstab` file contains the necessary information needed to automate the
process of mounting partitions, block devices, and/or remote filesystems.

```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

### Change Root

Change the apparent root directory to the newly installed system. Set the root
password.

```sh
arch-chroot /mnt
passwd
```

### Install VirtualBox Guest Additions

VirtualBox Guest Additions provides drivers and applications to optimize the
guest operating system. The `.xinitrc` file from my dotfiles will automatically
start the guest services that will interact with X server.

```sh
pacman -S virtualbox-guest-utils
systemctl enable vboxservice.service
```

### Create the `swapfile`

Create a 1 GiB `swapfile` and update the `fstab` file.

```sh
fallocate -l 1G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo -e "# swapfile\n/swapfile none swap defaults 0 0" >> /etc/fstab
```

### Create the `initramfs`

Create the initial RAM file system. I found [these][] [resources][] useful for
obtaining a rudimentary understanding of what an `initramfs` is.

```sh
mkinitcpio -p linux
```

[these]: https://wiki.archlinux.org/index.php/Mkinitcpio#Overview
[resources]: https://www.linux.com/learn/kernel-newbie-corner-initrd-and-initramfs-whats

### Create a New User

If you want, you can create an administrative user. Replace `nutty` with your
username and `password` with your password. Then, use `visudo` to allow the
user to gain full root privileges when they precede a command with `sudo`.

```sh
useradd -m -G wheel -s /bin/bash nutty
passwd password
visudo # and add the following line: nutty ALL=(ALL) ALL
```

### Configure the Time Zone

Use `tzselect` to identify your `Zone/SubZone`. Then, create a `/etc/localtime`
symlink that points to the zoneinfo file in `/usr/share/zoneinfo/`.

```sh
tzselect
ln -s /usr/share/zoneinfo/Zone/SubZone /etc/localtime
hwclock --systohc --utc
```

### Configure the Locale

Uncomment `en_US.UTF-8 UTF-8` in `/etc/locale.gen`. Then, generate the
localizations.

```sh
locale-gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
```

### Configure the Network

Enable the network interface service. On my machine, the network interface card
name is `enp0s3`. On yours, it may be different. Use `ip link` to get the
current names.

```sh
systemctl enable dhcpcd@enp0s3.service
```

### Configure the Hostname

Create `/etc/hostname` and add a matching entry to `/etc/hosts`. Replace
`snowshoe` with your hostname.

```sh
echo snowshoe > /etc/hostname
echo -e "127.0.1.1\tsnowshoe.localdomain\tsnowshoe" >> /etc/hosts
```

### Configure the Bootloader

Exit out of the `arch-chroot` to install the `systemd-boot` bootloader.

```sh
exit # out of the arch-chroot
bootctl --path=/mnt/boot install
```

Then, configure the bootloader.

```sh
ls -l /dev/disk/by-partuuid/ > /mnt/boot/loader/entries/arch.conf
vim /mnt/boot/loader/entries/arch.conf
```

I redirect the output of `ls -l /dev/disk/by-partuuid/` to `arch.conf` as a way
to copy and paste the `UUID` of the root partition into `arch.conf`. Your
`arch.conf` should end up looking something like:

```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options root=PARTUUID=14420948-2cea-4de7-b042-40f67c618660 rw
```

### Reboot the Virtual Machine

Shutdown the machine and unmount the Arch Linux disk image in the VirtualBox
virtual machine settings.

```sh
shutdown -h 0
```

That's it! You now have a barebones installation of Arch Linux in a virtual
machine.