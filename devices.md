---
layout:      devices
title:       Device comparison
description: Comparison between past and current devices supported by CopperheadOS
---

# CopperheadOS device comparison
{: .no_toc}

* TOC
{:toc}

This is a comparison of the security features of devices currently or previously supported by
CopperheadOS along with those we plan to support in the future.

Obsolete devices no longer supported by Android or CopperheadOS like the Nexus 5 are included to
document historical progress.

HiKey and HiKey 960 are supported devices but aren't compared here because they're primarily
development platforms and it wouldn't make sense to compare the (lack of) security properties.

The Samsung Galaxy S4 International LTE variant (jfltexx) used to be supported by CopperheadOS
Alpha but we see that as a distinct operating system from CopperheadOS Beta and later where it
became directly based on the Android Open Source Project and moved to fully signed production
builds.

## Minimum requirements for CopperheadOS support

The following security properties are provided by our current generation hardware targets and are
considered hard requirements for expanding CopperheadOS support to future devices:

* Verified boot, including:
    * verification of all firmware
    * verification of the entire operating system
    * public key fingerprint display
    * public key enforcement for the operating system via tamper evident storage
    * rollback protection for the operating system via tamper evident storage
* Hardware key derivation support
    * key for a substantial portion of the derivation work unavailable to firmware/software
    * enforcement of escalating delays via a dedicated HSM
* Hardware-backed keystore (TEE or better)
    * key + verified boot attestation (does not need to use the Google key attestation root)
* Hardware random number generator for use as an input to the kernel CSPRNG
    * Must provide adequate entropy in early boot before drivers to use specialized hardware are
      available. Implementing EFI\_RANDOM\_PROTOCOL works. Setting /chosen/kaslr-seed in the
      device tree isn't good enough as it's only enough for KASLR (64-bit).
* Treble driver model (uninvasive support and sandboxed HALs)
* Ongoing maintenance including security updates for all firmware and device-specific components
  with at least 2 years of support remaining for firmware, etc. from every vendor involved.
* A/B update support including automatic rollback on initial boot failure and verified boot
  integration for rollback protection
* 64-bit CPU
* LPDDR4 or later with TRR support to mitigate rowhammer
* Driver support for at least Linux 4.4 (ideally mainline drivers). Linux 4.9 is the baseline for
  devices launched in 2018.
* No closed-source drivers in the kernel (binary drivers can be audited, but not easily hardened)
* Proper scanning MAC randomization support avoiding identifiers other than the MAC address like a
  non-randomized probe sequence number

