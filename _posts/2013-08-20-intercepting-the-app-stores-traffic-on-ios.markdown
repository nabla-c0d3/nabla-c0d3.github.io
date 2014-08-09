---
layout: post
title: "Intercepting the App Store's traffic on iOS"
date: 2013-08-20 11:07
comments: true
categories: ios
---


_TL;DR: By default, MobileSubstrate tweaks do not get injected into system
daemons on iOS which explains why my SSL Kill Switch tool wasn't able to
disable SSL certificate validation in the iTunes App Store._


### The problem

Last year I released the [iOS SSL Kill Switch][killswitch-gh], a tool designed
to help penetration testers decrypt and intercept an application's network
traffic, by disabling the system's default SSL certificate validation as well
as any kind of custom certificate validation (such as [certificate pinning
][killswitch-slides]).

While the tool worked well on most applications including SSL-pinning apps
such as Twitter or Square, [users reported][killswitch-issue] that it didn't
work the iTunes App Store, which would still refuse to connect to an
intercepting proxy impersonating the iTunes servers. Other similar tools such
as [Intrepidus Group's trustme ][intrepidus-blog] also seemed to have the same
limitation.


### A quick look at the App Store on iOS

The first step was to get the right setup:

* An intercepting proxy (Burp Pro) running on my laptop.
* An iPad with the SSL Kill Switch installed, and configured to use my laptop as the device's proxy.

After starting the App Store app, I noticed that I could already intercept and
decrypt specific SSL connections initiated by the App Store: all the HTTP requests
to query iTunes for available apps (as part of the App Store's tabs such as
``Featured'', ``Top Charts'', etc.) as well as app descriptions (``Details'',
``Reviews'').

However, more sensitive operations including user login or app installation
and purchase would fail by rejecting my intercepting proxy's invalid SSL
certificate. From looking at logs on the device, it turns out that two
distinct processes are behind the App Store's functionality:

    AppStore[339] <Warning>: JS: its.sf6.Bootstrap.init: Initialize
    itunesstored[162] <Error>: Aug 22 11:29:10  SecTrustEvaluate  [root AnchorTrusted]

* _AppStore_ is the actual App Store iOS application that you can launch
from the Springboard. It is responsible for displaying the App Store UI
to the user.
* _itunesstored_ is a daemon launched at boot time by [launchd][launchd-wiki],
the process responsible for booting the system and managing services/daemons.
_tunesstored_ seems to be responsible for the more sensitive operations
within the App Store (login, app purchase, etc.) and possibly some of the
DRM/Fairplay functionality.


### Why SSL Kill Switch didn't work

I initially thought the issue to be that [the strategy used by the SSL Kill
Switch][killswitch-slides] to disable certificate validation somehow wasn't
enough to bypass _itunesstored_'s certificate pinning.
However, it turns out that the SSL Kill Switch was just not being injected
into the _itunesstored_ process at all, for a couple reasons:

* The _itunesstored_ process is started as a daemon by [_launchd_][launchd-wiki]
early during the device's boot sequence, before MobileSubstrate and MobileLoader
get started. Therefore, none of the MobileSubstrate tweaks installed on the device,
including the SSL Kill Switch, get injected into this process.
* The SSL Kill Switch had a [MobileLoader filter][mobileloader-wiki] so that
the code disabling certificate validation would only be loaded into apps
linking the UIKit bundle (ie. applications with a user interface). This was
initially done to restrict the effect of the SSL Kill Switch to App Store apps
only. However, _itunesstored_ is a daemon that doesn't have a user interface,
hence the filter prevented MobileLoader from injecting the SSL Kill Switch
into the process.


### Man-in-the-Middle on _itunesstored_

After figuring this out, getting _itunesstored_ to stop validating SSL
certificates was very straightforward.
First of all, make sure you're using the latest version of the [SSL Kill
Switch][killswitch-gh] (at least v0.5). Then, all you need to do is kill the
_itunesstored_ process:

    iPad-Mini:~ root# ps -ef | grep itunesstored
    501   170     1   0   0:00.00 ??         0:01.95 /System/Library/PrivateFrameworks/iTunesStore.framework/Support/itunesstored
      0   432   404   0   0:00.00 ttys000    0:00.01 grep itunesstored

    iPad-Mini:~ root# kill -s KILL 170

When doing so, _launchd_ will automatically restart _itunesstored_. This time
however, MobileLoader will inject the SSL Kill Switch's code into the
process. You can validate this by looking at the device's logs, for example
using the xCode console. You should see something like this:

    itunesstored[1045] <Notice>: MS:Notice: Loading: /Library/MobileSubstrate/DynamicLibraries/SSLKillSwitch.dylib
    itunesstored[1045] <Warning>: SSL Kill Switch - Hook Enabled.

If you restart the App Store app, you should then be able to proxy all the
traffic and see app store transactions such as logins or app downloads.

![](/images/posts/burp_app_store.png)

If you try to install an app while proxying, your proxy might crash or freeze
when the App Store tries to download the app because IPA files can be fairly
large (200+ MB).


### Takeaway

A similar methodology could be used to proxy other system daemons including
for example _accountsd_, which is responsible for the Twitter and Facebook
integration that was added to iOS 5 and iOS 6.

While working on this, I also discovered a [better way][killswitch-v5] to
disable SSL certificate validation and certificate pinning in iOS apps.
Hence, SSL Kill Switch v0.5 is actually a complete rewrite. If you're interested
in knowing how it works, I wrote a [blog post][killswitch-v5] explaining what the
tool does.



[killswitch-issue]: https://github.com/iSECPartners/ios-ssl-kill-switch/issues/6
[killswitch-slides]: http://media.blackhat.com/bh-us-12/Turbo/Diquet/BH_US_12_Diqut_Osborne_Mobile_Certificate_Pinning_Slides.pdf
[killswitch-gh]: https://github.com/iSECPartners/ios-ssl-kill-switch
[intrepidus-blog]: http://intrepidusgroup.com/insight/2013/01/scorched-earth-how-to-really-disable-certificate-verification-on-ios/
[mobileloader-wiki]: http://iphonedevwiki.net/index.php/MobileSubstrate#MobileLoader
[launchd-wiki]: http://en.wikipedia.org/wiki/Launchd
[killswitch-v5]: /blog/2013/08/20/ios-ssl-kill-switch-v0-dot-5-released/

