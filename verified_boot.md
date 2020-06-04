---
layout:      default
title:       Verified boot / remote attestation
description: Technical information on verified boot and remote attestation
---

# Verified boot / remote attestation

Verified boot is an important security feature, primarily aimed at making it substantially harder
for an attacker to persistently compromise the OS. It also provides basic resistance against
tampering with a device after gaining physical access. For more user-oriented information such as
details on the user experience and instructions on using the Auditor app for attestation, see [the section in the usage guide](/android/docs/usage_guide#verified-boot).

In the primary use case, it serves as an extra line of defence after an attacker has gained remote
code execution (either with an exploit chain or by getting the user to install an app) and then
escalated to kernel level access. Verified boot aims to prevent the attacker from making
modifications to any firmware or the operating system, along with preventing downgrades to past
vulnerable versions. It aims to force the attacker into persisting via state outside the operating
system (i.e. the userdata partition) and exploiting the OS again from there each time it boots. An
important security property it aims to provide is that a factory reset can fully wipe away malware
even if it has used exploits to gain root access. A much harder path to privileged persistence
would be to exploit the verification process as it happens.

CopperheadOS implements full verified boot like stock Android operating systems, and also improves
it by reducing the trust placed in persistent state (userdata partition).

The main weakness of verified boot is not the verification implementation itself but the fact that
there's a lot of persistent state in userdata. As a thought experiment, consider what would end
up happening if the OS supported app-accessible root access (which isn't even provided by either
AOSP or CopperheadOS in a debug build). An attacker could simply persist as root by granting root
access to their malicious app, which would wipe out many of the benefits of verified boot. The
situation is much better than that, but still far from ideal:

- an attacker that has gained root access can generally still *brick* the device
- they can modify any persistent state: files, POSIX permissions SELinux labels, raw disk structure
- can exploit the OS via persistent state, which is why it's important for verified boot to
  include downgrade protection so an attacker can't simply install a more vulnerable past release
  of the OS
- /data/dalvik-cache holds code / trusted data for the base system
    - mostly solved by CopperheadOS already, but more work is required for completion
    - even standard AOSP now avoids using this for ```system_server```
- various ways to block the automatic updater from working
- could do things like partially corrupting the userdata filesystem to prevent the OS from trying
  to repair or validate a portion of it
- can target the file system implementation for exploitation via the disk format
- install non-system apps
- grant apps runtime permissions
- mess with package manager metadata and other information that may be trusted to a large extent
    - may be able to get system / signature gated permissions, etc. if this isn't done well, so
      this needs to be audited
    - should also make sure it's not possible to do a 'fake' system app update, which should
      already be impossible due to apk signatures and verification on every boot but this really
      needs auditing from a verified boot perspective
- accessibility service
- device manager
- direct boot to start apps before the first unlock
    - the updater uses this, so it's notable that without direct boot being available to
      non-system apps, the updater gets to run before any other app could run
- direct boot + accessibility service quite soundly rules out any kind of reporting to the user in
  late boot unless it's something that can actually be verified (i.e. the attestation support now
  provided by TEE)
- developer options, adb key whitelist

## Summary of current generation verified boot

### Early firmware

Early firmware (i.e. loaded before the OS) is verified from a hardware root of trust. Standard
mobile SoC platforms allow the device vendor to irreversibly flash their keys into fuses so that
early firmware is signed by the device vendor rather than the SoC vendor. Early firmware includes
the boot chain leading up to the OS kernel, TrustZone, the cellular baseband, fastboot mode, etc.

