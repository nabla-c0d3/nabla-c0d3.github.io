---
layout: post
title: "TrustKit Android Released"
date: 2017-01-25 13:02:06 -0700
post_author: Alban Diquet
categories: android ssl
---

A couple days ago, a cowoker and I released [TrustKit for Android][trustkit-android-gh]: an open source library that makes it easy to deploy SSL public key pinning and reporting in any Android App. 

This follows the release of TrustKit iOS two years ago, which has now been deployed in many iOS Apps, in order to make their network connections more secure. We hope that TrustKit Android will help developers get more visibliity in their App's network behavior through TrustKit's reporting mechanism, and more security via SSL pinning.

### Overview

TrustKit Android works by extending the [Android N Network Security Configuration](https://developer.android.com/training/articles/security-config.html) in two ways:

* It provides support for the `<pin-set>` (for SSL pinning) and `<debug-overrides>` functionality of the Network Security Configuration to earlier versions of Android, down to API level 17. This allows Apps that support versions of Android earlier than N to implement SSL pinning in a way that is future-proof.
* It adds the ability to send reports when pinning validation failed for a specific connection. Reports have a format that is similar to the report-uri feature of [HTTP Public Key Pinning](https://developer.mozilla.org/en-US/docs/Web/HTTP/Public_Key_Pinning) and [TrustKit iOS](https://github.com/datatheorem/trustkit).

For better compatibility, TrustKit will also run on API levels 15 and 16 but its functionality will be disabled.

### Getting Started

Check out the the [project's page][trustkit-android-gh] on GitHub or jump straight into [the documentation](https://datatheorem.github.io/TrustKit-Android/documentation/).

[trustkit-android-gh]: https://github.com/datatheorem/TrustKit-Android
