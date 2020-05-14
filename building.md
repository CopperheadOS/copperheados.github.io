---
layout:      default
title:       Building
description: Instructions on building CopperheadOS from source and signing releases
---


# Building CopperheadOS
{: .no_toc}

* TOC
{:toc}

## Community Builders Initiative (CBI)
### What is it
Building Android from source can be an intimidating task especially when compared
to building Android applications or compiling components from system languages. CBI
acts as a bridge between Copperhead's OS experts, who have years of experience with
Android, and users who are looking to benefit from building CopperheadOS from source
in-house.

The community builders initiative is a channel from Copperhead to assist external builders
looking to expand on a CopperheadOS deployment for commercial and/or non-profit purposes. Personal
device users may benefit from this channel as well though the initiative will really
benefit larger (>10 devices) deployments.

### Who
There are many types of individuals, organisations and businesses that benefit from
involvement in CBI. Having control over the source code of an internal deployment
is beneficial in that the company controls the signing keys and may have policies
in place that won't keep up with official CopperheadOS releases.

The possibilities are endless and some examples include:
  * Government agencies with strict internal compliance procedures
  * Individuals looking to be CopperheadOS distributors
  * Non-profits with donated hardware
  * Organisations selling software solutions (ie: encrypted messaging)
  * Companies looking to white-label build CopperheadOS
  * Those looking to port CopperheadOS to a non-supported device
  * .. and many more.

### Benefits
Members of CBI receive:
  * Support channel access
  * 3 free commercial licenses for demo/testing purposes
  * Policy assistance for internal organisation deployments
  * Usage tips and integration guides for the latest security software

