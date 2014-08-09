---
layout: post
title: "SSLyze v0.7 Released"
date: 2013-08-14 11:34
comments: true
categories: sslyze
---

A new version of [SSLyze][sslyze-gh] is now available. SSLyze is a Python tool that can analyze the SSL configuration of a server by connecting to it.


### Changelog

* Complete rewrite of the OpenSSL wrapper as a [C extension][nassl-gh]
   * SSLyze is now statically linked with the latest version of OpenSSL instead of using the system's (potentially outdated/broken) OpenSSL library
    * All of SSLyze's features are now available on all supported platforms (including SSL 2.0, TLS 1.1 and TLS 1.2)
    * Scans are slightly faster
    * Python 2.6 is no longer supported
* Support for StartTLS FTP, POP, IMAP, LDAP and "auto". See --starttls
* Support for OCSP Stapling. See --certinfo
* Other various improvements that results in SSLyze being more stable/robust


### Packages

SSLyze requires Python 2.7; the supported platforms are Windows 7 32/64 bits,
Linux 32/64 bits and OS X 64 bits.
SSLyze is statically linked with OpenSSL 1.0.1e. For this reason, the easiest
way to run SSLyze is to download one the following pre-compiled packages:


#### Linux
The following packages were tested on Debian 7 and Ubuntu 13.04.

* [Package for Python 32 bits][sslyze-dl-linux32]
* [Package for Python 64 bits][sslyze-dl-linux64]


#### OS X Mountain Lion

* [Package for Python 64 bits][sslyze-dl-osx64]


#### Windows 7

* [Package for Python 32 bits][sslyze-dl-win32]
* [Package for Python 64 bits][sslyze-dl-win64]


[sslyze-dl-linux32]: https://www.dropbox.com/s/u6h4u50daejuz5q/sslyze-0_7-linux32.zip?dl=1
[sslyze-dl-linux64]: https://www.dropbox.com/s/zf0e8oolkpkcuhu/sslyze-0_7-linux64.zip?dl=1
[sslyze-dl-osx64]: https://www.dropbox.com/s/v4vb2q7h5cb3tl3/sslyze-0_7-osx64.zip?dl=1
[sslyze-dl-win32]: https://www.dropbox.com/s/wmmgny3cz3tsido/sslyze-0_7-win32.zip?dl=1
[sslyze-dl-win64]: https://www.dropbox.com/s/xt526dgsyz1utid/sslyze-0_7-win64.zip?dl=1
[sslyze-gh]: https://github.com/iSECPartners/sslyze
[nassl-gh]: https://github.com/nabla-c0d3/nassl


