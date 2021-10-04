---
layout: single
classes: wide
title: Raspberry Pi OS full disk encryption from scratch
permalink: /configure_fde/
toc: true
toc_icon: "cog"
---

## Burn and boot Rasperry Pi Os

On a Windows 10 PC:
1. Using Raspberry pi imager, burn Raspberry Pi OS (32 bit) on a target USB drive of 16Gb.
2. Using "Create and format hard disk partitions" functionality of Windows 10, create an empty partition at the end of the just flashed USB drive. This can be done with right click on the unallocated space, new simple volume, next, next, yes, Do not format this volume, next, finish.

In this way, when the Raspberry Pi OS will boot for the first time, it will not be able to expand the filesystem to the whole drive. Now you can boot Raspberry Pi OS from the USB. You will see an error saying "Could not expand filesystem" this is OKAY, proceed. Perform the initial configuration, setup the wifi, skip the update for now, enable SSH server. Now you will be able to connect via SSH to the Pi.

## Configure the new root partition

Open a terminal in the Pi, and install the cryptsetup utility, which we will use to encrypt the partitions.

```bash
pi@hostname:~$ ssh pi@...
pi@hostname:~$ sudo su -
root@hostname:/home/pi# apt install cryptsetup
```

If you list the partitions of the USB drive (sda) you should see three partitions, the first one (sda1) is used for boot, the second one (sda2) contains the operating system, and the third one (sda3) is an empty partition.

```bash
root@hostname:/home/pi# fdisk -l
/dev/sda1          8192   532479   524288  256M  c W95 FAT32 (LBA)
/dev/sda2        532480  7774207  7241728  3.5G 83 Linux
/dev/sda3       7774208 30031871 22257664 10.6G 83 Linux
```

Encrypt and format sda3.

```bash
root@hostname:/home/pi# cryptsetup -v luksFormat -c xchacha20,aes-adiantum-plain64 /dev/sda3
root@hostname:/home/pi# cryptsetup -v luksOpen /dev/sda3 sda3_crypt
root@hostname:/home/pi# mkfs -t ext4 /dev/mapper/sda3_crypt 
```

Copy files from the original root (sda2) to the new encrypted root partition (sda3).

```bash
root@hostname:/home/pi# mkdir -p /mnt/newroot
root@hostname:/home/pi# mount /dev/mapper/sda3_crypt /mnt/newroot
root@hostname:/home/pi# sudo mkdir -p /mnt/oldroot
root@hostname:/home/pi# sudo mount /dev/sda2 /mnt/oldroot
root@hostname:/home/pi# rsync -avhP /mnt/oldroot/ /mnt/newroot/
root@hostname:/home/pi# sync && sync
```

## Ensure the new root partition is used on boot

Using blkid, keep note of the luks and ext4 partition UUID

```bash
root@hostname:/home/pi# blkid
/dev/sda1: LABEL_FATBOOT="boot" LABEL="boot" UUID="7616-4FD8" TYPE="vfat" PARTUUID="f4481065-01"
/dev/sda2: LABEL="rootfs" UUID="87b585d1-84c3-486a-8f3d-77cf16f84f30" TYPE="ext4" PARTUUID="f4481065-02"
/dev/sda3: UUID="2a78c011-94b0-4127-8c4d-927bf39ca018" TYPE="crypto_LUKS" PARTUUID="f4481065-03"
/dev/mapper/sda3_crypt: UUID="73194a05-ac76-4218-b5db-6e4117bae75e" TYPE="ext4"
```

Edit crypttab file to decrypt sda3 using user provided password. Be careful in using the UUID of /dev/sda3.

```bash
root@hostname:/home/pi# nano /mnt/newroot/etc/crypttab

# new line:
sda3_crypt UUID=2a78c011-94b0-4127-8c4d-927bf39ca018 none luks
```

Edit the fstab file, to mount the partition on boot. Replace the line with / with the new line using sda3_crypt.

```bash
root@hostname:/home/pi# nano /mnt/newroot/etc/fstab

# new line to replace:
/dev/mapper/sda3_crypt   /            ext4    defaults,noatime  0       1
```

Change /boot/cmdline.txt, check the PARTUUID value.

```bash
root@hostname:/home/pi# cp /boot/cmdline.txt /boot/cmdline.txt.normal
root@hostname:/home/pi# nano /boot/cmdline.txt

# new line to replace:
console=serial0,115200 console=tty1 root=/dev/mapper/sda3_crypt rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait quiet plymouth.ignore-serial-consoles cryptdevice=PARTUUID="f4481065-03":sda3_crypt
```

Change /boot/config.txt, by appending the new line at the end of the file

```bash
nano /boot/config.txt

# new line to insert at the end:
initramfs initramfs.gz followkernel
``` 

## Configure a kernel postinst script

Create a new kernel post install script, so that the initiramfs is updated upon kernel update.

```bash
root@hostname:/home/pi# nano /etc/kernel/postinst.d/initramfs-rebuild
```

The content being the following script.

