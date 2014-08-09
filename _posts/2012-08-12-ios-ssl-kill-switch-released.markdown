---
layout: post
title:  "iOS SSL Kill Switch released"
date:   2012-08-12 17:13:22
categories: ios ssl
---

_Note: this article was originally posted on [iSEC Partners' blog][original-post]._

### iOS Blackbox testing VS Certificate pinning

When performing a black box assessment of an iOS App, one of the main tasks of the tester is to intercept the application's network communications using a proxy. This gives the tester the ability to see what is happening behind the scenes and how the application and the server communicate with each other.

Successfully proxying the application's traffic can be challenging when the application uses SSL combined with certificate pinning in order to validate the server's identity. Without access to the application's source code to manually disable certificate validation, the tester is left with no simple options to intercept the application's traffic.


### iOS SSL Kill Switch

At [iSEC Partners][isec], I've been working on a tool to simplify the process of bypassing certificate pinning when performing black box testing of iOS Apps: [iOS SSL Kill Switch][killswitch-gh]. This tool hooks specific SSL functions at runtime that perform certificate validation. Using Cydia, it can easily be deployed on a jailbroken device, allowing the tester to disable certificate validation for any app running on that device in a matter of minutes.

The tool was successfully tested against the Twitter, Square and card.io iOS applications which all use certificate pinning to secure their network traffic.


### Project page

The iOS SSL Kill Switch was presented at [BlackHat Vegas 2012][killswitch-bh] along with a [similar tool][androidssl-gh] for Android. It is available on the GitHub [project page][killswitch-gh].


[isec]: https://www.isecpartners.com
[killswitch-gh]: https://github.com/iSECPartners/ios-ssl-kill-switch
[androidssl-gh]: https://github.com/iSECPartners/android-ssl-bypass
[killswitch-bh]: http://www.blackhat.com/usa/bh-us-12-briefings.html#Diquet
[killswitch-slides]: https://github.com/iSECPartners/ios-ssl-kill-switch/blob/master/BH2012_MobileCertificatePinning.pdf?raw=true
[original-post]: https://www.isecpartners.com/tools/mobile-security/ios-ssl-killswitch.aspx
