---
layout: post
title: "SSLyze 2.0.0 Released"
date: 2018-10-06 11:30:06 -0700
post_author: Alban Diquet
categories: ssl
---

I just released [SSLyze 2.0.0][sslyze-gh], my Python library for scanning the SSL/TLS configuration of a server.

This release adds support for the final version of TLS 1.3, and also introduces a lot of behind-the-scene improvements that I am going to describe in this article.

### Changelog

* Dropped support for Python 2 and older versions of Python 3; **only Python 3.6 and 3.7 are supported**.
* Added support for the final/official release of TLS 1.3 (RFC 8446).
* Added beta support for [TLS 1.3 early data (0-RTT) testing](https://tools.ietf.org/html/draft-ietf-httpbis-replay); see `--early_data` and `EarlyDataScanCommand`.
* Significantly improved [the documentation for the Python API][sslyze-documentation].
* SSLyze can now be installed via Docker.
* Bug fixes.
* Switched to a more modern Python tool chain (pipenv, pytest, pyinvoke).
* Removed legacy Python 2/3 code and ported the code base to Python 3 only.

### A modern Python toolchain

A lot of the changes I've implemented for this release had to do with using new/better Python tools that have been released since I initially started working on SSLyze eight years ago:

* **Type checker**: I added type annotations to the whole code base using the [typing module](https://docs.python.org/3/library/typing.html); strict type checking is then enforced in CI with [mypy](https://mypy.readthedocs.io/en/latest/).
* **Build system**: I re-implemented the build and tasks (testing, etc.) system using [Invoke](https://www.pyinvoke.org/). SSLyze's [C module for accessing OpenSSL](nass-gh) requires compiling various libraries (Zlib, OpenSSL, etc.) and the previous implementation was using custom Python code. With Invoke, the whole C module can be built using one command, on all supported platforms (Linux, Windows, etc.).
* **Test runner**: I switched to [pytest](https://docs.pytest.org/en/latest/) as the test runner; it provides more options and a lot more details when a test fails, and is overall superior to the standard library's unittest module. Even the [unittest's documentation](https://docs.python.org/3.7/library/unittest.html) mentions pytest as a better solution. The next step will be to migrate the actual code within SSLyze's test suite from unittest to pytest (which provides an API for tests that's a lot cleaner).
* **Dependencies management**: I switched to [Pipenv](https://pipenv.readthedocs.io/en/latest/) for dependency and virtual environment management. It replaces `pip` and `virtualenv`, and makes things a lot simpler. Additionally, GitHub's [Dependency graph feature](https://github.com/nabla-c0d3/sslyze/network/dependencies) supports Pipenv, and can automatically detect dependencies that have known vulnerabilities; pretty cool!

Modernizing the toolchain will make it a lot easier to maintain and extend SSLyze, and makes the code base a lot more approachable to developers who may be interested in contributing.

### OpenSSL: double the fun

Following the discovery of the Heartbleed vulnerability in 2014, the OpenSSL team decided to start aggressively dropping support for TLS features or protocols that are insecure and should not be used. This is obviously a very good thing for the Internet, but it also makes the job of scanning servers for TLS issues more difficult.

For example, SSLyze relies on OpenSSL to try to perform SSL 2.0 handshakes in order to find servers that support this legacy protocol. Once OpenSSL stopped supporting SSL 2.0 (which, again, is a good thing), any future release of OpenSSL could no longer be used by SSLyze.

The solution I ended up implementing is to package not one but two (!!) versions of OpenSSL within SSLyze (more specifically within [nassl, its C module for accessing OpenSSL](https://github.com/nabla-c0d3/nassl)):

* A "legacy" version of OpenSSL, 1.0.1e. This is the last OpenSSL release that supports all the insecure features and protocols, and the version SSLyze uses to scan for things like Heartbleed, SSL 2.0, CCS injection, etc.
* A "modern" version of OpenSSL, 1.1.1. This version was released only a few days ago, and is the version SSLyze uses to scan for modern TLS features, such as [TLS 1.3 and early data](https://blog.cloudflare.com/introducing-0-rtt/).

This approach ensures that moving forward, SSLyze can scan for both legacy TLS issues, and new features and protocols.

### More details

For more details, head to [the project's page][sslyze-gh] or the [Python documentation][sslyze-documentation].

[nass-gh]: https://github.com/nabla-c0d3/nassl
[sslyze-documentation]: https://nabla-c0d3.github.io/sslyze/documentation/
[sslyze-gh]: https://github.com/nabla-c0d3/sslyze/releases
