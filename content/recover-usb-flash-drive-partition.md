---
title: Recover USB Flash Drive Partition After Using dd Command
date: 2025-10-14
description: This guide is to solve a problem where the USB Flash Drive partition changes after writing a ISO image using dd command.
---

I have a [video](https://www.youtube.com/watch?v=23I8GAweVxU) walkthrough on
how to solve this problem with Indonesian language, but there I also
provide an English subtitle so you can follow the steps with the subtitle on.

{{< youtube 23I8GAweVxU >}}

---

Let's start with the problem first. So every time you write a Linux ISO image
using `dd` command to a storage device such as a USB flash drive, the partition
size becomes different. You can take a look by using the `lsblk` or `fdisk -l`
command to show information about your storage device and its partition.

In my case, every time I write a Void Linux ISO image using `dd` command
(described in the [Void documentation](https://docs.voidlinux.org/installation/live-images/prep.html)),
there are two partitions made. `/dev/sda1` with 1 GB size and type "Empty",
and `/dev/sda2` with 32 MB size and type "EFI". While my USB flash drive
originally was 8 GB, after using `dd` the partition became smaller and this is
quite a problem.

While you can recover or fix this problem easily using [Rufus](https://rufus.ie)
on Windows with a single click through the UI, you can also use a `fdisk` command
on Linux. I found an [Arch Linux forum](https://bbs.archlinux.org/viewtopic.php?id=204366)
on how to solve this problem, but I will give a bit more detail on how to recover it.

First, when you plug in the USB flash drive, make sure it is not mounted. To
do this, you need to look into what your device name is by using `fdisk -l`.
Remember the `fdisk` command needs root permission, so you can use `sudo` or
`doas` like `sudo fdisk -l` or `doas fdisk -l`. There you will see any device name
started with `/dev/sdX`. In this example, my USB flash drive was `/dev/sdb`.
To unmount it, enter `sudo umount /dev/sdX`. Keep in mind to change the `X`
with the correct device. For me it will be `sudo umount /dev/sdb`.

After it is unmounted, we need to wipe all partitions from the flash drive using
`dd` and it will erase all data. Careful, and remember to replace the `X`:

```sh
dd if=/dev/zero of=/dev/sdX bs=512 count=1
```

If you take a look at the flash drive using `sudo fdisk -l /dev/sdX` again, you
will see there are no partitions left; it is because we "zeroed" the disk.

Then enter the interactive command of `fdisk` with `sudo fdisk /dev/sdX`. In
the interactive command, the prompt will change into `Command (m for help):`.
To list all available interactive commands, simply enter `m` in the prompt.

In the interactive command session, we need to make a single primary partition
first by entering `n` to add a new partition. If you are prompted with a question
like "Partition type", "Partition number", "Partition sector", you can simply
just press "Enter" for all questions to use the default choice.

After the partition is successfully created, we need to change the partition
type from "Linux" (the default) to "W95 FAT32 (LBA)". It is the standard
partition type for Windows, so both Windows and Linux can read and know the
partition type. To do this, enter `t` in the prompt, then enter `L` to list
all available partition types. If you take a look at the list, there is a code
and the type name, such as `0c W95 FAT32 (LBA)`, `0e W95 FAT16 (LBA)`, and so on.
The list may be different with the code name, so make sure you take a look at
the list. For me, it shows that the code is `0c` for `W95 FAT32 (LBA)`, so
enter `0c` in the prompt.

To look at the detail of the partition, enter `p` in the prompt to print the partition
table. There you can see the new partition was made with the device name `/dev/sdX1`,
the size is the same as with the flash drive, and the type is "W95 FAT16 (LBA)".

After that is done, enter `w` in the prompt to save or write to the disk, and
you will automatically exit from the interactive command.

Last, we need to format the partition that has been made. But make sure you
format the correct device name with the number. Use `lsblk` or
`sudo fdisk -l /dev/sdX` to print the disk information, and you will see the
list of all partitions started with `/dev/sdX1`. So to format the flash drive
partition, enter `mkfs -t vfat -F 32 /dev/sdX1`. I remind you again to be careful
and change the `X` with your exact device name, like `/dev/sdb1`.

So that is pretty much it. It is not really hard, but you just need to be
careful which device name you are going to use. Thank you for reading!

Reference: https://bbs.archlinux.org/viewtopic.php?id=204366
