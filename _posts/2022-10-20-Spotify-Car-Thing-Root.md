---
layout: post
title: Spotify Car Thing Root
---

## "NotABug" - [superbird-bulkcmd](https://github.com/oddsolutions/superbird-bulkcmd)

[Spotify Car Thing](https://carthing.spotify.com/) (superbird) resources to access U-Boot, and subsequently a root shell over USB. But according to Spotify this is not a bug. Therefore that means it must be a ["feature"](https://miro.medium.com/max/1200/1*KDfUqn6c66axcbsTPPWSpQ.jpeg).

![Hacked Car Thing](https://i.imgur.com/VRjOR5v.jpg)

*Note: this method has been tested on the factory firmware (device never used/updated : App Version 0.24.107 - OS Version 6.3.29), but should work on all firmware versions released as of this article's writing.*

# Standard Disclaimer
You are solely responsible for any potential damage(s) caused to your device by this exploit.

# Exploit Resources
* [superbird-bulkcmd](https://github.com/oddsolutions/superbird-bulkcmd)

# Whitepaper

## Problem Statement
When the Car Thing launched, it largely flew under most people's radar, and comically it wasn't until Spotify _deeply_ discounted it in late 2022, to $29.99 that it caught our eyes.

This device was designed to be a simple music selection device that mounts to your car dashboard or air-vents. It is unfortunately very underpowered, with a lower-end Amlogic chip, the S905D2, paired with 500 _MB_ of RAM - ouch.

Sadly, it has no documented method to run custom-code, let alone custom OSes.

## Background

When the device was discounted, I picked up a few units for security research, and messaged Fred shortly after starting to ask about collaborating on it - and comically he had independently already started.

To start, U-boot and Linux kernel source code for this device is [public](https://github.com/spsgsb/) but advertised nowhere by Spotify.

## Getting initial access
We discovered shortly into research, that holding buttons 1 & 4 on boot put the deivce into Amlogic's USB mode, where you can upload BL2 images! Sweet.

We were able to upload a signed BL2, and then from there, upload a signed BL33, which kicked us into Amlogic's Burn Mode.

From here we were able to execute U-Boot shell commands via Amlogic's `update` command, and the `bulkcmd` feature it houses.

## Getting UART access
At this point, it became clear UART would aid our efforts, and with some simple voltage sniffing and an educated guess, we discerned the UART has the following pin-out:
![Car Thing UART Pin-out](https://i.imgur.com/LpP9VgB.jpg)

For our development case, we wanted more persistent access to the UART pins, so we removed the sticker on the rear of the device, dissasembled, removed the rear heat-shield, and then filed out part of the case, as shown below
![Car Thing UART Setup](https://i.imgur.com/vpUnuvx.jpg)

Once we had UART console, we crafted a method to enable a root shell over UART:
```
sudo update bulkcmd 'amlmmc env'
sudo update bulkcmd 'setenv initargs init=/sbin/pre-init'
sudo update bulkcmd 'setenv initargs ${initargs} ramoops.pstore_en=1'
sudo update bulkcmd 'setenv initargs ${initargs} ramoops.record_size=0x8000'
sudo update bulkcmd 'setenv initargs ${initargs} ramoops.console_size=0x4000'
sudo update bulkcmd 'setenv initargs ${initargs} rootfstype=ext4'
sudo update bulkcmd 'setenv initargs ${initargs} console=ttyS0,115200n8'
sudo update bulkcmd 'setenv initargs ${initargs} no_console_suspend'
sudo update bulkcmd 'setenv initargs ${initargs} earlycon=aml-uart,0xff803000'
sudo update bulkcmd 'setenv storeargs ${storeargs} setenv avb2 0\;'
sudo update bulkcmd 'setenv initargs ${initargs} ro root=/dev/mmcblk0p15'
sudo update bulkcmd 'env save'
```

This gave us a local root shell, but still required UART access which is inconvienient for end-users.

## Getting an ADB root shell
We took note that the device happened to have `adbd` locally installed, but not running.

We realized it wasn't as simple as _just_ starting the daemon, we had to [disable](https://github.com/oddsolutions/superbird-bulkcmd/blob/main/scripts/disable-avb2.sh) [Android Verified Boot](https://source.android.com/docs/security/features/verifiedboot), and configure the device's USB connection in an `init.d` script, as shown in [scripts/enable-adb.sh.client](https://github.com/oddsolutions/superbird-bulkcmd/blob/main/scripts/enable-adb.sh.client).

At this point we had full u-boot access, as well as persistent ADB (root) access, we initially wanted to try to bring-up Android Automotive on the device, but 500 MB of RAM made Android near-impossible to port.

We also tried to get other GUI applications _cough_ maybe doom _cough_ running, but this device utilizes a QT feature called [EGLFS](https://doc.qt.io/qt-6/embedded-linux.html), which doesn't have a window management system like X11 or Wayland, so it is hard to get additional applications running on the device, but hey, maybe someone in the community can get it working using the access we're providing!

We ended up settling on using a modified init-ramdisk loaded via USB to simplify attaining root-access for the end-user. Hope you enjoy!

## Warning
- Many developers may (as we did) think that the easiest path to running custom code on this device would be to use the provided burn-mode access to run `update bulkcmd fastboot` and then `fastboot flashing unlock` the device. *BE WARNED*, this bricked every device we tried it on. You will end up with a blank, black screen on boot, and we have yet to discern how to recover from this. This will be updated if this type of bricked device is recoverable.

# Disclosure Notes
- October 20, 2022 - Intitial notice sent to Spotify
- October 21, 2022 - Spotify responded on HackerOne stating that the product is unsupported, and end-of-life, and therefore no bugs would be accepted pertaining to the product

*Note: This "exploit" didn't technically warrant disclosure, as it doesn't leverage any specific vulnerabillities, utilizes existing functionallity to get root access over USB.*

# Credits
- Frederic Basse (frederic): The "exploit", writeup, debugging/developing/theorizing the methodologies used.
- Nolen Johnson (npjohnson): The "exploit", writeup, debugging/developing/theorizing the methodologies used.

# Special Thanks
- Sean Hoyt (deadman): The awesome hacked-logo image.

# Contribute to FOSS development on this device
- U-Boot: [superbird-uboot](https://github.com/spsgsb/uboot/tree/buildroot-openlinux-201904-g12a)
- GNU/Linux: [superbird-linux](https://github.com/spsgsb/kernel-common)