```bash
#!/bin/sh -e

# Rebuild initramfs.gz after kernel upgrade to include new kernel's modules.
# https://github.com/Robpol86/robpol86.com/blob/master/docs/_static/initramfs-rebuild.sh
# Save as (chmod +x): /etc/kernel/postinst.d/initramfs-rebuild

# Remove splash from cmdline.
if grep -q '\bsplash\b' /boot/cmdline.txt; then
  sed -i 's/ \?splash \?/ /' /boot/cmdline.txt
fi

# Exit if not building kernel for this Raspberry Pi's hardware version.
version="$1"
current_version="$(uname -r)"
case "${current_version}" in
  *-v7+)
    case "${version}" in
      *-v7+) ;;
      *) exit 0
    esac
  ;;
  *-v7l+)
    case "${version}" in
      *-v7l+) ;;
      *) exit 0
    esac
  ;;
  *-v8+)
    case "${version}" in
      *-v8+) ;;
      *) exit 0
    esac
  ;;
  *+)
    case "${version}" in
      *-v7+) exit 0 ;;
      *-v7l+) exit 0 ;;
      *-v8+) exit 0 ;;
    esac
  ;;
esac

# Exit if rebuild cannot be performed or not needed.
[ -x /usr/sbin/mkinitramfs ] || exit 0
[ -f /boot/initramfs.gz ] || exit 0
lsinitramfs /boot/initramfs.gz | grep -q "/$version$" && exit 0  # Already in initramfs.

# Rebuild.
mkinitramfs -o /boot/initramfs.gz "$version"
```

Update the permissions to make the file executable and copy to new partition.

```bash
root@hostname:/home/pi# chmod +x /etc/kernel/postinst.d/initramfs-rebuild
root@hostname:/home/pi# cp /etc/kernel/postinst.d/initramfs-rebuild /mnt/newroot/etc/kernel/postinst.d/initramfs-rebuild
```

## Configure an initramfs hook for cryptsetup

```bash
root@hostname:/home/pi# nano /etc/initramfs-tools/hooks/luks_hooks
```

The content being the following.

```bash
#!/bin/sh -e
PREREQS=""
case $1 in
        prereqs) echo "${PREREQS}"; exit 0;;
esac

. /usr/share/initramfs-tools/hook-functions

copy_exec /sbin/resize2fs /sbin
copy_exec /sbin/fdisk /sbin
copy_exec /sbin/cryptsetup /sbin
```

Update the permissions to make the file executable and copy to new partition.

```
root@hostname:/home/pi# chmod +x /etc/initramfs-tools/hooks/luks_hooks
root@hostname:/home/pi# cp /etc/initramfs-tools/hooks/luks_hooks /mnt/newroot/etc/initramfs-tools/hooks/luks_hooks
```

## Configure an initramfs kernel modules

```bash
root@hostname:/home/pi# nano /etc/initramfs-tools/modules
```

Ensure the following modules are present.

```
algif_skcipher
xchacha20
adiantum
aes_arm
sha256
nhpoly1305
dm-crypt
```

Copy file to new partition.

```
root@hostname:/home/pi# cp /etc/initramfs-tools/modules /mnt/newroot/etc/initramfs-tools/modules
```

## Build the initramfs

Do not care about warnings generated after the first instruction, because we have CRYPTSETUP=y...

``` bash
CRYPTSETUP=y mkinitramfs -o /boot/initramfs.gz
reboot
```

After reboot, it will drop into initramfs, and it's necessary to unlock the disk by hand.

```
cryptsetup -v luksOpen /dev/sda3 sda3_crypt
exit
```

Now the OS will boot, so it's possible to SSH again into the Pi. We can build again the initramfs and now there should be no warnings

```
sudo su -
mkinitramfs -o /boot/initramfs.gz
```

## Cleanup and reboot

```
sudo rmdir /mnt/oldroot
sudo rmdir /mnt/newroot
reboot
```

## Configure the operating system

Then sudo raspi-config
- install ssh server: raspi-config -> interfaces -> ssh
- enable desktop boot without autologin: raspi-config -> boot options -> desktop/cli -> Desktop
- change the hostname: raspi-config -> Network options -> hostname -> btcpay

Do not ask executable files: File manager, Edit, Preferences, General, Check "Do not ask option on executable launch"

Login as root

```
ssh pi@...
sudo su -
```

Install some utilities

```
apt-get purge realvnc-vnc-server
apt install git xrdp htop
```

Clone the btcpayserver repository.

```
mkdir -p /root/BTCPayNode
cd /root/BTCPayNode
git clone https://github.com/gradientskier/btcpayserver-docker.git
```

Create the setup icon.

```
nano /home/pi/Desktop/setup.desktop
```

Insert the following content in the setup icon.

```
[Desktop Entry]
Version=1.0

Type=Application

Terminal=true

Icon=/usr/share/icons/gnome/256x256/apps/terminal.png

Name=Setup Node

Exec=bash -c 'sudo /root/BTCPayNode/btcpayserver-docker/Tools/rpi-setup-node.sh'
```

