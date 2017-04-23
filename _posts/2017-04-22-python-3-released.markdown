---
layout: post
title: "SSLyze for Python 3 Released"
date: 2017-04-22 13:02:06 -0700
post_author: Alban Diquet
categories: ssl
---

I just released SSLyze v1.1.0, which finally adds support for Python 3! This means that both the command line tool and [the Python API][sslyze-documentation] can be called using Python 3.3+. Python 2.7 is still supported, for now.

Head to the [project's page][sslyze-gh] for more information.


### Full Changelog

* **Added support for Python 3.3+** on Linux and MacOS. Windows will be supported later.
* Added support for scanning for cipher suites on servers that require client authentication.
* Certificate transparency SCTs via OCSP Stapling will be now displayed when running a `CertificateInfoScanCommand`.
* Removed custom code for parsing X509 certificates, which was the source of numerous bugs and crashes when running a `CertificateInfoScanCommand`:
	* Certificates returned by the SSLyze Python API are now parsed using the [cryptography](https://github.com/pyca/cryptography) library, making further processing a lot easier and cleaner.
	* Certificates returned in the XML and JSON output when using `--certinfo` are no longer parsed. XML/JSON consumers should instead parse the PEM-formatted certificate available in the output using their language/framework's X509 libraries.
	* The `--print_full_certificate` option when using `--certinfo` is no longer available.
* Bug fixes for the Heartbleed check.	
* Added unit tests for SSL 2.0, SSL 3.0, Heartbleed and OpenSSL CCS injection checks.

The Python API can do a lot more than this (such as scanning StartTLS endpoints, connecting through a proxy, or enabling client authentication); head to the [project's page][sslyze-gh] or the [documentation][sslyze-documentation] for more information.


[sslyze-documentation]: https://nabla-c0d3.github.io/sslyze/documentation/
[sslyze-gh]: https://github.com/nabla-c0d3/sslyze