CopperheadOS can be [ported to other devices](/building#device-porting-process) but
will only officially support devices meeting these requirements.

### Expected near future requirements

We expect to make the following into requirements by the end of 2018:

* Public key fingerprint display for verified boot as a true security feature i.e. less truncation
* Hardware-backed keys via a dedicated HSM

### Further future requirements

* ECC memory
* Superior rowhammer mitigations for memory than TRR
* No closed-source drivers in userspace (binary drivers can be audited, but not easily hardened)
* Far simpler boot chain without UEFI again (regressed in 2017)
* Firmware information as part of the attestation capabilities
* Improved security for pinning the attestation certificate chain (i.e. not only batches) to
  provide more assurance for the OS version and OS patch level reported by our Auditor app
* Significantly more active use of existing anti-rollback features for firmware
* Hardware switch for audio recording (microphone + motion sensors). A camera switch can be
  implemented via a case with sliding covers instead, which is arguably a better approach as it's
  much easier to see the current state (harder to screw up) and it's easier to audit.
* Copperhead keys as an immutable root of trust (i.e. flashed into fuses) with Copperhead having
  control over the boot chain. Does not need to imply our involvement in making the hardware, as
  devices could be sold without the security fuses flashed yet.
    * Ideally, the hardware would also provide enough anti-rollback fuses for us to use fairly
      fine-grained rollback protection too, so we can use immutable rollback protection for OS
      versions.

## Support status

Note that the end-of-life date is a minimum guarantee from the OEM and is subject to change.
CopperheadOS can continue OS updates past end-of-life for a long time but full security updates
require continued releases of the firmware for components like the baseband, WiFi, TrustZone, etc.

| Device     | Branch            | End-of-life date |
| ---------- | ----------------- | ---------------- |
| Pixel 3a   | 10 (current)      | May 2022         |
| Pixel 3aXL | 10 (current)      | May 2022         |
| Pixel 3    | 10 (current)      | October 2021     |
| Pixel 3XL  | 10 (current)      | October 2021     |
| Pixel 2XL  | 10 (current)      | October 2020     |
| Pixel 2    | 10 (current)      | October 2020     |
| Pixel XL   | 10 (EOL)          | December 2019    |
| Pixel      | 10 (EOL)          | December 2019    |
| Nexus 6P   | Oreo M3 (EOL)     | November 2018    |
| Nexus 5X   | Oreo M2 (EOL)     | November 2018    |
| Nexus 9    | n/a               | October 2017     |
| Nexus 5    | n/a               | October 2016     |

## Driver model

| Device     | Treble |
| ---------- | ------ |
| Pixel 3aXL | Yes    |
| Pixel 3a   | Yes    |
| Pixel 3    | Yes    |
| Pixel 3XL  | Yes    |
| Pixel 2XL  | Yes    |
| Pixel 2    | Yes    |
| Pixel XL   | Yes    |
| Pixel      | Yes    |
| Nexus 6P   | No     |
| Nexus 5X   | No     |
| Nexus 9    | No     |
| Nexus 5    | No     |

In addition to [making future updates substantially
easier](https://source.android.com/devices/architecture/treble) and improving testing /
verification, Treble substantially improves security by [splitting up the HAL implementation into
many isolated processes and reducing kernel attack
surface](https://android-developers.googleblog.com/2017/07/shut-hal-up.html).

## Bootloader firmware

| Device     | Verified boot | Rollback protection | Key enforcement         | OS public key fingerprint display   | A/B update support | Serial debugging while locked | OEM unlocking toggle | Anti-theft protection |
| ---------- | ------------- | ------------------- | ----------------------- | ----------------------------------- | ------------------ | ----------------------------- | -------------------- | --------------------- |
| Pixel 2 XL | Full          | Yes                 | Direct + via encryption | Strong implementation in progress   | Yes                | Restricted                    | Yes                  | Yes                   |
| Pixel 2    | Full          | Yes                 | Direct + via encryption | Strong implementation in progress   | Yes                | Restricted                    | Yes                  | Yes                   |
| Pixel XL   | Full          | No                  | Only via encryption     | Strong implementation in progress   | Yes                | Restricted                    | Yes                  | Yes                   |
| Pixel      | Full          | No                  | Only via encryption     | Strong implementation in progress   | Yes                | Restricted                    | Yes                  | Yes                   |
| Nexus 6P   | Full          | No                  | Only via encryption     | Weak (64-bit via 16 hex characters) | No                 | Yes (despite toggle)          | Yes                  | Without boot password |
| Nexus 5X   | Full          | No                  | Only via encryption     | Weak (48-bit via 12 hex characters) | No                 | Yes                           | Yes                  | No                    |
| Nexus 9    | Partial       | No                  | n/a                     | n/a                                 | No                 | Yes                           | Yes                  | Without boot password |
| Nexus 5    | Partial       | No                  | n/a                     | n/a                                 | No                 | Yes                           | No                   | No                    |

2nd generation Pixels introduce verified boot rollback protection (Android Verified Boot 2.0)
rather than only enforcing it in the update client and recovery.

A/B update support provides two sets of OS partitions (A and B slots) with support for safe atomic
swaps between them. This requires a fair bit of firmware support and includes automatic rollbacks
of updates after they're installed if booting the OS fails multiple times. The freshly booted new
OS version will verify the installation late in the booting process and will then mark the boot as
successful, otherwise rollback will happen after multiple attempts without success.

CopperheadOS has a modern update client for A/B update devices implementing fully automatic
updates with automatic resume after failure at any point in the download, verification,
installation and post-installation verification process. Previously, CopperheadOS used a fork of
the CyanogenMod / LineageOS CMUpdater app which is still the update client on CopperheadOS Nexus
devices and it isn't very friendly or robust and it isn't able to resume downloads or attempt to
perform installation again without deleting the download or starting over, unlike the new update
system.

## WiFi driver / firmware

| Device     | Vendor           | Robust scanning MAC randomization | Robust associated MAC randomization               |
| ---------- | ---------------- | --------------------------------- | ------------------------------------------------- |
| Pixel 2 XL | Qualcomm Atheros | Yes                               | Yes, but only at boot for now (CopperheadOS only) |
| Pixel 2    | Qualcomm Atheros | Yes                               | Yes, but only at boot for now (CopperheadOS only) |
| Pixel XL   | Qualcomm Atheros | Yes                               | Yes (CopperheadOS only)                           |
| Pixel      | Qualcomm Atheros | Yes                               | Yes (CopperheadOS only)                           |
| Nexus 6P   | Broadcom         | Partial (only rotated random MAC) | No                                                |
| Nexus 5X   | Qualcomm Atheros | Yes                               | No                                                |
| Nexus 9    | Broadcom         | Partial (only rotated random MAC) | Partial (CopperheadOS only)                       |
| Nexus 5    | Broadcom         | Partial (only rotated random MAC) | Partial (CopperheadOS only)                       |

See the Android blog post on [Changes to Device Identifiers in Android
O](https://android-developers.googleblog.com/2017/04/changes-to-device-identifiers-in.html) for an
overview. Qualcomm Atheros WiFi on Nexus / Pixel devices has enhanced drivers / firmware providing
more robust MAC randomization than can be accomplished by CopperheadOS via the usual
device-agnostic kernel and userspace MAC randomization support.

Associated MAC randomization similarly requires cooperation from the firmware and drivers to avoid
leaking other identifiers or the radio broadcasting before the MAC is randomized. On Pixel phones,
CopperheadOS uses a custom implementation for Qualcomm Atheros after determining that there was no
way to achieve the desired results via the standard MAC changing API.

## Trusted Execution Environment (TEE) firmware

| Device     | Key / verified boot attestation | Disk encryption keys     | Disk encryption key tied to verified boot key |
| ---------- | ------------------------------- | ------------------------ | --------------------------------------------- |
| Pixel 2 XL | Yes                             | Encrypted by TEE         | Yes                                           |
| Pixel 2    | Yes                             | Encrypted by TEE         | Yes                                           |
| Pixel XL   | Partial                         | Encrypted by TEE         | Yes                                           |
| Pixel      | Partial                         | Encrypted by TEE         | Yes                                           |
| Nexus 6P   | No                              | Encrypted by OS with TEE | Yes                                           |
| Nexus 5X   | No                              | Encrypted by OS with TEE | Yes                                           |
| Nexus 9    | No                              | Encrypted by OS with TEE | No                                            |
| Nexus 5    | No                              | Encrypted by OS with TEE | No                                            |

## Weaver

| Device     | Hardware key escrow for authentication / encryption |
| ---------- | --------------------------------------------------- |
| Pixel 2 XL | Yes                                                 |
| Pixel 2    | Yes                                                 |
| Pixel XL   | No                                                  |
| Pixel      | No                                                  |
| Nexus 6P   | No                                                  |
| Nexus 5X   | No                                                  |
| Nexus 9    | No                                                  |
| Nexus 5    | No                                                  |

The Pixel 2 introduced a dedicated Hardware Security Module (HSM) with support for performing key
escrow used to improve encryption key derivation. The OS needs to provide valid credential-derived
tokens to get back randomly generated tokens stored in the HSM. These random tokens are an extra
input for credential-derived key derivation. The HSM has a secure internal timer and enforces
exponentially growing delays on authentication failures, which results in decryption having
hardware-enforced throttling without actually needing to grant any trust to the HSM.

### Disk encryption keys

Disk encryption keys are randomly generated and stored encrypted via key encryption keys derived
from user credentials and other inputs. On older devices, the TEE was used as part of the process
of deriving the key encryption key with scrypt in the OS. On newer devices, the OS uses scrypt to
perform similar key derivation with scrypt and passes the output to the TEE. This puts the TEE in
a better position as it can perform hardware-bound key derivation via features like using Qualcomm
Crypto Engine HMAC support for hardware-bound HKDF. Details on the current implementation are not
currently made available but it's at least as good as the old implementation and the TEE is no
longer crippled by a lackluster API at the boundary with the OS.

## Memory

| Device     | Memory standard | TRR | ECC | Rowhammer susceptibility |
| ---------- | --------------- | --- | --- | -------------------------|
| Pixel 2 XL | LPDDR4          | Yes | No  | Moderate (varies)        |
| Pixel 2    | LPDDR4          | Yes | No  | Moderate (varies)        |
| Pixel XL   | LPDDR4          | Yes | No  | Moderate (varies)        |
| Pixel      | LPDDR4          | Yes | No  | Moderate (varies)        |
| Nexus 6P   | LPDDR4          | Yes | No  | Moderate (varies)        |
| Nexus 5X   | LPDDR3          | No  | No  | Very high (varies)       |
| Nexus 9    | LPDDR3          | No  | No  | Very high (varies)       |
| Nexus 5    | LPDDR3          | No  | No  | Very high (varies)       |

## Kernel hardening

Features supported by every currently supported device like heap canaries are not listed. Unlike
the tables above, this is software related so this table is specific to CopperheadOS and in theory
these features could be backported further if we had more resources. However, the priority would
be backporting more changes from mainline / linux-hardened to the latest devices.

| Device     | LTS Branch | PXN      | PAN      | HARDENED\_USERCOPY | FORTIFY\_SOURCE | RO protection | SSP                | ASLR                           | Clang + -fsanitize=local-init |
| ---------- | ---------- | -------- | -------- | ------------------ | --------------- | ------------- | ------------------ | ------------------------------ | ----------------------------- |
| Pixel 2 XL | 4.4        | Hardware | Software | Yes                | Yes             | Basic         | Strong + zero byte | Hardened, 39-bit address space | Yes                           |
| Pixel 2    | 4.4        | Hardware | Software | Yes                | Yes             | Basic         | Strong + zero byte | Hardened, 39-bit address space | Yes                           |
| Pixel XL   | 3.18       | Hardware | Software | Yes                | Yes             | Basic         | Strong + zero byte | Hardened, 39-bit address space | No                            |
| Pixel      | 3.18       | Hardware | Software | Yes                | Yes             | Basic         | Strong + zero byte | Hardened, 39-bit address space | No                            |
| Nexus 6P   | 3.10       | Hardware | No       | No                 | No              | Weak          | Strong + zero byte | Basic, 39-bit address space    | No                            |
| Nexus 5X   | 3.10       | Hardware | No       | No                 | No              | Weak          | Strong + zero byte | Basic, 39-bit address space    | No                            |
| Nexus 9    | 3.10       | Hardware | No       | No                 | No              | Terrible      | Basic              | Basic, 39-bit address space    | No                            |
| Nexus 5    | 3.4        | No       | No       | Yes                | No              | Terrible      | Basic              | Hardened, 32-bit address space | No                            |

## Encryption

See the [relevant Android documentation](https://source.android.com/security/encryption/) for
details. File-based encryption (FBE) has per-profile encryption keys and splits up storage into
credential encrypted (default) and explicitly opt-in device encrypted storage available before the
device is unlocked. For example, the modern CopperheadOS Updater app marks itself as Direct Boot
aware and marks the update settings as device encrypted in order to perform updates before the
device is unlocked. It enables fully automatic maintenance of an idle device. In the future, FBE
will also enable authenticated encryption and storage classes for protecting data while locked by
dropping a set of keys derived from the user credentials from memory when it locks. Devices are
almost always turned on and keys can be physically extracted from a device that's turned on so FBE
is crucial for improving disk encryption to meet real world needs. It's possible to protect data
while locked today, but app developers are too lazy to do it via the keystore and need an easy
mechanism via a new FBE storage class.

The drawback of FBE is that it can leak more metadata when comparing devices that are turned off.
For example, even though file names are encrypted some information can be gained based on their
size. CopperheadOS increases the padding of file names from the default 4 bytes to 32 bytes. The
protection of other metadata will improve over time as part of Linux ext4 development. Device
encrypted data is very explicitly opt-in and few apps take advantage of it so that isn't a serious
concern for apps, but it is one for the base system since it uses it a fair bit to implement a
fully functional OS in early boot. Our view is that these issues are not major ones and can be
worked through. The advantages of FBE will far outweigh the disadvantages in the future. It would
be possible to layer FBE on top of FDE but there would need to be a boot password again for it to
accomplish much and that would lose the usability advantages of Direct Boot along with truly fully
automatic update support. It's something we can consider for the future, but mitigating the side
channel metadata leakage for devices that are turned off is far less important to us than
advancing protection of data while the screen is locked by moving as much as possible to a new
storage class.

File-based encryption requires firmware support to be fully implemented. There was an experimental
version ported by Google to the Nexus 5X and 6P but it shouldn't be used beyond testing and cannot
be considered robust or secure. It isn't quite the same as the real thing and the legacy update
client on CopperheadOS Nexus devices cannot perform updates if it's enabled.

| Device     | Mode |
| ---------- | ---- |
| Pixel 2 XL | FBE  |
| Pixel 2    | FBE  |
| Pixel XL   | FBE  |
| Pixel      | FBE  |
| Nexus 6P   | FDE  |
| Nexus 5X   | FDE  |
| Nexus 9    | FDE  |
| Nexus 5    | FDE  |
