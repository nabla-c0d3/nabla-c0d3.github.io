---
layout: post
title: "SDWebImage Now Validates SSL Certificates"
date: 2015-03-21 12:02:06 -0700
post_author: Alban Diquet
categories: ios ssl
---

SDWebImage is a [popular iOS library][sdwebimage] for implementing asynchronous image downloads. Last year, I reported to the development team that SDWebImage had [SSL certificate validation disabled][gh-issue] when fetching data over HTTPS.

The issue was introduced in June 2014 in version 3.7.0: certificate validation was disabled as a side-effect of adding support for NTLM authentication to the library's `NSURLConnectionDelegate`. The specific commit is available [here][issue-introduced].

The team just [released version 3.7.2][releases], which finally addresses this issue and re-enables certificate validation; the actual fix can be seen [here][issue-fixed]. If you're using SDWebImage in your App, make sure to update the latest version (as always)!


[sdwebimage]: https://github.com/rs/SDWebImage
[gh-issue]: https://github.com/rs/SDWebImage/issues/918
[issue-introduced]: https://github.com/rs/SDWebImage/commit/50c4d1d2eb11e86f69a72021098e9738f008c009#diff-7519dfc55f22e25bd87d757458d74b82R412
[issue-fixed]: https://github.com/rs/SDWebImage/commit/52b2b70abff4a83d7d1e77499511ae21799c4dc8
[releases]: https://github.com/rs/SDWebImage/releases
