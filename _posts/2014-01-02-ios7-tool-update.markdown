---
layout: post
title: "Tool updates for iOS 7"
date: 2014-01-02 13:44
post_author: Alban Diquet
categories: ios
---

With the availability of the evasi0n7 jailbreak and the subsequent release two
days ago of [Cydia Substrate][substrate] with support for iOS 7 and ARM64, a
full-blown iOS 7 penetration testing environment can now be setup. To this
extent, I've updated the two following iOS pentesting tools in order to add
support for iOS 7 and ARM64:

* [iOS SSL Kill Switch v0.5][killswitch-gh] which disables SSL certificate
verification/pinning within Apps.
* [Introspy-iOS v0.4][introspy-gh], an iOS Apps security profiler.

The pre-compiled packages for these tools now contain both an armv7 and an
arm64 slice, which means that they will work on 64 bits iOS Apps for devices
with an A7 chip (such as the iPhone 5s and the iPad Air).

Both tools were successfully tested on an iPhone 5s running iOS 7.0.4:

<center> <img src="/images/posts/introspy-ios7.png"></img> </center>


### Sandbox changes in iOS 7

While testing Introspy-iOS on iOS 7, I ran into issues with the _sandboxd_
daemon denying write access to specific files the tool was trying to create.
Interestingly enough, it seems like the Seatbelt profiles deployed on iOS 7
have been updated, compared to iOS 6. Specifically:

* AppStore Apps can no longer write to the root folder of their container
directory, for example
_/var/mobile/Applications/3152B928-D771-424C-AE39-F79EC4A79EC5/_
* System Apps can no longer write to _/var/mobile/_

Because of these changes, I had to modify the locations where Introspy-iOS
stores its files, to the following paths:

* _[App Container]/Library/_ for AppStore Apps.
* _/var/mobile/Library/Preferences/_ for System Apps.

It is unclear why the Seatbelt profiles were changed, although the ability to
write to these locations was not actually needed by Apps. More information
regarding the Seatbelt profiles used for various iOS Apps is available on the
[iphonedev wiki][seatbelt-wiki].

[substrate]: http://www.cydiasubstrate.com/
[isec-gh]: https://github.com/iSECPartners/
[killswitch-gh]: https://github.com/iSECPartners/ios-ssl-kill-switch/releases
[introspy-gh]: https://github.com/iSECPartners/Introspy-iOS/releases
[seatbelt-wiki]: http://iphonedevwiki.net/index.php/Seatbelt
