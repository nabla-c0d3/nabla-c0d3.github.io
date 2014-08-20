---
layout: post
title: "Multipeer Connectivity Follow Up"
date: 2014-08-20 12:02:06 -0700
post_author: Alban Diquet
categories: ios
---

Earlier this month, I [spoke at the Black Hat US conference][bh-link] about [Multipeer Connectivity][mc-doc] on iOS and OS X. The slides are available [here][slides-pdf] and this post is follow up article with things I did not have time to talk about during the talk.


### Multipeer Connectivity TL;DR

* The protocol used for data exchange (the "session phase") is a simple UDP-based protocol, wrapped in DTLS 1.0 in most cases.
* The authentication (on or off) and encryption settings (MCEncryptionNone, MCEncryptionOptional, MCEncryptionRequired) set by the App developer control wether DTLS is used and the list of cipher suites enabled during the DTLS handshake.
* Enabling authentication causes the protocol to perform a mutually-authenticated DTLS handshake (ie. each side/peer exchanges their certificate).
* Disabling authentication with encryption enabled results in the usage of Anonymous TLS cipher suites (ie. no certificates get exchanged), which are vulnerable to man-in-the-middle attacks.
* Disabling both authentication and encryption switches DTLS off; the plaintext UDP protocol is directly used.
* Overall, most of the authentication and encryption settings work as advertised by the Apple documentation except for MCEncryptionOptional when used with Authentication, which is vulnerable to downgrade attacks.

This diagram extracted from the slide deck summarizes my analysis of Multipeer Connectivity's security settings:

<center> <img src="/images/posts/multipeer-table.png"></img> </center>

The details on why I "blacklisted" some of the security settings are available in the [slide deck][slides-pdf].


### The Missing Slides
Multipeer Connectivity is a fairly complex Framework and there are quite a few interesting topics that I just did not have time to talk about during my Black Hat talk.

#### Implementing Authentication

When enabling authentication, the App developer has to deploy certificates and private keys to all instances of the App running on the users' devices. Every device/App then has to validate these certificates when pairing with other devices. Properly deploying a PKI is a very difficult task; here are some pointers on how to achieve this in the context of Multipeer Connectivity.


##### Authentication using Trust-On-First-Use

One option is to generate the certificate and private key on the device the first time the App is launched (using [SecKeyGenerate()][seckey-doc]). Then, the first time two devices pair with each other, they should store/pin each other's certificate and validate it on subsequent Multipeer Connectivity sessions. The advantage of this solution is that it does not require an Internet connection.

##### Authentication using an identity server

Another option is to instead rely on a backend identity server to fetch user/device certificates. This is a simpler approach but requires an Internet connection to fetch identities, which may be a challenge as Multipeer Connectivity is designed to work without any Internet connectivity.

Apple has already deployed a similar device/user PKI tied to the user's iCloud account; this is leveraged for example when using Airdrop with the "Contacts only" feature. It would have been pretty cool if Apple let developers use this existing PKI to authenticate devices based on the user's contacts, when implementing Multipeer Connectivity within an App.

##### Certificate validation

Remember that certificate validation is very tricky to get right. Be extra careful when implementing your [session:didReceiveCertificate:fromPeer:certificateHandler:][mcdeleg-doc] certificate validation method.


#### Implementing Customized Pairing
During the presentation, I only showed the default pairing process which displays a prompt to the user receiving the connection:

<center> <img src="/images/posts/multipeer-prompt.png"></img> </center>


However, as a developer, it is possible to fully customize the pairing process. For example, an App could be implemented to not even display a prompt to the user during pairing, basically allowing auto-pairing. This is what the [FireChat App][firechat] does in order to allow nearby users to chat anonymously without having to go through a confirmation prompt.
For a quite a few use cases (such as file sharing), auto-pairing could be a major security risk; as a developer, be very careful if you decide to customize the pairing process.


#### Real-world Apps Using Multipeer Connectivity

One of the questions I got asked was what kind of Apps currently available were using Multipeer Connectivity.

##### Non-Apple Apps

I found quite a few Apps on the store including:

* [Audibly][audibly]: Stream songs to other devices to use them as speakers.
* [iTranslate Voice][itranslate]: “AirTranslate”: translate voice in real-time and stream the translation to a nearby device.
* [FireChat][firechat]: Anonymous “off-the-grid“ chat.

Of course there are tons of other possible use cases: collaborative editing, file sharing, multiplayer gaming, etc.


##### Apple Apps

I haven't actually found any official Apple App that uses Multipeer Connectivity. However, I've only looked at a couple things so far:

* Airdrop: Airdrop uses very similar technologies (Bluetooth, Bonjour, etc.) but it's not Multipeer Connectivity.
* [HomeKit on iOS 8][homekit]: HomeKit is a new Framework on iOS 8 which "enable users to discover devices in their home and configure them"; it does not use Multipeer Connectivity either.

There are other Apple Apps/features that could be leveraging Multipeer Connectivity, including [Handoff on iOS 8][handoff], which I'll eventually look at when time allows.


[bh-link]: https://www.blackhat.com/us-14/briefings.html#it-just-networks-the-truth-about-ios-7s-multipeer-connectivity-framework
[slides-pdf]: /documents/BH_MultipeerConnectivity.pdf
[mc-doc]: https://developer.apple.com/library/ios/documentation/MultipeerConnectivity/Reference/MultipeerConnectivityFramework/Introduction/Introduction.html
[seckey-doc]: https://developer.apple.com/library/ios/documentation/security/Reference/certifkeytrustservices/Reference/reference.html
[mcdeleg-doc]: https://developer.apple.com/library/iOs/documentation/MultipeerConnectivity/Reference/MCSessionDelegateRef/Reference/Reference.html
[audibly]: https://itunes.apple.com/us/app/audibly-make-your-music-heard./id882170258?mt=8
[itranslate]: http://itranslatevoice.com/index.html
[firechat]: https://itunes.apple.com/us/app/firechat/id719829352?mt=8
[handoff]: https://www.apple.com/ios/ios8/continuity/
[homekit]: https://developer.apple.com/homekit/
