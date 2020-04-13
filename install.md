---
layout:      default
title:       Installation
description: Instructions on installing and updating CopperheadOS
---
*Proceed with caution: This documentation is being worked on and out of date.*

# Installing CopperheadOS
{: .no_toc}

* TOC
{:toc}

## Supported devices

CopperheadOS currently has official support for the following devices:

* Pixel 2 (walleye)
* Pixel 2 XL (taimen)
* Pixel 3 (blueline)
* Pixel 3 XL (crosshatch)
* Pixel 3a (sargo)
* Pixel 3a XL (bonito)

At one point CopperheadOS was supported on the:

* HiKey (hikey)
* HiKey 960 (hikey960)
* Nexus 5X (bullhead)
* Nexus 6P (angler)
* Pixel (sailfish)
* Pixel XL (marlin)


It can be ported to other Android devices with Treble support via the standard [device porting process](/building#device-porting-process). Most devices lack support for the
[security requirements](/devices#minimum-requirements-for-copperheados-support)
needed to match how it works on the officially supported devices.

For Pixel phones, users not buying a device from Copperhead with the official build need to [make a build signed with their own keys](/building) before flashing it to the device with
these instructions. Future releases can be similarly built from source and sideloaded as updates
via the instructions here. 

HiKey and HiKey 960 installation instructions are not covered here.

## Prerequisites

You should have at least 4GB of memory to avoid problems.

You can obtain the adb and fastboot tools from the Android SDK. Either install Android Studio or
use the standalone SDK. Do not use distribution packages for adb and fastboot. Distribution
packages are out-of-date and not compatible with the latest version of Android. An obsolete
fastboot will result in corrupted installations and potentially bricked devices. **Do not make the
common mistake of assuming that everything will be fine and ignoring these instructions.** Double
check that the first fastboot in your PATH is indeed from an up-to-date SDK installation:

    which fastboot

### Installing platform tools on your Mac, Windows or Linux

Download the latest SDK platform-tools directly from Google [here](https://developer.android.com/studio/releases/platform-tools)

To make your life easier, add the directories to your PATH in your shell profile configuration. This
will make it so you do not have to be inside a specific directory to get access to fastboot.

Adding to your $PATH in [Mac](https://www.architectryan.com/2012/10/02/add-to-the-path-on-mac-os-x-mountain-lion/#.Uydjga1dXDg), [Windows](https://helpdeskgeek.com/windows-10/add-windows-path-environment-variable/) and [Linux](https://opensource.com/article/17/6/set-path-linux).

Check your $PATH by typing `fastboot -h` and see if the output matches ..`usage: fastboot [OPTION...] COMMAND...`

With your $PATH correct and installation complete, you can now flash factory images
regardless of the directory.

### Setting up the SDK on Linux without Android Studio
To set up a minimal SDK installation without Android Studio on Linux:

    mkdir ~/sdk
    cd ~/sdk
    wget https://dl.google.com/android/repository/commandlinetools-linux-6200805_latest.zip
    unzip commandlinetools-linux-6200805_latest.zip

Run an initial update, which will also install platform-tools and patcher;v4:

    tools/bin/sdkmanager --update

For running the Compatibility Test Suite you'll also need the build-tools for aapt:

    tools/bin/sdkmanager 'build-tools;25.0.3'

To make your life easier, add the directories to your PATH in your shell profile configuration:

    export PATH="$HOME/sdk/tools:$HOME/sdk/tools/bin:$HOME/sdk/platform-tools:$HOME/sdk/build-tools/25.0.3:$PATH"
    export ANDROID_HOME="$HOME/sdk"

This is not mandatory, since you can run them from ~/sdk/platform-tools directly.

You should update the sdk before use from this point onwards:

    sdkmanager --update

## Enabling OEM unlocking

OEM unlocking needs to be enabled from within the operating system.

Enable the developer settings menu by going to Settings -> About device and pressing on the build
number menu entry until developer mode is enabled.

Next, go to Settings -> Developer settings and toggle on the 'Enable OEM unlocking' setting.

## Updating stock before using fastboot

It's important to have the latest bootloader firmware before installing CopperheadOS, due to bug
fixes for the fastboot mode used to flash CopperheadOS. There are known issues with older versions
of the bootloader that are likely to cause problems.

If you're only behind one release, updating within the stock OS makes sense to get an incremental
update. If you're behind multiple releases, updating within the OS will usually require installing
multiple updates to catch up to the current state of things. The quickest way to deal with that if
you have plenty of bandwidth is [sideloading the latest full over-the-air update from
Google](https://developers.google.com/android/ota).

## Obtaining factory images

The initial install should be performed by flashing the factory images. This will replace the
existing OS installation and wipe all the existing data.

### Latest CopperheadOS images

CopperheadOS currently provides factory images to the [Copperhead Partner network](https://copperhead.co/partners) and is not open
to the public for download.

### Legacy installation information (Nexus)
At one point Copperhead supported public downloading of official Nexus factory images on a [releases page](https://copperhead.co/android/releases). Public factory had to be unfortunately images removed because of mass violation of Copperhead's non-Commercial licensing.

## Flashing factory images

First, boot into the bootloader interface. You can do this by turning off the device and then
turning it on by holding both the Volume Down and Power buttons. Alternatively, you can use ```adb
reboot bootloader``` from Android.

### Supported devices unlocking directions
The bootloader now needs to be unlocked to allow flashing new images:

    fastboot flashing unlock

### Legacy Pixel 2 XL bootloader unlocking directions
On some older models of Pixel 2 XL it's necessary to unlock the
critical partitions.

    fastboot flashing unlock_critical

The command needs to be confirmed on the device.

Next, extract the factory images and run the script to flash them. Extracting the factory images
depends on the OS you're using and the applications available.

* [Using 7zip on Windows](https://itstillworks.com/untar-windows-8537010.html)
* [Extracting files in Mac](https://support.apple.com/en-ca/guide/terminal/apdc52250ee-4659-4751-9a3a-8b7988150530/mac)

*$DEVICE will either be crosshatch, taimen, walleye, sargo, bonito or blueline. The $DATE of the factory
image will correspond to our [releases](/updates).*

Note that the ```fastboot``` command run by the flashing script requires a fair bit of free
space in a temporary directory, which defaults to ```/tmp```:

    tar xvf $DEVICE-factory-$DATE.tar.gz
    cd $DEVICE-factory-$DATE
    ./flash-all.sh

Use a different temporary directory if your ```/tmp``` doesn't have 2GiB available:

    mkdir tmp
    TMPDIR=$PWD/tmp ./flash-all.sh

You should now proceed to locking the bootloader before using the device as locking wipes the data
again.

## Pixel 3/3a: setting custom AVB key

The custom AVB key for Pixel 3 and 3a series should automatically
be set upon flashing.

## Pixel 2 and 2 XL: Setting custom AVB key

* [Pixel 2 AVB key available here](https://releases.copperhead.co/images/walleye-avb_pkmd.bin)
* [Pixel 2XL AVB key available here](https://releases.copperhead.co/images/taimen-avb_pkmd.bin)

On the Pixel 2 and Pixel 2 XL, the public key needs to be set for Android Verified Boot 2.0 before
locking the bootloader again

    fastboot flash avb_custom_key $DEVICE-avb_pkmd.bin

To confirm that the key is set, verify that ```avb_user_settable_key_set``` is ```yes```:

    fastboot getvar avb_user_settable_key_set

## Locking the bootloader

Locking the bootloader is important as it enables full verified boot. It also prevents using
fastboot to flash, format or erase partitions.  Verified boot will detect modifications to any of
the OS partitions (boot, recovery, system, vendor) and it will prevent reading any modified /
corrupted data. If changes are detected, error correction data is used to attempt to obtain the
original data at which point it's verified again which makes verified boot robust to non-malicious
corruption.

Reboot into the bootloader menu and set it to locked:

    fastboot flashing lock

The command needs to be confirmed on the device since it needs to perform a factory reset.

### Legacy bootloader locking

If your Pixel 2 XL is the same older model that requires unlocking of the critical_partition,
lock the critical_partitions again:

    fastboot flashing lock_critical


### Post installation precautions
OEM unlocking should be disabled again in the developer settings menu within the operating system.
This prevents unlocking the bootloader without access to the owner account. CopperheadOS prevents
bypassing the OEM unlocking toggle by wiping the data partition from the hidden recovery menu,
unlike stock Android.  You can still trigger factory resets from within the OS. Note that this
means that recovering a device with a forgotten password is not possible without Copperhead doing
it, which is the main purpose of this feature (anti-theft).  Stock Android can be more forgiving
because it's tied to a Google account.

**Note: Unlocking the bootloader again will perform a factory reset. Do not plan to do
so unless otherwise requried.**

You should now disable OEM unlocking and factory reset the CopperheadOS device
for optimal hardware security.



## Updating

### Update client

See [the section in the usage guide](/usage_guide#updates-on-pixel-phones).

### Sideloading

Updates can also be downloaded from Copperhead and installed via
recovery with adb sideloading. The zip files are signed and will be verified by the CopperheadOS
recovery image. Sideloading should only be done if OTA updates are not working and there is a
critical issue with Updater.

First, boot into recovery. You can do this either by using ```adb reboot recovery``` from the
operating system or selecting the Recovery option in the bootloader menu.

You should see an Android lying on their back being repaired, with the text "No command" meaning
that no command has been passed to recovery.

Next, access the recovery menu by holding down the power button and pressing the volume up button
a single time. This key combination toggles between the GUI and text-based mode with the menu and
log output.

Finally, select the "Apply update from ADB" option in the recovery menu and sideload the update
with adb:

    adb sideload $DEVICE-ota_update-$DATE.zip

## Clearing custom AVB key

On the Pixel 2 and Pixel 2 XL, reverting back to stock requires clearing the configured public
key after unlocking the bootloader and before locking it again with the stock factory images:

    fastboot erase avb_custom_key

To confirm that the key is unset, verify that ```avb_user_settable_key_set``` is ```no```:

    fastboot getvar avb_user_settable_key_set