Change permissions for the setup icon.

```
chown pi:pi /home/pi/Desktop/setup.desktop
chmod 700 /home/pi/Desktop/setup.desktop
```

Remove the wifi password

```
cat /etc/wpa_supplicant/wpa_supplicant.conf
# Copy all but the relevant wifi network block, including the ‘network=’ and opening/closing braces.

apt install wipe
wipe -f /etc/wpa_supplicant/wpa_supplicant.conf

nano /etc/wpa_supplicant/wpa_supplicant.conf
# Paste the previously copied section

reboot
```

## Create the operating system image

Now you have a USB drive that boots from the third partition, and there is a second partition in between that can be removed. It's possible to create an image
that does not have the second partition in between. We take the USB drive into another linux machine and we create a disk image for the third partition.
In the example, the USB drive has been mounted as /dev/sdc and the resulting image is saved into another drive at /media/ubuntu/RaspiImages.

```bash
apt install pv pigz
dd if=/dev/sdc3 conv=sync,noerror bs=4M | pv -s 16G | pigz > /media/ubuntu/RaspiImages/2020.07.02-sandisk-16-raspios-crypted-v01-sdc3.gz
```

After saving an image for sdc3, you can recreate partitions and restore sdc3 upon sdc2. On the 16GB drive, using fdisk, delete partitions 2,3 and recreate partition 2 as partition 2+3. Finally restore partion 3 upon new partition 2

```bash
pigz -dc /media/ubuntu/RaspiImages/2020.07.02-sandisk-16-raspios-crypted-v01-sdc3.gz | pv -s 12G | dd of=/dev/sdd2
```

Using a text editor for /boot/cmdline.txt change the partition that decrypts into sda3_crypt, with same PARTUUID except with 02 at the end.
Boot from the Raspberry pi and it should work. From the raspi, change accordingly:

```
nano /etc/fstab
nano /etc/crypttab
nano /boot/cmdline.txt
```

Rebuild initramfs and reboot

```
CRYPTSETUP=y mkinitramfs -o /boot/initramfs.gz
reboot
```

It will drop (again) into initramfs.

```
cryptsetup -v luksOpen /dev/sda2 sda2_crypt
exit
```

Then rebuild initramfs and reboot
```
sudo su -
mkinitramfs -o /boot/initramfs.gz
reboot
```

If desirable, we need to resize the sda2 crypt.

```
sudo su -
cryptsetup -v luksOpen /dev/sdb2 sdb2_crypt
cryptsetup resize /dev/mapper/sdb2_crypt
fsck -f /dev/mapper/sda1_crypt
resize2fs /dev/mapper/sdb2_crypt
shutdown
```

Create the final image into another linux machine. Always check the name of the drive with `fdisk -l`, now assuming it's /dev/sdb and the target image is created into an exFat file system mounted at /media/ubuntu/winmaclinux

```bash
dd if=/dev/sdb conv=sync,noerror bs=4M | pv -s 13G | pigz > /media/ubuntu/winmaclinux/2020.07.18-sandisk-16-raspios-crypted-v05-prod2.gz
```

## Final notes

Performed on kernel 5.10.17-v7l+ on a usb drive
* initramfs before kernel update: 464 -rwxr-xr-x  1 root root  15M Jun 30 19:08  initramfs-5.10.17-v7l+.gz
* initramfs after kernel update: 

Performed on 4.19.118-v7l+ on a microsd, with a different cypher (aes-xts-plain)
* initramfs original: 400 -rwxr-xr-x  1 root root 14944793 Jun 30 12:38  initramfs.gz
* initramfs after kernel update: 400 -rwxr-xr-x  1 root root 15483048 Jun 30 13:01  initramfs.gz

Some useful commands to remember

``` bash
# To check the content of initramfs
lsinitramfs /boot/initramfs.gz | grep crypt

# To restore an image
dd if=/dev/sdc conv=sync,noerror bs=4M | pv -s 1000G | sudo dd of=/dev/sdd bs=4M
```

Some useful links, that helped in the creation of this guide.

* https://mutschler.eu/linux/install-guides/raspi-btrfs/
* https://rr-developer.github.io/LUKS-on-Raspberry-Pi/
* https://raspberrypi.stackexchange.com/questions/92557/how-can-i-use-an-init-ramdisk-initramfs-on-boot-up-raspberry-pi
* https://robpol86.com/raspberry_pi_luks.html
* https://www.raspberrypi.org/documentation/configuration/config-txt/boot.md
* https://www.cyberciti.biz/faq/unix-linux-dd-create-make-disk-image-commands/
* https://unix.stackexchange.com/questions/31669/is-it-possible-to-mount-a-gzip-compressed-dd-image-on-the-fly
* https://www.raspberrypi.org/forums/viewtopic.php?f=91&t=248380&p=1516491
* https://virtual-reality-piano.medium.com/how-to-forget-a-saved-wifi-network-on-a-raspberry-pi-4cbbcf53b128
