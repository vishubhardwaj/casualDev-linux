# A Personal Arch Installation Guide

This is a personal guide so if you are lost and just found this guide from somewhere, I recommend you to read the [official wiki](https://wiki.archlinux.org/)! This guide will focus on me setting up Arch linux according to my use. This guide exists so that I can remember a bunch of things when reinstalling Archlinux.

### Pre-installation

Before installing, make sure to:

- Read the [official wiki](https://wiki.archlinux.org/). It is advisable to read that instead. I wrote this guide for myself.
- Acquire an installation image from [here](https://archlinux.org/download/).
- Verify signature.
- Prepare an installation medium.
- Boot the live environment.

### Set the console keyboard layout, font and other stuff.

- Verify the Boot mode, check the UEFI bitness:

  ```
  root@archiso~# cat /sys/firmware/efi/fw_platform_size
  ```

  If the command returns `64`, then system is booted in UEFI mode and has a 64-bit x64 UEFI. If the command returns `32`, then system is booted in UEFI mode and has a 32-bit IA32 UEFI; while this is supported, it will limit the boot loader choice to systemd-boot. If the file does not exist, the system may be booted in [BIOS](https://en.wikipedia.org/wiki/BIOS "wikipedia:BIOS") (or [CSM](https://en.wikipedia.org/wiki/Compatibility_Support_Module "wikipedia:Compatibility Support Module")) mode. If the system did not boot in the mode you desired (UEFI vs BIOS), refer to your motherboard's manual.
- The default [console keymap](https://wiki.archlinux.org/title/Console_keymap "Console keymap") is [US](https://en.wikipedia.org/wiki/File:KB_United_States-NoAltGr.svg "wikipedia:File:KB United States-NoAltGr.svg"). I'm going to keep it US.

  ```
  root@archiso~# localectl list-keymaps  // to check available keymaps
  root@archiso~# loadkeys us             // load keymap
  ```
- I'll change the default [Console font](https://wiki.archlinux.org/title/Console_fonts "Console font"

  ```
  root@archiso~# showconsolefont                      // to see the font console is using
  root@archiso~# ls /usr/share/kbd/consolefonts/      // to see the available fonts
  root@archiso~# setfont ter-120n                     // to set the font
  ```
- Connect to the internet, I'm configuring wifi. For lan, use `ip link`.

  ```
  root@archiso~# iwctl                     //to start iwctl utility
  -[iwd]# device list                       //list wifi devices, mine is wlan0
  -[iwd]# station wlan0 get-networks        //get the available network list
  -[iwd]# station wlan0 connect *wifi_name* //connect to wifi and enter password on prompt
  -[iwd]# exit

  root@archiso~# ping archlinux.org        // test the connection
  ```
- Update system clock

  Use [timedatectl(1)](https://man.archlinux.org/man/timedatectl.1) to ensure the system clock is accurate:

  ```
  root@archiso~# timedatectl
  ```
- (Optional) Edit the `/etc/pacman.conf` by removing `#` from `#PARALLELDOWNLOADS = 5`. This is only to enable parallel downloads.
- (Optional) Make a backup of `/etc/pacman.d/mirrorlist`. Then update the mirrorlist using `reflector`

  ```
  root@archiso~# cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
  root@archiso~# reflector --latest 20 --protocol https --sort rate --country India,Singapore, --save /etc/pacman
  ```

### Partition the disk

- When recognized by the live system, disks are assigned to a [block device](https://wiki.archlinux.org/title/Block_device "Block device") such as `/dev/sda`, `/dev/nvme0n1` or `/dev/mmcblk0`. To identify these devices, use [lsblk](https://wiki.archlinux.org/title/Lsblk "Lsblk") or *fdisk* .

  ```
  root@archiso~# fdisk -l
  root@archiso~# gdisk /dev/sda
  ```

  `gdisk` is a simple utility. Use `d` to delete a partition, `n` to create a new one. `w` to write the changes. I'll use the following hex codes

  - EFI boot partition - 1G - ef00
  - Linux x86-64 /root - 100G - 8304
  - Linux /home - 364.8G - 8302

### Format the partition and mount

```
root@archiso~# mkfs.fat -F 32 /dev/sda1
root@archiso~# mkfs.ext4 /dev/sda2
root@archiso~# mkfs.ext4 /dev/sda3
root@archiso~# mount /dev/sda2 /mnt
root@archiso~# mount --mkdir /dev/sda1 /mnt/boot
root@archiso~# mount --mkdir /dev/sda2 /mnt/home
```

### Installation

```
root@archiso~# pacstrap -K /mnt base base-devel linux linux-zen linux-firmware
```

### Generate fstab

```
root@archiso~# genfstab -U /mnt >> /mnt/etc/fstab
```

Check the resulting /mnt/etc/fstab file, and edit it in case of errors.

### Chroot

Now, change root into the newly installed system

`root@archiso~# arch-chroot /mnt /bin/bash`

Update and install some utilities
`[root@archiso /]# pacman -Syu`
`[root@archiso /]# pacman -S nano vim git amd-ucode dhcpcd iwd`

### Time Zone

A selection of timezones can be found under /usr/share/zoneinfo/. Since I am in the India, I will be using /usr/share/zoneinfo/Asia/Kolkata. Select the appropriate timezone for your country
`[root@archiso /]# ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime`
Run hwclock to generate /etc/adjtime:
`[root@archiso /]# hwclock --systohc`
This command assumes the hardware clock is set to UTC.

### Localisation

The locale defines which language the system uses, and other regional considerations such as currency denomination, numerology, and character sets. Possible values are listed in /etc/locale.gen. Uncomment en_US.UTF-8, as well as other needed localisations.

Uncomment en_US.UTF-8 UTF-8 and other needed locales in /etc/locale.gen, save, and generate them with:

`[root@archiso /]# locale-gen`
Create the locale.conf file, and set the LANG variable accordingly:
`[root@archiso /]# locale > /etc/locale.conf`

### Network configuration

Create the hostname file. In this guide I'll just use MYHOSTNAME as hostname. Hostname is the host name of the host. Every 60 seconds, a minute passes in Africa.

`[root@archiso /]# echo "MYHOSTNAME" > /etc/hostname`
Open /etc/hosts to add matching entries to hosts:

```
127.0.0.1    localhost
::1          localhost
127.0.1.1    MYHOSTNAME.localdomain	  MYHOSTNAME
```

If the system has a permanent IP address, it should be used instead of 127.0.1.1.

### Root password

Set the root password:

`[root@archiso /]# passwd`

##### Add a user account

Add a new user account. In this guide, I'll just use MYUSERNAME as the username of the new user aside from root account. (My phrasing seems redundant, eh?) Of course, change the example username with your own:

`[root@archiso /]# useradd -m -g users -G wheel -s /bin/bash MYUSERNAME`
This will create a new user and its home folder.

Set the password of user MYUSERNAME:

`[root@archiso /]# passwd MYUSERNAME`

##### Add the new user to sudoers:

If you want a root privilege in the future by using the sudo command, you should grant one yourself:

```
[root@archiso /]# EDITOR=vim visudo
```
Uncomment the line (Remove #):
```
%wheel ALL=(ALL) ALL
```

##### Install the boot loader

Yeah, this is where we install the bootloader. We will be using systemd-boot, so no need for grub2.

Install bootloader:

We will install it in /boot mountpoint (/dev/sda1 partition).

`bootctl --path=/boot install`
Create a boot entry /boot/loader/entries/arch.conf, then add these lines:

```
title Arch Linux  
linux /vmlinuz-linux-zen
initrd /amd-ucode.img
initrd  /initramfs-linux-zen.img  
options root=/dev/sda2 rw
```

If your `/` is not in `/dev/sda2`, make sure to change it.

Save and exit.

### Update boot loader configuration

Update bootloader configuration

vim /boot/loader/loader.conf
Delete all of its content, then replaced it by:

default arch.conf
timeout 0
console-mode keep
editor no

### Microcode

Processor manufacturers release stability and security updates to the processor microcode. These updates provide bug fixes that can be critical to the stability of your system. Without them, you may experience spurious crashes or unexpected system halts that can be difficult to track down.

If you didn't install it using pacstrap, install microcode by:

For AMD processors:

pacman -S amd-ucode

For Intel processors:

pacman -S intel-ucode

If your Arch installation is on a removable drive that needs to have microcode for both manufacturer processors, install both packages.

Load microcode. For systemd-boot, use the initrd option to load the microcode, before the initial ramdisk, as follows:

sudoedit /boot/loader/entries/entry.conf

title   Arch Linux
linux   /vmlinuz-linux
initrd  /CPU_MANUFACTURER-ucode.img
initrd  /initramfs-linux.img
...

# Enable internet connection for the next boot

To enable the network daemons on your next reboot, you need to enable dhcpcd.service for wired connection and iwd.service for a wireless one.

systemctl enable dhcpcd iwd

##### Exit chroot and reboot:

Exit the chroot environment by typing exit or pressing Ctrl + d. You can also unmount all mounted partition after this.

Finally, reboot.
