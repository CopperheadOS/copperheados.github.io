---
layout:      default
title:       Technical Overview
description: Technical overview of the currently implemented features in CopperheadOS
---

# Technical overview of CopperheadOS
{: .no_toc}

This is a technical overview of the currently implemented features in CopperheadOS.
The scope will be expanded as more features are implemented.

CopperheadOS is based on the latest stable release of the Android Open Source Project and has the
baseline privacy and security features from there, which are already far ahead of any traditional
desktop / mobile Linux distribution. Unlike other variants of Android including aftermarket OSes
and the forks vendors make for their devices, CopperheadOS doesn't disable or weaken the baseline
security features like verified boot and the SELinux policy. This document doesn't attempt to
cover the Android baseline beyond a brief introduction at the start.

This overview covers the latest generation devices (Pixel 2 , Pixel 2 XL). Past generation devices
have no remote attestation, weaker verified boot without rollback protection, a different
implementation of encryption, a different update system and only a subset of the kernel hardening
is present.

The [usage guide](/usage_guide) covers the user experience of
the small subset of user-facing features in more depth.

* TOC
{:toc}

## Prelude: Android baseline

The Android Open Source Project provides a robust base to build upon. The baseline security model
and features are not documented here, only the CopperheadOS improvements. To summarize some of the
standard security features inherited from Android:

* Full Disk Encryption at the filesystem layer is always enabled, covering all data (AES-256-XTS)
  and metadata (AES-256-CBC+CTS). Encryption keys are randomly generated, and then encrypted
  with a key encryption key derived via scrypt from the passphrase the verified boot key and the
  hardware-bound Trusted Execution Environment key which also implements rate limiting below the OS
  layer.
* Separate disk encryption keys for each user profile based on the credentials. User profiles are
  very useful even on single-user devices as isolated workspaces. A profile can also be dedicated
  to usage in situations where there would be a risk of the password (recording entry) or encryption
  key (dumping data from the hardware after unlock) important profile being compromised if it was
  entered.
* Full verified boot, covering all firmware and OS partitions. The unverified userdata partition
  is encrypted and wiped by a factory reset. Rollback protection is implemented via the Replay
  Protected Memory Block along with direct enforcement of the CopperheadOS public keys. Very basic
  remote attestation of verified boot results including the public key fingerprint is supported
  via keys provisioned by the device vendor accessible to the Trusted Execution Environment for
  signing. See the [documentation on verified boot](/verified_boot) for more details.
* Baseline app isolation via unique uid/gid pairs for each app.
* App permission model including the ability to revoke permissions and supply fake data. Most
  permissions are based on dynamic checks for IPC requests, while a small subset make use of
  secondary groups.
* Full system SELinux policy with fine-grained domains. There is no unconfined domain, and MLS
  provides very strong multi-user isolation.
* Fine-grained ioctl command filtering is done via SELinux. This is better than using seccomp-bpf
  to filter ioctl parameters since it's per-device-type and lower overhead.  This is being
  expanded to cover all usage of ioctls including device-specific whitelists for graphics
  drivers, etc.
* Kernel attack surface reduction via seccomp-bpf. There are strong sandboxes for the media
  codec endocing / decoding and extractor services, along with the Chromium browser and WebView
  used to render web content in other apps. A much more basic seccomp policy is applied to the
  entire app layer, disallowing system calls not explicitly exposed as a public API via Bionic
  libc.
* Widespread adoption of memory-safe languages, including within the base OS.
* Higher-entropy ASLR than the upstream Linux kernel defaults paired with library load order
  randomization in the linker.
* SSP, full RELRO and PIE and \_FORTIFY\_SOURCE=2 are enabled by default. \_FORTIFY\_SOURCE=2
  checks for read overflows rather than only write overflows like glibc. The dynamic linker only
  supports position independent executables. Text relocations are not supported on 64-bit or modern
  app API levels or the base OS for 32-bit and neither are RWX sections in libraries.
* etc.

## Exec-based spawning model

CopperheadOS spawns applications from the Zygote service in the traditional Unix way with ```fork```
and ```exec``` rather than Android's standard model using only ```fork```. This results in the address
space layout and stack/setjmp canaries being randomized for each spawned application rather than
having the same layout and canary values reused for all applications until reboot. In addition to
hardening applications from exploitation, this also hardens the base system as the large, near
root equivalent system\_server service is spawned from the Zygote.

Supporting exec-based spawning requires reimplementing the same infrastructure, debugging hooks,
etc. that are provided by the standard spawning model.

Support for shared RELRO sections has been removed, since it's unusable with exec-based spawning
and results in a reduction of complexity and SELinux permissions. Similarly, the AtlasAsset
service has been removed since it wouldn't be possible for it to work as currently designed. There
are various other disabled optimizations along with some emulation of the standard spawning model
by doing a tiny subset of the preloading that's usually done in the Zygote when apps are spawned.

## Chromium / WebView

CopperheadOS ships a custom build of Android's WebView along with a standalone build of Chromium
from the same source tree. Unlike Google's Chrome app, the CopperheadOS Chromium app is 64-bit.

Chromium supports per-site-instance sandboxing via isolatedProcess and seccomp-bpf. The WebView
has early support for sandboxing via a single isolatedProcess renderer for each app using it. It
isn't yet enabled in stock Android, but CopperheadOS force enables it. The sandboxed processes
aren't tied to the architecture of the app, so on CopperheadOS they are set to always run as
64-bit unlike vanilla Chromium where they will always be 32-bit. Once enabling multiple renderer
processes is stable, CopperheadOS will enable more of them since the memory cost is acceptable for
the niche. It currently results in stability issues due to apparent race conditions.

