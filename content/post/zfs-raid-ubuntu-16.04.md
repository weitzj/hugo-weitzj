+++
date = "2016-05-05T15:48:21+02:00"
draft = false
title = "Creating a ZFS zroot Raid 10 on Ubuntu 16.04"
description = "My journey on how to install ZFS on Ubuntu 16.04"
tags = [ "zfs", "ubuntu", "raid", "zpool", "linux"]

+++

[Ubuntu 16.04 Xenial Xerus](https://insights.ubuntu.com/2016/02/16/zfs-is-the-fs-for-containers-in-ubuntu-16-04/)
is finally out with full ZFS support. So I embarked on a journey to get a
fileserver up and running with the goal to:

- Boot from ZFS
- Have a Raid 10
- (Raid 10 is not a requirement for this tutorial to work)

### tl;dr

- Use the [Ubuntu Desktop DVD](http://www.ubuntu.com/download/desktop) for the
  installation
- ZFS UEFI installation does not work in VirtualBox
- Putting the boot partition inside a zpool does not work. Use a *separate* UEFI
  partition for `/boot`
- Grub2 has to be installed _after_ the first time you booted your newly installed system
- Follow [this tutorial](https://github.com/zfsonlinux/pkg-zfs/wiki/HOWTO-install-Ubuntu-16.04-to-a-Native-ZFS-Root-Filesystem)

# Setup

###### Overview

The general idea is to use the Ubuntu Installer DVD but install Ubuntu by hand,
since the installer GUI does not handle ZFS, yet. We are using UEFI instead of
legacy Bios Boot.

Installing _by hand_ means, we will manually create a new *zpool*, install
everything inside inside the zpool and `chroot` to finish up the installation.

Installation of the Grub bootloader inside the ZFS zpool does not work during
chroot. Therefore we need to reboot the system with the Ubuntu Installer DVD,
boot our installed system and _then_ install Grub.

### Boot Live DVD

- Use the [ubuntu-16.04-desktop-amd64.iso](http://www.ubuntu.com/download/desktop)
since we need to install additional ZFS tools during installation. I am using an
USB stick, which contains the DVD:
```sh
dd if=ubuntu-16.04-desktop-amd64.iso of=/dev/disk2 bs=16m
```

- Use _Try Ubuntu_
- Get a terminal prompt, e.g. `<CTRL><ALT><F2>`
- Username: `ubuntu`, no password

```sh
# become root
sudo -i

# create password for root
passwd

# enable root user login
passwd -u root
```

### Partition Setup

###### Goal

- We want a Raid10 using 4 drives. So all 4 drives will get the __same
  partition layout__
- Here we are using `sda, sdb, sdc, sde` (Be aware: `sdd` is the __USB-stick__)
- For Grub we need a __separate EFI partition__. It is not possible to have ZFS
  containing the EFI partition as a dataset.

###### Install ZFS-Tools

ZFS is in the Ubuntu kernel, boot the tools are not on the DVD. So we need to
install them first.
```sh
# Add zfstools repo
echo "deb http://archive.ubuntu.com/ubuntu/ xenial universe restricted
" >> /etc/apt/sources.list

# install tools
apt-get update
apt-get install zfsutils-linux zfs-initramfs
```

###### Create partitions

Delete any old MBR/GPT partition tables on your harddrives (e.g. `mdadm`
  superblocks), ottherwise ZFS might not load your storage pool.
```sh
# delete old partition tables
sgdisk --zap-all /dev/sda
sgdisk --zap-all /dev/sdb
sgdisk --zap-all /dev/sdc
sgdisk --zap-all /dev/sde
```

Create partition table for `sda`
```sh
parted /dev/sda
mklabel gpt

# root partition
mkpart zfs zfs 0% -512MB

# boot partition
mkpart efi fat32 -512MB 100%

# mark EFI partition as bootable
set 2 boot on

# quit `parted`
quit
```

Copy partition table to other drives
```sh
# copy from sda to sdb !
sgdisk --replicate=/dev/sdb /dev/sda
sgdisk --replicate=/dev/sdc /dev/sda
sgdisk --replicate=/dev/sde /dev/sda
```

Since `sdb, sdc, sde` are now __identical__ to `sda`, we need to generate new
UUID's for the other drives
```sh
sgdisk --randomize-guids /dev/sdb
sgdisk --randomize-guids /dev/sdc
sgdisk --randomize-guids /dev/sde
```

Format EFI partitions

```sh
mkfs.vfat -F32 /dev/sda2
mkfs.vfat -F32 /dev/sdb2
mkfs.vfat -F32 /dev/sdc2
mkfs.vfat -F32 /dev/sde2
```

### Create zpool and dataset

In the previous step we created 2 partitions on each harddrive. Now we use the
__first partition of each drive__ to form a new zpool.

```sh
# This command should not fail! Otherwise the kernel will refuse to mount
# the zpool during boot.
# If you have to use '-f', try to wipe your superblock using:
# wipefs --force --all /dev/sda or
# sgdisk --zap-all /dev/sda or
# mdadm --stop /dev/md1

# create the zpool using 2 mirrors. ZFS will stripe the mirrors automatically.
zpool create -o autoexpand=on -O compression=lz4 storage \
      mirror sda1 sdb1 \
      mirror sdc1 sde1

# check your harddrive's block size. For SSD's it might make sense to use
# `zpool create -o ashift=12 ...`

# Verify zpool
# Should show sda1, sdb1, sdc1, sde1
zpool status

  pool: storage
 state: ONLINE
  scan: scrub repaired 0 in 0h0m with 0 errors on Thu May  5 11:27:31 2016
config:

        NAME        STATE     READ WRITE CKSUM
        storage     ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sda1    ONLINE       0     0     0
            sdb1    ONLINE       0     0     0
          mirror-1  ONLINE       0     0     0
            sdc1    ONLINE       0     0     0
            sdd1    ONLINE       0     0     0

errors: No known data errors
```

Export zpool and reimport `by-id`
```sh
zpool export storage
zpool import -d /dev/disk/by-id storage

# Should show disk ids
zpool status
```

Create ZFS datasets
```sh
zfs create storage/ROOT
zfs create storage/ROOT/ubuntu-1
zfs create storage/HOME
zfs create storage/HOME/home-1

# Update mountpoints. Ignore warnings.
zfs set mountpoint=none storage
zfs set mountpoint=none storage/ROOT
zfs set mountpoint=none storage/HOME

# we need to use legacy mount points, since `mountall` does not
# work with systemd (Ubuntu 16.04), yet.
zfs set mountpoint=legacy storage/HOME/home-1
```

### Install Ubuntu in ZFS-ROOT using `chroot`

###### Goal

- Install Ubuntu into ZFS-ROOT which we created in the last part
- Boot again from USB-Stick and boot the installed system
- Install Grub

###### Prepare chroot

By default ZFS will mount to `/storage`. We want it
to be under `/mnt`. So we need to unmount zpool and reimport.
```sh
zfs umount -a
zpool export storage
zpool import -d /dev/disk/by-id -R /mnt storage
```

Check mount points
```sh
mount

# should look like
storage/ROOT/ubuntu-1 on /mnt type zfs (rw,relatime,xattr,noacl)
storage/HOME/home-1 on /mnt/home type zfs (rw,relatime,xattr,noacl)
```

Create `boot` mountpoint for `sda2` (We are installing Grub only on sda2)
```sh
mkdir -p /mnt/boot/efi
mount /dev/sda2 /mnt/boot/efi
```

Install Ubuntu in ZFS-ROOT

```sh
# install ubuntu
unsquashfs -f -d /mnt/ /media/cdrom/casper/filesystem.squashfs

# setup networking
cp /run/resolvconf/resolv.conf /mnt/run/resolvconf/resolv.conf

# copy kernel
cp /media/cdrom/casper/vmlinuz.efi /mnt/boot/vmlinuz-`uname -r`
```

### chroot

Chroot into ZFS-ROOT

```sh
cd /
for i in /dev /dev/pts /proc /sys; do mount -B $i /mnt$i; done
chroot /mnt /bin/bash --login
```

Install `zfsutils` in chroot
```sh
# Add zfstools repo
echo "deb http://archive.ubuntu.com/ubuntu/ xenial universe restricted
" >> /etc/apt/sources.list

# install tools
apt-get update
apt-get install zfsutils-linux zfs-initramfs
```

Get UUID of `/boot` harddrive, `sda2`
```sh
# get UUID for sda2
ls -la /dev/disk/by-uuid | grep sda | awk '{print $9}'
```

Edit `/etc/fstab` and add `/boot` and `/home` mountpoints
```sh
# Add the UUID from the last step
UUID=9383-EFFA  /boot/efi       vfat    umask=0077      0       1

# Add `legacy` zfs volumes
# DO NOT USE A PREFIX SLASH!
# zfs set mountpoint=legacy storage/HOME/home-1
storage/HOME/home-1 /home zfs defaults    0       0
```

###### Execute normal Ubuntu setup

Add user
```sh
cd /usr/lib/ubiquity/user-setup
./user-setup-ask
./user-setup-apply
```

Set timezone
```sh
cd /usr/lib/ubiquity/tzsetup
./tzsetup
./post-base-installer
```

Setup language
```sh
cd /usr/lib/ubiquity/localechooser
./locale-chooser
./post-base-installer
```

Enable root user
```sh
# create root password
passwd

# enable root login
passwd -u root
```

Install packages
```sh
apt-get install grub-efi-amd64 openssh-server
```

Since we want to use the zpool using `disk/by-id` instead of `sda1, sdb1,...`
but zfs 'forgets' this, we need to add an udev rule to `/etc/udev/rules.d/60-zpool.rules`
```sh
# This creates symlinks directly in /dev with the same names as those in
# /dev/disk/by-id, but only for ZFS partitions.  This is necessary for GRUB to
# locate the partitions using the output of `zpool status`.
KERNEL=="sd*[0-9]", IMPORT{parent}=="ID_*", ENV{ID_PART_ENTRY_SCHEME}=="gpt", ENV{ID_PART_ENTRY_TYPE}=="6a898cc3-1dd2-11b2-99a6-080020736631", SYMLINK+="$env{ID_BUS}-$env{ID_SERIAL}-part%n"
```

Setup hostname in `/etc/hostname`
```sh
echo 'myhostname' > /etc/hostname
```

Grub will be installed later after reboot, since the `zpool` command inside the
chroot does not work with the mounted `/dev/zfs` device.

### Leave chroot/prepare reboot

```sh
exit
cd /
umount /mnt/dev/pts
umount /mnt/dev
umount /mnt/proc
umount /mnt/sys
umount /mnt/home
umount /mnt/boot/efi
zfs umount -a

# very important!
zpool export -a

reboot # and launch from USB-stick bootloader again
```

### Reboot and Grub install

###### Status

- You should have installed Ubuntu into a zpool named `storage` using a dataset
  `storage/ROOT/ubuntu-1`
- The partition layout of `sda` contains 2 partitions, `/dev/sda1`, which is
  part of the zpool and an empty formatted EFI partition `/dev/sda2`
- You rebooted the Live DVD and are in the Grub boot menu

###### Boot installed system manually from Live DVD Grub

Enter command mode
```sh
c
```
List all devices
```sh
ls

# Should give you a list like:
(hd0) == The USB-stick, which you loaded
(hd1) == /dev/sda
(hd1, gpt1) == /dev/sda1
(hd1, gpt2) == /dev/sda2
```

Boot zpool/ROOT/ubuntu-1
```sh
# load zfs module
insmod zfs

# Specify ZFS root, e.g. /dev/sda1
root=(hd1,gpt1)

# load kernel [...] is a tab keypress. You should have tab-completion.
# If tab completion does not work, check if `root=(hd1,gpt1)` is correct
linux /ROOT/.../.../.../boot/vmlinuz... root=ZFS=storage/ROOT/ubuntu-1 ro
# e.g.
# linux /ROOT/ubuntu-1/@/boot/vmlinuz-4.4.0-generic root=ZFS=storage/ROOT/ubuntu-1 ro

# load initrd [...] is a tab keypress. You should have tab-completion
# If tab completion does not work, check if `root=(hd1,gpt1)` is correct
initrd /ROOT/.../.../.../boot/initrd...

# e.g
# initrd /ROOT/ubuntu-1/@/boot/initrd.img-4.4.0-21-generic

# boot the system
boot
```

Install grub from booted system
```sh
# login as root
# verify `mount` has mounted /dev/sda2 as /boot/efi
# verify `mount` has mounted storage/HOME/home-1 as /home
mount

# install Grub
cd /
grub-install

# reboot without USB-stick
reboot
```

## You are done!

Further steps, which you might want to look at:

- Install Grub on `/dev/sdb2`, `/dev/sdc2`... as well
- Install Grub on an USB-stick

### Links on which this tutorial is based upon:

- https://github.com/zfsonlinux/pkg-zfs/wiki/HOWTO-install-Ubuntu-16.04-to-a-Native-ZFS-Root-Filesystem
- https://github.com/glasen/zfs_root_german/blob/master/zfs_neuinstallation.md

