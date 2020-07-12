---
layout: post
title: Installing Void Linux with Full-Disk Encryption
category: "Systems"
---

A comprehensive guide to installing Void Linux on your system with <abbr title="Full-Disk Encryption">FDE</abbr>, including an encrypted boot partition.

This is an extension to the official Void documentation for Full Disk Encryption[^void-fde] in an attempt to make the steps easier to follow.

## Prerequisites

Make sure you have a Void Linux installation disk ready to plug in and boot.[^prepare-install] This is also a good opportunity to back up your current installation and start afresh, as this tutorial is aimed towards a clean system install.


## Installation

First, we shall boot into the installation disk and login as root. When you login, you will see that the default shell for root on Void Linux is set to dash. I suggest switching to bash for convenience---although it is entirely up to you to do that.


### Partitioning the Drive

We shall format and create partitions for our new system using `parted`. We'll be creating these two basic and self-explanatory partitions:

* **[UEFI Only]** An unencrypted ESP.
* An encrypted main partition for everything else.

The <abbr title="EFI System Partition">ESP</abbr> consists of the bootloader and other things that we'll require to decrypt the main partition on startup and then boot the system. The main partition will be encrypted, consisting of all the remaining partitions including `/boot`.

In this guide, we'll be referring to the drive as `/dev/sdX`. You'll need to replace `X` in the commands with the appropriate drive you want to set up. The commands are pretty straight forward:

```sh
$ parted /dev/sdX mklabel gpt # set to "msdos" for legacy mode
$ parted -a optimal /dev/sdX mkpart primary 2048s 100M # 100% for legacy mode
$ parted -a optimal /dev/sdX mkpart primary 100M 100% # UEFI ESP Partition
$ parted /dev/sdX set 1 boot on
```

**Note**: If your system's BIOS compatibility is set to Legacy mode, you are not required to create the ESP partition.
{: .message }


### Setting up LVM and LUKS for Encryption

We've created the basic partitions that we need for our system. Now we will install `cryptsetup` and `lvm2`. This will help us set up the <abbr title="Logical Volume Manager">LVM</abbr> partition and encrypt the drive using LUKS[^luks].

```sh
$ xbps-install -S cryptsetup lvm2
```

We'll be creating a LUKS encrypted partition using `cryptsetup`. For this guide, we'll be naming the LUKS partition `crypt-pool`. You may choose to replace it with whatever you wish. Also replace `X` and `N` in `/dev/sdXN` with the appropriate drive name and partition number.

Type in a strong password when requested.

```sh
$ # Replace N with 1 for Legacy, and 2 for UEFI
$ cryptsetup luksFormat -c aes-xts-plain64 -s 512 /dev/sdXN
$ cryptsetup luksOpen /dev/sdXN crypt-pool
```

We will now create a logical volume group, `vgpool`, inside `crypt-pool`. Again, the name for the volume group can be set to whatever you want. This volume group will consist of all our system partitions. You may also create additional partitions for `/home` or `/var` if you wish.

```sh
$ pvcreate /dev/mapper/crypt-pool
$ vgcreate vgpool /dev/mapper/crypt-pool
$ lvcreate -L 300M -n boot vgpool
$ lvcreate -C y -L <2x RAM size> -n swap vgpool
$ lvcreate -l 100%FREE -n root vgpool
```

This will create for you:

* a 300MB partition marked as `boot`.
* a \<2x RAM size\> partition marked as `swap`.
* remaining space marked as `root`.

These markings are just names for convenience and not actual mount points. We'll be defining the mount points for these partitions in a while.

Confirm the partition table you just created using `lvs`.

```sh
$ lvs -o lv_name,lv_size -S vg_name=vgpool
```

Next, we need to format and setup filesystems for our partitions. We also need to setup a `vfat` filesystem for the ESP on UEFI.

```sh
$ mkfs.vfat -F32 /dev/sdX1 # UEFI only!
```

Setup the remaining filesystems regardless of a UEFI or legacy system.

```sh
$ mkfs.ext4 -L boot /dev/mapper/vgpool-boot
$ mkfs.ext4 -L root /dev/mapper/vgpool-root
$ mkswap -L swap /dev/mapper/vgpool-swap
$ swapon /dev/mapper/vgpool-swap
```


### Installing Void Linux

We then mount the partitions and begin the installation process. We'll be using the markings we've defined previously to reference the partitions inside our volume group.

```sh
$ mount /dev/mapper/vgpool-root /mnt
$ mkdir /mnt/{dev,proc,sys,boot,home,var}
$ mount /dev/mapper/vgpool-boot /mnt/boot
```

If you're on an UEFI system, you also need to mount your ESP to `/boot/efi`

```sh
$ mkdir /mnt/boot/efi
$ mount /dev/sdX1 /mnt/boot/efi
```

We also need to mount some special filesystems from the live installation disk--- `/dev`, `/proc`, and `/sys`.

```sh
$ mount -t proc /proc /mnt/proc
$ mount -o bind /sys /mnt/sys
$ mount -o bind /dev /mnt/dev
```

Now finally, we can install Void Linux on our system. You'll need to add `grub-x86_64-efi` and `efibootmgr` to the install list for UEFI support.

```sh
$ xbps-install -Sy -R https://alpha.de.repo.voidlinux.org -r /mnt grub \
			base-system lvm2 cryptsetup vim \
			grub-x86_64-efi efibootmgr # UEFI Only!
```