CopperheadOS also builds Chromium with -fstack-protector-strong rather than -fstack-protector and
removes mremap from the system call whitelist since CopperheadOS is currently not using it as an
optimization in the system allocator and it's not used elsewhere. It also *temporarily* makes use
of -fwrapv to prevent optimizations from introducing security issues due to reliance on undefined
signed overflow. However, this will be replaced by automatic integer overflow checking via usage
of -fsanitize=shift,signed-integer-overflow -fsanitize-trap=shift,signed-integer-overflow once
Chromium makes further progress towards eliminating bugs caught by UBSan.

Using Chromium or a WebView-based browser has the advantage of reducing attack surface compared to
another browser engine, since the Chromium-based WebView is already a baseline Android component.
The standalone Chromium also has a far superior sandbox to the one available for the WebView at
the moment, although it will continue to catch up.

Chromium's default settings have been adjusted to be more suited to CopperheadOS. Navigation error
correction, contextual search, network prediction, metrics and hyperlink auditing are disabled by
default. The welcome page with the opt-out for metrics is omitted since it's already disabled, as
are the 3 forms of promotion of Google's data reduction proxy. The form autofill feature is also
disabled by default until it has a saner implementation less vulnerable to phishing.

Note that while no features have been removed, Chromium for Android depends on Play Services for
Google account integration and some other features based on Google services so those features are
not available on CopperheadOS.

DuckDuckGo has been added as a search engine option and is set as the default. It supports the
search suggestion API used by Chromium. A missing feature is search-by-image, which is available
with Bing or Google set as the search engine.

CopperheadOS also disables the Location permission group for the browser by default since Chromium
knows how to request it when needed and many users don't use it. Separately from that, the default
browser search engine is also not granted internal browser geolocation permissions by default and
it needs to be explicitly enabled just as it needs to be for a regular site.

## Bionic libc improvements

### Hardened allocator

CopperheadOS replaces the system allocator with a port of OpenBSD's malloc implementation. Various
enhancements have been made to the standard OpenBSD allocator, some of which have been upstreamed.

The allocator doesn't use any inline metadata, so traditional allocator exploitation techniques
are not possible. Hash tables track page spans for each parallel allocation pool and point to
bitmaps tracking slot metadata within pages for small allocations. The out-of-line metadata
results in full detection of invalid free calls. It is guaranteed to abort for pointers that are
not active malloc allocations.

The configuration is made read-only at runtime and the rest of the global state is protected to
some extent via randomization and canaries. CopperheadOS has some small extensions to improve the
randomization and may do more in the future.

In order to mitigate vulnerabilities caused by offsets from unchecked malloc calls, CopperheadOS
sets the allocator to abort on out-of-memory by default. It can be turned off on a case by case
basis in processes verified to check for out-of-memory for all allocator calls. Android uses full
overcommit paired with a userspace out-of-memory killer, so the abort only occurs when a process
exhausts the virtual address space. In practice, it's not possible for an attacker to trigger
virtual memory exhaustion without also being able to trigger the out-of-memory killer, and this
mitigates many severe vulnerabilities. The address space is very large on all of the first-tier
CopperheadOS devices (47-bit) so it can only really be exhausted if there's a severe logic error
or an attacker has control of the allocated size.

#### Regions

OpenBSD malloc is a zone-based allocator, similar in design to Android's standard jemalloc
allocator. Unlike jemalloc, it uses page-aligned regions instead of 2MiB aligned regions so it
doesn't cause a loss of mmap randomization entropy. The standard allocator loses 9 bits of mmap
entropy on systems with 4096 byte pages.

A randomized page cache provides a layer on top of mmap and munmap. Spans of pages are cached in
an array with up to 256 slots using randomized searches. Regions are split at this layer but not
merged together. It's meant to provide a thin layer over mmap, partly to benefit from fine-grained
randomization by the kernel. Linux only randomizes the mmap base, unlike OpenBSD, so for now it's
not on par with how it works there.

A user-facing setting is exposed for enabling page cache memory protection to trigger aborts for
use-after-free bugs. For small allocations, a whole page needs to be cleared out for this to work,
but another technique is used to provide a comparable mitigation (see below).

#### Small allocations

Fine-grained randomization is performed for small allocations by choosing a random pool to satisfy
requests and then choosing a random free slot within a page provided by that pool. Freed small
allocations are quarantined before being put back into circulation via a randomized delayed
allocation pool. These features raise the difficulty of exploiting vulnerabilities by making the
internal heap layout and allocator behavior unpredictable.

CopperheadOS uses a ring buffer to extend the randomized quarantine with a deterministic layer
enforcing a minimum delay before allocations are put back into circulation. The memory dedicated
to the quarantine is evenly split between the two layers. CopperheadOS also adds a hash table
providing enhanced double-free detection by tracking all delayed allocations within the ring
buffer and randomized array. Previously, double-free could not be detected if the previously freed
allocation was still in the quarantine. CopperheadOS makes the quarantine size configurable, with
the size limit exposed as a user-facing setting.

Small allocations are filled with junk data upon being released. In production builds, zeroing is
used rather than filling with the usual 0xdf value to be more conservative. The 0xdf value isn't
able to guarantee that 32-bit pointers fault on 64-bit (0xdfdfdfdf) and tends to uncover bugs
which is good in debug builds but not great when it triggers crashes from uses of uninitialized
memory or other bugs in production. This prevents many information leaks caused by use without
initialization and can make the exploitation of use-after-free and double free vulnerabilities
more difficult. When allocations leave the quarantine, the junk data is validated to detect
write-after-free. By default, 32 bytes are checked and full validation can be enabled via a
user-facing setting. Junk validation was implemented in CopperheadOS and then successfully
upstreamed into OpenBSD.

