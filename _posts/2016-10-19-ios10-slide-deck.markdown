---
layout: post
title: "iOS 10 Security Changes Slide Deck"
date: 2016-09-19 13:02:06 -0700
post_author: Alban Diquet
categories: ios
---

I hosted an online Tech Talk on October 19th about the security & privacy changes introduced in iOS 10. I talked about both user-facing and developer-facing changes that I thought were interesting. The last section also provides some guidelines on how to be ready for the App Transport Security requirements that Apple [will start enforcing on January 2017][ats-post].

You can download the slide deck [here][slide-deck].

As I was preparing the presentation, I discovered that the Apple documentation about App Transport Security has changed significantly over the last couple months. For example, ATS exemptions for third-party domains (such as `NSThirdPartyExceptionAllowsInsecureHTTPLoads`) have been removed from the documentation, implying that developers can just use the regular exemptions even for third-party domains. 

Hence, the slide deck contains the most up-to-date information about iOS 10 and ATS, but I will update the previous blogs posts with these additional changes.


[slide-deck]: /documents/ios10_security_changes.pdf
[ats-post]: /blog/2016/08/14/ats-enforced-2017/


