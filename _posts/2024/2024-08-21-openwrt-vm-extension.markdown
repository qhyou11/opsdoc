---
layout: post
title:  "openwrt x86虚拟机扩容"
date:   2024-08-21 12:00:13 +0800
category: Linux
---
网上给了一些方法，主要是两个思路：
一个是在系统安装后，增加一个硬盘，把这个硬盘作为外置overlay引入。 https://www.51it.wang/ll/1901 ；
另外一个思路是从镜像入手，把镜像的空间扩大了，通过这个镜像创建的虚拟硬盘空间自然也变大。 https://macbruins.com/2011/11/15/expanding-x86-openwrt-root-partition/ 

这里提供另外一个方案，既然我们的环境是虚拟机，可以把硬盘从原实例中detach出来，作为一个新硬盘加载到另外一个linux虚拟机上。
挂载完成后，在linux虚机上可以通过growpart 扩展分区，再执行文件系统扩容。
```bash
[root@alma287 ~]# yum install cloud-utils-growpart

[root@alma287 ~]# lsblk
NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                  8:0    0   30G  0 disk
├─sda1               8:1    0    1G  0 part /boot
└─sda2               8:2    0   29G  0 part
  ├─almalinux-root 253:0    0   26G  0 lvm  /
  └─almalinux-swap 253:1    0    3G  0 lvm  [SWAP]
sdb                  8:16   0    1G  0 disk
├─sdb1               8:17   0   16M  0 part
└─sdb2               8:18   0  104M  0 part
sr0                 11:0    1 1024M  0 rom
[root@alma287 ~]# growpart /dev/sdb 2
CHANGED: partition=2 start=33792 old: size=212992 end=246783 new: size=2063327 end=2097118

[root@alma287 ~]# fdisk -l /dev/sdb
Disk /dev/sdb: 1 GiB, 1073741824 bytes, 2097152 sectors
Disk model: Virtual Disk
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x80cdd189

Device     Boot Start     End Sectors    Size Id Type
/dev/sdb1  *      512   33279   32768     16M 83 Linux
/dev/sdb2       33792 2097118 2063327 1007.5M 83 Linux

[root@alma287 ~]# resize2fs /dev/sdb2
resize2fs 1.46.5 (30-Dec-2021)
Please run 'e2fsck -f /dev/sdb2' first.

[root@alma287 ~]# e2fsck -f /dev/sdb2
e2fsck 1.46.5 (30-Dec-2021)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
Padding at end of inode bitmap is not set. Fix<y>? yes

rootfs: ***** FILE SYSTEM WAS MODIFIED *****
rootfs: 1464/6656 files (0.0% non-contiguous), 4955/26624 blocks

[root@alma287 ~]# resize2fs /dev/sdb2
resize2fs 1.46.5 (30-Dec-2021)
Resizing the filesystem on /dev/sdb2 to 257915 (4k) blocks.
The filesystem on /dev/sdb2 is now 257915 (4k) blocks long.

[root@alma287 ~]# lsblk
NAME               MAJ:MIN RM    SIZE RO TYPE MOUNTPOINTS
sda                  8:0    0     30G  0 disk
├─sda1               8:1    0      1G  0 part /boot
└─sda2               8:2    0     29G  0 part
  ├─almalinux-root 253:0    0     26G  0 lvm  /
  └─almalinux-swap 253:1    0      3G  0 lvm  [SWAP]
sdb                  8:16   0      1G  0 disk
├─sdb1               8:17   0     16M  0 part
└─sdb2               8:18   0 1007.5M  0 part
sr0                 11:0    1   1024M  0 rom
```