Canaries can be placed at the end of small allocations to absorb small overflows and catch various
forms of heap corruption upon free. This was a successfully upstreamed CopperheadOS extension. In
CopperheadOS, this is enabled by default despite the memory usage cost since the supported devices
have lots of memory and there's a negligible performance impact. On 64-bit, the leading byte of
heap canaries is zeroed to mitigate non-NUL-terminated string overflows at the expense of losing 8
of the 64 bits of entropy.

#### Large allocations

In OpenBSD, only the leading 2048 bytes of large allocations are junk filled by default and junk
filling isn't used at all if page cache memory protection is enabled. CopperheadOS removes these
optimizations, so it either uses full junk filling (default) or none. It makes sense for the cost
to scale up with allocation size and junk filling is important for preventing information leaks
via uninitialized malloc usage even with page cache memory protection enabled.

A user-facing setting is exposed for placing guard pages at the end of large allocations to
prevent and detect overflows at the cost of higher memory usage and reduced performance. It may be
enabled by default on 64-bit in the future. Large allocations can be moved as close as possible to
the end of the allocated region in order to trigger faults for small overflows, and CopperheadOS
enables this by default when guard pages are enabled but not otherwise, unlike OpenBSD where it is
simply enabled by default.

### Extended ```_FORTIFY_SOURCE``` implementation

The ```_FORTIFY_SOURCE``` feature provides buffer overflow checking for standard C library functions
in cases where the compiler can determine the buffer size at compile-time.

Copperhead has added fortified implementations of the following functions in addition to the
coverage Bionic already had:

