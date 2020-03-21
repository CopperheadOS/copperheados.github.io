---
layout:      default
title:       Copperhead Auditor app privacy policy
description: Privacy policy for the attestation app
---

# Copperhead Auditor app privacy policy

This app uses the camera permission to scan QR codes from the app running on another device. The
images captured by the camera are not stored and the data is not used beyond decoding QR codes
used to directly implement the functionality of the app.

The QR codes do not contain any personal information. The QR codes contain data for performing
verification of OS integrity and the identity of the app installation. The identity is randomly
generated and reset by clearing the app data or uninstalling / reinstalling the app. The codes
also contain some information gathered from the OS to display to the user on the device doing
verification, which may be extended in the future:

* Whether the user profile is secured with an authentication mechanism (yes or no)
* Whether an accessibility service is activity (yes or no)
* Whether a device manager is active (yes or no)

The app does not work with any other personal or sensitive data. It doesn't make network requests
or communicate with other apps. Other than the user-initiated QR code generation, the app keeps
all data internal and inaccessible to other apps or any servers.

On the Auditee side, the only persistent data is a signing key. On the Auditor side, the only
persistent data is the pairing information for each verified device:

* Pinned public key certificate chain
* Device variant (Pixel 2 vs. Pixel 2 XL)
* Operating system (Stock vs. CopperheadOS)
* OS version (at the time of the last verification)
* OS patch level (at the time of the last verification)
* App version (at the time of the last verification)
* First verification time
* Last verification time

No data is stored if the device does not pass verification. More data may be pinned in the future.

Alternatives to QR codes may be provided in the future and will be similarly user-initiated. For
example, if support for remote attestation via the network is added, it is going to require the
user to explicitly set it up with a chosen server.

Technical details on the operation of the app are [documented in the source
code](https://github.com/copperhead/Attestation/blob/18fc1c9040ab25b79656299aa04d2453a372a22c/app/src/main/java/co/copperhead/attestation/AttestationProtocol.java#L80-L148)
which is published in full for non-commercial use primarily so that the implementation can be
audited and tested. Paying others to conduct commercial audits is permitted despite the
non-commercial clause and you can contact us for explicit permission if you feel it's necessary.
