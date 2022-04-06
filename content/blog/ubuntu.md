---
title: "Installing Ubuntu from FreeBSD"
date: 2022-04-06
description: "Installing Ubuntu on external drive from FreeBSD"
slug: "installing-ubuntu-from-freebsd"
tags: [ "freebsd", "desktop" ]
categories: [ "FreeBSD" ]
---

I want to install Ubuntu for playing games on external SSD, so I don't have to touch my boot loader. I got inspired by [Ruben's post](https://rubenerd.com/my-lazy-approach-to-freebsd-dual-booting/). I decided to buy a new SSD external drive and install Ubuntu on it. I'm pretty sure I've told the Ubuntu installer to NOT override boot record on my main drive, but sadly the installer ignored my choice and happily installed GRUB on it. At this point, I was really scared of not being able to boot my FreeBSD anymore.

## Restoring order
Luckily, I was able to quickly find a remedy for my tragedy on FreeBSD forum. Here it comes, taken from [Zirias' post](https://forums.freebsd.org/threads/restoring-bsdinstalls-boot-loader.79486/):

```
# newfs_msdos /dev/ada0p1
# mount -t msdosfs /dev/ada0p1 /boot/efi
# mkdir -p /boot/efi/efi/boot
# cp /boot/loader.efi /boot/efi/efi/boot/BOOTx64.efi
# umount /boot/efi
```

Before I found this advise, I managed to screw my boot record even more by incorrectly following different advise. I had to therefore use FreeBSD installation media to repair the boot loader. I downloaded the installer and used `dd` command to copy it on USB flash drive. Only then I was able to restore the original boot record, what a joy!

## Using Bhyve for installation
I was then wondering how to prevent the installer for causing me the pain again. I realised that if I constrain the installer to Bhyve instance, it will see only the external drive and nothing else. I created a new host and pointed it's disk to the device file, which represents my external disk. The configuration is really simple:

```
loader="uefi"
graphics="yes"
graphics_res="1280x720"
xhci_mouse="yes"
cpu=2
memory=4G
network0_type="virtio-net"
network0_switch="public"
disk0_type="virtio-blk"
disk0_name="/dev/da0"
disk0_dev="custom"
```

I quickly found that it's actually more convenient to install OS in this way, because I could still use my laptop for browsing and chatting, while I installed Ubuntu on the external drive. Another advantage of this approach is that when I wanted to copy data from the laptop's drive to Ubuntu, I could simply mount my directory as shared folder. I just added following bit into the Bhyve configuration file:

```
disk1_type="virtio-9p"
disk1_name="sharename=/home/marian/Games/Shadowrun1"
disk1_dev="custom"
```

With this setup, I can simply reboot the laptop, select the external SSD as boot drive and enjoy playing games on Linux. Mission accomplished!

