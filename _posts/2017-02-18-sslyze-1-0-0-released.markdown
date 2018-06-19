---
layout: post
title: "SSLyze v1.0.0 Released"
date: 2017-02-08 13:02:06 -0700
post_author: Alban Diquet
categories: ssl
---

I just released a new version of SSLyze, my Python tool/library which can analyze the SSL configuration of a server by connecting to it and detect various issues (bad certificates, dangerous cipher suites, lack of session resumption, etc.). 

After almost 20 releases over the past 6 years, SSLyze is now at version 1.0.0. This is a major release as I have completed a significant refactoring of SSLyze's internals,in order to clean its Python API. The API should be considered stable and is now [fully documented][sslyze-documentation]! 

Using SSLyze as a Python module makes it easy to implement SSL/TLS scanning as part of continuous security testing platform, and detect any misconfiguration across a range of public and/or internal endpoints.


### Sample Code

This sample code scans the `www.google.com:443` endpoint to detect the list of SSL 3.0 cipher suites it accepts, whether it supports secure renegotiation, and will also review the server's certificate:

<script src="https://gist.github.com/nabla-c0d3/989e37e6204a5e689eeb988321b48ca3.js"></script>

### Getting Started

The Python API can do a lot more than this (such as scanning StartTLS endpoints, connecting through a proxy, or enabling client authentication); head to the [project's page][sslyze-gh] or the [documentation][sslyze-documentation] for more information.



[sslyze-documentation]: https://nabla-c0d3.github.io/sslyze/documentation/
[sslyze-gh]: https://github.com/nabla-c0d3/sslyze
