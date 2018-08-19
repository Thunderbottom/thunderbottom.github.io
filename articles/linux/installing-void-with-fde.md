Installing Void Linux with Full Disk Encryption
===============================================

A comprehensive guide to installing Void Linux on your system with full-disk encryption (with `/boot` encryption). At the end of this tutorial, you'll have a completely encrypted and a (hopefully) booting Void Linux installation.


# Preparation

Make sure you have a Void Linux installation disk ready to plug in and boot. This is also a good opportunity to back up your current install and start afresh, as this tutorial is aimed towards a clean system install.


# Installation

First, we boot into the installation disk and login as root. The default shell for root on Void Linux is dash and not bash. I suggest switching to bash for convenience — although, it's entirely up to you to do that.

We need to format and create partitions for our new system using `parted`. Two (or just one on legacy systems) basic self-explanatory partitions will be made:

* An unencrypted ESP. (Only on UEFI systems)
* An encrypted partition for everything else.

__For UEFI systems:__

```sh
	$ parted /dev/sdX mklabel gpt
	$ parted -a optimal /dev/sdX mkpart primary 2048s 100M
	$ parted -a optimal /dev/sdX mkpart primary 100M 100%
	$ parted /dev/sdX set 1 boot on
```

__For BIOS systems:__

```sh
	$ parted /dev/sdX mklabel msdos
	$ parted -a optimal /dev/sdX mkpart primary 2048s 100%
	$ parted /dev/sdX set 1 boot on
```

Now, we will install `cryptsetup` and `lvm2`. This will help us setup LVM partitions and LUKS encryption.

```sh
	$ xbps-install -S cryptsetup lvm2
```

In the next step, we create and open a LUKS encrypted partition, `crypt-pool`. Replace `N` in `/dev/sdXN` with the the appropriate partition number. (1=Legacy, 2=UEFI).

Type in a strong password when requested.

```sh
	$ cryptsetup luksFormat -c aes-xts-plain64 -s 512 /dev/sdXN
	$ cryptsetup luksOpen /dev/sdXN crypt-pool
```

We will now create a logical volume group, `vgpool`, inside `crypt-pool`. Then we partition the logical volume into multiple system partitions. You may also create additional partitions for `/home` or `/var` if you wish.

```sh
	$ pvcreate /dev/mapper/crypt-pool
	$ vgcreate vgpool /dev/mapper/crypt-pool
	$ lvcreate -L 300M -n boot vgpool
	$ lvcreate -C y -L <2x RAM size> -n swap vgpool
	$ lvcreate -l 100%FREE -n root vgpool
```

Confirm the partition table you just created using `lvs`.

```sh
	$ lvs -o lv_name,lv_size -S vg_name=vgpool
```

Next, we need to format and setup filesystems for our partitions. We also need to setup a `vfat` filesystem for the ESP on UEFI.

```sh
	$ mkfs.vfat -F32 /dev/sdX1
```

Setup the remaining filesystems regardless of a UEFI or legacy system.

```sh
	$ mkfs.ext4 -L boot /dev/mapper/vgpool-boot
	$ mkfs.ext4 -L root /dev/mapper/vgpool-root
	$ mkswap -L swap /dev/mapper/vgpool-swap
	$ swapon /dev/mapper/vgpool-swap
```

We then mount the partitions and begin the installation process.

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

We also need to mount some special filesystems - `/dev`, `/proc`, and `/sys`.

```sh
	$ mount -t proc /proc /mnt/proc
	$ mount -o bind /sys /mnt/sys
	$ mount -o bind /dev /mnt/dev
```

Now finally, we can install Void Linux on our system. You should add `grub-x86_64-efi` and `efibootmgr` to the install list for UEFI support.

```sh
	$ xbps-install -Sy -R http://repo.voidlinux.eu/current -r /mnt base-system lvm2 cryptsetup grub vim
```


# Configuring the System

The first thing to configure on the newly installed system is to set the root password.

```sh
	$ passwd -R /mnt root
```

Next, we need to setup our system language, hostname, timezone, etc. You need to change the values in the example in accordance to your system.

