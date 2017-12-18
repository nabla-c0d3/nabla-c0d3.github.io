---
layout: post
title: "Scanning for the ROBOT Vulnerability at Scale"
date: 2017-12-17 13:02:06 -0700
post_author: Alban Diquet
categories: ssl
---

I just released [SSLyze 1.3.0][sslyze-gh], which adds support for scanning for [the ROBOT vulnerability](https://robotattack.org/) that was disclosed last week.

Using [SSLyze's Python API][sslyze-documentation], it is possible to easily and quickly scan a lot of servers for the vulnerability. From my own testing and depending on the network conditions, it takes about 5 seconds to scan 20 servers. SSLyze also has the ability to scan servers that use StartTLS-based protocols (such as SMTP, XMPP, etc.), which the test script released along with ROBOT does not support.

The following script (tested on Python 3.6) demonstrates how it can be done:

<script src="https://gist.github.com/nabla-c0d3/db3610017aff20941c29fe693744e919.js"></script>

Enjoy and happy scanning!


[sslyze-documentation]: https://nabla-c0d3.github.io/sslyze/documentation/
[sslyze-gh]: https://github.com/nabla-c0d3/sslyze/releases
