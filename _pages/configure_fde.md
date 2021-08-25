---
layout: single
classes: wide
title: Raspberry Pi OS Full Disk Encryption from scratch
permalink: /configure_fde/
toc: false
toc_label: "Table of content"
toc_icon: "cog"
---

# Burn and boot Rasperry Pi Os

On a Windows 10 PC:
1. Using Raspberry pi imager, burn Raspberry Pi OS (32 bit) on the target USB drive.
2. Using "Create and format hard disk partitions" functionality of Windows 10, create an empty partition at the end of the just flashed USB drive. This can be done with right click on the unallocated space, new simple volume, next, next, yes, Do not format this volume, next, finish.

In this way, when the Raspberry Pi OS will boot for the first time, it will not be able to expand the filesystem to the whole drive. Now you can boot Raspberry Pi OS from the USB. You will see an error saying "Could not expand filesystem" this is OKAY, proceed. Perform the initial configuration, setup the wifi, skip the update for now, enable SSH server. Now you will be able to connect via SSH to the Pi.

# Configure new root partition

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

```
root@hostname:/home/pi# cryptsetup -v luksFormat -c xchacha20,aes-adiantum-plain64 /dev/sda3
root@hostname:/home/pi# cryptsetup -v luksOpen /dev/sda3 sda3_crypt
root@hostname:/home/pi# mkfs -t ext4 /dev/mapper/sda3_crypt 
```

Copy files from the original root (sda2) to the new encrypted root partition (sda3).

```
root@hostname:/home/pi# mkdir -p /mnt/newroot
root@hostname:/home/pi# mount /dev/mapper/sda3_crypt /mnt/newroot
root@hostname:/home/pi# sudo mkdir -p /mnt/oldroot
root@hostname:/home/pi# sudo mount /dev/sda2 /mnt/oldroot
root@hostname:/home/pi# rsync -avhP /mnt/oldroot/ /mnt/newroot/
root@hostname:/home/pi# sync && sync
```

# Ensure new root partition is decrypted and mounted during boot

Using blkid, keep note of the luks and ext4 partition UUID

```
root@hostname:/home/pi# blkid
/dev/sda1: LABEL_FATBOOT="boot" LABEL="boot" UUID="7616-4FD8" TYPE="vfat" PARTUUID="f4481065-01"
/dev/sda2: LABEL="rootfs" UUID="87b585d1-84c3-486a-8f3d-77cf16f84f30" TYPE="ext4" PARTUUID="f4481065-02"
/dev/sda3: UUID="2a78c011-94b0-4127-8c4d-927bf39ca018" TYPE="crypto_LUKS" PARTUUID="f4481065-03"
/dev/mapper/sda3_crypt: UUID="73194a05-ac76-4218-b5db-6e4117bae75e" TYPE="ext4"
```

Edit crypttab file to decrypt sda3 using user provided password. Be careful in using the UUID of /dev/sda3.

```
root@hostname:/home/pi# nano /mnt/newroot/etc/crypttab
sda3_crypt UUID=2a78c011-94b0-4127-8c4d-927bf39ca018 none luks
```

Edit the fstab file, to mount the partition on boot. Replace the line with / with the new line using sda3_crypt.

```
root@hostname:/home/pi# nano /mnt/newroot/etc/fstab
/dev/mapper/sda3_crypt   /            ext4    defaults,noatime  0       1
```

Change /boot/cmdline.txt, check the PARTUUID value

```
cp /boot/cmdline.txt /boot/cmdline.txt.normal

nano /boot/cmdline.txt
console=serial0,115200 console=tty1 root=/dev/mapper/sda3_crypt rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait quiet plymouth.ignore-serial-consoles cryptdevice=PARTUUID="f4481065-03":sda3_crypt
```

and /boot/config.txt, by appending the new line at the end of the file

```
nano /boot/config.txt
initramfs initramfs.gz followkernel
``` 

# Add kernel postinst script

nano /etc/kernel/postinst.d/initramfs-rebuild

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
lsinitramfs /boot/initramfs.gz |grep -q "/$version$" && exit 0  # Already in initramfs.

