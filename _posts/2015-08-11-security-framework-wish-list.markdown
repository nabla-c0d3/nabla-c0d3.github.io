---
layout: post
title: "Apple Security Framework Wish List"
date: 2015-08-11 13:02:06 -0700
post_author: Alban Diquet
categories: ios
---

While working on [TrustKit][bh2015-conf], I ran into significant challenges when trying to implement SSL pinning the right way. The best practice is to pin the certificate's Subject Public Key Info (ie. the public key and the key's algorithm/type). However, the Security framework on iOS and OS X does not provide comprehensive APIs for parsing certificates or interacting with the OS' trust store. 

While I did manage to implement SPKI pinning in TrustKit, the workarounds I had to use are unsatisfying, and I hope that Apple can eventually extend the Security framework to make it easier to implement SSL pinning code.


### Extracting the public key data from a certificate

On iOS, the only way to retrieve the public key bits from a certificate is convoluted:

1. Create a `SecTrustRef` trust object with the certificate, and call `SecTrustEvaluate()` on it.
2. Call `SecTrustCopyPublicKey()` on the trust object to retrieve a `SecKeyRef` of the public key.
3. Add the public key to the device's Keychain using `SecItemAdd()`.
4. Retrieve the public key from the Keychain and set `kSecReturnData` to `YES` in order to specify that the key's data should be returned (instead of an opaque `SecKeyRef`).

Having to put data in Keychain just to retrieve the public key bits of a certificate seems really complex, slow and inefficient. On OS X, the key's' data can be retrieved by just calling `SecCertificateCopyValues()` on the certificate with `kSecOIDX509V1SubjectPublicKey` as the list of OIDs.

I had to implement both techniques in TrustKit [within the pinning logic][pubkey-gh], which shows how different the code is depending on the platform. 

[See the bug on Open Radar][pubkey-rdar].


### Extracting the SPKI data from a certificate

On both iOS and OS X, there is no API to extract the Subject Public Key Info bits. On OS X, the `SecCertificateCopyValues()` function seemed like a good candidate in combination with the `kSecOIDX509V1SubjectPublicKeyAlgorithm` and `kSecOIDX509V1SubjectPublicKeyAlgorithmParameters` OIDs. However the function only returns a parsed output (such as the OID corresponding to the key algorithm) instead of the actual bytes of the SPKI.

Overall, the possible solutions here are far from ideal:

* Parsing ASN1 manually, a non-starter
* Embedding OpenSSL, which tends to quickly get dangerously outdated

In TrustKit, I took an unsatisfying but less dangerous approach: the developer has to specify the public key algorithm (RSA 2048, ECDSA, etc.) in the configuration. Then, TrustKit re-generates the SPKI by picking the bits corresponding to the algorithm [from a hardcoded list][spki-gh], and combining it with the public key bits.

[See the bug on Open Radar][spki-rdar].


### Detecting private trust anchors

When implementing SSL pinning, it is useful to __not__ enforce pinning when a private trust anchor (ie. one that was manually added to the OS' trust store) is detected in the server's certificate chain. This is needed for allowing SSL connections through corporate proxies or firewalls and it is [how Chrome operates in that scenario][chrome-secu]; stopping attackers with the ability to modify the OS (by adding certificates to the OS trust store, hooking SSL APIs, etc.) is not part of SSL pinning's threat model.

There are no APIs on iOS to differentiate private trust anchors from system trust anchors (the ones shipped with the OS). Consequently, pinning can only be implemented in a way that will always block connections where pinning should not actually be enforced.

On OS X, the list of private trust anchors can be retrieved using `SecTrustSettingsCopyCertificates()` with the
`kSecTrustSettingsDomainUser` and `kSecTrustSettingsDomainAdmin` domain settings. Then, this list can be used to detect a private trust anchor in the server's certificate chain, [as implemented in TrustKit][anchor-gh].

[See the bug on Open Radar][anchor-rdar].


[chrome-secu]: https://www.chromium.org/Home/chromium-security/security-faq
[pubkey-rdar]: https://openradar.appspot.com/22205652
[pubkey-gh]: https://github.com/datatheorem/TrustKit/blob/master/TrustKit/Pinning/public_key_utils.m#L63
[spki-rdar]: https://openradar.appspot.com/22209593
[spki-gh]: https://github.com/datatheorem/TrustKit/blob/master/TrustKit/Pinning/public_key_utils.m#L30
[anchor-rdar]: https://openradar.appspot.com/22206007
[anchor-gh]: https://github.com/datatheorem/TrustKit/blob/master/TrustKit/Pinning/ssl_pin_verifier.m#L146
[bh2015-conf]: https://www.blackhat.com/us-15/briefings.html#trustkit-code-injection-on-ios-8-for-the-greater-good
