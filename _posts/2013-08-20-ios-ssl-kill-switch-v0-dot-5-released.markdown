---
layout: post
title: "iOS SSL Kill Switch v0.5 Released"
date: 2013-08-20 10:48
comments: true
categories: ios ssl
---

Version 0.5 of the [iOS SSL Kill Switch][killswitch-blog] is now available.
iOS SSL Kill Switch is a tool to disable SSL certificate validation -
including certificate pinning - within iOS Apps in order to facilitate
blackbox testing.

The main goal for this version was to add the ability to [disable
certification within the iTunes App Store app][appstore-gh]. While working on
this feature, I discovered a new way to disable certificate validation that
worked on many more applications than previous versions of the tweak.
As a consequence, version 0.5 of the SSL Kill Switch is a complete rewrite.

__The debian package is available [here][killswitch-deb]__ and was tested on iOS 6.1.
Instructions on how to install it are available on the [project page][killswitch-gh].


### How it works

Just like the previous versions of the tool, the SSL Kill Switch uses [MobileSubstrate][mobilesubstrate-wiki]
to patch system functions. However, this new version of the tweak hooks
functions within the [Secure Transport API ][securetransport-api] instead of
hooking _NSURLConnection_ methods and _SecTrustEvaluate()_.

The Secure Transport API is ``the lowest-level TLS implementation on iOS''
which makes it an interesting target because other higher level APIs such as
_NSURLConnection_ internally rely on the Secure Transport API for their
certificate validation routines. This means that disabling SSL certificate
validation in the Secure Transport API should affect most (if not all) of the
network APIs available within the iOS framework.

According to the [documentation][securetransport-doc], disabling or performing
custom certificate validation is implemented the following way when using the
Secure Transport API:

1. Before starting the connection, call _SSLSetSessionOption()_ to set the _kSSLSessionOptionBreakOnServerAuth_ option to ``true'' on the SSL context. Setting this option to ``true'' disables the framework's built-in certificate validation to let the application perform its own certificate verification.
2. Run the Secure Transport handshake as per usual using the _SSLHandshake()_ function.
3. When _SSLHandshake()_ returns _errSSLServerAuthCompleted_, call _SSLCopyPeerTrust()_ to get a trust object for the connection and use that trust object to implement whatever custom server trust evaluation you desire.
4. Either continue the Secure Transport handshake by calling _SSLHandshake()_ again, or shut down the connection.

The SSL Kill Switch removes the ability to do any kind of certificate
validation by hooking and modifying three functions within the [Secure
Transport API][securetransport-api].

#### Patch _SSLCreateContext()_: Disable the built-in certificate validation in all SSL contexts
_SSLCreateContext()_ is used to create a new SSL context. SSL Kill Switch modifies this function
so that all new SSL contexts have the _kSSLSessionOptionBreakOnServerAuth_ set to true
by default:

{% highlight c %}
static SSLContextRef replaced_SSLCreateContext (
   CFAllocatorRef alloc,
   SSLProtocolSide protocolSide,
   SSLConnectionType connectionType
) {
    SSLContextRef sslContext = original_SSLCreateContext(alloc, protocolSide, connectionType);

    // Immediately set the kSSLSessionOptionBreakOnServerAuth option in order to disable cert validation
    original_SSLSetSessionOption(sslContext, kSSLSessionOptionBreakOnServerAuth, true);
    return sslContext;
}
{% endhighlight %}


#### Patch _SSLSetSessionOption()_: Remove the ability to re-enable the built-in certificate validation
_SSLSetSessionOption()_ can be called to set the value of specific options on a given SSL context. The tweak
patches this function in order to prevent the _kSSLSessionOptionBreakOnServerAuth_ from being set to any value.
The goal here is to ensure that all SSL contexts keep the default value of ``true'' for the
_kSSLSessionOptionBreakOnServerAuth_ option (as set in the previous section):


{% highlight c %}
static OSStatus replaced_SSLSetSessionOption(
    SSLContextRef context,
    SSLSessionOption option,
    Boolean value
 ) {
    // Remove the ability to modify the value of the kSSLSessionOptionBreakOnServerAuth option
    if (option == kSSLSessionOptionBreakOnServerAuth)
        return noErr;
    else
        return original_SSLSetSessionOption(context, option, value);
}
{% endhighlight %}


#### Patch _SSLHandshake()_: Force a trust-all custom certificate validation
Lastly, _SSLHandshake()_ is modified in order to prevent this function from ever returning
_errSSLServerAuthCompleted_, which is the return value that will trigger the caller's certificate
checking/pinning code:


{% highlight c %}
static OSStatus replaced_SSLHandshake(
    SSLContextRef context
) {
    OSStatus result = original_SSLHandshake(context);

    // Hijack the flow when breaking on server authentication
    if (result == errSSLServerAuthCompleted) {
        // Do not check the cert and call SSLHandshake() again
        return original_SSLHandshake(context);
    }
    else
        return result;
}
{% endhighlight %}

That's it ! After patching those three functions, certificate validation was disabled in all
the applications that I tried including Safari, Twitter, Square as well as the iTune App
Store ([with a few additional steps][killswitch-appstore]).


### Project page

Have a look at the full code by browsing to the [project page][killswitch-gh] on GitHub.



[killswitch-blog]: /blog/2012/08/12/ios-ssl-kill-switch-released/
[securetransport-api]: https://developer.apple.com/library/ios/DOCUMENTATION/Security/Reference/secureTransportRef/Reference/reference.html
[securetransport-doc]: https://developer.apple.com/library/ios/technotes/tn2232/_index.html#//apple_ref/doc/uid/DTS40012884-CH1-SECSECURETRANSPORT
[killswitch-deb]: https://www.dropbox.com/s/p81z1l0ygnl35vw/com.isecpartners.nabla.sslkillswitch_v0.5-iOS_6.1.deb?dl=1
[killswitch-gh]: https://github.com/iSECPartners/ios-ssl-kill-switch
[mobilesubstrate-wiki]: http://iphonedevwiki.net/index.php/MobileSubstrate
[killswitch-appstore]: /blog/2013/08/20/intercepting-the-app-stores-traffic-on-ios/
[appstore-gh]: https://github.com/iSECPartners/ios-ssl-kill-switch/issues/6
