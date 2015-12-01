---
layout: post
title: "Burp and iOS 9 App Transport Security"
date: 2015-12-01 13:02:06 -0700
post_author: Alban Diquet
categories: ios
---

When trying to intercept an iOS 9 App's traffic using Burp (or any other proxy software), you might run into issues caused by App Transport Security, a new security feature [introduced in iOS 9][ats-blog].

By default, App Transport Security will cause most Apps on iOS 9 to only allow network connections to servers with a strong SSL configuration:

* TLS version 1.2
* Perfect forward secrecy cipher suites
* RSA 2048+ bits for the certificate chain

By default Burp does not support connections with these settings - TLS 1.2 is disabled by default - so most iOS 9 Apps will not be able to connect to Burp, when trying to proxy the App's network traffic.

To address that, enable TLS 1.2 and the right cipher suites in Burp's SSL settings:

![](/images/posts/burp-ats1.png)

Also, Burp's default CA certificate is only 1024 bits, so you will have to [generate a 2048 bits certificate and private key][create-ca] and import it into Burp (Proxy -> Options -> Import / Export CA certificate).

![](/images/posts/burp-ats2.png)

After updating Burp's configuration and CA key pair, you should then be able to proxy iOS 9 Apps without any problem.

[ats-blog]: https://nabla-c0d3.github.io/blog/2015/06/16/ios9-security-privacy/
[create-ca]: https://jamielinux.com/docs/openssl-certificate-authority/create-the-root-pair.html
