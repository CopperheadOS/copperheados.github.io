---
layout:      default
title:       CopperheadOS usage guide
description: Recommendations on how to use CopperheadOS effectively.
---
*Proceed with caution: This documentation is being worked on and out of date.*

# CopperheadOS usage guide
{: .no_toc}

CopperheadOS is a Linux-based mobile operating system with a focus on privacy and security. It
builds on the latest stable release of the Android Open Source Project (10) which is Android
without any Google apps and services.

This usage guide documents the user experience of the OS with a focus on how it compares to the
Android operating system. Most security features are kept unintrusive without a user-facing
impact. See the [technical overview](/android/docs/technical_overview) for
documentation on the security features rather than only changes to the user experience.

CopperheadOS can run Android apps unless they have a hard dependency on Google services. It ships
with the F-Droid app store to provide access to thousands of open source apps but it can also run
many apps from the Amazon Appstore and Google Play Store.

Since it doesn't include Google apps and services, CopperheadOS isn't limited by the requirements
imposed by Google for an operating system to be considered Android compatible. It disregards the
constraints required to actually be Android while still retaining the ability to run nearly all
real world Android apps not tied to Google. Android devices aren't permitted to ship many of the
features documented below like the Network permission toggle.

* TOC
{:toc}

## Permission model

Apps run in an application sandbox with access to little more than their internal data and
interfaces explicitly exported to them from the base system and other apps. Most of their
capabilities need to be granted by the permission system. Permissions are divided into normal
permissions like the ability to set an alarm or start at boot and dangerous permissions that need
to be explicitly granted by users.

CopperheadOS and stock Android use the same runtime permission model for apps targeting the modern
Android platform. Dangerous permission groups (camera, contacts, microphone, etc.) are disabled by
default and apps need to request them from the user as needed with the expectation that they
attempt to handle not getting the permissions. The dangerous permission groups can also be toggled
via Settings -> Apps & notifications -> App info -> App name -> Permissions and modern apps can be
expected to handle this and request the permissions again as needed. Dangerous permission groups
can also be audited / toggled by group instead of by app using Settings -> Apps & notifications ->
App permissions.

For apps targeting old versions of the Android platform, stock Android grants all requested
dangerous permissions at install time. The dangerous permission groups can still be toggled off
via Settings -> Apps & notifications -> App info -> App name -> Permissions and provide empty data
where possible rather than failing for compatibility. By contrast, CopperheadOS presents the user
with a menu where they can review and disable dangerous permission groups before the app is able
to run. The substantial difference is that stock Android doesn't give the user a chance to toggle
off permissions before the app is able to run. Installing it and toggling them off before manually
launching the app is not adequate as it's possible for it to be started before that.

Note that Android makes it possible for apps to get by without permissions in most cases. An app
can have the user pick a contact, take a picture, store / open a file in a storage provider, etc.
without any permissions required. Be skeptical about the justification given by app developers for
their permission usage. For example, many claim to need to read the phone state (Phone permission
group) to detect an active phone call but it's almost always a false justification as they could
be using the standard audio focus mechanism.

### Network permission

Unlike stock Android, CopperheadOS treats full network access as a user-facing permission with a
toggle. For compatibility, it's enabled by default for apps targeting the modern Android platform
unlike other runtime permissions.

There are many known cases of apps exporting interfaces to other apps for making limited network
requests, so this toggle will become more useful when further isolation options are available. For
example, web browsers almost all expose an interface allowing other apps to open URLs and choose
not to require either the INTERNET permission or explicit user consent before making the request.

### Sensors permission

Unlike stock Android, CopperheadOS treats sensor access as a user-facing permission with a toggle.
For compatibility, it's enabled by default for apps targeting the modern Android platform unlike
other runtime permissions. Stock Android only treats heartbeat sensor access as dangerous.

Unlike the Network toggle, apps aren't marked with an existing low-level permission when they make
use of sensor access. CopperheadOS adds a new permission and treats all apps as requesting it so
there's a sensor permission toggle for every app.

