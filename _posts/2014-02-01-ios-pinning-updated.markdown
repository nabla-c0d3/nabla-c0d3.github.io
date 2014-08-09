---
layout: post
title:  "iOS certificate pinning code updated for iOS 7"
date:   2014-02-01 07:51:00
post_author: Alban Diquet
categories: ios ssl
---


I've updated the iOS certificate pinning code which is part of iSEC Partners'
[SSL Conservatory][ios-github] project on Github. This new version brings the
following changes:

* The Xcode project was re-created as a static library (instead of an iOS App)
to facilitate integration. Sample code demonstrating how to use the library has
been moved the project's unit tests.
* A new convenience delegate class for _NSURLSession_, the HTTP connection
framework introduced in iOS 7, was added to the project. Similarly to the
existing convenience class for _NSURLConnection_ this class makes it easy to add
certificate pinning to connections relying on _NSURLSession_.

### Project page

Code and instructions are available on the project's [Github page][ios-github].

[ios-github]: https://github.com/iSECPartners/ssl-conservatory/tree/master/ios
