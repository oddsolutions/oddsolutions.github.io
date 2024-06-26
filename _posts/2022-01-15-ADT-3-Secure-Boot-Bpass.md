---
layout: post
title: Google (Askey) ADT-3 - Secure Boot Bypass
---

## whoops - CVE-2021-39672

An accidental oversight in the u-boot of various Amlogic Android devices that enables the user to boot custom kernels.

To analogize it, there's a 6-inch thick fully titanium security door between you and the keys to the kingdom, it has been built up, and fortified by thousands of people over many years.

But as it turns out, the last vendor that worked on it installed a button under the handle that says "Temporarily Disable Security Door". Whoops.

# Standard Disclaimer

You are solely responsible for any potential damage(s) caused to your device by this exploit.

# Exploit Resources
* [lineage-recovery](https://mirrorbits.lineageos.org/full/deadpool/20230916/recovery.img)

# Whitepaper

## Background

When the Google ADT-3 launched, a lot of us in the Android community were excited to see a publicly available Android TV reference box, and the variety of devices that used it as a template!

On launch, though kernel source was provided, no kernel module/u-boot source was provided, and the no AOSP device-trees were released, at least not with hardware-backed OMX, generic Amlogic targets like [`yukawa`](https://android.googlesource.com/device/amlogic/yukawa/) do exist.

Inquiring with Google ultimately led to the legally required GPL license releases of kernel modules/u-boot source code.

The Google ADT-series of devices are developer-centric Android TV devices that, much alike their Nexus betherin, saw their hardware/low-level firmware developed by an OEM partner, and software from the application's boot-loader up developed by Google.

Thus far, three ADT series devices have been released:

```
Device | Codename      | Processor      | Release    | Android Version
------ | ------------- | -------------- | ---------- | ---------------
ADT-1  | molly         | NVIDIA Tegra 4 | 2014       | 4.4, 5.0.2 
ADT-2  | beast         | Amlogic s905x  | 2018       | 9
ADT-3  | adt3/deadpool | Amlogic s905y2 | 2022       | 10, 11, 12, 13
ADT-4  | adt4          | Amlogic s905x4 | Unreleased | 14
```

The Google ADT-1 was given to all people who attended Google I/O 2014, and shipped to app developers upon request to help build support for the fledgling Android TV platform. I requested one on the reccomendation of my friend Greg Willard (r3pwn).

I had no application development background, and no skills that warranted being sent one, but someone at Google decided to send me one anyway, and I can't thank them enough, as it sent me down my Android development and research path.

The Google ADT-2 was a Google internal device used to develop Android TV. They also shipped it to application developers upon request, and I was able to requisition one again.

The Google ADT-3 was sold by [Askey](https://store.askey.com/adt-3.html) and was an integral device, having a variety of devices based direclty off its design, such as the ONN 4K Box (2021), the Dynalink Box, and many others.

There will be no [Google ADT-4](https://9to5google.com/2023/07/05/android-tv-adt-4-report/), but you can buy the same hardware that Googler's use to develop Android TV [here](https://droidlogic.tv/products/amlogic-s905x4-developer-box-android-2?variant=41678529528004).

## Discovery
I was in the process of porting LineageOS to the ADT-3, and had bricked my first unit by locking the bootloader on a build that failed to boot.

When I tried to bootloader unlock the device, it would ask me to "Enable the OEM Unlocking option in developer settings" which I could not do due to the unbootable OS that was installed.

In a last ditch effort to repair the device, I went to temporarily boot recovery on a different ADT-3 that was functional to pull some images to try to repair the bricked one.

But I forgot to move the USB cable over to the functional device, and when I ran `fastboot boot lineage-recovery.img`, to my great surprise, the device booted up the unsigned recovery image.

That... can't be right, can it? To my great surprise, it was correct.

## Remediation
The underlying vulnerabillity is actually quite simple. Google has created an amazingly versatile bootloader interface they've coined `fastboot`, alongside a host-side tool to communicate with devices making use of this protocol. `fastboot` has been implemented in a variety of well known bootloader implementations, such as the Android Bootloader, U-Boot, and NvBoot.

Now, the implementations of `fastboot` varies widely, given that bootloader images, for the most part, have no CTS/VTS equivilant unit tests enforced by Google to certify builds.

Due to what seems to be an Amlogic/Askey oversight, the device doesn't properly check the devices `lock` status before allowing `fastboot boot` or `fastboot oem` commands.

This means that on a device with a locked bootloader, a user can arbitrarily `fastboot boot` any modified image, such as the one linked above in the "Exploit Resources" section.

Lets look at the ADT-3 u-boot source code release's relevant function `check_lock`, which can be viewed [here](https://android.googlesource.com/platform/external/u-boot/+/refs/heads/android-tv-11.0.0_r1/drivers/usb/gadget/f_fastboot.c#532).

Though `check_lock` properly limits some function, and there is a seperate configuration guarding `fastboot flash`, it doesn't limit the following commands:
```
static const struct cmd_dispatch_info cmd_dispatch_info[] = {
        {
                .cmd = "reboot",
                .cb = cb_reboot,
        }, {
                .cmd = "getvar:",
                .cb = cb_getvar,
        }, {
                .cmd = "download:",
                .cb = cb_download,
        }, {
                .cmd = "boot",
                .cb = cb_boot,
        }, {
                .cmd = "continue",
                .cb = cb_continue,
        }, {
                .cmd = "flashing",
                .cb = cb_flashing,
        },
#ifdef CONFIG_FASTBOOT_FLASH
        {
                .cmd = "flash",
                .cb = cb_flash,
        },
#endif
        {
                .cmd = "update",
                .cb = cb_download,
        },
        {
                .cmd = "flashall",
                .cb = cb_flashall,
                .cb = cb_reboot,
        },
        {
                .cmd = "reboot-fastboot",
                .cb = cb_reboot,
        },
        {
                .cmd = "set_active",
                .cb = cb_set_active,
        },
        {
                .cmd = "oem",
                .cb  = cb_oem_cmd,
        },
        {
                .cmd = "snapshot-update",
                .cb  = cb_snapshot_update_cmd,
        }
};
```

In contrast to this, lets look at the analog to this on the ChromeCast with Google TV (4K), the function in question can be found [here](https://github.com/oddsolutions/android_external_u-boot/blob/360f2993f14ca2523c7d854d00828f1f5e12d2b5/drivers/usb/gadget/f_fastboot.c#L1295), called `require_unlock`.

This is utilized in the following analogous function to properly limit specific commands to devices that are bootloader unlocked:
```
static const struct cmd_dispatch_info cmd_dispatch_info[] = {
        {
                .cmd = "reboot",
                .cb = cb_reboot,
                .require_unlock = false,
        }, {
                .cmd = "getvar:",
                .cb = cb_getvar,
                .require_unlock = false,
        }, {
                .cmd = "download:",
                .cb = cb_download,
                .require_unlock = true,
        }, {
                .cmd = "boot",
                .cb = cb_boot,
                .require_unlock = true,
        }, {
                .cmd = "continue",
                .cb = cb_continue,
                .require_unlock = false,
        }, {
                .cmd = "flashing",
                .cb = cb_flashing,
                .require_unlock = false,
        },
#ifdef CONFIG_FASTBOOT_FLASH
        {
                .cmd = "flash",
                .cb = cb_flash,
                .require_unlock = true,
        },
#endif
        {
                .cmd = "erase",
                .cb = cb_erase,
                .require_unlock = true,
        },
        {
                .cmd = "reboot-bootloader",
                .cb = cb_reboot,
                .require_unlock = false,
        },
        {
                .cmd = "reboot-fastboot",
                .cb = cb_reboot,
                .require_unlock = false,
        },
        {
                .cmd = "set_active",
                .cb = cb_set_active,
                .require_unlock = true,
        },
};
```

Somewhat comical that a Google device on an earlier firmware revision has this issue patched.

## Onto bigger and better things

The larger problem here is the oem commands also aren't limited, and on Amlogic devices these act as a direct pass-through to u-boot shell commands.

Oh man, I am going to some [dangerous things](https://imgur.com/a/44LBQcO) with this.

This means that we can run `fastboot oem "setenv lock 10100000"` to unlock the bootloader directly, regardless of the user-space OEM unlock toggle's state.

Upon trying that, it worked. The device can be arbitrarily locked and unlocked without any authentication. I then tried this on the ADT-3's sister devices, the Dynalink 4K Box, and onn. 4K Box, on both of which it functioned similarly.

## Affected devices

```
| Device             | Codename      | Processor      | Patched?                                                           |
|--------------------|---------------|----------------|--------------------------------------------------------------------|
| ADT-3              | adt3/deadpool | Amlogic S905y2 | Yes - As of `adt3-user-12-STT1.211025.001.Z4-7928920-release-keys` |
| Dynalink 4K Box    | sti6130d350   | Amlogic S905y2 | Yes - As of `sti6130d350-user 12 SC 20221221 release-keys`         |
| onn. 4k Box (2021) | sti6140d360   | Amlogic S905x2 | No                                                                 |
```

## Remediation

The following patch was pushed to the `android-tv-12.0.0_r1` tag [here](https://android.googlesource.com/platform/external/u-boot/+/refs/heads/android-tv-12.0.0_r1%5E%21/#F0).

This properly guards both the `fastboot boot` and `fastboot oem` families of commands on locked bootloader devices.

## Disclosure Timeline
- 09-30-2021 - Initial disclosure to Google
- 10-13-2021 - Google marks it as "Won't Fix (Not Reproducible)", I argue this, and it is immeadiately changed to "Won't Fix (Infeasible)"
- 10-27-2021 - Google passes the ticket along to "Amlogic TV"
- 11-29-2021 - Google alerts me that remediation is taking longer than expected
- 01-24-2022 - Google marks thet ticket - "Assigned"
- 02-08-2022 - Google awards a bug bounty and alerts me that it will be included in the February ASB for this device and a CVE is assigned (CVE-2021-39672), and a build for `adt3` is released remediating the issue
- 04-05-2022 - Dynalink releases a remediated build for `sti6130d350`

# Credits
- Nolen Johnson (npjohnson): The writeup, helping debug/develop/theorize the bypass method.

# Special Thanks
- Jan Altensen (Stricted): Teaching me a bucketload about Amlogic platforms.
- Chris Dibona: Being an awesome advocate of OSS software and helping ensure that we got all the source-code pertinent to the device.
- Pierre-Hugues Husson (phh): For pointing me down the Amlogic road to begin with by letting me know Google had decided to make the ADT-3 bootloader unlockable.

# Contribute to FOSS development on this device
- U-Boot: [deadpool-uboot](https://android.googlesource.com/platform/external/u-boot/+/refs/heads/android-tv-12.0.0_r1)
- Android: [android_device_google_deadpool](https://github.com/LineageOS/android_device_askey_deadpool)
- GNU/Linux: [amlogic-linux-4.9](https://github.com/LineageOS/android_kernel_amlogic_linux-4.9)
