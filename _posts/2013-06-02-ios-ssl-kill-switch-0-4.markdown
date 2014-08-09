---
layout: post
title:  "iOS SSL Kill Switch v0.4 released"
date:   2013-06-02 15:12:29
categories: ios ssl
---

Version 0.4 of the [iOS SSL Killswitch][killswitch-blog] now available.

The iOS SSL Kill Switch is a tool to disable SSL certificate validation - including certificate pinning - within iOS Apps in order to facilitate blackbox testing.

In addition to patching _NSURLConnection_ at runtime, this new release implements another strategy to disable certificate validation: it modifies the _SecTrustEvaluate()_ function of the Security Framework in order to make it accept all certificate chains (similar to Intrepidus Group's [trustme][trustme] tool). Overall, with both patching strategies available (_NSURLConnection_ and _SecTrustEvaluate()_ ), the iOS SSL Kill Switch will successfully disable certificate validation on more iOS applications.

The debian package can be downloaded [here][killswitch-dl]. See also:

* [Black Hat 2012 Slides][killswitch-slides]
* [Github Project Page][killswitch-gh]


[killswitch-dl]: https://www.dropbox.com/s/95gm7h89owumywk/com.isecpartners.nabla.sslkillswitch_v0.4-iOS_6.1.deb?dl=1
[killswitch-gh]: https://github.com/iSECPartners/ios-ssl-kill-switch
[killswitch-blog]: /blog/2012/08/12/ios-ssl-kill-switch-released/
[killswitch-slides]: https://github.com/iSECPartners/ios-ssl-kill-switch/blob/master/BH2012_MobileCertificatePinning.pdf?raw=true
[trustme]: http://intrepidusgroup.com/insight/2013/01/scorched-earth-how-to-really-disable-certificate-verification-on-ios/
