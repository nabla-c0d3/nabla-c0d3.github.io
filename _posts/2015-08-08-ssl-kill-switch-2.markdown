---
layout: post
title: "Introducing SSL Kill Switch 2"
date: 2015-08-08 13:02:06 -0700
post_author: Alban Diquet
categories: ios ssl
---

I recently started working on [iOS SSL Kill Switch][old-gh-page] again, a tool which disables SSL validation and pinning in iOS Apps. I've implemented some significant changes including enabling support for OS X Apps, as well as adding the ability to disable _TrustKit_ (the SSL pinning library [I released at Black Hat][bh2015-conf]).

To reflect all these changes, I've renamed the tool to "SSL Kill Switch 2" and moved the repository to a [new GitHub page][new-gh-page]; the [old repository][old-gh-page] will no longer be maintained.


[old-gh-page]: https://github.com/iSECPartners/ios-ssl-kill-switch
[new-gh-page]: https://github.com/nabla-c0d3/ssl-kill-switch2
[bh2015-conf]: https://www.blackhat.com/us-15/briefings.html#trustkit-code-injection-on-ios-8-for-the-greater-good
