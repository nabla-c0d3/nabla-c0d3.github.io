---
layout: post
title:  "Certificate Validation with OpenSSL"
date:   2012-10-16 15:12:29
categories: ssl openssl
---

### The most dangerous code in the world

Stanford just released a whitepaper about the lack of proper SSL certificate validation in client applications: ["The most dangerous code in the world: validating SSL certificates in non-browser software"][ssl-client-bugs]. This papers calls out numerous, popular applications and libraries that do not properly validate certificates, thereby eliminating the confidentiality and integrity guarantees of SSL. According to the paper, one of the root cause of this issue is "badly designed APIs of SSL implementations (such as JSSE, OpenSSL, and GnuTLS) [...] which present developers with a confusing array of settings and options".


### The most dangerous API in the world: OpenSSL

Undoubtedly, OpenSSL's APIs for SSL certificate validation are confusing and poorly documented. Even worse, as opposed to other SSL libraries, OpenSSL does not provide any utility function to perform hostname validation: the library only validates the server's certificate chain, not the name the certificate was issued for.

Throughout my work as a security consultant, I have seen numerous applications relying on OpenSSL to secure their network connection but failing to properly validate server certificates, resulting in the connections being vulnerable to man-in-the-middle attacks. This has led me to work on a whitepaper to provide application developers with clear and simple guidelines on how to properly perform certificate validation within an SSL client using the OpenSSL library. This whitepaper is now available on GitHub: ["Everything You've Always Wanted to Know About Certificate Validation With OpenSSL (but Where Afraid to Ask)"][mywp].


### The SSL Conservatory

This project is part of iSEC Partners' [SSL Conservatory][ssl-cons].

[ssl-client-bugs]: https://crypto.stanford.edu/~dabo/pubs/abstracts/ssl-client-bugs.html
[mywp]: https://github.com/iSECPartners/ssl-conservatory/blob/master/openssl/everything-you-wanted-to-know-about-openssl.pdf?raw=true
[ssl-cons]: https://github.com/iSECPartners/ssl-conservatory/

