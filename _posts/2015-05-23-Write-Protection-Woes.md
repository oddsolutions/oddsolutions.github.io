---
layout: post
title: ZTE Majesty Root and Write Protection Exploit
---

## How a $30 phone caused me a headache

A research paper/narrative one of my first exploit creation processes.

# Standard Disclaimer

You are solely responsible for any potential damage(s) caused to your device by this exploit.

# Exploit Resources
* [z976c-root-wp](https://github.com/oddsolutions/z976c-root-wp)

# Whitepaper

## Problem Statement
Upon Googling "ZTE Majesty Root", I  was greeted by a _single_ XDA thread about the device, with no success stories.

No big surprise, it wasn't a popular device when it was sold. The thread was about 8 pages of people complaining about how it hadn't yet been rooted. 

## Background

A few months ago while I was playing cards in my Physics class, my good friend named Quincy Jones (no relation) comes up to me and says "Hey, I have a crappy old ZTE phone my grandma gave me, and I know that you like messing around with this kind of stuff. You want it?"

This should have been a fun, quick exploitation effort, then I could start working porting LineageOS. But alas, this device had some additional [armour](https://imgur.com/a/lLLmj1T).

To begin, this phone runs ZTE's skin on top of Android 4.1.2, Jellybean. The model of this device is `Z796C`, which, after a quick Google search, is called the ZTE Majesty, and is sold by Straight Talk Wireless.

## Getting root access
This phone runs Android 4.1.2, which means that, in theory, this device is vulnerable to hundreds of different local (and even some remote) code execution and privilege escalation exploits.

To begin, I tried to adapt [SafeRoot](https://forum.xda-developers.com/t/root-saferoot-root-for-vruemj7-mk2-and-android-4-3.2565758/), which seemed to succeed, but upon reboot, no root. Strange, but not out of the realm of possibillities that they patched the underlyingg vulnerabillity.

Onto attempt two, [Cydia Impactor](http://www.cydiaimpactor.com/), which uses [Master Key Bug](https://web.archive.org/web/20210413034520/http://www.saurik.com/id/17), and although it seemed to work, again upon reboot, no `su` binary installed.

The more astute of you may already have some ideas as to what the underlying cause is.

Onto the universal one-click root solution of the time period, [TowelRoot](https://towelroot.com/). Downloaded the APK and ran it's exploit, it seemed to succeed... or so I thought.

After TowelRoot displayed its success message, I left the app and jumped over to SuperSU, which said that the SU Binary needed to be updated. I let it run its setup while I went for a run.

I returned back about 30 minutes later, and it still displayed the "Installing..." prompt! I closed the app, and pulled up an ADB shell to debug the issue:

`shell@android$`

Wow. That is a painfully generic codename.

I then ran `su` which returned: `/system/bin/sh: su: not found`

This greatly confused me, as I thought we had root earlier? I teh rebooted an reran TowelRoot, again, it reported success.

I then opened an ADB shell and ran `su`. To which I was greeted with an SuperSU prompt, to which I agreed, and was dropped into a root shell:

`root@android:/ #`

Wow. That's still a painfully generic codename.

## Getting a partition map

I started surfing around from my ADB root shell to get a lay of the land:

```
root@android:/ # cat /proc/partitions
# Returns the block listing with no associated partition names
root@android:/ # ls -al /dev/block/platform/msm_*/by-name
# Returns: /dev/block/platform/msm_*/by-name: No such file or directory
root@android:/ # cat /proc/mtd
# Returns: "dev:  size  erasesize  name"
```

So, this device has no `by-name` partition listing, meaning that we need to get more creative.

The next plan of action for me would normally be to examine the recovery.img and get an fstab, which commonly lists all of a devices updatable partitions.

The bad news is that ZTE doesn't distribute any factory images or OTA images for this device.

Next I tried to get a partition listing from fastboot:

`C:\Users\nolenjohnson> adb reboot bootloader`

The device rebooted normally.

I then discovered on XDA that all Straight Talk phones have fastboot disabled.

This not only prevents disaster recovery, but just made a seemingly easy job much, **much** harder.

I got a bit more creative, knowing that recovery knows how to mount all partitions.

`root@android:/ # reboot recovery`

Then, from recovery, I initiated a Factory Data Reset, and after it completed, I reboot to system, re-reooted, then ran:

```
root@android:/ # cat /cache/recovery/last_log
```

This log had a lot more information than we needed, but it contained this snippet:
```
  0 /tmp ramdisk (null) (null) 0
  1 /qcsblhd_cfgdata emmc /dev/block/platform/msm_sdcc.3/by-num/p1 (null) 0
  2 /qcsbl emmc /dev/block/platform/msm_sdcc.3/by-num/p2 (null) 0
  3 /amss emmc /dev/block/platform/msm_sdcc.3/by-num/p3 (null) 0
  4 /oemsbl emmc /dev/block/platform/msm_sdcc.3/by-num/p5 (null) 0
  5 /emmcboot emmc /dev/block/platform/msm_sdcc.3/by-num/p6 (null) 0
  6 /ssd emmc /dev/block/platform/msm_sdcc.3/by-num/p7 (null) 0
  7 /boot emmc /dev/block/platform/msm_sdcc.3/by-num/p8 (null) 0
  8 /cefs1 emmc /dev/block/platform/msm_sdcc.3/by-num/p10 (null) 0
  9 /cefs2 emmc /dev/block/platform/msm_sdcc.3/by-num/p11 (null) 0
  10 /system ext4 /dev/block/platform/msm_sdcc.3/by-num/p12 (null) 0
  11 /data ext4 /dev/block/platform/msm_sdcc.3/by-num/p13 (null) 1002438656
  12 /persist ext4 /dev/block/platform/msm_sdcc.3/by-num/p14 (null) 0
  13 /cache ext4 /dev/block/platform/msm_sdcc.3/by-num/p15 (null) 0
  14 /recovery emmc /dev/block/platform/msm_sdcc.3/by-num/p16 (null) 0
  15 /misc emmc /dev/block/platform/msm_sdcc.3/by-num/p17 (null) 0
  16 /FOTA emmc /dev/block/platform/msm_sdcc.3/by-num/p18 (null) 0
  17 /splash emmc /dev/block/platform/msm_sdcc.3/by-num/p19 (null) 0
  18 /sdcard2 vfat /dev/block/platform/msm_sdcc.3/by-num/p20 (null) 0
  19 /sdcard vfat /dev/block/mmcblk1p1 /dev/block/mmcblk1 0
```

**Yay! A partition map!**

About 5 minutes into further brainstorming, the device randomly rebooted.

## Discovering write protection

When it rebooted, I got back on an ADB shell:

```
shell@android$ su
Returned: /system/bin/sh: su: not found
```

**This means war.**

For those who haven't figured it out yet, it was now apparent the phone has write protection on the `system` partition. I'll need to disable this to do anything meaningful.

But in the mean time I could continually get temporary root.

## Temporarily disabling write protection

I was aware of an old trick found by John Sawyer (jcase) to temporarily remove write protection targetted at HTC devices.

We write the boot partition to the recovery partition, and then reboot to recovery, because write protection is disabled when the boot-reason is `recovery` or any of it's related boot-modes, as recovery needs to be able to write to system on older file-based OTA devices.

First, we back up our boot/recovery partitions:
```
root@android:/# dd if= /dev/block/mmcblk0p8 of=/sdcard/boot.img
root@android:/# dd if= /dev/block/mmcblk0p16 of=/sdcard/recovery.img
```

Now we write the contents of the boot partition to the recovery partition and reboot to recoveyr
```
root@android:/# dd if= /dev/block/mmcblk0p8 of=/dev/block/mmcblk0p16
root@android:/# reboot recovery
```

Upon reboot, it seems completely normal. All data is intact, and system is booted successfully as expected, now to test write protection. To do so, we re-run TowelRoot, then ADB Shell:
```
root@android:/# mount -o remount,rw /system
Returns: `mount -o remount,rw /system`
```
Success! We have RW access to /system!

Now to make root access persistent, we can just open the SuperSU app and allow it to update the SU binary and install it to the system partition.

I then rebooted to the normal boot image by running
```
root@android:/# reboot
```

## (Accidentally) Permently disabling write protection
On reboot I expected to have root access, but to be unable to remount `/system` due to write protection.

However, upon reboot:

```
shell@android$ su
root@android:/# mount -o remount,rw /system
mount -o remount,rw /system
```

How are we still able to remount system RW? To make sure I wasn't still booted to the recovery partition, I ran:

```
root@android:/# cat /proc/cmdline | grep bootmode
Returns: androidboot.bootreason=reboot
```

This implies that we're booted from the actual boot partition, and not recovery, which is what we expected.

Well... Now I am really confused, but I can live with it! We defeated write protection on the ZTE Majesty permanently, even if by accident.

## After action report
I intended to work on a bootloader unlock method for this device, and then port LineageOS. Unfortunately fate was not on my side, and the device slipped from my pocked during a football game, down into the ether below the bleachers, never to be seen again.

Maybe we'll cross paths again someday, but for now, goodnight, sweet prince.

# Credits
- Nolen Johnson (npjohnson): The writeup, and exploit creation

# Special Thanks
- Quincy Jones: Challenging me with a fun project, and a free device

# Contribute to FOSS development on this device
- Sadly no one seems to care about FOSS for this device, and ZTE wouldn't even provide kernel source :(
