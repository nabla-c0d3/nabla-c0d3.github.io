---
layout: post
title: "Introducing the Trust Stores Observatory"
date: 2018-01-16 13:02:06 -0700
post_author: Alban Diquet
categories: ssl
---

For anyone interested in SSL/TLS, certificates, and trust, it has always been surprisingly difficult to get the list of root certificates trusted on each of the major platforms (Mozilla, Microsoft, etc.). 

The only tool that I am aware of is the [Certification Authority Trust Tracker (CATT)](https://github.com/kirei/catt), which I have been using for many years in order to retrieve the root stores to be used in [SSLyze](https://github.com/nabla-c0d3/sslyze), the SSL scanning tool I work on. However and as useful as it has been, CATT has to be run manually every time, and is not easy to extend or troubleshoot as it relies on several scripts written in Bash or Perl.

Because it shouldn't be this hard to retrieve and monitor the content of the main platforms' root stores, I have been working on a new project called the [**Trust Stores Observatory**](https://github.com/nabla-c0d3/trust_stores_observatory); it provides the following features:

* An easy way to download the most up-to-date root certificate stores, via a permanent link: [https://nabla-c0d3.github.io/trust_stores_observatory/trust_stores_as_pem.tar.gz](https://nabla-c0d3.github.io/trust_stores_observatory/trust_stores_as_pem.tar.gz).
* The ability to record any changes made to the root stores, by committing such changes to Git. This way we can keep the history of the root stores and for example keep track of when a new root certificate was added.
* The ability to review and compare the content of the different root stores, by storing the content of each store in a YAML file.

### Supported platforms

The Trust Stores Observatory currently supports the following platforms:

* **Microsoft Windows**: every two months, Microsoft releases the list of root certificates via an Excel spreadsheet available at [https://aka.ms/trustcertpartners](https://aka.ms/trustcertpartners). The actual certificates (in PEM or DER format) are not provided and a few rows within the spreadsheet are missing some information (such as the root certificate's fingerprint).
* **Mozilla NSS**: the list of root certificates is stored in source code, in a custom format which is convenient for NSS to build from, at [https://hg.mozilla.org/mozilla-central/raw-file/tip/security/nss/lib/ckfw/builtins/certdata.txt](https://hg.mozilla.org/mozilla-central/raw-file/tip/security/nss/lib/ckfw/builtins/certdata.txt).
* **Apple iOS and macOS**: Apple maintains one web page [for iOS](https://support.apple.com/en-us/HT204132) and one [for macOS](https://support.apple.com/en-us/HT202858) with the list of root certificates for each major release. The actual certificates (in PEM or DER format) are not provided.
* **Google AOSP**: the list of trusted root certificates is available in PEM format in the Android Open Source Project's repository at [https://android.googlesource.com/platform/system/ca-certificates](https://android.googlesource.com/platform/system/ca-certificates).

### How it works

The project is implemented using Python 3.6. Each root store is stored in a YAML file in the project's repository; the YAML file contains the subject name and the fingerprint of every trusted and blocked root certificate.

Once a week, a [Travis cron](https://travis-ci.org/nabla-c0d3/trust_stores_observatory) is automatically run in order to retrieve the latest version of each root store, and to commit any changes to the observatory's repository.

### What's next?

* Support for additional platforms and root stores (Java, Ubuntu, etc.).
* Support for also retrieving the list of EV OIDs.
* Better handling of special restrictions (name constraints, notBefore, etc.) as several platforms have implemented custom restrictions for some CA certificates.

### Check it out

Head to the [project's page](https://github.com/nabla-c0d3/trust_stores_observatory) for more information and feel free to reach out if you have questions or feedback!
