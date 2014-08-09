---
layout: post
title:  "Exploring SSL Pinning on iOS"
date:   2013-02-19 13:11:22
categories: ios ssl
---

_Note: this article was originally posted on [iSEC Partners' blog][original-post]._

### SSL Pinning


When an iOS app only needs to communicate to a well-defined set of servers over SSL, the security of the app's network communications can be improved through SSL pinning. By requiring a specific certificate to be part of the server's certificate chain, the threat of a rogue CA or a CA compromise is significantly reduced.

To simplify the process of adding this security feature to iOS apps, I released some source code as part of iSEC Partners' [SSL conservatory project][sslcons-gh]. The two source files can easily be added to an existing iOS app and provide a simple API to pin certificates to the domains the app needs to connect to.


#### _The SSLCertificatePinning_ class

This implementation allows a developer to pin a certificate for any number of domains the application needs to connect to. Specifically, developers can whitelist a certificate that will have to be part of the certificate chain sent back by the server during the SSL handshake. This gives additional flexibility as developers can decide to pin the CA/anchor certificate, the server/leaf certificate, or any intermediate certificate for a given domain. Each option has different advantages and limitations; for example, pinning the server/leaf certificate provides the best security but the certificate is going to change more often than the CA/anchor certificate. A change in the certificate (for example because it expired) will result in the app being unable to connect to the server. When that happens, the new certificate can be pushed to users by releasing a new version of the iOS app.

The _SSLCertificatePinning_ API only exposes two methods and a convenience class:

* _+(BOOL)loadSSLPinsFromDERCertificates:(NSDictionary \*)certificates_

This method takes a dictionary with domain names as keys and DER-encoded certificates as values and stores them in a pre-defined location on the filesystem.

* _+(BOOL)verifyPinnedCertificateForTrust:(SecTrustRef)trust andDomain:(NSString \*)domain_

This method accesses the certificates previously loaded using the _loadSSLPinsFromDERCertificates:_ method and looks in the trust object's certificate chain for a certificate pinned to the given domain.
_SecTrustEvaluate()_ should always be called before this method to ensure that the certificate chain is valid.


#### The _SSLPinnedNSURLConnectionDelegate_ class

The _SSLPinnedNSURLConnectionDelegate_ class is designed to be subclassed and extended to implement the _NSURLConnectionDelegate_ protocol and be used as a delegate for _NSURLConnection_ objects. This class implements the _connection:willSendRequestForAuthenticationChallenge:_ method so that it automatically validates that the certificate pinned to the domain the NSURLConnection object is accessing is part of the server's certificate chain.


### Cocoa Touch Shortcomings


Specific limitations due to what functions the iOS API exposes to developers and how much freedom they have when dealing with certificate validation prevented me from implementing some of the ideas that I had in mind when I started this project.


#### Public key pinning

For various reasons, it is actually better to pin a public key to a domain rather than the whole certificate. Unfortunately, public key pinning can only be partially implemented on iOS. While it is possible to extract the public key from a given certificate using a convoluted series of APIs calls (_SecTrustCreateWithCertificates()_ and _SecTrustCopyPublicKey()_), the anchor certificate for a given certificate chain cannot be accessed. Therefore, as the anchor/CA public key cannot be extracted and validated, only the public key of the leaf or any intermediate certificates can be pinned to a domain.


#### SSL pinning in webviews

On iOS, the _UIWebView_ class can be used to directly embed and display web content inside an app. However, as opposed to regular network communication APIs such as _NSURLConnection_, authentication challenges cannot be reliably handled when using the _UIWebView_ class. Various tricks exist, for example to perform HTTP Basic authentication or to disable certificate validation within a webview through a credential caching mechanism. However, the lack of explicit APIs to configure the webview's behavior makes it unsuitable for SSL pinning.


#### Revocation

Revocation checking is needed even when doing certificate pinning. However, there is no way on iOS to configure how revocation should be handled. The default behavior is to only check revocation for EV certificates, and this validation is "best attempt", meaning that if the OCSP server cannot be reached, the validation will still not fail.


### Project Page


The [SSL Conservatory][sslcons-gh] on GitHub.


[original-post]: https://www.isecpartners.com/news-events/news/2013/february/ssl-pinning-on-ios.aspx
[sslcons-gh]: https://github.com/iSECPartners/ssl-conservatory/tree/master/ios
