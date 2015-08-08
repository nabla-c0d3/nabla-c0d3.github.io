---
layout: post
title: "TrustKit: Effortless SSL Pinning for iOS and OS X"
date: 2015-08-08 12:02:06 -0700
post_author: Alban Diquet
categories: ios ssl
---

Earlier this week, Angela Chow (from the Yahoo security team), Eric Castro and I spoke at the Black Hat US conference in a session titled ["TrustKit: Code Injection on iOS 8 for the Greater Good"][bh2015-conf] (slide deck is available [here][bh2015-pdf]).

We presented [TrustKit][gh-page], a new open-source library that makes it very easy to deploy SSL pinning in iOS or OS X Apps. The approach we used to solve SSL pinning is novel in several ways, as it is based on techniques such as function hooking and code injection, which are generally used for reverse-engineering and customizing Apps on a jailbroken device.

The result is what we like to call "Drag & Drop" SSL pinning: TrusKit can be deployed in any App in minutes, without having to modify the App's source code. 

Additionally, to ensure TrustKit would work in real-world Apps with millions of users and strict requirements in terms of performance, battery life and overall usability, we collaborated with the Yahoo mobile and security teams. They helped us design a library that would solve the challenges most companies are facing with SSL pinning, such as detecting how many users are seeing unexpected certificate chains in the wild, or ensuring that 100% of the App's connections are protected. 

TrustKit is already live on the App Store in some of the Yahoo Apps, and we open-sourced the project on [GitHub][gh-page] at the end of our presentation. Have a look and let us know what you think! The more developers use it, the better it will get.



[gh-page]: https://github.com/datatheorem/TrustKit
[bh2015-pdf]: https://datatheorem.github.io/TrustKit/files/TrustKit-BH2015.pdf
[bh2015-conf]: https://www.blackhat.com/us-15/briefings.html#trustkit-code-injection-on-ios-8-for-the-greater-good
