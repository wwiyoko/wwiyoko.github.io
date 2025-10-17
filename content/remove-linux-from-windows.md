---
title: How to Remove Linux Completely From Windows
date: 2025-10-17
description: A guide on how to remove Linux completely from Windows.
---

You may be thinking, *"Why would you use Windows to remove Linux while you
yourself are a [Linux zealot](https://www.fastslang.com/linux-zealot)?"*.
Well, not all cases using Linux can run smoothly.[^1] You either just want to
fresh install a new Linux distribution to the previous partition because you
ran out of storage, or something fatal happen that made you unable to log in to
[GRUB](https://en.wikipedia.org/wiki/GNU_GRUB).

[^1]: We should be realistic. No operating system is perfect for every use case.

> **Note**: This guide is only for UEFI systems. For those of you who are using
> BIOS legacy, there is an easy way to manage it by using
> [EasyBCD](https://neosmart.net/EasyBCD/).

The first thing, obviously, is to remove the Linux partitions. In Windows,
search for "Disk Management" and remove any Linux partitions that you recognize
by the size, because Windows does not know the Linux partition type.

Second, we need to remove the bootloader from the EFI partition. To do this, run
the Command Prompt or CMD (in short) as administrator. Then enter these commands:

```
diskpart
list disk
```

There you will enter the `diskpart` interactive command and list all available
disks. Then you need to select which disk contains the EFI partition. In this
example, I only had one disk, so it will be number 0. You can take a look at
the `Disk ###` field on the table to know the disk number.

```
sel disk 0
list vol
```

Select the disk number and list all disk volumes to know which volume number is
for EFI. If you see a volume with the FAT32 filesystem and the info "System",
that is the EFI partition that we want to mount. For example, my EFI partition
will be number 2:

```
sel vol 2
```

Then assign the volume to a new label that is **not** the same letter as all active
volume labels. For example, if you have `C:` for Windows partition and another
disk volume/partition is active with label `D:`, then you can assign the EFI
partition as `E`:

```
assign letter=E:
exit
cd /d E:
dir
cd EFI
dir
```

Execute the command one by one in every line. In the command above, we
assign the letter, exit from the `diskpart` session, enter the `E:` volume,
list the directories, and if you see there is a `EFI` directory then enter using
`cd EFI`.

In the `EFI` directory, you may see there are a few operating system
names, such as "Microsoft", "boot", or "debian". The directory names may be
different, but usually the first name started with the distribution or operating
system name.

After that you can remove the bootloader directory for the Linux distribution
you want to remove. For example, to remove the Debian bootloader directory:

```
rmdir /S debian
```

You will be prompted with a confirmation to remove. Type `y` to confirm.

If you reboot and enter the BIOS/UEFI firmware settings and there is still
a Linux bootloader in the "Boot" section, then we need a next step, which is to
delete the bootloader entry. To do this, open Command Prompt or CMD (in short)
as administrator again and list all available firmware entries:

```
bcdedit /enum firmware
```

There you can see a list of boot options from the firmware menu. First,
you need to look at `description` field and it should match with the bootloader
name in boot options for "debian", for example. Then delete the entry with the
corresponding entry identifier:

```
bcdedit /delete {12345678-9abc-def0-1234-56789abcdef}
```

`{12345678-9abc-def0-1234-56789abcdef}` is an example of an entry identifier.
The entry identifier may be different, so you should be careful to copy and paste
the identifier.

After all of that, you can reboot, and the boot name should already be gone in
the BIOS/UEFI firmware settings.

That is pretty much it. Thank you for reading!

Reference: https://unix.stackexchange.com/a/552768