Sensor access has been shown to provide the capability to perform [location
tracking](http://www.ccs.neu.edu/home/noubir/publications-local/NVBN2016.pdf), [input
logging](https://link.springer.com/article/10.1007/s10207-017-0369-x) and crude audio recording
that's [still capable of recognizing
speech](https://www.usenix.org/system/files/conference/usenixsecurity14/sec14-paper-michalevsky.pdf)
despite the 200 Hz sampling limit enforced by the OS.

### Background access

CopperheadOS adds special permissions for controlling access to data in the background in Settings
-> Security -> Apps with background access.

This feature will become more strict in the future. It's an early implementation aimed at fleshing
out the full set of functionality and working towards disabling access by default.

The powerful 'draw over other apps' special permission can be used to remain in the foreground. It
needs to be explicitly granted by users in a more manual way than the usual dangerous permission
request dialog and a persistent notification is displayed while it's being used. It also can't be
used to draw over the system ui to hide the persistent notification. It's outside of the scope of
this feature as it's quite dangerous already and should only be granted to an app in exceptional
circumstances just like accessibility services and device administrators.

#### Clipboard

CopperheadOS disables access to the clipboard by apps in the background by default. This can be
enabled per-app via Settings -> Security & location -> Apps with background access -> Access
clipboard in the background. For example, you may need to whitelist a clipboard manager.

#### Audio recording

Initiating audio recording in the background can be disabled per-app via Settings -> Security &
location -> Apps with background access -> Record audio in the background. It isn't yet disabled
by default but will be in the future.

For now, apps can still initiate audio recording in the foreground and continue in the background
even with background audio recording disabled. The feature will eventually be improved to kill
apps with an open recording stream when they're sent to the background.

#### Sensors

Sensors access in the background can be disabled per-app via Settings -> Security & location ->
Apps with background access -> Access sensors in the background. It isn't yet disabled by default
but will be in the future.

#### Location

Location access in the background can be disabled per-app via Settings -> Security & location ->
Apps with background access -> Access location in the background. It isn't yet disabled by default
but will be in the future.

### Serial number

CopperheadOS requires that apps have the Phone permission group (read phone state) to access the
serial number just like the IMEI on stock Android. Android is moving towards this but doesn't yet
enforce it even for apps targeting Android Oreo.

### Display over other apps

Unlike stock Android, CopperheadOS doesn't permit any automatic grants of the special "Display
over other apps" permission. It needs to be very explicitly granted by users.

### Storage

Apps have their own private storage directories and can share files with other apps using content
providers. Apps can act as storage providers to provide structured requests to retrieve and store
data including for the shared storage directory. Direct scoped access can also be requested for
the shared storage directory (since 7.0). Unfortunately, many apps require the storage permissions
for direct, full access to shared storage so it's unwise to store sensitive data there.

In the future, CopperheadOS will offer the ability to isolate shared storage rather than toggling
access. Isolated shared storage will provide an app with a dedicated shared storage directory
accessible only to themselves and the built-in file manager. Ideally, apps would already use the
available tools to provide this kind of functionality on their own.

#### File management

The built-in file manager for shared storage is recommended. It's available as the Files app on
the home screen and is also accessible via Settings -> Storage -> Files. Shared storage can be
shown by enabling by "Show internal storage" in the app menu. It has proper integration with
storage providers (apps providing storage to other apps, with explicit user control) and external
storage. It will also be the only app able to access isolated shared storage directories of other
apps once that feature is implemented.

### Location

The location icon is displayed in the status bar when an app is monitoring the device's location.

Settings -> Security & Location -> Location provides a global toggle to disable location detection
along with a list of apps that have made recent location requests.

## User profiles

User profiles offer isolated workspaces within the operating system.

Fully opening the notification tray will show an avatar for the main user profile. Touching it
opens the menu for creating and switching between profiles. Further configuration is available via
'More settings' which is a shortcut to Settings -> Users & accounts -> Users.

The special Guest profile is meant to be used as a scratch space with a quick shortcut in the
notification tray for deleting everything in the profile. It also offers the option to clear it
when switching to it.

Apps cannot communicate or share data across user profiles. On Pixel phones, each user profile has
separate disk encryption keys for all user data rather than there being a shared boot password. A
profile also cannot record audio while another profile is active. By default, only the Owner
account has access to phone calls and SMS. It can be enabled for other accounts via the user
settings menu on the owner account.

In addition to the baseline functionality, CopperheadOS exposes option to disallow audio recording
and to disallow installing apps in the user configuration menu for the owner account.

## Force always on Tor / VPN

In Settings -> Network & Internet -> VPN, the gear icon next to installed VPN apps like Orbot
(Tor) can be used to set one as an "Always-on VPN" to make it start automatically. However, it's
also necessary to toggle on "Block connections without VPN" to disallow connections being made if
the VPN dies. This will be enabled by default for an "Always-on VPN" on CopperheadOS in the near
future.

## Native debugging

Support for native debugging features can be disabled by toggling off Settings -> Security &
Location -> Enable native code debugging. It's left enabled by default like stock in order to
generate useful logs and crash dump information from crashes tied to bugs in native code.
Disabling it will make the information generated by the 'Take bug report' feature nearly useless
for crashes caused by native code but it will still produce useful information for uncaught Java
exceptions. There are currently no known app compatibility issues caused by disabling this, but it
isn't compatible with some nasty tricks used by obfuscated apps to make themselves harder to
inspect.

## USB peripherals

CopperheadOS defaults to ignoring connected USB peripherals if the screen is locked. This can be
controlled in Settings -> Security & Location -> USB accessories. The options are:

1. Disallow new USB peripherals
2. Allow new USB peripherals when unlocked (default)
3. Allow new USB peripherals (like stock Android)

This option has no impact on the device acting as a USB peripheral itself when connected to a
computer. However, Android already defaults to charge only mode and requires explicit opt-in to
the device exposing itself as an MTP, PTP or MIDI device.

## Camera on lockscreen

CopperheadOS adds a toggle at Settings -> Security & Location -> Camera on lockscreen for reducing
lockscreen attack surface by disallowing camera usage. It disables both the camera launch icon in
the lower right corner of the lockscreen and camera launch gestures while locked.

## Quick Settings restrictions

CopperheadOS restricts usage of sensitive Quick Settings tiles while the screen is securely locked.

The following Quick Settings tiles have an unlocking requirement in stock Android:

* Cast
* Location

CopperheadOS extends this requirement to more tiles:

* Bluetooth
* NFC (not present as a tile in stock)
* Airplane mode
* Wi-Fi
* Auto-rotate
* Data Saver
* Hotspot
* Cellular data
* Battery

Some tiles still have no unlocking requirement:

* Flashlight
* Night Light
* Invert colors
* Do not disturb
* Work

The 'Flashlight' tile is quite useful from the lockscreen and adds minimal attack surface.

'Night Light' and 'Invert colors' are comparable to the brightness slider. They add minimal attack
surface and are easy to notice and toggle off. Similarly, 'Do not disturb' (DND) is comparable to
the volume rocker which works while locked and both are quite useful functionality to have there.
The volume rocker already allows setting the 'Alarms only' DND mode too.

Android for Work isn't currently in scope for CopperheadOS hardening and the Work tile isn't
available when it's not in use, so it has been left as is. Android for Work isn't aimed at
businesses deploying dedicated work devices but rather Bring Your Own Device (BYOD).

## Wi-Fi

### Scanning

MAC randomization is always enabled for Wi-Fi scanning. The Nexus 5X and Pixel phones have fairly
unique firmware support for scanning MAC randomization going [above and beyond the usual
implementation](https://android-developers.googleblog.com/2017/04/changes-to-device-identifiers-in.html).
On most other devices, there are identifiers exposed by Wi-Fi scanning beyond the MAC address such
as the packet sequence number and assorted identifying information in the probe requests.

Wi-Fi scanning is never performed when Wi-Fi is disabled without explicitly enabling it in Settings
-> Security & Location -> Location -> Scanning, unlike stock Android. The same thing applies to
Bluetooth.

Avoid using hidden APs (i.e. APs not broadcasting their SSID) since known hidden SSIDs end up
being broadcast to find them again. SSIDs are not broadcast for standard non-hidden APs.

### Privacy when associated with an Access Point (AP)

The DHCP client uses the anonymity profile rather than sending a hostname so it doesn't compromise
the privacy offered by MAC randomization.

Unlike stock Android, CopperheadOS also provides MAC randomization when associated with an AP.
This is currently exclusive to Pixel phones and can be toggled off in Settings -> Network &
Internet -> Wi-Fi -> Wi-Fi preferences -> Randomize MAC address which will restore the hardware
MAC address when Wi-Fi is toggled off and on again.

## LTE only mode

If you have a reliable LTE connection from your carrier, you can reduce attack surface by
disabling 2G / 3G connectivity in Settings -> Network & Internet -> Mobile network -> Preferred
network type. Traditional voice calls will only work in the LTE only mode if you have either an
LTE connection and VoLTE (Voice over LTE) support or a Wi-Fi connection and VoWi-Fi (Voice over
Wi-Fi) support. VoLTE / VoWi-Fi on Pixel phones is expected to work on all carriers where it's
supported on stock (T-Mobile, Rogers, Fido, etc.) other than Verizon. VoLTE / VoWi-Fi compatibility
is substantially worse on Nexus devices for now.

This feature is not intended to improve the confidentiality of traditional calls and texts, but
may somewhat raise the bar for some forms of interception. It's not a substitute for end-to-end
encrypted calls / texts or even transport layer encryption. LTE does provide basic network
authentication / encryption but it's for the network itself. The intention of the LTE only feature
is only hardening against remote exploitation by disabling an enormous amount of legacy code.

## Network connection information / statistics

CopperheadOS prevents third party apps from obtaining detailed network information without any
permissions as they can on stock Android. The Net Monitor app is built into CopperheadOS and has a
special exception from this rule. Network information can also be accessed via the Android Debug
Bridge shell. Third party apps can only access information that's made explicitly available via
documented interfaces with permission controls and there's essentially no access to it right now
as valid use cases for it by third party apps not covered by Net Monitor haven't been presented.
The only known examples of using the information in good faith haven't been correct, such as the
attempts to implement a user-facing firewall via a VPN service rather than properly integrated via
the existing OS firewall infrastructure so that it actually works properly.

## Default connections made by CopperheadOS

Net Monitor can be used to monitor connections by the OS in addition to those done by apps. The
DownloadManager service will do HTTP / HTTPS downloads on behalf of third party apps and those
shouldn't be confused with downloads triggered by the OS itself.

CopperheadOS makes connections to the outside world to test connectivity, detect captive portals
and download updates. No data varying per user / installation is sent in these connections. There
aren't analytics / telemetry in CopperheadOS.

The expected connections by CopperheadOS (including all base system apps) are the following:

* The CopperheadOS Updater app fetches update metadata from https://SERVER/DEVICE-CHANNEL every
  hour when connected to a permitted network for updates. The Nexus 5X and Nexus 6P use
  legacy.copperhead.co (144.217.14.61) and Pixel phones use release.copperhead.co
  (144.217.14.110). These are currently hosted on OVH like the Copperhead site.
* F-Droid performs update checks for enabled repositories. By default, the standard F-Droid
  repository and Copperhead repository are enabled. The Copperhead repository is hosted at
  fdroid.copperhead.co (142.44.162.254) on OVH.
* As with other devices with a Qualcomm baseband (which provides GPS), [GPS
  almanac](https://en.wikipedia.org/wiki/GPS_signals#Almanac) updates from Qualcomm are downloaded
  from https://xtrapath1.izatcloud.net/xtra3grc.bin, https://xtrapath2.izatcloud.net/xtra3grc.bin or
  https://xtrapath3.izatcloud.net/xtra3grc.bin. CopperheadOS has modified all references to these
  servers to use HTTPS rather than a mix of HTTP and HTTPS.
* Connectivity checks designed to mimic a web browser user agent are performed by using HTTP and
  HTTPS to fetch standard URLs generating an HTTP 204 status code. This is used to detect when
  internet connectivity is lost on a network, which triggers fallback to other available networks
  if possible. These checks are designed to detect and handle captive portals which substitute
  the expected empty 204 response with their own web page. These need use a very common domain and
  URL in order to bypass whitelisting systems only permitting access to common domains / URLs so a
  domain like copperhead.co would be inadequate. CopperheadOS leaves these set to the standard four
  URLs to blend into the crowd of billions of other Android devices with and without Google Mobile
  Services performing the same empty GET requests:
  * HTTPS: https://www.google.com/generate\_204
  * HTTP: http://connectivitycheck.gstatic.com/generate\_204
  * HTTP fallback: http://www.google.com/gen\_204
  * HTTP other fallback: http://play.googleapis.com/generate\_204
* DNS connectivity and functionality tests
* DNS resolution for other connections

The connectivity checks are performed by both the OS itself and the hardened Chromium browser.

## DNS

Unlike stock Android, CopperheadOS uses Cloudflare's DNS servers for network connectivity tests
and the default fallback when the network doesn't provide DNS servers.  This decision is based on
Cloudflare providing [a stellar privacy
policy](https://developers.cloudflare.com/1.1.1.1/commitment-to-privacy/privacy-policy/privacy-policy/)
compared to [the decent one](https://developers.google.com/speed/public-dns/privacy) for Google
Public DNS. Cloudflare DNS supports DNS-over-HTTPS and DNS-over-TLS so it will be possible to use
it with Android's upcoming DNS-over-TLS support.

In practice, the default fallback is rarely used. DNS servers provided by the local network are
preferred for both mobile data and Wi-Fi, since it avoids trusting an additional party with the
information. The local network and ISP can still obtain all of the information when using an
alternate DNS server. Hiding the information from them requires using a VPN to move the same trust
to the VPN provider which is hopefully more worthy of that trust than the ISP. Even when using
DNS-over-HTTPS or DNS-over-TLS, tracking IP addresses leaks nearly as much information and even
HTTPS connections leak the hostname (unlike the path) so very little is truly being hidden by
those technologies.

Android 9.0 is adding support for overriding the default DNS servers with a DNS-over-TLS server
chosen by the user. However, as noted above this won't be able to hide much information from the
local network and ISP. It has some value but is limited, particularly before there's some form of
encrypted SNI extension for TLS 1.3 to avoid leaking exact hostnames for HTTPS rather than only
having imprecise reverse DNS lookups.

It's possible for apps implementing the VPN service feature to provide alternative DNS servers and
it can be done without intercepting and routing network traffic like a real VPN. Some apps use
this to implement features like domain-based content filtering.

## App spawning time

You may notice that cold start app spawning time takes a bit longer (i.e. in the ballpark of
100ms) than stock Android due to security centric [exec spawning
feature](/android/docs/technical_overview#exec-based-spawning-model). This is most noticeable on the Nexus 5X
due to it being the slowest supported device and is far less significant on current generation
hardware. It doesn't cause any performance cost after launching an app, and similarly doesn't
cause any extra latency if the app was already running / cached in the background.

## Updates on Pixel phones

The update system implements automatic background updates. It checks for updates once per hour
when there's network connectivity and then downloads and installs updates in the background. It
will pick up where it left off if downloads are interrupted, so you don't need to worry about
interrupting it. Similarly, interrupting the installation isn't a risk because updates are
installed to a secondary installation of CopperheadOS which only becomes the active installation
after the update is complete. Once the update is complete, you'll be informed with a notification
and simply need to reboot with the button in the notification or via a normal reboot. If the new
version fails to boot, the OS will roll back to the past version and the updater will attempt to
download and install the update again.

The updater will use incremental updates to download only changes rather than the whole OS unless
the current version is behind the current release by more than 3 versions. As long as you have
working network connectivity on a regular basis and reboot when asked, you'll almost always be on
one of the past couple versions of the OS which will minimize bandwidth usage since incrementals
will always be available. If you fall more than 3 versions behind, it will download a large full
update shipping the full OS so it can update from any version.

The updater works while the device is locked / idle, including before the first unlock since it's
explicitly designed to be able to run before decryption of user data.

### Settings

The settings are available in the Settings app in System -> About phone -> Update settings.

The "Release channel" setting can be changed from the default Stable channel to the Beta channel
if you want to help with testing. The Beta channel will usually simply follow the Stable channel,
but the Beta channel may be used to experiment with new features.

The "Permitted networks" setting controls which networks will be used to perform updates. It
defaults to using any network connection. It can be set to "Non-roaming" to disable it when the
cellular service is marked as roaming or "Unmetered" to disable it on cellular networks and also
Wi-Fi networks marked as metered.

The "Require battery above warning level" setting controls whether updates will only be performed
when the battery is above the level where the warning message is shown. The standard value is at
15% capacity.

Enabling the opt-in "Automatic reboot" setting allows the updater to reboot the device after an
update once it has been idle for a long time. When this setting is enabled, a device can take care
of any number of updates completely automatically even if it's left completely idle.

### Update security

The update server isn't a trusted party since updates are signed and verified along with downgrade
attacks being prevented. The update protocol doesn't send identifiable information to the update
server and works well over a VPN / Tor. Copperhead isn't able to comply with a government order to
build, sign and ship a malicious update to a specific user's device based on information like the
IMEI, serial number, etc. The update server only ends up knowing the IP address used to connect to
it and the version being upgraded from based on the requested incremental.

Android updates support serialno constraints to make them validate only on a certain device but
CopperheadOS rejects any update with a serialno constraint for both the Stable and Beta channels.

### Disabling updates

It's highly recommended to leave automatic updates enabled and to configure the permitted networks
if the bandwidth usage is a problem on your mobile data connection. However, it's possible to turn
off the update client by going to Settings -> Apps, enabling Show system via the menu, selecting
CopperheadOS Updater and disabling the app. If you do this, you'll need to remember to enable it
again to start receiving updates.

## Authentication / encryption

Using a strong passphrase is recommended. CopperheadOS extends the arbitrary default maximum
passphrase length from 16 characters to 64 characters.

Pattern unlock is strongly discouraged and may be turned into a hidden option in the future.

Fingerprint unlock acts as an extremely convenient secondary unlock mechanism. However, it opens
up weaknesses compared to knowledge-based authentication. It's not usable after a reboot for the
first unlock or after 48 hours and it doesn't reduce authentication or encryption security in
those cases. It has the important redeeming quality of making a strong passphrase as the main
unlock mechanism very convenient and the placement of the scanner can make it even more convenient
than swipe to unlock or even the no unlock mechanism option (i.e. power button only).

The long-term plan for CopperheadOS is to build upon fingerprint unlock by [adding support for
setting an optional knowledge-based 2nd
factor](https://github.com/copperhead/bugtracker/issues/451). Once that's implemented, the only
recommended authentication setups will be the following, from strongest to weakest:

1. Strong passphrase
2. Strong passphrase with fingerprint + PIN (or weaker passphrase) as a secondary unlock mechanism
3. Strong passphrase with fingerprint as a secondary unlock mechanism

Until it's implemented, using a PIN or a weak passphrase instead of the second option can make
sense. It's much weaker in the case where fingerprint unlock isn't available, i.e. after a reboot
or the 48 hour timeout.

CopperheadOS supports randomizing the PIN entry layout via a toggle in Settings -> Security &
Location -> Passwords -> Scramble PIN layout, which is disabled by default.  This will apply to
the planned 2nd factor fingerprint unlock mechanism in addition to using a PIN as the main unlock
method, which will be discouraged once there's a better option available.

### Fingerprint unlock attempts

CopperheadOS disables fingerprint unlock after 5 failed attempts, unlike stock Android which
allows 20 attempts with 30 second delays after each 5 failed attempts.

You can use this to disable the fingerprint scanner by intentionally making invalid unlocking
attempts. The device will vibrate on invalid unlocking attempts and will stop vibrating once
fingerprint unlock has been disabled.

## Keyboard personalized suggestions

The keyboard has the option of maintaining an internal database to improve suggestions based on
past input. It's entirely local and inaccessible to any other apps like all internal app data, but
CopperheadOS disables it by default to avoid gathering persistent statistics about user input that
may be valuable to an attacker that has compromised the device. It can be enabled again in
Settings -> Languages & input -> Virtual keyboard -> CopperheadOS Keyboard -> Text correction ->
Personalized suggestions, which is also accessible by holding the comma key on the keyboard and
pressing CopperheadOS Keyboard Settings.

## F-Droid repository

The CopperheadOS F-Droid repository is included in the default set. For reference:

* Repository URL: https://fdroid.copperhead.co/repo
* Repository fingerprint: F0D4EB1193AD82FEB224BD1174B6FBD89A39D8ED988C9FFF2ADD0DCD1C4E271B

It's only intended to be useful to CopperheadOS users. Nothing from there is guaranteed to work
elsewhere and issues on other operating systems should not be reported.

## App recommendations

### Built-in user-facing apps

CopperheadOS includes most of the standard Android Open Source Project (AOSP) apps.

The following apps are actively developed as part of AOSP and receive both bug fixes and new
features. Using these apps rather than third party ones is highly recommended:

* Calculator
* Clock
* Contacts
* Files
* Launcher
* Phone
* Settings

The following apps are no longer actively developed as part of AOSP and only receive important
security fixes. These apps will keep working indefinitely since Android is backwards compatible
but they won't receive new features or overhauls. These are candidates for replacement down the
road, but the replacements need to be a good fit for CopperheadOS:

* Camera
* Email
* Gallery
* Music
* Search (launcher icon is removed in CopperheadOS, but it's included for compatibility)

Some of the AOSP apps have been replaced in CopperheadOS:

* Calendar -> Etar (maintained derivative of the backend-agnostic AOSP Calendar app, can be used
  with any service exposing a calendar backend via an app)
* Browser -> Chromium (hardened browser developed by CopperheadOS based on Chromium, see [the
  section on browsing](#browsing))
* Messaging -> Silence (replaced to provide end-to-end SMS encryption)

CopperheadOS also includes some additional built-in apps. These apps are included to provide
functionality missing in AOSP compared to stock Android or in the case of Net Monitor because it
can't work if it's not integrated with the OS [due to CopperheadOS privacy
enhancements](#network-connection-information--statistics).

* F-Droid: app repositories and updates
* Offline Calendar: local calendar storage backend as an alternative to cloud-based backends
* Net Monitor: monitoring network connections
* PDF Viewer: hardened PDF viewer developed alongside CopperheadOS

### Browsing

CopperheadOS ships hardened builds of Chromium and the robust rendering sandbox makes it the most
secure option available. Since the Android WebView is provided by Chromium, most apps render web
content using the CopperheadOS Chromium builds. For example, DuckDuckGo uses the WebView in their
search app and the CopperheadOS PDF Viewer uses it to render PDF documents. CopperheadOS enables
the rendering sandbox for the WebView since Android Nougat and Google will be enabling it with
Android O, but the standalone browser sandbox is currently more restrictive. The WebView is also
at the mercy of the security of the app using it, since apps can add their own interfaces and can
grant access to the same files and other content they can access outside of the WebView.

Brave is based on Chromium with additional features like built-in ad-blocking, HTTPS Everywhere
and fingerprinting protection. However, it's not currently available in the main F-Droid
repository and would be missing the extra hardening. In the future, we'll likely provide hardened
builds of Brave for CopperheadOS via our F-Droid repository.

Avoid Gecko-based browsers like Firefox. They're significantly less secure and are among the few
apps not able to benefit from the full set of CopperheadOS hardening features due to shipping
their own linker and custom JIT compiler within the app process. The WebView is inherently
Chromium-based so using Gecko also means exposing the attack surface of two browser engines rather
than one. Firefox Focus currently uses the system WebView rather than Gecko but Mozilla [plans to
change that](https://github.com/mozilla-mobile/focus-android/issues/13).

### Red shift

On Pixels, use Settings -> Display -> Night Light instead of a third party app requiring a grant
of the special "Display over other apps" permission.

### Messaging

Recommended messaging app preference list:

1. Conversations + OMEMO
2. Noise or Signal to communicate with Signal users and for encrypted calls
3. Silence encrypted SMS to communicate with Android users without data connections
4. Other apps with end-to-end encryption if you can't convince contacts to install
one of the above (Wire, WhatsApp, etc.)
5. Apps with transport encryption without end-to-end encryption
6. Unencrypted SMS or apps without transport encryption

#### Conversations

The recommended messaging client is Conversations. It's an XMPP client interoperable with other
XMPP clients and servers. It supports end-to-end encryption via robust cryptography (OMEMO) based
on the Signal protocol along with OTR and PGP for backwards compatibility with lesser clients.
It's one of very few apps with efficient push messaging without needing Google Cloud Messaging
(GCM). It also supports end-to-end encrypted group chat.

Conversations has an official XMPP server with all of the necessary extensions for full
functionality. It costs 8 EUR / year after the 6 month free trial.  Using the official server to
support the project is recommended, but there are [other
options](https://gultsch.de/compliance_ranked.html) without a subscription fee.  We don't
currently have a recommendation about which ones to prefer, beyond sticking to those with support
for every XEP other than XEP-0357 (which is for GCM, rather than the standard push mechanism).

#### Signal

Noise, a rebranded build of Signal available outside the Play Store is available in the Copperhead
F-Droid repository. It has full support for all of the Signal features including voice and video
calls but it isn't optimized for low impact on battery life like Conversations. It used to be a
fork removing the hard dependency on Google Play Services but since Signal 3.30.0 that is not a
hard dependency anymore. Noise doesn't change a single line of code in Signal, only the branding.

The official website releases of Signal can be installed [from their
site](https://signal.org/android/apk/) but this requires temporarily enabling installing apps from
your browser (it still requires consent each time). Signal will check for updates to itself and
notify about them but it can't support the option of automatic upgrades. The upgrades need to be
installed by hand and require enabling app installs from Signal (it still requires consent each
time).

You can choose between installing Noise via F-Droid and receiving updates via F-Droid or using the
Signal website release and manually installing the updates via the notifications it creates.
Ideally, the official releases of Signal would be available via an official Signal F-Droid
repository so that we wouldn't need Noise anymore, at least after Signal ships a more complete
backup / import mechanism so a full migration is possible.

#### Silence

CopperheadOS replaces the AOSP Messaging app with Silence to provide support for encrypted SMS. It
isn't really recommended to prefer it over data-based encrypted messaging apps, but rather to make
use of it for communicating with contacts without data connections, or for all messaging if you
don't have a data connection yourself. It makes sense to leave it as the default SMS app even if
you're using an app like Noise able to act as the default SMS client.

#### WhatsApp

WhatsApp works on CopperheadOS, but it isn't currently available in a convenient way. The best way
to use it is probably installing the Amazon Appstore as an apk and then installing it from there,
so that you have updates for it along with the Appstore which will update itself.

We might consider trying to convince Facebook to either host an F-Droid repository or permit
redistribution of it.

### YouTube

Use the NewPipe app from F-Droid if you want more functionality than the mobile site.

### Maps

#### Simple, fast and highly usable

The all around best option for basic map and navigation functionality is
[maps.me](https://maps.me/) but it has some privacy and security issues.

The com.github.axet.maps fork is available in the standard F-Droid repository with the ads,
binaries, etc. stripped out.

The official maps.me apk is [available on their site](https://maps.me/apk/) and the Play Store
version works perfectly well on CopperheadOS too.

The main issue with it is that there's no way to have it use internal app storage instead of using
shared storage for internal app data like settings and map downloads. The data is exposed to other
apps with storage access and could be tampered with or simply watched for location information. It
might be something that the fork would be willing to address. Feel free to encourage both projects
to take a more secure approach or at least offer it as an option.

#### Functionality rich

OsmAnd (OpenStreetMap Automated Navigation Directions) can be installed from F-Droid and provides
map viewing and mobile navigation. It has the killer feature of optional support for downloading
the OpenStreetMap database for chosen regions. In addition to the obvious advantage of not having
a dependency on an internet connection, offline mapping offers more privacy. It's recommended to
use the offline mode if you have enough storage space to spare. **Note that it's important to
configure OsmAnd to use the internal storage directory:** go into the menu, then Settings, General
settings, select the "Data storage folder" option, select the edit button and set it to the
"Internal application memory" option.

Note that OsmAnd sends the semi-persistent ANDROID\_ID to their server on connections. ANDROID\_ID
will become less identifying in future CopperheadOS releases by default and further user control
will be offered, but it reflects poorly on OsmAnd.

#### Google maps

If you really need Google Maps, you can use their web application. It's not as nice as the mobile
app but the core functionality other than turn-by-turn navigation is there.

### Advanced camera features

Install the Open Camera app from F-Droid and enable "Use Camera2 API" in the settings menu. This
enables support for features like manual ISO configuration and HDR (**not** HDR+) mode.

On the Pixel 2 and Pixel 2 XL, HDR+ via the Pixel Visual Core is available to apps on CopperheadOS
with support for it. Unfortunately, the AOSP Camera app doesn't currently support it.

### Uber

Use [https://m.uber.com/](https://m.uber.com/) instead of the Android app. Open it with Chromium
and select 'Add to Home screen' from the menu. Unlike most sites, the Uber web app is set up to
act as a standalone app when added to the home screen rather than the launcher only acting as a
bookmark and opening a Chromium tab. The map isn't enabled by default since the web app is aimed
at the niche of users with low bandwidth but it can be enabled.

### Chromecast

VLC supports casting to Chromecast devices, so support within the OS provided by Play Services
isn't required.

### Installing apps from the Play Store

The Yalp Store app on F-Droid can be used to install apps from the Play Store. Before using it to
install an app, press the menu button, press Settings and enable the toggle for 'Download to
internal storage' to make sure Yalp doesn't download apps to shared storage where other apps with
storage access could potentially tamper with them before installation. Yalp will still demand the
Storage permission even though it isn't required though.

It's not clear how Google feels about the Yalp Store app so we don't recommend using your own
account credentials unless it's a throwaway account. Stick to the Yalp Store credentials. If you
need apps that you've purchased, consider using F-Droid's support for swapping apps between
devices to get them from a trusted device with the Play Store. Note that many paid apps use DRM
which won't work without Play Services.

Many apps from the Play Store use Google Play Services, but lots of those will still work fine on
CopperheadOS. Some functionality like push notifications may not be available if the apps don't
implement fallbacks for systems without Play Services included in the OS.

Be careful when installing apps from the Play Store and granting permissions to them since overall
they're a lot less trustworthy than the apps in the F-Droid repositories enabled by default on
CopperheadOS. There are apps in F-Droid with serious privacy and security issues but there aren't
any known cases of malicious apps and they tend to be a lot more respectful of privacy with most
of the major deviations from that marked in the F-Droid entries for those apps.

## Verified boot

CopperheadOS only supports devices with verified boot support. The hardware verifies authenticity
and integrity of firmware and the core operating system on every boot. The core operating system
verifies the entire rest of the operating system, providing full verified boot for the OS. Every
installation is guaranteed to be bit-for-bit identical at a storage level, otherwise it wouldn't
work. Updating the OS updates it to a pristine installation of a new release.

A fresh installation or an incremental update shipping only changes to blocks provide exactly the
same result. If any block fails to verify, the OS has error correction data to attempt to fix the
issue before verifying the fixed block again. If the corruption cannot be fixed, an error will be
displayed and it will reboot.

An attacker that has successfully gained root access via an exploit chain *cannot* simply modify
any of the OS partitions. They would need to have an exploit for the verified boot process to
exploit it as it reads their altered data which is an extremely high bar. This greatly raises the
bar for privileged persistence. An attacker is forced to persist as an unprivileged third party
app, requiring exploitation of the OS via an unpatched vulnerability to gain back control. Factory
resets wipe userdata, guaranteeing that an attacker loses persistence unless they have a verified
boot bypass. OS updates have a high chance of breaking any attempt to exploit the OS again too.

On the Pixel 2 and later, rollback protection is provided for the OS. This prevents an attacker
with root access from downgrading the OS to a previous version of the OS as a way to roll it back
to a version where they have working exploits for verified boot, etc. The Pixel 2 and later also
provide substantially better enforcement of the CopperheadOS public key.

Verified boot is primarily about preventing attacker persistence, but it also greatly raises the
cost of tampering with a device based on physical access. This is especially true for the Pixel 2.

For more technical details, see the [documentation on verified boot](/verified_boot).

### Verified boot attestation (Pixel 2 and later)

Copperhead's Auditor app uses hardware security features on supported devices to validate the
integrity of the operating system from another Android device.  It will verify that the device is
running either the stock operating system or CopperheadOS with the bootloader locked and that no
tampering with the operating system has occurred. It will also detect downgrades to a previous
version. Supported CopperheadOS devices:

* Google Pixel 2
* Google Pixel 2 XL

It cannot be bypassed by modifying or tampering with the operating system (OS) because it receives
signed device information from the device's Trusted Execution Environment (TEE) including the
verified boot state, operating system variant and operating system version. The initial
verification has some security provided by the Google root certificate.  The verification is much
more meaningful after the initial pairing as the app primarily relies on Trust On First Use via
pinning.

Usage instructions:

The device being verified (Auditee) must be one of the supported devices. The device performing
verification (Auditor) just needs to be any Android 7.0+ compatible device with a camera.

1. press Auditor on the device that will be verifying the Auditee
2. press Auditee on the device that's going to be verified
3. point the camera of the Auditee at the QR code on the Auditor to read the challenge
4. tap the QR code on the Auditor to advance ahead (if you do this too early, you can press back)
5. point the camera of the Auditor at the QR code on the Auditee to read the attestation
6. view verification of the attestation results

An Auditor can verify any number of different Auditee devices. It shows a fingerprint and the
first / last verification time in successful paired attestation results. An Auditee can be
verified by any number of Auditors but there will be a different fingerprint for each unique
pairing rather than the same fingerprint shown on each Auditor for the same Auditee.

<!-- The Auditor app is available in the default enabled Copperhead F-Droid repository. It has similar
licensing as the operating system, i.e. free non-commercial usage for devices not purchased from
Copperhead. Customers that have bought CopperheadOS devices are granted a free commercial license
for the app on those devices. For usage outside CopperheadOS, it's [available on the Play Store as
a paid app permitting commercial
usage](https://play.google.com/store/apps/details?id=co.copperhead.attestation) and [free on
GitHub for non-commercial usage](https://github.com/copperhead/Attestation/releases). -->

### Verified boot fingerprint

The bootloader will display a notice with the fingerprint of the verified boot key since it isn't
the built-in OEM key. Current devices use the TEE to prevent the key from being changed, but the
implementation has limitations and there is some value in manually confirming it. The Pixel 2 (XL)
have much more stringent enforcement of the key along with rollback protection.