### Prerequisites
Before enrolling in CBI members are expected to:
* Fully read and comprehend our [build documentation](/building#copperheados-build-instructions)
* Understand Copperhead's [licensing](/licensing)
* Build CopperheadOS/Chromium from source
* Have the pre-existing hardware and access to devices they are building for

### Get involved

Email us [builders@copperhead.co](mailto:builders@copperhead.co) to get started
in the enrollment process. If you
experience issues building CopperheadOS from source please let us know
on our official bugtrackers. General OS bugs [can be filed here](https://github.com/CopperheadOS/bugtracker)
while device specific bugs can be filed on their respective issue trackers (ie: [Pixel 2 XL](https://github.com/CopperheadOS/device_google_taimen/issues)).  We look
forward to working with you.

## CopperheadOS build instructions
## Build dependencies

This documentation assumes you're using Ubuntu 18.04. Contact
[os_team@copperhead.co](mailto:os_team@copperhead.co) if you've had success on other OS's.

* x86/64 Linux build environment (OS X is not supported unlike AOSP)
* Android Open Source Project build dependencies
* Linux kernel build dependencies
* 16GiB of memory or more
* 200GiB of free storage space


## Supported devices

CopperheadOS currently has official build support for the following devices:

* Pixel 2 (walleye)
* Pixel 2 XL (taimen)
* Pixel 3 (blueline)
* Pixel 3 XL (crosshatch)
* Pixel 3a (sargo)
* Pixel 3a XL (bonito)

In the past CopperheadOS supported:

* HiKey (hikey)
* HiKey 960 (hikey960)
* Nexus 5X (bullhead)
* Nexus 6P (angler)
* Pixel (sailfish)
* Pixel XL (marlin)

It can be ported to other Android devices with Treble support via the standard [device porting process](/building#device-porting-process). Most devices lack support for the
[security requirements](/devices#minimum-requirements-for-copperheados-support)
needed to match how it works on the officially supported devices.

## Downloading source code

Since this is syncing the sources for the entire operating system and application layer, it will
use a lot of bandwidth and storage space.

You likely want to use the most recent stable tag, not the development branch, even for developing
a feature. It's easier to port between stable tags that are known to work properly than dealing
with a moving target.

### Development branch

The Android10 branch is used for all the currently supported Pixel family devices (1 through 3a and XLs) and other devices:

    mkdir copperheados
    cd copperheados
    repo init -u git@gitlab.com:copperheadsec/copperhead-source-partner-access/manifest.git -b android10 --manifest-name partner.xml
    repo sync -j32

If your network is unreliable and ```repo sync``` fails, you can run the ```repo
sync``` command again as many times as needed for it to fully succeed.

### Updating and switching branches/tags

To update the source tree, run the ```repo init``` command again to select the branch or tag and then
run ```repo sync -j32``` again. You may need to add ```--force-sync``` if a repository from switched from
one source to another, such as when CopperheadOS forks an additional Android Open Source Project
repository. You don't need to start over to switch between different branches or tags. You may
need to run ```repo init``` again to continue down the same branch since CopperheadOS only provides a
stable history via tags.

## Generating release signing keys

Keys need to be generated for resigning completed builds from the publicly available test keys.
The keys must then be reused for subsequent builds and cannot be changed without flashing the
generated factory images again which will perform a factory reset. Note that the keys are used for
a lot more than simply verifying updates and verified boot. Keys must be generated before building
for the Pixel and Pixel XL due to needing to provide the keys to the kernel build system, but this
step can be done after building for Nexus devices.

The keys should not be given passwords due to limitations in the upstream scripts. If you want to
secure them at rest, you should take a different approach where they can still be available to the
signing scripts as a directory of unencrypted keys. The sample certificate subject can be replaced
with your own information or simply left as-is.

Pixel and Pixel XL use Android Verified Boot 1.0. The Pixel 2, 3, 3a and the XL models
all use Android Verified Boot 2.0 (AVB). Use the command below to generate all keys necessary.

    ./script/gen-all-keys

## Building

The syntax for building a signed build is.

    ./build.sh -r -t target-files-package -d <device>

The following are the complete list of script options.

    -d  --device     set device to build"
    -t  --target     set make target"
    -s  --sign       sign package"
    -r  --release    build release build"
    -b  --build_id   set build number"
    -c  --clean      clean build"

## Prebuilt code

Like the Android Open Source Project, CopperheadOS contains some code that's built separately and
then bundled into the source tree as binaries. Ideally, everything would be built-in tree with the
AOSP build system but it's not always practical.

### Kernel

Unlike AOSP, CopperheadOS builds the kernel as part of the operating system rather than bundling a
pre-built kernel image.

### F-Droid

F-Droid is bundled as an apk in the external/F-Droid repository.

The privileged extension built from source from the [privileged-extension](https://github.com/CopperheadOS/platform_packages_apps_F-Droid_privileged-extension)
repository as part of the normal build process.

## Device porting process

Early AOSP port:

- implement userdebug test-keys builds, likely with android-prepare-vendor
- implement full signing for proper release-keys builds (including verified boot)
- configure and enable verified boot on the device

AOSP testing:

- test verified boot for all OS partitions (vbmeta, boot, dtbo, system, vendor for the Pixel 2)
- test verified boot rollback protection
- test key attestation support for verified boot attestation
- test that over-the-air updates work properly including updating all firmware images and vendor.img (with verified boot enabled to catch issues)
- test that incremental over-the-air updates work properly
- run CTS and record results

AOSP polish:

- improve port as needed until it's solid enough to work on CopperheadOS support

Early CopperheadOS port:

- minimal changes in the device repository if necessary to get it working with CopperheadOS changes
- work around or fix any latent device-specific memory corruption bugs that are occurring in
  regular use and break with CopperheadOS exploit mitigations

Early CopperheadOS testing:

- run through the AOSP testing step again with CopperheadOS
- compare CTS results with AOSP and make sure the difference matches the expected list of failures from intentional breaks in compatibility and AOSP bugs uncovered by mitigations that are not yet fixed

Full CopperheadOS port:

- port all device-specific changes to the device repository (Pixel 2 as the reference)
- integrate device into release, repository management and over-the-air update server scripting
- adapt device-specific extensions to SELinux policy to CopperheadOS SELinux policy hardening as necessary
- implement custom kernel builds
- Implement stable-base branch with all kernel.org LTS kernel changes cherry-picked + conflicts and other issues resolved. Keep empty commits, mark each commit with conflicts resolved at the bottom of the commit message with \[CopperheadOS\] and uncomment the automatically generated Conflicts list below that. See [the Pixel 2 kernel stable-base branch](https://github.com/CopperheadOS/kernel_google_wahoo/tree/oreo-m2-s2-release-stable-base) for an example. This branch should be kept rebased with history preserved via tags for each release.
- strip down kernel configuration to a minimum (Pixel 2 defconfig as the reference)
- implement monolithic kernel builds with modules disabled
- implement Clang-compiled kernel builds using the CopperheadOS Clang toolchain
- enable -fsanitize=local-init compiler feature for the kernel
- apply other CopperheadOS kernel hardening changes on top of the kernel.org LTS cherry-picks. Use the [Pixel 2 kernel branch](https://github.com/CopperheadOS/kernel_google_wahoo/tree/oreo-m2-s2-release) as a reference. This branch should be kept rebased with history preserved via tags for each release.
- implement specialized scanning MAC randomization for the Wi-Fi driver if not already implemented
- implement specialized associated MAC randomization for the Wi-Fi driver if not already implemented (Pixel as the reference for qcacld-2.0, Pixel 2 as the reference for qcacld-3.0)

Full CopperheadOS testing:

- repeat the early CopperheadOS testing step again now that device-specific changes are ported

## Redistribution

CopperheadOS kernel code is GPL2, so derivatives using only the kernel changes simply need to
respect the usual GPL2 rules by making the sources needed to build the kernel available, etc.

CopperheadOS userspace code is primarily licensed under the [Creative Commons
Attribution-NonCommercial-ShareAlike 4.0
International](https://creativecommons.org/licenses/by-nc-sa/4.0/) license so a commercial license
is required to earn money from derivatives using the userspace code. Licensing can be based on
revenue sharing so don't be afraid to contact sales@copperhead.co for small scale commercial
licensing.

CopperheadOS art and branding is primarily licensed under the [Creative Commons
Attribution-NonCommercial-NoDerivatives 4.0
International](https://creativecommons.org/licenses/by-nc-nd/4.0/) license. Derivatives that are
being distributed need to replace the boot animation, wallpaper and other art / branding. The
current branding to replace, which will expand over time:

* CopperheadOS boot animation (remove to use AOSP boot animation, or replace it)
* CopperheadOS wallpapers (remove to use AOSP default wallpapers)
* CopperheadOS Chromium branding (revert to Chromium branding, or replace it)

The 'CopperheadOS' branding itself is trademarked. Derivatives should come up with their own
project name and globally replace 'CopperheadOS' with 'NewProjectName' if they're being
distributed. They should state that they use code based on CopperheadOS / ported from CopperheadOS
but shouldn't claim to actually be CopperheadOS itself. The current list of strings to replace,
which will expand over time:

* ro.build.user and ro.build.host system properties are set to copperheados for reproducible
  builds and should be set to a different value
* kernel build user and host are set to copperheados for reproducible builds and should be set to
  a different value
* Clang toolchain version string (Android was replaced with CopperheadOS)
* keyboard (packages/inputmethods/LatinIME) branding strings (Android Open Source Project was
  replaced with CopperheadOS)
* update client (packages/apps/Updater) branding strings
* recovery branding string (Android was replaced with CopperheadOS)
