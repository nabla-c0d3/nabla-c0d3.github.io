---
layout: post
title: "TrustKit 1.2.0 for iOS 9 Released"
date: 2015-10-20 13:02:06 -0700
post_author: Alban Diquet
categories: ios
---

I just released TrustKit 1.2.0, which adds support for iOS 9. As explained in a previous blog post, Apple implemented a [behind-the-scene change in iOS 9 that broke TrustKit][ios9-post] (and other tools).


### Changelog

* Complete re-write of the hooking strategy to automatically add SSL pinning to the App's connections. TrustKit now swizzles `NSURLSession` and `NSURLConnection` delegates to add pinning validation to the delegate's authentication handler methods; for developers who want to call into TrustKit manually, this behavior can be disabled using the `TSKSwizzleNetworkDelegates` setting. This change was made due to the previous hooking strategy (targeting _SecureTransport_) not working on iOS 9.
* The pinning policy format has slightly changed, in order to add new global settings: `TSKSwizzleNetworkDelegates`, `TSKIgnorePinningForUserDefinedTrustAnchors`, `TSKPinnedDomains`. If you have an existing pinning policy for TrustKit 1.1.3, all you need to do is put it under the `TSKPinnedDomains` key.
* Greatly simplified the `TSKPinningValidator` API to make it easy to write authentication handlers that enforce the App's SSL pinning policy. Sample code describing how to do it is available in the documentation.
* Updated Xcode project settings: stricter warnings, enabled bitcode, separate iOS and OS X build schemes.
* Pinning failure reports now also send the IDFV in order to simplify the troubleshooting of errors, by being able to detect a single, malfunctioning device.

More information is available on the project's [github page][trustkit-gh].


### Migrating from a previous version

If you were already using TrustKit in your App, here are a few things to take into account when updating to 1.2.0:

* The new hooking technique based on swizzling `NSURLSession` and `NSURLConnection` delegates will only protect connections initiated via these APIs, while the previous implementation would also work on other APIs such as `UIWebView` and `NSStream`. The updated [Getting Started][getting-started] guide has guidelines on how to leverage TrustKit for these network APIs.
* As explained in the changelog, the policy format has slightly changed. For your existing policy to work on 1.2.0, you just need to put it under the `TSKPinnedDomains` key.
* If you don't want TrustKit to auto-magically try to enforce SSL pinning on your App's connections, you can now disable this behavior by setting the `TSKSwizzleNetworkDelegates` to `NO`, and instead call into TrustKit manually within your App's authentication handlers, using the `TSKPinningValidator` class.


### TrustKit and PayPal

Unrelated to the 1.2.0 release but of particular interest is the fact that TrustKit was featured in a post about "Key Pinning in Mobile Applications" on [PayPal's engineering blog][trustkit-paypal]!


[getting-started]: https://datatheorem.github.io/TrustKit/getting-started/
[ios9-post]: https://nabla-c0d3.github.io/blog/2015/10/17/trustkit-ios-9-shared-cache/
[trustkit-paypal]: https://www.paypal-engineering.com/2015/10/14/key-pinning-in-mobile-applications/
[trustkit-gh]: https://datatheorem.github.io/TrustKit
