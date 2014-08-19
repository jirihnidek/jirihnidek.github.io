---
title: Linux LVM at Disk Image
layout: post
---

When you use some virtualization platform (KVM, Xen, etc.), then it is useful to know how to modify content at disk image of the virtualized Linux. When disk images contains partitions formated with some file system format, then it is simple and straightforward, but when virtualized Linux uses [LVM](http://en.wikipedia.org/wiki/Logical_volume_management), then it becomes little bit tricky. This tutorial will guide you through this process.

First of all it is necessary to stop virtual machine.

## Using Loop Device ##

There are several ways how to modify content on the disk image, but we will use loop device. We pretend that your image is in _/var/lib/libvirt/images_. Thus we will switch to this directory:

```bash
cd /var/lib/libvirt/images
```

When your image disk is in _tester1.img_, then we will create loop device using following command:

```bash
losetup /dev/loop0 tester1.img
```

> Be aware that _/dev/loop0_ is not used by other image.

## Command kpartx ##

Linux installation usually has several partitions and we will map them to other loop devices:

```bash
kpartx -a /dev/loop0
```

> Note: we can use kpartx directly at file with disk image (tester1.img).

This will map partitions from disk image to following loop devices:

    /dev/loop0p1 (/dev/mapper/loop0p1)
    /dev/loop0p2 (/dev/mapper/loop0p2)

We can investigate meaning of partitions using _fdisk_ command:

```bash
fdisk /dev/loop0
```

and then type **p**:

    Command (m for help): p
    
    Disk /dev/loop0: 21.5 GB, 21474836480 bytes
    255 heads, 63 sectors/track, 2610 cylinders, total 41943040 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x00082a8c
    
          Device Boot      Start         End      Blocks   Id  System
    /dev/loop0p1   *        2048     1026047      512000   83  Linux
    /dev/loop0p2         1026048    41943039    20458496   8e  Linux LVM

## LVM ##

We can see that device _/dev/loop0p2_ is partition with LVM. This LVM contains partitions, where we want to modify some files (e.g.: /etc/passwd and /etc/shadow ;-)). First of all we need to know what is the name of _volume group_ on this partition. We will use _pvscan_ for this purpose:

```bash
pvscan
```

The output of _pvscan_ should contain something like this:

    PV /dev/mapper/loop0p2   VG vg_tester1   lvm2 [19.51 GiB / 0    free]

We know now that _volume group_ has name **vg_tester1** and we will use it for mapping its logical volumes to loop devices:

```bash
vgchange -a y vg_tester1
```

We can see now that there are two new files in /dev/mapper:

    /dev/mapper/vg_tester1-lv_root
    /dev/mapper/vg_tester1-lv_swap

We can mount the block device /dev/mapper/vg_tester1-lv_root:

```bash
mkdir /mnt/tester1_root
mount /dev/mapper/vg_tester1-lv_root /mnt/tester1_root
```

Finally we can edit any file in /mnt/tester1_root.