# Rebuild.
mkinitramfs -o /boot/initramfs.gz "$version"
```

chmod +x /etc/kernel/postinst.d/initramfs-rebuild
cp /etc/kernel/postinst.d/initramfs-rebuild /mnt/newroot/etc/kernel/postinst.d/initramfs-rebuild

# Add initramfs hook for cryptsetup

nano /etc/initramfs-tools/hooks/luks_hooks

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

chmod +x /etc/initramfs-tools/hooks/luks_hooks
cp /etc/initramfs-tools/hooks/luks_hooks /mnt/newroot/etc/initramfs-tools/hooks/luks_hooks

# Add initramfs kernel modules

nano /etc/initramfs-tools/modules

```
algif_skcipher
xchacha20
adiantum
aes_arm
sha256
nhpoly1305
dm-crypt
```

cp /etc/initramfs-tools/modules /mnt/newroot/etc/initramfs-tools/modules

# Finally build the initramfs

CRYPTSETUP=y mkinitramfs -o /boot/initramfs.gz

Do not care about warnings, because we have CRYPTSETUP=y...

reboot

# It will drop into initramfs ...

```
cryptsetup -v luksOpen /dev/sda3 sda3_crypt
exit
```
And it will start ...

# SSH back into the pi

sudo su -

mkinitramfs -o /boot/initramfs.gz

Now there should be no warnings

sudo rmdir /mnt/oldroot
sudo rmdir /mnt/newroot

reboot

# End of procedure! Now some useful notes and commands:

Performed on kernel 5.10.17-v7l+ on a usb drive
* initramfs before kernel update: 464 -rwxr-xr-x  1 root root  15M Jun 30 19:08  initramfs-5.10.17-v7l+.gz
* initramfs after kernel update: 

Performed on 4.19.118-v7l+ on a microsd, with a different cypher (aes-xts-plain)
* initramfs original: 400 -rwxr-xr-x  1 root root 14944793 Jun 30 12:38  initramfs.gz
* initramfs after kernel update: 400 -rwxr-xr-x  1 root root 15483048 Jun 30 13:01  initramfs.gz

update-initramfs -v -c -k `uname -r`

update-initramfs -v -c -k all

lsinitramfs /boot/initramfs.gz | grep crypt

# Now let's create a disk image for the third partition

```
dd if=/dev/sdc3 conv=sync,noerror bs=4M | pv -s 16G | pigz > /media/ubuntu/RaspiImages/2020.07.02-sandisk-16-raspios-crypted-v01-sdc3.gz
```

# Recreate partitions and restore sdc3 upon sdc2

Then on the 16GB drive Delete partitions 2,3 and recreate partition 2 as partition 2+3

Restore partion 3 upon new partition 2

pigz -dc /media/ubuntu/RaspiImages/2020.07.02-sandisk-16-raspios-crypted-v01-sdc3.gz | pv -s 12G | dd of=/dev/sdd2

Change in /boot/cmdline.txt the partition that mounts into sda3_crypt, PARTUUID same with 02 at the end

Boot from raspi, it should work... On the raspi:

nano /etc/fstab
nano /etc/crypttab
nano /boot/cmdline.txt

CRYPTSETUP=y mkinitramfs -o /boot/initramfs.gz
There are warnings but we do not care...

reboot

# It will drop (again) into initramfs ...

```
cryptsetup -v luksOpen /dev/sda2 sda2_crypt
exit
```

sudo su -

mkinitramfs -o /boot/initramfs.gz

Now there should be no warnings

reboot

# Finally we need to resize

shutdown 

Here we may need resize...

sudo su -

cryptsetup -v luksOpen /dev/sdb2 sdb2_crypt

cryptsetup resize /dev/mapper/sdb2_crypt
fsck -f /dev/mapper/sda1_crypt
resize2fs /dev/mapper/sdb2_crypt

# Configuration

Then sudo raspi-config
- install ssh server: raspi-config -> interfaces -> ssh
- enable desktop boot without autologin: raspi-config -> boot options -> desktop/cli -> Desktop
- change the hostname: raspi-config -> Network options -> hostname -> btcpay

ssh pi@...

apt-get purge realvnc-vnc-server
apt install git xrdp htop

mkdir -p /root/BTCPayNode
cd /root/BTCPayNode
git clone https://github.com/gradientskier/btcpayserver-docker.git

touch /home/pi/Desktop/setup.desktop
nano /home/pi/Desktop/setup.desktop

```
[Desktop Entry]
Version=1.0

Type=Application

Terminal=true

Icon=/usr/share/icons/gnome/256x256/apps/terminal.png

Name=Setup Node

Exec=bash -c 'sudo /root/BTCPayNode/btcpayserver-docker/Tools/rpi-setup-node.sh'

```

chown pi:pi /home/pi/Desktop/setup.desktop
chmod 700 /home/pi/Desktop/setup.desktop

Do not ask executable files: File manager, Edit, Preferences, General, Check "Do not ask option on executable launch"

Remove the wifi password
nano /etc/wpa_supplicant/wpa_supplicant.conf
Delete the relevant wifi network block (including the ‘network=’ and opening/closing braces.

# Create the final image

Always check the name of the drive
fdisk -l

dd if=/dev/sdd conv=sync,noerror bs=4M | pv -s 16G | pigz > /media/ubuntu/RaspiImages/2020.07.15-sandisk-16-raspios-crypted-v03.gz

dd if=/dev/sdc conv=sync,noerror bs=4M | pv -s 16G | pigz > /media/ubuntu/WinMacShare/2020.07.18-sandisk-16-raspios-crypted-v04.gz


dd if=/dev/sdc conv=sync,noerror bs=4M | pv -s 1000G | sudo dd of=/dev/sdd bs=4M

# And some useful links

https://mutschler.eu/linux/install-guides/raspi-btrfs/

https://rr-developer.github.io/LUKS-on-Raspberry-Pi/

https://raspberrypi.stackexchange.com/questions/92557/how-can-i-use-an-init-ramdisk-initramfs-on-boot-up-raspberry-pi

https://robpol86.com/raspberry_pi_luks.html

https://www.raspberrypi.org/documentation/configuration/config-txt/boot.md

https://www.cyberciti.biz/faq/unix-linux-dd-create-make-disk-image-commands/

https://unix.stackexchange.com/questions/31669/is-it-possible-to-mount-a-gzip-compressed-dd-image-on-the-fly

https://www.raspberrypi.org/forums/viewtopic.php?f=91&t=248380&p=1516491

https://virtual-reality-piano.medium.com/how-to-forget-a-saved-wifi-network-on-a-raspberry-pi-4cbbcf53b128
