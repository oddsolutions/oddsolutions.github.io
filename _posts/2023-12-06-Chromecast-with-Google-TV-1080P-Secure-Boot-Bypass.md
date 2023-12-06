---
layout: post
title: Chromecast with Google TV (1080P) Secure-Boot Bypass
---

## _Round 2, fight!_ - boreal-secure-boot-bypass

A chain of 3 exploits intended to allow one to run a custom OS/unsigned code on the Chromecast with Google TV (CCwGTV) 1080P.

Security researchers Nolen Johnson (npjohnson), Jan Altensen (Stricted), and Ray Volpe (Functioner) developed this chain of vulnerabilities as a group effort.

The injection vector, as well as persistence bug appear to affect multiple other Nest devices, including the Nest Wi-Fi Pro, some Google/Nest Home Hub models, and some Google/Nest Home models.

# Standard Disclaimer

You are solely responsible for any potential damage(s) caused to your device by this exploit.

# Whitepaper

## Background

Back in 2021, Jan and I released a persistent secure-boot bypass ([sabrina-unlock](https://github.com/oddsolutions/sabrina-unlock)) for the original CCwGTV (Chromecast with Google TV).

In September of 2022, Google released a "Full HD" (1080p) model of Chromecast with Google TV, and silently rebranded the original CCwGTV as the Chromecast with Google TV 4K.

Given the names of these devices are confusingly similar, we'll refer to them by their internal codenames, with the 4K model being "sabrina" and the 1080P model being "boreal".

Sabrina utilized a high-end Amlogic S905D3G (SM1 family) chipset, while boreal utilizes a much lower end, but much newer Amlogic S805X2G (S4 family) chipset. This decision was likely made to hit a specific price point; and unlike SM1 series chips, the S4 series supports [AV1](https://en.wikipedia.org/wiki/AV1) hardware decoding.

So, of course we had to purchase a few to see if we could support it in LineageOS like we do [sabrina](https://github.com/LineageOS/android_device_google_sabrina).

Inquiring with Google ultimately led to the <u>legally</u> required GPL license releases of kernel/modules/u-boot source code.

The Nest team has (finally) migrated to utilizing GoogleSource Git repositories instead of Google Drive tarball releases. All Nest products with relevant OSS licensed code can be found in that devices specific manifest [here](https://nest-open-source.googlesource.com/manifests).

## Mitigations

The first hurdle to overcome is that boreal's development team learned from our work on it's 4K sibling.

Both the injection vector in the form of the BootROM exploit, and the persistence method in unlocking the bootloader by writing to the `env` partition 

The BootROM vulnerability is no longer viable, as this board utilizes a much newer BootROM version, and even has a fancy new low-level interface tool called `adnl` that replaces the old `update` tool.

The ability to persistently write the `env` partition to change the `lock` and `unlock_ability` variables is mitigated by "force locking" the device (more on that later, ugh).

## Getting a foot in the door

Firstly, we need an access venue to u-boot, or a similarly privileged injection vector.

UART was easy to identify, though Amlogic got creative with their baud-rate, setting it to `921600`, in contrast to previous Amlogic SoC's which utilize `115200`.

Unfortunately, Google has hardened both of the useful access venues UART offered before:

- u-boot's `CONFIG_AUTOBOOT_DELAY` is set to `-2`, which _should_ prevent us from interrupting the boot-sequence to access a u-boot shell over serial (more on that later...) 
- Android's console service is not started, and we have no privileged venue from which to start it, so we can't access an Android shell over serial either

Sadly, we were stuck here for some months, then a new friend popped out of the woodwork with just such an injection vector!

By strategically shorting the eMMC `DAT5` pin during boot when the device reaches a specific stage of u-boot, we can get u-boot to gracefully pause progression of the boot sequence, and create an artificial boot-delay that allows us to access a u-boot shell!

Given it was just shorted out, the eMMC is rightfully, very angry with us, and in an unusable state. This can be simply fixed by running Amlogic's useful built-in tool `amlmmc rescan 1`.

We initially thought that the device would be easy to unlock from here - little did we know what was in store for us.

## "The device is **force lock**ed"

We then tried to set the `lock` variable by hand as we did on `sabrina`, to discern what to set it to, we again looked at the `LockData` struct:

```
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

---- `u-boot/include/emmc_partitions.h`

So, just as we did on sabrina, we ran ` setenv lock 10100000` from u-boot shell, then attempted to continue the boot process.

Unfortunately, Google _really_ learned from last go around. We can't usefully set variables relevant to the lock state, as when you try to set them, the runtime check loops and states "The device is force locked", not progressing the boot process any further.

Lets take a look at how "force locking" is handled:

```c
static AvbIOResult read_is_device_unlocked(AvbOps* ops, bool* out_is_unlocked)
{
    if (get_board_variant() != BOARD_VARIANT_DEV) {
        printf("The device is force locked\n");
        *out_is_unlocked = false;
        return AVB_IO_RESULT_OK;
    }
```

---- `u-boot/cmd/amlogic/cmd_avb.c`

During the AVB (Android Verified Boot) process, if `get_board_variant` doesn't return `BOARD_VARIANT_DEV`, the device will forcibly report that it is locked regardless of the current `env` values.

Diving a bit deeper into how `get_board_variant` works:

```c
int get_board_variant(void)
{ 
#ifdef CONFIG_DEBUG_BOARD_VARIANT_PROD
        return BOARD_VARIANT_PROD;
#else
        static int board_variant = BOARD_VARIANT_UNKNOWN;

        if (board_variant == BOARD_VARIANT_UNKNOWN) {
                if (IS_FEAT_DIS_NORMAL_DEVICE_ROOTCERT_0() == 0 &&
                    IS_FEAT_DIS_DFU_DEVICE_ROOTCERT_0() == 0)
                        board_variant = BOARD_VARIANT_DEV;
                else   
                        board_variant = BOARD_VARIANT_PROD;
        }

        return board_variant;
#endif
}
```

---- ` u-boot/board/amlogic/s4_t211/s4_t211.c`

The board will only identify as `BOARD_VARIANT_DEV` when `IS_FEAT_DIS_DFU_DEVICE_ROOTCERT_0` equals 0.

```c
int IS_FEAT_DIS_DFU_DEVICE_ROOTCERT_0(void)
{
        // Do double-read to prevent hardware glitch attack
        if (OTP_BIT_CHECK(FEAT_DISABLE_DFU_DEVICE_ROOTCERT_0))
                return 1;
        if (OTP_BIT_CHECK(FEAT_DISABLE_DFU_DEVICE_ROOTCERT_0) == 0)
                return 0;
        return 1;
}
```

---- `u-boot/arch/arm/mach-meson/s4/aml_efuse.c`

Annnnnnd unfortunately `IS_FEAT_DIS_DFU_DEVICE_ROOTCERT_0`  utilizes `OTP_BIT_CHECK`, which directly reads an EFUSE, which is a one time writable software-programmable fuse that cannot be rewritten.

This sadly means that there is no direct way to disable signature checks on the device from u-boot shell.

Now, yes, in theory, from here, we could just dump u-boot out of memory, calculate function offsets, and byte-patch u-boot in memory to bypass this, but that would be a messy process that ultimately wouldn't help make it persistent - and we shortly discovered something much more useful anyway.

## Amlogic's upgrade process is strange, but (accidentally) useful

Lets take a look at how the actual AVB checks are actually handled:

```c
    avb_init();

    upgradestep = env_get("upgrade_step");

    if (is_device_unlocked() || !strcmp(upgradestep, "3"))
        flags |= AVB_SLOT_VERIFY_FLAGS_ALLOW_VERIFICATION_ERROR;

    if (!strcmp(ab_suffix, "")) {
        for (i = 0; i < AVB_NUM_SLOT; i++) {
            if (requested_partitions[i] == NULL) {
                requested_partitions[i] = "recovery";
                break;
            }
        }
        if (i == AVB_NUM_SLOT) {
            printf("ERROR: failed to find an empty slot for recovery");
            return AVB_SLOT_VERIFY_RESULT_ERROR_INVALID_ARGUMENT;
        }
    }

    if (!strcmp(ab_suffix, ""))
        result = avb_slot_verify(&avb_ops_, requested_partitions, ab_suffix,
            flags,
            AVB_HASHTREE_ERROR_MODE_RESTART_AND_INVALIDATE, out_data);
    else
        result = avb_slot_verify(&avb_ops_, requested_partitions_ab, ab_suffix,
            flags,
            AVB_HASHTREE_ERROR_MODE_RESTART_AND_INVALIDATE, out_data);

    if (!strcmp(upgradestep, "3"))
        result = AVB_SLOT_VERIFY_RESULT_OK;

    return result;
}
```

---- `u-boot/cmd/amlogic/cmd_avb.c`

So, wow, that's fun - for those of you who want to challenge yourself, look closely, the bug isn't that hard to spot.

_I'll wait._

If you guessed `what the hell is upgradestep` - you're on the right track.

The issue here is a logical operator misuse:

`if (is_device_unlocked() || !strcmp(upgradestep, "3"))`

This function is intended to temporarily bypass AVB during step 3 of AML Upgrades, but can be maliciously used to disable AVB while leaving the device in a state that it believes it is secure, and locked.

Now, as we noted earlier, `is_device_unlocked` will always unconditionally return `false` unless we have an `DEV` unit.

But `upgradestep` is simply controlled by the following function:

````c
upgradestep = env_get("upgrade_step")
````

---- `u-boot/cmd/amlogic/cmd_avb.c`

Awesome! So we can simply `setenv upgrade_step 3`, then `saveenv`, and continue the boot process by running `run storeboot`!

By utilizing this, u-boot will not only allow AVB verification errors (`AVB_SLOT_VERIFY_FLAGS_ALLOW_VERIFICATION_ERROR`), but also return a result of success (`AVB_SLOT_VERIFY_RESULT_OK`)!

This means that Android won't believe it is unlocked, or that AVB checks have been violated/skipped - it will report secure, and not wipe user-data like typical bootloader unlock would for data security purposes.

At this point, we could utilize the eMMC fault injection and the `upgradestep` bug to temporarily boot any kernel image we chose, but that process is "tethered" as each boot, `upgradestep` is reset forcibly by u-boot, even if it's written to the `env` partition.

## Making it persistent

If we want to run a full custom operating system, we need to find a method to _permanently_ bypass secure-boot.

Now, we noted that u-boot lacks any whitelist or sanity checking surrounding data passed to it via Android's `misc` partition [BCB](https://docs.u-boot.org/en/v2021.04/android/bcb.html) (Bootloader Control Block) data.

The BCB usually passes critical boot-state data, such as reboot-reason like rebooting to recovery or bootloader mode, etc.

u-boot interprets the data in the BCB as direct u-boot shell commands to be executing in the `preboot` stage of u-boot.

This inherent trust can be abused to feed malicious u-boot commands into the boot-sequence via root access Android-side.

So, by dumping the `misc` partition, then examining, and carefully writing data into the BCB to set the `upgrade_step` variable to `3` we can make the secure-boot bypass persist to at least the next boot, as shown below:

![](https://i.imgur.com/xLoo4fe.png)

At this point, we can violate AVB on the next boot, and therefore and introduce our own init script to continually set that variable each and every boot, or more conveniently, root with Magisk, and introduce a module to do it easily for the end user.

This means that so long as the BCB data isn't cleared forcibly by the user, the device will bypass secure-boot persistently!

A few caveats to this method:

- Factory Data Resetting, issuing `reboot recovery`, `reboot bootloader`, or anything that would modify the BCB will reset this value, and AVB will be enforced on the next reboot, meaning you will need to re-do the eMMC fault injection and re-set the variable
- There are no full factory images for boreal, so backup everything via a ADB root shell before modifying anything on the eMMC

This was an extremely fun process, and we made a new researcher friend along the way. LineageOS currently boots, but it multiple flavors of broken, so, say it with me ETA _soon_<sup>TM</sup>.

## How do I perform the exploit?

We significantly overcomplicated the eMMC fault injection and UART for convenience and testing purposes, here is an annotated diagram of our set up:

The bottom of the board:

![The bottom of the board](https://i.imgur.com/sFzg0QN.jpg)

The top of the board:

![The top of the board](https://i.imgur.com/6Ta6h12.jpeg)

In short, we wired a header for UART to the side of the device, wiring both Rx and Tx from the exposed pins, and creating ground by smearing solder from the nearby ground pad.

We then wired the eMMC `DAT5` pin to a physical switch that we installed on the board for ease of use.

The final product is shown below:

![The final product](https://i.imgur.com/LJWX2Ii.jpg)

The eMMC pin-out for those who are interested:

![The eMMC pin-out](https://i.imgur.com/QDq9UZQ.jpg)

## How the hell do you expect me to do this myself?!?!

It actually isn't as hard as one might expect.

The exploit can be performed with the following supplies:

- A host machine with USB
- FTDI UART adapter (or any equivalent)
- 2* Female to male jumper wires OR something like a PCBite kit
- 1* Male to male jumper wire OR a paper clip

The only "hard" part of the exploit chain is the eMMC fault injection, so we've written up some instructions to help simplify it for the end user:

Hook the FTDI UART adapter to your machine and open a serial terminal with the baud rate set to `921600`, and "Hardware Flow Control" disabled.

Then wire the pins labeled Rx on the board -> Tx on the adapter, and Tx on the board -> Rx on the adapter. At this point you should be able to view full boot logs on power on.

Now, prepare a ground wire from the adapter (or a paper clip that is seated on a reasonable ground), and put it in a position that allows you to tap the `DAT5` pin.

For some reason, it is easier to get a u-boot shell if the HDMI port is not connected, so unplug HDMI at this time.

Now, plug the device in and shortly into the boot process you will see something akin to the following message via UART:

```log
2d-eye soc_vref 0039 0039 0039 0032 0034 0041 0037 0038 0035 0034 0040 0040 0036 0038 0036 0038 0038 0035 0036 0033 0032 0032 0035 0035 0034 0035 0035 0032 0033 0032 0032 0036 0032 0032 0032 0035 0035 0047 0035 0042 0035 0044 0035 0040 average_value_dec 0035 0765 mv
2d-eye dram_vref average_value_dec 0000 vref_ave_voltage 0720 mv range_0 0720 mv range_1 0540 mv
```

Have in mind a series of three presses done in a single sequence, each about 1/2 second apart.

The first of the presses in this series should be done at the very tail end of the delay of the above referenced text, or at the very beginning of when the text starts moving again, leaning towards the latter of these two.

One of the three presses should drop you into a shell that shows:

```````
boreal#
```````

If you see errors about earlier bootloader stages failing to load, or other errors, the presses were done too soon, but the device should reboot on its own so that you can try again safely.

If you struggle with the timing, I like to put on the song "Wow" by Post Malone and do it to the rhythm of the song (I am _not_ joking, I am able to trigger the bug 9/10 times using this, so, thanks, Post).



# Demo

![An example of successful eMMC fault injection.](https://i.imgur.com/cIxf8M8.mp4)

Context: Audio removed from the video above for Copyright reasons, but "Wow" by Post Malone is in fact playing in the background.

![The serial output of both a failed and successful attempt in sequence.](https://i.imgur.com/NuDXAjM.mp4)

# Disclosure Timeline

- Early Q1-2023 -  Initial chain of 3 exploits developed
- Mid Q2-2023 - Initial report draft sent to Google
- 07-JUN-2023 - Google flagged the issue as "Triaged", and stated that the issue may qualify for the Android and Google Devices Security Reward Program
- 29-JUN-2023 - We followed up based on lack of response
- 08-AUG-2023 - We offered to provide a pre-hacked device as a demonstration
- 16-AUG-2023 - Google alerts us that the VRP team awarded us with a bug bounty for the vulnerabillity
- 29-AUG-2023 - Google asks that we delay disclosure until "the end of 2023" to allow the multiple low-level components to be patched by the relevant vendors
- 05-SEP-2023 - Google schedules a video call to discuss the proposed mitigations with us
- 07-SEP-2023 - We discussed mitigation proposals with Google, and they offered to sponsor our team to attend [hardwear.io NL 2023](https://hardwear.io/netherlands-2023/), and are alerted that Thomas Roth (stacksmashing) [HexTree.io](https://www.hextree.io), working with lennert and rquof reported a similar eMMC fault injection bug on boreal around the same time we did - the disclosure date is set for 15-NOV-2023.

​	NOTE: Only the injection vector (eMMC fault injection) was shared between Thomas's report and ours, we both went a completely different direction after that, and their chain did not intend to achieve persistence

- ~ Q3-2023 - The bug bounty pays out
- 10-OCT-2023 - Google's VRP team asks us (Jan, myself, and Thomas) to extend the disclosure deadline to mid-December - we agree on 06-DEC-2023 - the day after the December ASB (Android Security Bulletin) is posted
- ~ Early-December 04-DEC-2023 - A build that patches all 3 vulnerabilities and enables ARB was deployed via OTA for boreal.
- 06-DEC-2023 - This writeup was published and CVE’s were published by MITRE (links will be added when available)

Note: My dayjob also cross-posted this writeup [here](https://www.directdefense.com/executing-a-chromecast-exploit-times-three).

## CVE Tracking

[CVE-2023-48424](https://source.android.com/docs/security/bulletin/chromecast/2023-12-01#amlogic) - eMMC fault injection (attributed to both our team and the HexTree.io team)

[CVE-2023-48425](https://source.android.com/docs/security/bulletin/chromecast/2023-12-01#amlogic) - AVB (upgradestep) vulnerability in u-boot

[CVE-2023-6181](https://source.android.com/docs/security/bulletin/chromecast/2023-12-01#amlogic) - Proper whitelisting of BCB commands passed by the user-space

# Credits

- Nolen Johnson (npjohnson), Jan Altensen (Stricted), and Ray Volpe (Functioner): Theorizing, developing, and chaining together these vulnerabilities into an exploit chain.
- Philip Valvo Schnotalla: QA on this post and listening to me rant for hours.

# Special Thanks

- Angelina Sosa and the VRP team: Seriously, the best team in Google hands down, all are responsive, helpful, and enable security researchers to keep doing the things we love! Can't thank Google enough for the opportunity to attend hardwear.io.
- Thomas Roth of [hextree.io ](https://hextree.io): I have watched his [videos](https://www.youtube.com/channel/UC3S8vxwRfqLBdIhgRlDRVzw) for many years, and learned a lot about hardware hacking from his channel! Super stoked, and honored to find similar vulnerabilities to them! stacksmashing, lennert, and rqu all contributed to their version of the finding.
- Chris Dibona: Being an awesome advocate of OSS software and helping ensure that we got all the source-code pertinent to the device. Sad to see you depart Google, best of luck in your future endeavors!

# Contribute to FOSS development on this device

- U-Boot & Linux: https://nest-open-source.googlesource.com/manifests/#Chromecast-with-Google-TV-HD