Each component of the boot chains verifies the next component before it's loaded, leading up to
the late stage bootloader (such as Qualcomm's [lk-based
bootloader](https://source.codeaurora.org/quic/la/kernel/lk/) or [TianoCore-based
bootloader](https://source.codeaurora.org/quic/la/abl/tianocore/edk2/)) which is responsible for
verifying and loading the OS. Sample verification implementations are available in Qualcomm's open
source bootloader projects on the CodeAurora site (see the previous links). Device vendors usually
make tweaks to the bootloaders, but Qualcomm's reference implementations aren't very far from what
most devices using their SoC are shipping so it's a good place to look.

Later firmware components and TrustZone apps may have their rollback protection enforced via the
Replay Protected Memory Block instead of fuses if the device vendor wants more flexibility, such
as being able to unlock the bootloader and downgrade components since increasing the version in
the fuses cannot be undone. Ideally, fuses would be used for everything, but security needs to
coexist with development needs, especially on devices supporting unlocking.

The main limitation of the early verified boot process is reluctance from device vendors to make
use of the available rollback protection by actually increasing the rollback version. They may
want to support using the devices for testing past releases of the OS paired with older firmware
and this form of security is in conflict with that.

### OS (Android Verified Boot 2.0)

Verified boot becomes more interesting with the OS, where it shifts into Android Verified Boot 2.0
from the SoC platform implementation of verified boot.

The late stage bootloader starts by verifying vbmeta, which is a tiny signed operating system
partition containing metadata needed to verify the rest of the OS. RSA2048 / SHA256 is the
baseline standard signature algorithm and is used by the Pixel 2. The vbmeta data structure
contains a rollback index, which is used to provide rollback protection for the OS. On the Pixel
2, the rollback index gets stored in the Replay Protected Memory Block.

If the device isn't unlockable, verifying the OS uses a hard-wired key. If the device is
unlockable like the Pixel 2, it also supports a non-verified mode when the bootloader is unlocked
which shows a warning for at least 5s (10s on the Pixel 2). Devices supporting locking the
bootloader with another OS (like the Pixel 2) have support for flashing a public key when the
bootloader is unlocked and using that as an alternative to the hard-wired key, with a notice about
an alternate OS being installed showing the fingerprint of the key on boot (for at least 5s, and
the Pixel 2 also uses 10s for this). The Pixel 2 stores the public key in the Replay Protected
Memory Block with the rollback indexes.

The vbmeta data structure hash hashes needed to verify the other partitions and to bootstrap
verification of large partitions via hash trees. On the Pixel 2, it has hashes for the boot and
dtbo partitions and hash tree metadata for system and vendor, which together with vbmeta are the
full set of OS partitions.

Once the bootloader is finished with verification, it passes the verified boot state and verified
boot public key to the Trusted Execution Environment TEE and loads / runs the kernel from the boot
image. The kernel is responsible for using dm-verity to verify the rest of the OS via hash trees.

For more details see the [official AVB
documentation](https://android.googlesource.com/platform/external/avb/+/master/README.md).

### TEE and hardware-backed keystore integration

The TEE uses the verified boot key as input for disk encryption keys. It also makes use of the
verified boot state, OS version and OS patch level to provide protection for the hardware-backed
keystore as an indirect form of verified boot enforcement and downgrade protection. Even if the
rollback index isn't incremented for a new version, there's still downgrade protection.

The integration with the keystore means that hardware-backed keys can be used for remote
attestation of the verified boot state. In fact, there's a fancy [key
attestation](https://developer.android.com/training/articles/security-key-attestation.html)
feature for having the TEE sign the public key certificate of a newly generated hardware-backed
key with an attestation key chaining to a known attestation root. An attacker that has gained root
/ kernel access can't fake this, but they could exploit the TEE to bypass this. Android 9.0 brings
support for a [StrongBox keymaster implemented via a dedicated
HSM](https://developer.android.com/preview/features/security#hardware-security-module), which will
probably be present on the Pixel 3. Ideally, various other improvements will be made to these
attestation capabilities too. The current implementation is far from ideal, but it's already a
very useful feature and a huge improvement over not being able to meaningfully inspect devices
like this.

## Remote attestation

Copperhead uses the hardware-backed keystore with key attestation to implement [our Auditor
app](https://github.com/copperhead/Auditor) which provides both local verification from another
Android device (via QR codes). The app also has support for regularly scheduled remote
verification using our [attestation server](https://github.com/copperhead/AttestationServer)
hosted at [https://attestation.copperhead.co/](https://attestation.copperhead.co/).

Our Auditor app builds on the baseline verified boot for firmware and the entire operating system
by gathering information from the OS. For example, it can detect that an attacker persisted via an
accessibility service or device administrator. The TEE attestation provides the app id and key
fingerprint, which is what bootstraps the OS enforced checks. An attacker with root / kernel
access could fake the OS enforced checks, but the OS version / OS patch level are provided by the
key attestation feature and mitigate simply holding back the OS version to continue reliably
exploiting it. The Auditor app capabilities will be substantially expanded in the future. It's not
magic, but it does offer real security properties and it will be very useful for monitoring the
health of a fleet of Android devices, or just one device.

For more details on the Auditor app, see the [documentation on the attestation protocol in the
sources](https://github.com/copperhead/Auditor/blob/1e255dec90f1cec34823fbbd1d08706e6a30b541/app/src/main/java/co/copperhead/attestation/AttestationProtocol.java#L109-L177).
