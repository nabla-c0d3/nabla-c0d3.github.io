---
layout: post
title: "Multipeer Connectivity on iOS 9"
date: 2015-09-19 13:02:06 -0700
post_author: Alban Diquet
categories: ios
---

Apple just released iOS 9, which fixes a [good number of security issues][ios-security], and includes a mitigation for a man-in-the-middle attack on Multipeer Connectivity [I presented last year at Black Hat US][blog-multipeer]:

![](/images/posts/multipeer-ios9.png)

The issue was that an attacker on the network could downgrade the encryption level of a Multipeer Connectivity session configured with `MCEncryptionOptional` to `MCEncryptionNone` (ie. plaintext communication), even if authentication was enabled.

![](/images/posts/multipeer-attack.png)

Apple did not fix the core issue with `MCEncryptionOptional`, as it would require significant changes to the implementation, in order to have each peer validate the security settings exchanged during the handshake _after_ authentication is performed. 

However they changed the default encryption level (used when the App does not explicitly specify one) from `MCEncryptionOptional` to `MCEncryptionRequired`, which is not vulnerable to the downgrade attack. Overall, this change should protect most Apps, but the ideal solution would have been to remove `MCEncryptionOptional` altogether.


[ios-security]: https://support.apple.com/en-us/HT205212
[blog-multipeer]: /blog/2014/08/20/multipeer-connectivity-follow-up/