* fread - [upstreamed](https://android-review.googlesource.com/#/c/160350/)
* fwrite - [upstreamed](https://android-review.googlesource.com/#/c/160350/)
* getcwd - [upstreamed](https://android-review.googlesource.com/#/c/162664/)
* memchr - [upstreamed](https://android-review.googlesource.com/#/c/147372/)
* memrchr - [upstreamed](https://android-review.googlesource.com/#/c/147372/)
* memcmp
* memmem
* pread - [upstreamed](https://android-review.googlesource.com/#/c/147114/)
* pread64 - [upstreamed](https://android-review.googlesource.com/#/c/147114/)
* pwrite - [upstreamed](https://android-review.googlesource.com/#/c/162665/)
* pwrite64 - [upstreamed](https://android-review.googlesource.com/#/c/162665/)
* readlink - [upstreamed](https://android-review.googlesource.com/#/c/147291/)
* readlinkat - [upstreamed](https://android-review.googlesource.com/#/c/147291/)
* realpath - [upstreamed](https://android-review.googlesource.com/#/c/147365/)
* send - [upstreamed](https://android-review.googlesource.com/#/c/169960/)
* sendto - [upstreamed](https://android-review.googlesource.com/#/c/169960/)
* write - [upstreamed](https://android-review.googlesource.com/#/c/162665/)

Additionally, the dlmalloc API has been annotated with ```alloc_size``` attributes to provide buffer
overflow checking for the remaining code using the extended API.

Some [false positives in jemalloc](https://github.com/jemalloc/jemalloc/pull/250) were fixed in
order to land support for ```write``` fortification in AOSP.

### Dynamic object size queries

In CopperheadOS, system calls perform dynamic overflow checks in addition to the ```_FORTIFY_SOURCE```
checks based on compile-time buffer sizes from compiler analysis. The standard system call symbols
are really wrappers querying ```__dynamic_object_size``` and then calling through to the usual raw
system call wrappers. The makes the feature a drop-in enhancement even for precompiled third party
code, much like the hardened allocator.

The main component of the ```__dynamic_object_size``` feature is querying malloc for the object size.
OpenBSD malloc tracks allocation metadata entirely via out-of-line data structures so it can
accurately respond to these queries with either the size class of the allocation (minus the offset
into it) or a sentinel value. Recursion would trigger deadlocking, so CopperheadOS tracks whether
it is inside a malloc call via thread-local storage. The malloc queries are are disabled within
malloc calls. This allows the dynamic object size checks to be used even for libc, after early
initialization. Note that this currently returns a sentinel value for addresses beyond the first
page of an allocation, but improving this is on the roadmap.

Before querying malloc, there's special handling for addresses within the calling thread's stack,
the executable and the isolated library region. These paths can only give a rough upper bound or
abort the process if the address isn't part of any valid object. It reduces the performance cost of
the feature because querying malloc is relatively expensive.

It's restricted to system calls as it would be too expensive elsewhere. The read-only-after-init
global used to enable this feature after early init code may be extended into a configurable
feature. Calls like fread and fwrite sit in a middle ground between calls like memcpy and system
calls so there could be 3 levels to the performance vs security compromise: off, system calls,
system calls + fread/fwrite and everything. This would end up being part of a high level
performance vs. security slider exposed to users.

### Function pointer protection

Writable function pointers in libc have been eliminated, removing low-hanging fruit for hijacking
control flow with memory corruption vulnerabilities.

The ```at_quick_exit``` and ```pthread_atfork``` callback registration functions have been extended with
the same memory protection offered by the ```atexit``` implementation inherited from OpenBSD.

The vDSO function pointer table is made read-only after initialization, as is the pointer to the
function pointer table used to implement Android's malloc debugging features. This has been
[upstreamed](https://android-review.googlesource.com/#/c/159730/).

Mangling of the setjmp registers was [implemented
upstream](https://android-review.googlesource.com/#/c/170157/) based on input from Copperhead.

### Miscellaneous improvements

The deprecated ```Build.SERIAL``` field is always set to ```Build.UNKNOWN```, rather than the device's
serial number. Android Oreo introduced Build.getSerial() requiring the ```READ_PHONE_STATE```
permission, but still allows using ```Build.SERIAL``` unless ```targetSandboxVersion``` is set to 2.
Direct access to the underlying properties, etc. was already removed in Android Oreo.

Allocations larger than ```PTRDIFF_MAX``` are prevented, preventing a class of
overflows. This has been upstreamed for
[mmap](https://android-review.googlesource.com/#/c/170800/) and
[mremap](https://android-review.googlesource.com/#/c/181202/).

A dedicated memory region is created for mapping libraries, to isolate them from the rest of the
mmap heap. It is currently 128M on 32-bit and 1024M on 64-bit. A randomly sized protected region
of up to the same size is placed before it to provide a random base within the mmap heap. The
address space is reused via a simple address-ordered best-fit allocator to keep fragmentation at a
minimum for dynamically loaded/unloaded libraries (plugin systems, etc.).

Secondary stacks are randomized by inserting a random span of protected memory above the stack and
choosing a random base within it. This has been [submitted
upstream](https://android-review.googlesource.com/#/c/161453/).

Secondary stacks are guaranteed to have at least one guard page above the stack, catching sequential
overflows past the stack mapping. The ```pthread_internal_t``` structure is placed in a separate mapping
rather than being placed within the stack mapping directly above the stack. It contains thread-local
storage and the value compared against stack canaries is stored there on some architectures.

Signal stacks were given guard pages to catch stack overflows. This was
[upstreamed](https://android-review.googlesource.com/#/c/144365/).

On 64-bit, the leading byte of stack canaries is zeroed to mitigate non-NUL-terminated string
overflows at the expense of losing 8 of the 64 bits of entropy.

Assorted small improvements:

* have getauxval(...) set errno to ENOENT for missing values, per glibc 2.19 -
  [upstreamed](https://android-review.googlesource.com/#/c/142250/)
* fix the mremap signature - [upstreamed](https://android-review.googlesource.com/#/c/180130/)
* implementations of ```explicit_memset``` and ```secure_getenv``` for use elsewhere
* replaced the broken implementations of ```issetugid``` and ```explicit_bzero```
* larger default stack size on 64-bit (1MiB -> 8MiB)
* and various other assorted improvements

## Android Runtime

As part of moving towards reducing trust in /data/ (i.e. state not covered by verified boot),
```WITH_DEXPREOPT``` and ```WITH_DEXPREOPT_PIC``` are always enabled and ```WITH_DEXPREOPT_BOOT_IMG_ONLY``` is
always disabled to reduce usage of /data/dalvik-cache by the base system to a minimum. There is no
usage of executable data in /data/dalvik-cache for the base system since even boot.oat is position
independent. The boot.art file still has to be relocated via patchoat and is still stored in
/data/dalvik-cache but isn't executable. Avoiding that is also planned. The Android Runtime has
been taught not to look for executable code (oat and odex files) in /data/dalvik-cache and execute
and symlink read permissions for the dalvik cache label have been removed for system\_server and
domains only used by the base system, leaving it permitted by the policy only for untrusted\_app,
isolated\_app and the shell domain for adb shell.

CopperheadOS disables the ART JIT compiler and JIT profiling for security reasons. It also
disables the usage of any generated profiles for compilation by switching the defaults from
profile-based ahead-of-time verification/optimization to full verification/optimization. It
currently leaves in place the default of using interpret-only verification at
first-boot/boot/install and only doing optimized compilation in the background and for the new
online A/B update system. This may be changed due to the cost of the JIT being disabled.

## Compiler hardening

* A custom -fsanitize=local-init sanitizer is used to zero all uninitialized local variables to
  prevent information leaks. In some cases, it even prevents code execution vulnerabilities caused
  by attacker control over uninitialized memory. Existing optimization passes are quite good at
  removing it when it's unnecessary. It fits in well with the hardened malloc since that takes
  care of zeroing allocations on free when they aren't being directly unmapped.
* Lightweight bounds checking for statically sized arrays via -fsanitize=bounds
  -fsanitize-trap=bounds is enabled by default in userspace for C and C++. It
  is currently disabled for a few libraries where it catches out-of-bounds
  access bugs in regular use.
* For some sub-projects, lightweight object size bounds checking is performed
  (including extended checks for arrays) via -fsanitize=object-size
  -fsanitize-trap=object-size. This has a lot of overlap with the bounds
  sanitizer, but the redundant checks are optimized out when both are set to trap
  on error. It would be nice to enable this globally, but there's too much code
  relying on undefined out-of-bounds accesses.
* For some sub-projects, both unsigned and signed integer overflow checking via -fsanitize=integer
  -fsanitize-trap=integer (mostly backported from AOSP master).
* Stack overflow checking for supported architectures via -fstack-check (not on ARM yet due to a
  [severe bug](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=65958)) - submitted upstream [for the
  NDK](https://android-review.googlesource.com/#/c/163710/)
* For code where signed integer overflow checking is not enabled, overflow is made into
  well-defined behavior via -fwrapv to avoid security vulnerabilities like incorrectly written
  overflow checks being optimized out.
* Expanded stack overflow canaries to all relevant functions via -fstack-protector-strong for both
  the NDK and base system. It has also been upstreamed for both the
  [NDK](https://android-review.googlesource.com/#/c/143084/) and [base
  system](https://android-review.googlesource.com/#/c/185410/). CopperheadOS also enables
  -fstack-protector-strong for the Linux kernel. The lack of SSP for the kernel
  on ARM64 was reported upstream and -fstack-protector-strong has now been enabled.
* Added ```-Wsuggest-attribute=format``` warning to audit for missing format attributes - dozens of
  cases have been found and fixed, providing more coverage for warnings about exploitable format
  string bugs.

## Enhanced SELinux policies

* dalvikcache\_data\_file execute access limited to domains for third party apps (isolated\_app
  and untrusted\_app), preventing a trivial form of malware persistence without re-exploitation
* dalvikcache\_data\_file symlinks forbidden

See [the Android Runtime section](#android-runtime) for details on how this works.

* eliminated code injection holes
    * Removed gpu\_device execute access - upstreamed
    * Removed ashmem and tmpfs execute access (traditionally used by the Dalvik JIT and
      CopperheadOS disables the usage in Firefox's custom linker via an environment variable - no
      other compatibility issues have been found)
    * Removed execmod (text relocations) from domains other than untrusted\_app -
      [upstreamed](https://android-review.googlesource.com/#/c/269631/)
    * Removed execmem from all but the untrusted\_app, isolated\_app and
      mediadrmserver domains (note: mediadrmserver will likely be removed).
      This is possible due to disabling the ART JIT compiler and forcing usage
      of the multiprocess/sandboxed WebView which runs as isolated\_app.
    * Removed priv\_app app\_data\_file execute (upstreamed) / execute\_no\_trans access
    * The remaining hole in full (not just in-memory) w^x is app\_data\_file
      execution for untrusted\_app. The intention is to allow apps to opt-in to
      removing it and then upstream that, with the goal of changing the default
      as a breaking change in a future API level to force opting into the less
      secure policy along with discouraging the pattern.
* Split out untrusted\_apps signed with the release/media/shared keys (i.e. all of the base system
  untrusted\_apps) into untrusted\_base\_app, allowing the removal of execmod, execmem,
  app\_data\_file execute, dalvikcache\_data\_file execute and asec access
* Split out isolated\_apps signed with the release/media/shared keys (i.e. all of the base system
  isolated\_apps) into isolated\_base\_app, allowing the removal of dalvikcache\_data\_file execute
* removed mediaserver's write access to sysfs - upstreamed
* reported a hole in the isolation for system apps running as the system user upstream, which was
  prompted fixed: ptrace could be used to take control of other system apps running as the system
  user

Access to timing information via /proc/ has been limited to a few core services in order to close
sensitive information from being obtained via timing side channels. This has been
[upstreamed](https://android-review.googlesource.com/#/c/252368/). This is being gradually
extended to enforce fine-grained restrictions for the rest of the sensitive information exposed in
/proc: [zoneinfo](https://android-review.googlesource.com/#/c/255380/), vmstat.

Basic network interface and routing information (```/proc/net/dev```, ```/proc/net/if_inet6```,
```/proc/net/ipv6_route```, ```/proc/net/route```) is split out from the ```proc_net``` label into a new
```proc_net_devroute``` label with it permitted in the same domains and ```proc_net``` access is removed
for ```untrusted_app``` / ```untrusted_base_app```. Since monitoring network connections is a useful
feature, the org.secuso.privacyfriendlynetmonitor app is bundled and given a dedicated netmonitor
SELinux domain based on ```untrusted_base_app``` with ```proc_net``` access added back. Access is also
still permitted via the disabled by default ADB shell. Since monitoring by users is covered well
by the base system, there's little reason for any other apps to have access to this information.
A small subset of the information is available via NetworkStatsManager, but it requires the
```PACKAGE_USAGE_STATS``` permission to retrieve information for other apps which has to be manually
granted to third party apps by the user and can't simply be requested by an app directly.

A dedicated SELinux domain is used for the over-the-air updater app and access to
```update_engine_service``` and ```ota_package_file``` is removed from ```priv_app```, preventing other apps
from downgrading the OS version since ```update_engine``` only performs a signature check.

## Encryption and authentication

### Support for longer passwords

The maximum password length is raised from 16 characters to 64 characters.

### Reduced side channel leakage for file-based encryption

Padding for ext4 encryption file names is increased from the default of 4 bytes to 32 bytes. This
reduces the amount of information leaked about file name length. There's lots of room for further
improvements reducing metadata leakage but it involves changes the ext4 encryption format and
would need to happen in the Linux kernel.

### PIN entry scrambling

A setting is offered for randomizing the layout of PIN entry. It will become significantly more
useful once the 2 factor authentication feature above is implemented, since the combination of a
strong passphrase with a fingerprint + PIN as a secondary unlock method for convenience will be
the recommended setup for most people. It currently only impacts the lockscreen but may be
extended.  Note that the owner account setting also currently controls this for the lockscreen as
a whole, but it will eventually use the per-user setting instead.

### Fingerprint unlock

The number of retries before permanent lockout from fingerprint unlock is reduced from 20 to 5.
Since the timed lockout threshold is 5, only permanent lockout is used as it takes precedence.

## Privacy-focused defaults

The default settings have been altered to emphasize privacy over small conveniences.

Location tagging is disabled by default in the Camera app, and there is no longer a prompt about
choosing whether to enable it on the first launch. It can still be enabled using the Camera's
settings menu.

Personalized keyboard suggestions based on gathering input history are disabled by default.

Sensitive notifications are hidden on the lockscreen by default>

Passwords are hidden by default.

## Over-the-air updates

The new update system is a minimal, clean layer on top of the ```update_engine``` infrastructure from
ChromeOS and Brillo. It exclusively implements automatic background updates. User control is
offered via simple toggles to control whether it downloads on metered data connections or when
roaming is detected rather than updating ever being a manual procedure. Newer devices have dual OS
partitions rather than only firmware partitions and can update the alternate set of partitions in
the background at low priority without causing any disruption to users. By the time the user needs
to be informed about an update, it's already complete along with the bulk of the post-install work
like ahead-of-time compilation for apps not bundled preoptimized within the OS, which is important
for CopperheadOS since it disables JIT compilation / profiling in favor of simply performing full
AOT compilation with no JIT and minimal use of the interpreter for cold code like one-time
initialization to save space / memory.

The updater fetches a device-specific/channel-specific plain text file from a static file server
with a space-separated version and build date on each line with the most recent builds at the top.
Only the first line of the file is used by the update client within the OS. The client will try to
fetch an incremental for the current version to the new version. If an incremental isn't
available, it will fall back to a full update package able to update from any past version.
Currently, incrementals from the past 3 versions to the current are generated and then published.
The incremental or full update is then downloaded, the signature is verified and the build date is
then checked against the build date claimed by the server to prevent downgrade attacks by an
attacker with control of the server. The payload is then passed to ```update_engine```, which performs
another layer of signature verification on the payload. When ```update_engine``` reports completion, a
notification will be presented to the user with a reboot button, although the update will kick in
when the device is booted again whether or not it's via the notification.

The updater currently checks for updates every hour. If a failure occurs while trying to download
an update, it has the current download state saved and will resume the download when it next runs
if it's still an update to the most recent version. Rather than waiting a whole hour to retry if a
failure occurs, a new update check is also scheduled to occur after at least 30 seconds. It will
usually end up being longer due to the conservative nature of Android's JobScheduler, but not
nearly as long as waiting for the next regular interval.

The dual partition system makes updates significantly more robust since the updates are performed
on a set of unused partitions and have no impact until they are verified and marked as ready. The
next boot of the OS will then boot into the new version. A verification step is run fairly late in
the late boot process where every block from the OS partitions is read (```update_verifier```) in
order to leverage dm-verity to detect any corruption. If anything goes wrong during booting and
verification, it can automatically fall back to the previous version since it's still installed on
the previously used set of partitions.

## Networking

Interfaces are given a random MAC address whenever they are brought up. This can be disabled via a
toggle in the network settings. Wireless MAC addresses are also unconditionally randomized during
scanning (pre-associated). The hostname is randomized at boot by default, and it can also be
disabled in order to use the persistent hostname based on ```ANDROID_ID``` instead.

The kernel TCP/IP settings are adjusted to prioritize security. A minimal firewall is provided
with some options that are always sane including dropping traffic with conntrack's INVALID state
and reverse path filtering. Android has group-based control over networking so basic control over
networking is in the realm of permissions, but more advanced firewall features might be provided
down the road.

<!--
Support for IP sets is enabled in the kernel and the ipset utility is provided, but not yet
integrated in an automated way.
-->

## Permissions

Platform key signature permissions are restricted to system applications, to prevent old
applications signed with the platform key remaining as a threat.

Infrastructure is implemented for disabling permissions when apps are in the background. This is
used to deny access to the clipboard to apps running in the background, with a per-app toggle for
enabling it (such as for clipboard managers) in the Settings app. Similarly, apps cannot begin an
audio recording stream in the background and this will likely be extended to fully preventing
audio recording in the background even when it was started in the foreground. Toggles for
background location access and background sensors access are also implemented.

CopperheadOS enables Android's hidden
[```PERMISSIONS_REVIEW_REQUIRED```](https://android.googlesource.com/platform/frameworks/base/+/9c165d76010d9f79f5cd71978742a335b6b8d1b4)
feature, enforcing that the user must sign off on every dangerous permission after installing
apps. It has no impact on modern apps targeting API level 23 or later. For apps targeting API
level 22 or lower, it prevents apps from starting until the user proceeds through a prompt
allowing each dangerous permission to be toggled off, nearly identical to the user interface in
Settings -> Apps -> App name -> Permissions but with the app not able to run before the user has
made their desired changes and accepted it. Since this feature isn't widely used, some issues were
encountered with the implementation which have been fixed or worked around in CopperheadOS. These
fixes are being [submitted
upstream](https://android-review.googlesource.com/#/c/platform/packages/apps/PackageInstaller/+/452540/)
where applicable.

The ```INTERNET``` permission is marked as dangerous to make it user-facing and added to a new NETWORK
permission group with a user-facing toggle. Unlike other dangerous permissions, it's enabled by
default for apps targeting API level 23 or later for compatibility.

An ```OTHER_SENSORS``` dangerous permission is introduced and added as a requirement for access to
sensors without an existing permission (```BODY_SENSORS``` for heartbeat sensors). It's paired with an
```OTHER_SENSORS``` permission group used to expose a user-facing toggle. Like the redone ```INTERNET```
permission, it's enabled by default for apps with all API levels for compatibility. Since this
isn't an existing permission, there isn't a way to know which apps require it so it's considered
to be requested by every app.

The ```SET_TIME_ZONE``` permission is changed to a signature|privileged permission from a normal
permission to match ```SET_TIME```. This was implemented upstream by Google for Android Oreo.

## Attack surface reduction

NFC, NDEF Push and Bluetooth are disabled by default. An NFC quick tile is included to more easily
toggle it as needed.

A toggle is provided to disable the camera on the lockscreen.

Some quick tiles (bluetooth, nfc, airplane mode, WiFi, auto-rotate, data saver, hotspot, cellular
data, battery saver) have an authentication requirement added matching the standard authentication
requirement for the cast and location quick tiles.

An "LTE only" option is added to Settings -> Wireless & networks -> More -> Cellular networks ->
Preferred network type. It disables usage of the assorted 2G and 3G protocols.

A minimal port of grsecurity's ```PERF_HARDEN``` feature extends the ```kernel.perf_event_paranoid```
sysctl with an additional level (3). This is enabled by default on production builds, with a
system property (```security.perf_harden```) accessible to the ADB shell user for controlling it. The
userspace integration has been upstreamed:
[1](https://android-review.googlesource.com/#/c/233736/),
[2](https://android-review.googlesource.com/#/c/234352/),
[3](https://android-review.googlesource.com/#/c/234400/) and the kernel functionality was merged
alongside it.

CopperheadOS disables allows disabling unprivileged access to ptrace by enabling the stackable
Yama LSM and setting the ```kernel.yama.ptrace_scope``` sysctl to 2 by default. It's exposed in the
user interface as disabling native debugging since it will disable other debugging features with
added attack surface too.

A port of grsecurity's ```DENYUSB``` feature provides a ```kernel.deny_new_usb``` sysctl for disabling the
recognition of new USB devices. It reduces the attack surface exposing by USB drivers when active.
Integration in the lockscreen infrastructure provides automatic toggling based on whether the
screen is locked. A system property is exposed to users via Settings -> Security -> Device
security for choosing between it being enabled, controlled by lock state or disabled. The current
threat model only involves protecting the OS after the initial decryption process. It doesn't
provide any protection for the early boot environment or the recovery image as that would be a
different kind of feature.

Unlike Android, CopperheadOS uses separate kernel builds for production (user) and developer
(userdebug, eng) builds. This will make it possible to disable unnecessary debugging features in
production.

CopperheadOS disables the kernel's ```CONFIG_AIO``` feature. It isn't used or exposed by the base
system and is a dubious feature. It performs no better than thread pools and it can still block,
along with having coverage of only a tiny portion of blocking system calls even when considering
only commonly used system calls for IO. There are no known compatibility issues caused by having
this disabled. Since this is such a dubious niche feature, it's also very poorly tested and it
doesn't get much attention. Proposed improvements have been blocked based on the concern that
POSIX AIO is such a bad interface that trying to improve/extend it would be harmful. Following the
lead of CopperheadOS on this front has been proposed and accepted upstream for the recommended
Android kernel configuration used to derive device specific configurations.

### Recovery

Various debug options are removed from the recovery menu in production builds.

## Security-focused apps

### Chromium

See the [top level section](#chromium--webview).

### Silence

The AOSP SMS app (Messaging) is replaced with Silence to provide the option of end-to-end
encryption over SMS when a data connection is not available to either contact to enable better
options.

### PDF Viewer

CopperheadOS includes a custom PDF Viewer based on pdf.js in a WebView. It doesn't require any
permissions since it relies on content providers via ```ACTION_VIEW``` and ```ACTION_OPEN_DOCUMENT``` for
the application/pdf mime type rather than requesting storage access. It only has access to data
that the user explicitly provides to it.

Since JavaScript is memory safe, memory safety bugs in the PDF implementation itself are not a
concern and the performance is still tolerable. The app exposes a very small subset of the
Chromium attack surface and the WebView sandbox acts as an additional layer of isolation that's
significantly stronger than the usual app sandbox. It renders via a 2D canvas and loads fonts via
the FontFace API using the hardened Chromium font rendering stack (font sanitization, etc.). It
loads a new viewer for each document in order to improve isolation between documents compared to
reusing the same pdf.js environment.

The app uses Content Security Policy (CSP) to permit only static content from the app assets,
image ```blob:``` data and a placeholder HTTPS URL used to receive data from the app. The policy
prevents dynamic and inline JS/CSS and provides some attack surface reduction for the DOM. JS API
attack surface is limited indirectly due to the code being static. Content access, file access and
network access are also disabled for the WebView, so it can only access apk assets and resources.
It receives the PDF data from the app by making a placeholder HTTPS request which is substituted
with the PDF input stream by the app without needing to permit the WebView to have direct access
to content providers. Various unnecessary WebView features are disabled including caching,
cookies, saving form data and loading other URLs.

An alternative to this would be a custom isolatedProcess / seccomp-bpf sandbox to contain a pure
Java PDF library and some associated native code, but pdf.js was deemed to be a more secure and
viable option than any other existing open source libraries. The widely used libraries are written
in C and C++ (mupdf, poppler, pdfium). The pdf.js solution is also nearly ideal when the source of
the PDFs is Chromium or a Chromium-based browser due to presenting only a small subset of the same
attack surface.

The PDF Viewer app also uses targetSandboxVersion 2 which uses a substantially more restricted
SELinux domain barely permitting anything beyond communication using intents.

## Kernel hardening

The linux-hardened project was being developed
alongside CopperheadOS to provide a publicly available hardened Linux kernel. Most changes are
intended to land upstream with it remaining a minimal set of out-of-tree changes. The focus is on
protecting the kernel itself from exploitation. It also offers an improved implementation of
Address Space Layout Randomization and miscellaneous hardening features. Android devices use
long-term support kernel branches so these changes are being backported when applicable to the
current generation CopperheadOS kernels after being implemented in linux-hardened for the latest
mainline kernel stable branch.

CopperheadOS also benefits from backporting work done by Google from the mainline kernel such as
PAN emulation preventing the kernel from accessing userspace memory outside of userspace accessor
functions, HARDENED\_USERCOPY providing dynamic checks of copies to/from userspace and
```__ro_after_init``` which CopperheadOS currently applies from Google's Android O preview kernel
releases.

On current generation devices, kernels are compiled with Clang rather than GCC. This means
compiler hardening work is shared with the CopperheadOS userspace. The -fsanitize=local-init
feature mentioned in the [compiler hardening section](#compiler-hardening) is also used for the
kernel builds to zero all uninitialized variables. Google will likely be moving towards using a
subset of trapping UBSan, CFI and SafeStack in the kernel similar to userspace, so CopperheadOS is
focused on other areas.

Pages are zeroed upon being freed to reduce the lifetime of sensitive data. Pages are also
verified to be zero before being doled out again to catch write-after-free bugs.

The slab allocator (SLUB) is hardened via disabling slab merging, additional memory corruption
sanity checks for non-kmalloc slab caches, per-cache XOR encryption for the free lists and zeroing
data when freed. On allocation, write-after-free is detected by verifying that the memory is still
zeroed from the page and slab sanitization. A pointer-size random value is placed after memory
allocations in order to absorb small buffer overflows, rendering them harmless. The value is
checked on free in order to detect heap corruption. A different value is used for free and used
slots, providing basic double-free detection that's usually missing due to inline metadata. The
upstream ```HARDENED_USERCOPY``` feature is extended to check the cookies in ```__check_heap_object```
which provides basic use-after-free detection for the user copy functions in addition to more
frequent heap corruption checks. Ideally, a hardened allocator with out-of-line metadata would be
implemented along the same lines as the malloc implementation in userspace.

An equivalent to ```_FORTIFY_SOURCE``` in userspace is implemented, currently covering the string.h
str/mem family functions.

The ASLR implementation is improved via stronger stack randomization and randomizing the lower
bits of the argument block. This builds upon the higher entropy randomization already provided by
Android compared to Linux kernel defaults for the mmap and executable bases. The executable base
is moved from the middle of the address base to very near the start of it to make room for the
higher stack entropy and to reduce address space fragmentation in general.

A port of grsecurity's ```DEVICE_SIDECHANNEL``` feature prevents information leakage via timing data
leaked by character and block devices. Processes without the MKNOD capability are provided with
the creation time as the access and modify time, along with not receiving notifications for either
access or modify events.

Extra entropy is gathered from uninitialized memory in early boot feature) to improve the
effectiveness of probabilistic kernel mitigations relying on entropy available before the hardware
random number is initialized. This isn't an issue for userspace after init/ueventd since init
blocks until 512 bytes is read from the hardware random number generator and also restores saved
entropy from previous boots.

There's also an assortment of minor hardening changes being expanded over time.

For kernel attack surface reduction features, see [the relevant
section](#attack-surface-reduction).

## Miscellaneous features

SQLite's ```SECURE_DELETE``` feature is enabled, resulting in deleted content being overwritten with
zeros. This prevents sensitive data from lingering around in databases after it's deleted. SQLite
is widely used by Android's base system is the standard storage mechanism for applications, so
this results in lots of coverage. This has been
[upstreamed](https://android-review.googlesource.com/#/c/209123/). The default journal mode is
also set to ```TRUNCATE``` rather than ```PERSIST``` to stop data from lingering in the journal after
transactions. This change has also been
[upstreamed](https://android-review.googlesource.com/#/c/210080/).

The hidepid=2 option is enabled for procfs, hiding processes owned by other UIDs. Since non-system
apps each have a unique UID, this prevents apps from obtaining sensitive information about each
other via /proc. There are exceptions for a few of the core services via the gid mount option
(lmkd, servicemanager, keystore, debuggerd, logd, system\_server) but not for apps. A subset of
this was provided by SELinux, but it isn't fine-grained enough. This enhancement was [adopted
upstream](https://android-review.googlesource.com/#/c/181345/) based on the implementation in
CopperheadOS (it had been planned, but they were unaware of the gid mount option).

On the Pixel 2 and later, the getentropy implementation is switched to using a blocking call to
getrandom to wait until the kernel CSPRNG is properly seeded in early boot. After very early boot,
it never blocks again. On stock Android, a non-blocking call is used with a fallback to
/dev/urandom if the kernel CSPRNG is not yet properly seeded. On the Pixel 1, getrandom is used
with the blocking disabled because it can block for very long periods of time / indefinitely on
boot.

Some misuses of ```memset``` for sanitizing data were replaced with ```explicit_memset```, to stop the
compiler from optimizing them out.

Many global function pointers in Android's codebase have been made read-only. This is ongoing work
and will need to be complemented with Control Flow Integrity (CFI) as many are compiler-generated.
Some of this work has been upstreamed: [1](https://android-review.googlesource.com/#/c/172190/),
[2](https://android-review.googlesource.com/#/c/172140/).

# Past features

These features were available on CopperheadOS in the past and are still relevant but are currently
unavailable.

## Security level and advanced configuration

A slider is exposed in Settings -> Security -> Advanced for controlling the balance between
performance and security. By default, it starts at 50%. It provides high level control over
various performance vs. security tunables exposed there. All of the options can also be set
manually rather than using the slider.

## Encryption and authentication

### Support for a separate encryption password

***Note: this feature needs to be redesigned for Android 10***

In vanilla Android, the encryption password is tied to the lockscreen password. That's the default
in CopperheadOS, but there's full support for setting a separate encryption password. This allows
for a convenient pattern, pin or password to be used for unlocking the screen while using a very
strong encryption passphrase. If desired, the separate encryption password can be removed in favor
of coupling it to the lockscreen password again.

When a separate encryption password is set, the lockscreen will force a reboot after 5 failed
unlocking attempts to force the entry of the encryption passphrase. This makes it possible to use a
convenient unlocking method without brute force being feasible. It offers similar benefits as wiping
after a given number of failures or using a fingerprint scanner without the associated drawbacks.
