---
title: Partitioning Schemes for Arch Linux 
date: 2020-07-08
tags: [arch]
description: my arch linux partitioning schemes 
draft: false
---

### Laptop 1: hp-4540s-i5 (BIOS/MBR)

```bash
$ sudo dmidecode | grep -A3 '^System Information'
```
```
System Information
	Manufacturer: Hewlett-Packard
	Product Name: HP ProBook 4540s
	Version: A1018C1100

	Memory Size - 16 GB

	Disk Size - 500 GB
```
```bash
$ sudo dnf install lshw
```
Intel Core i5 -3230M @ 2.60 GHz
```
$ sudo lshw -short
```


**Graphics Card**
```bash
$ lspci | grep -e VGA -e 3D
00:02.0 VGA compatible controller: Intel Corporation 3rd Gen Core processor Graphics Controller (rev 09)
01:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Thames [Radeon HD 7550M/7570M/7650M]
```

As you can see from above, I have 2 graphics cards for this laptop.


### Laptop 2: hp-450g3-i7 (UEFI/GPT)
 Product Name : HP ProBook 450 G3
 Processor 1 Intel(R) Core(TM) i7 - 6500U CPU @ 2.50 GHz
 Memory Size  8192 MB (8 GB)

 Disk Size 1TB =  1000 GB approx. - 900 GB


**Graphics Card**
```bash
$ lspci | grep -e VGA -e 3D
VGA compatible controller: Intel Corporation Skylake GT2 (HD Graphics)
```

**partitioning tools**

I will be using 

1. fdisk -l  - check my partitions 

2. fdisk /dev/sda - to do the actual partition

So the partitioning command

```
$ fdisk /dev/sda
```
where /dev/sda/ is the disk we are partitioning. 

### Partition Scheme for 4540s (Legacy System)
This will be a single boot (Arch Linux).
The partition is to be observed in the order shown
Disk Size - 500 GB

Disk Drive - /dev/sda
 

Device Name | bootable    | Partition Type | Size (GB/MB)| Type                  | Partition Type  |
|-------------|------------ |----------      |  -----------| -----                 |  ------------   |
| /dev/sda1   |    *        |/boot           | 550MB       |  primary              |  83 - Linux     |
| /dev/sda2   |             |/var            | 110GB       |  primary              |  83 - Linux     | 
| /dev/sda3   |             |/               | 90GB        |  primary              |  83 - Linux     | 
| /dev/sda4   |             |/home           | 230 GB      |  primary              |  83 - Linux     |
|             |             |                | 80GB        | Free Space            |                 |

 
Using `cfdisk` make `dev/sda1` bootable.shown by an asterisk. 

Use partprobe to update partition tables,if you happen to make changes
```
# partprobe /dev/sda
```


### Partition Scheme for 450 G3 (UEFI system)

This will be a dual boot(Arch Linux + Windows).
The partition is to be observed in the order shown

Disk Size HDD - 1000 GB = 1TB
```
fdisk -l
```
 **To wipe dev/sda for UEFI System**
```
 #parted /dev/sda -s mklabel gpt 
```
**To wipe dev/sda for Legacy System**
```
 #parted /dev/sda -s mklabel msdos 
```

Disk Drive - /dev/sda  - fdisk /dev/sda

Use t - to change partition type
 

|Device Name | Partition Type | Size (GB/MB)| Type                  |                      | Partition Type |
| :----------:| :--------:     |  :---------:| :-----:               | :-------------:      | :---------:    |
| /dev/sda1   | /boot          | 250MB       | UEFI System           | 1024 First Sector    |   1            |
| /dev/sda2   | /var           | 180GB       | Linux Filesystem(ex4) | +120G Last Sector    |   20           |
| /dev/sda3   | /              | 40GB        | Linux Filesystem(ex4) | +100G Last           |   20           |
| /dev/sda4   | /home          | 190 GB      | Linux Filesystem(ex4) | +190G Last           |   20           |
| /dev/sda5   | /swap          | 32 GB       | Linux Swap            | +32G Last Sector     |   19           |
|             |                | 489 GB      | Free Space(windows)   |                      |                |

                                                                                                
/var -  for docker containers
/swap - 2 * RAM

### Why this partition scheme?
- Personal preference - need for speed!
- `/var` is after boot because we want to start docker containers automatically.
   The other reasoning behind this partition is disk space, we want them(docker containers), to be on their own partition for performance.
   The size of `/var` usually reflects the intended purpose of the machine.Obviuosly mine is intended for development and as we know
   docker containers tend to grow in size and numbers and having this default docker directory a separate partition will greatly improve
   performance of my laptops and also iwill be protecting my `root partition - /` for running out of space. 

- `/boot` - the boot partition should be within the first 1024 cylinders, coz you  never know when a kernel update or disk deframentation may
		  occur making you system completely unbootable. 
		  -as a matter of fact I will be installing the rolling release and the stable release linux firmwares for Arch Linux.

- `/swap` - i did put it at the outer track still,so that when an hibernating/putting my laptop to sleep, on wake up it is easier to find it.
  Hard disks are usually faster on the outer tracks.

- `/home` - Personal files.I don't have to be worried about losing my files in case of OS updates.

- `/` - Root partition - other software installation goes here. 


Actually if you create many partitions instead of one `/` root partition,upgrades becomes easier.

The mountable partitions here will be: `/home`,`/`,& `/var`.

For instance this command will suffice:

```bash
root@archiso ~$ mount /dev/sda3  /mnt

root@archiso # arch-chroot /mnt
```

### References
- https://www.dell.com/support/article/en-us/sln152018/the-types-and-definitions-of-ubuntu-linux-partitions-and-directories-explained?lang=en