```sh
	$ echo "YOUR-HOSTNAME" > /mnt/etc/hostname
	$ echo "TIMEZONE=Europe/Zurich" >> /mnt/etc/rc.conf
	$ echo "KEYMAP=us" >> /mnt/etc/rc.conf
	$ echo "TTYS=2" >> /mnt/etc/rc.conf
	$ echo "LANG=en_US.UTF8"
	$ echo "en_US.UTF8 UTF8" >> /mnt/etc/default/libc-locales
	$ chroot /mnt xbps-reconfigure -f glibc-locales
```

We then need to add the filesystems entries to `fstab`. Do not copy this blindly or bad things will happen.

```
	/dev/mapper/vgpool-boot		/boot	ext4	defaults			0 2
	/dev/mapper/vgpool-swap		none	swap	sw				0 0
	/dev/mapper/vgpool-root		/	ext4	defaults			0 1
	tmpfs				/tmp	tmpfs	tsize=1G,defaults,nodev,nosuid	0 0
```

We also need to add the ESP to the fstab for an UEFI system.

```
	/dev/sdX1		/boot/efi	vfat	defaults	0 0
```

Now, we will `chroot` into our newly installed system and set up the bootloader.

```sh
	$ chroot /mnt /bin/bash
```

We now need to create a keyfile to automatically decrypt our encrypted partition on root. Not doing so will require you to type the crypt-pool password twice. On UEFI systems, this is because only `/boot/efi` is unencrypted, and the `/boot` partition still needs to be decrypted to continue booting. You then need to enter the password again to decrpt the rootfs. So to save yourself from the hassle, it is better to have the boot drive auto-decrypt using keyfile.

```sh
	$ dd bs=512 count=4 if=/dev/urandom of=/crypto_keyfile.bin
	$ cryptsetup luksAddKey /dev/sdXN /crypto_keyfile.bin
	$ chmod 000 /crypto_keyfile.bin
	$ chmod -R g-rwx,o-rwx /boot
```

And then we will add this keyfile to `/etc/crypttab`. To do that, we need to first note down the UUID of the encrypted disk. Replace X and N with the disk letter and the partition number.

```sh
	$ lsblk -o NAME,UUID | grep sdXN | awk '{print $2}'
```

We will then add the disk to the crypttab. Open `/etc/crypttab` in your favorite text editor and fill it with the content as follows.

```
	luks-<disk UUID here>	/dev/sdXN	/crypto_keyfile.bin	luks
```

We need to notify dracut about these things. Dracut is the tool used to generate initramfs images. This is also a good time to set `hostonly=yes` for dracut. This will  make dracut generate initramfs only for the current system (host) instead of a generic image. We will put the two things in separate files for better organization.

```sh
	$ mkdir -p /etc/dracut.conf.d/
	$ echo 'hostonly=yes' > /etc/dracut.conf.d/00-hostonly.conf
	$ echo 'install_items+="/etc/crypttab /crypto_keyfile.bin"' > /etc/dracut.conf.d/10-crypt.conf
```

In the next step, we add encryption support to GRUB and make the bootloader aware of the LUKS encrypted disk.

```sh
	$ echo "GRUB_PRELOAD_MODULES=\"cryptodisk luks\"" >> /etc/default/grub
	$ echo "GRUB_ENABLE_CRYPTODISK=y" >> /etc/default/grub
	$ echo "GRUB_CMDLINE_LINUX=\"cryptdevice=/dev/sdXN rd.luks.crypttab=1 rd.md=0 rd.dm=0 rd.lvm=1 rd.luks=1 rd.luks.allow-discards rd.luks.uuid=<device-UUID>\"" >> /etc/default/grub
```

The only thing remaining is to install the bootloader and reconfigure the kernel using dracut.

```sh
	$ grub-mkconfig -o /boot/grub/grub.cfg
	$ grub-install /dev/sdX
	$ xbps-reconfigure -f $(xbps-query -s linux4 | cut -f 2 -d ' ' | cut -f 1 -d -)
```

That's it. Now all you have to do is close the volume group and the LUKS disk, and hope that it reboots into the system.

```sh
	$ exit && umount -R /mnt
	$ vgchange -a n vgpool
	$ cryptsetup luksClose crypt-pool
	$ reboot
```

At this point, I'd like to point out to the [Post Installation Guide](https://wiki.voidlinux.eu/Post_Installation) to further help set your system up, and call it a day.
