---
layout: post
title: "Security and Privacy Changes in iOS 9"
date: 2015-06-16 12:02:06 -0700
post_author: Alban Diquet
categories: ios
---

I just watched the WWDC 2015 sessions about [security](https://developer.apple.com/videos/wwdc/2015/?id=706) and [privacy](https://developer.apple.com/videos/wwdc/2015/?id=703) and put together some notes about the changes brought by iOS 9 that I thought were interesting.


### App Transport Security

This is a big one: by default on iOS 9, Apps will no longer be allowed to initiate plaintext HTTP connections, and will be required to use HTTPS with the strongest TLS configuration (TLS 1.2 and PFS cipher suites):

![](/images/posts/ios9-ats.png)

It is possible to lift these restrictions and still retrieve data over plaintext HTTP by adding [some configuration keys to the App's _Info.plist_](http://www.neglectedpotential.com/2015/06/working-with-apples-application-transport-security/).
Also, App Transport Security seems to only be available for connections initiated using `NSURLSession`. While `NSURLConnection` is being deprecated (forcing everyone to switch to `NSURLSession` for HTTP), I wonder if plaintext connections initiated through other network APIs (such as `NSStream`) will fail as well.

A great change overall, and this might even be the first step to mandatory HTTPS as part of the App Store policy.


### Detection of Installed Apps Blocked

Apple has closed three privacy gaps that allowed Apps to detect which other Apps were installed on the device.

* The first technique was to use the [`sysctl()`](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man3/sysctl.3.html) function to retrieve the process table (a remnant of OS X), which includes the list of running Apps.
In iOS 9, `sysctl()` was modified to no longer allow sandboxed Apps to retrieve information about other running processes.

* The second technique relied on the [`UIApplication canOpenUrl`](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIApplication_Class/#//apple_ref/occ/instm/UIApplication/canOpenURL:) method to try known URI schemes implemented by specific Apps, in order to detect if these Apps were installed on the device. This was made famous by Twitter, which used [a list of 2500 URI schemes](https://nikf.org/blog/twitter-ads-app-detection) to detect which Apps were installed on the device.
In iOS 9, Apps have to explicitly declare which schemes they would like to use in their _Info.plist_ file. For Apps targeting iOS 8 but running on an iOS 9 device, there is also a hard limit of 50 URI schemes that can be checked at most.

* There was a third technique which relied on the [icon cache being accessible to sandboxed Apps](https://twitter.com/aykay/status/586104297420632064). Although it wasn't even mentionned in the WWDC video, this privacy leak [has also been addressed](https://twitter.com/aykay/status/608612622653632514) in iOS 9.

Overall, closing these privacy gaps is a great move for users as these APIs were being abused by various Apps and analytics/ads SDKs.


### Universal Links

Universal Links are meant to replace URI schemes for communicating with other Apps.
In a nutshell, an App can specify a list of web domains they're associated with:

![](/images/posts/ios9-applink.png)

 Then, if the App is installed on the device, the App will have the ability to "receive" and open any HTTP or HTTPS link to the associated domains, when the link gets clicked on the device. If the App is not installed, the link will be opened in Safari instead.

This new mechanism is better than URI schemes for a few reasons:

* Strong ownership of links/content: enabling Universal Links requires uploading a specific file to the web domain to be associated with the App, which proves ownership. Conversely, a URI scheme could be claimed by any App that gets installed on the device.
* Because Universal Links rely on standard HTTP and HTTPS URLs, no need for specific, iOS-only URLs. Also, the links work regardless of whether the App is installed or not.
* Better privacy as Universal Links do not leak whether a specific App is installed on the device.

For more information, there's a full [WWDC session](https://developer.apple.com/videos/wwdc/2015/?id=509) dedicated to Universal Links.


### Mac Address Randomization Extended

Mac address randomization was introduced in iOS 8 in order to prevent the tracking of users through their device's WiFi MAC address. This feature was initially criticized for being enabled [only if location services were turned off](http://blog.airtightnetworks.com/ios8-mac-randomization-analyzed/) on the device.

To further prevent user tracking, Apple has extended MAC address randomization to additional services, and seems to now include location services scans:

![](/images/posts/ios9-macrandom.png)


### Misc. Keychain Improvements

Apple made some notable changes to the Keychain on iOS 9:

* The physical store where the Keychain cryptographic data is persisted has been moved to the Secure Enclave, the device's cryptographic co-processor (available since the iPhone 5S).
* The weakest Keychain accessibility class, [`kSecAttrAccessibleAlways`](https://developer.apple.com/library/ios/documentation/Security/Reference/keychainservices/#//apple_ref/doc/constant_group/Keychain_Item_Accessibility_Constants) will be deprecated in iOS 9. This protection class causes the data to not be encrypted at all when stored in the Keychain.
* When using TouchID to protect Keychain items, [`touchIDAuthenticationAllowableReuseDuration`](https://developer.apple.com/library/prerelease/ios/releasenotes/General/iOS90APIDiffs/modules/LocalAuthentication.html) can be used to avoid re-prompting the user for their fingerprint if they already had a match some time ago.


### Application Passwords

Keychain items can be encrypted using both the device's passcode and an "Application password". Both values are then needed to decrypt and retrieve the item. This allows Apps to control when the data is accessible, for example by storing the Application password on a server, or having it being displayed via a hardware token. Application passwords can be configured using the  [LACredentialType.ApplicationPassword](https://developer.apple.com/library/prerelease/ios/releasenotes/General/iOS90APIDiffs/modules/LocalAuthentication.html) setting.


### Secure Enclave Protected Private Keys

`SecGenerateKeyPair()`, which is used to generate RSA and ECDSA key pairs, can now be configured to directly store the generated private key in the device's Keychain (within the Secure Enclave). The key can then be subsequently used to sign data or verify signatures directly within the Secure Enclave by specifying additional parameters to `SecKeyRawSign()` and `SecKeyRawVerify()`. This means that the private key can be used without ever leaving the device's Secure Enclave. The  [`kSecAttrTokenIDSecureEnclave` attribute](https://developer.apple.com/library/prerelease/ios/releasenotes/General/iOS90APIDiffs/frameworks/Security.html) needs to be used when generating the key pair.


### Network Extension Points

Extensions were introduced in iOS 8, and are architected around the concept of ["extension points"](https://developer.apple.com/library/ios/documentation/General/Conceptual/ExtensibilityPG/): specific areas of the OS that can be customized using a designed API. Extension points from iOS 8 include for example "Custom Keyboard" and "Share", respectively for customizing the device's keyboard and adding new share buttons to receive content from other Apps. iOS 9 brings several new extension points geared toward network filtering and VPNs:

* The Packet Tunnel Provider extension point, to implement the client-side of a custom VPN tunneling protocol.
* The App Proxy Provider extension point, to implement the client-side of a custom transparent network proxy protocol.
* The Filter Data Provider and the Filter Control Provider extension points, to implement dynamic, on-device network content filtering.

These extension points can only be used with a special entitlement, thereby requiring Apple to approve the extension first.