This should install all the required things for our system to boot. Although, we're not done yet.


### Configuring the System

The first thing to configure on the newly installed system is to set the root password.

```sh
$ passwd -R /mnt root
```

This will ask you for a new root password for the system mounted at `/mnt`.

Next, we need to setup our system language, hostname, timezone, etc. You need to change the values as per your convenience.

```sh
$ echo "YOUR-HOSTNAME" > /mnt/etc/hostname
$ echo "TIMEZONE=Europe/Zurich" >> /mnt/etc/rc.conf
$ echo "KEYMAP=us" >> /mnt/etc/rc.conf
$ echo "TTYS=2" >> /mnt/etc/rc.conf
$ echo "LANG=en_US.UTF8"
$ echo "en_US.UTF8 UTF8" >> /mnt/etc/default/libc-locales
$ chroot /mnt xbps-reconfigure -f glibc-locales
```

We then need to add the filesystems entries to `/mnt/etc/fstab`. I expect you to responsibly replace all the values in the configuration below, as per your system's configuration.

```
/dev/mapper/vgpool-boot		/boot	ext4	defaults			0 2
/dev/mapper/vgpool-swap		none	swap	sw				0 0
/dev/mapper/vgpool-root		/	ext4	defaults			0 1
tmpfs				/tmp	tmpfs	tsize=1G,defaults,nodev,nosuid	0 0
```

We also need to add the ESP to the fstab for UEFI systems.

```
/dev/sdX1	/boot/efi	vfat	defaults	0 0
```

We now need to `chroot`[^chroot] into our newly installed system and set up the bootloader.

```sh
$ chroot /mnt /bin/bash
```

This should put you in a shell inside your newly installed system.

We now need to create a keyfile to automatically decrypt the encrypted partition on booting up the system. Not doing so will require you to type the password twice on UEFI systems. This is because only the `/boot/efi` is unencrypted, and the `/boot` partition still needs to be decrypted to continue booting. You then need to enter the password again to decrpt the rootfs.

To save yourself from the hassle, it is better to have the boot partition auto-decrypt using the keyfile, and enter the password once for rootfs. Generate the keyfile and set permissions to it:

```sh
$ dd bs=512 count=4 if=/dev/urandom of=/crypto_keyfile.bin
$ cryptsetup luksAddKey /dev/sdXN /crypto_keyfile.bin
$ chmod 000 /crypto_keyfile.bin
$ chmod -R g-rwx,o-rwx /boot
```

And then we need to add this keyfile to `/etc/crypttab`. This will ensure that the system uses the keyfile on boot to decrypt the `/boot` partition.

To do so, we need to first note down the UUID of the encrypted disk. Replace `X` and `N` with the drive name and the partition number.

```sh
$ lsblk -o NAME,UUID | grep sdXN | awk '{print $2}'
```

We then need to add the disk UUID and the key to `/etc/crypttab`:

```
luks-<disk UUID here>	/dev/sdXN	/crypto_keyfile.bin	luks
```

We need to notify `dracut` about the crypttab. Dracut is the tool used to generate initramfs images.[^dracut] This is also a good time to set `hostonly=yes` for dracut. This will ensure that dracut generates initramfs only for the current system (host) instead of generating a generic image. We will put the two things in separate files for better organization.

```sh
$ mkdir -p /etc/dracut.conf.d/
$ echo 'hostonly=yes' > /etc/dracut.conf.d/00-hostonly.conf
$ echo 'install_items+="/etc/crypttab /crypto_keyfile.bin"' > /etc/dracut.conf.d/10-crypt.conf
```

In the next step, we enable encryption support inside GRUB and make the bootloader aware of the LUKS encrypted disk.

```sh
$ echo "GRUB_PRELOAD_MODULES=\"cryptodisk luks\"" >> /etc/default/grub
$ echo "GRUB_ENABLE_CRYPTODISK=y" >> /etc/default/grub
$ echo "GRUB_CMDLINE_LINUX=\"cryptdevice=/dev/sdXN rd.luks.crypttab=1 rd.md=0 rd.dm=0 rd.lvm=1 rd.luks=1 rd.luks.allow-discards rd.luks.uuid=<device-UUID>\"" >> /etc/default/grub
```

The only thing that remains is to install the bootloader and then reconfigure the kernel:

```sh
$ grub-mkconfig -o /boot/grub/grub.cfg
$ grub-install /dev/sdX
$ xbps-reconfigure -f $(xbps-query -s linux4 | cut -f 2 -d ' ' | cut -f 1 -d -)
```


### Coda

That's it. Now all you have to do is close the volume group and the LUKS disk, reboot, and pray that it boots right into the system.

```sh
$ exit && umount -R /mnt
$ vgchange -a n vgpool
$ cryptsetup luksClose crypt-pool
$ reboot
```

At this point, I'd like to point you to the [Configuration Guide](https://docs.voidlinux.org/config/index.html) for further setup and call it a day.

[^void-fde]: [Void Linux: Full Disk Encryption](https://docs.voidlinux.org/installation/guides/fde.html)
[^prepare-install]: [Void Linux: Prepare Installation Media](https://docs.voidlinux.org/installation/live-images/prep.html)
[^luks]: [Linux Unified Key Setup](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup)
[^chroot]: [chroot: arch wiki](https://wiki.archlinux.org/index.php/Chroot)
[^dracut]: [dracut: man page](https://linux.die.net/man/8/dracut)
