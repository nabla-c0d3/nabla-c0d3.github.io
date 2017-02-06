---
layout: post
title: "SSL Kill Switch and the iOS 10 Network Stack"
date: 2017-02-05 13:02:06 -0700
post_author: Alban Diquet
categories: ios ssl
---

After the release of the latest iOS 10 jailbreak, I started [getting reports][ios10-report] that SSL Kill Switch, my iOS tool [to disable SSL pinning in Apps][ssl-kill-switch], was not working at all on this version of iOS. 

This was very surprising since the tool [directly patches the OS' low-level TLS stack][kill-switch-5] (called _SecureTransport_) which is used by all the higher-level network APIs on iOS/macOS (_CFNetwork_, _NSURLSession_, etc.) to setup a TLS/HTTPS connection with a server. The functions SSL Kill Switch patches are part of _SecureTransport_'s public API and therefore do not really change across iOS versions, making it unlikely that the tool would just stop working on a new release of iOS.

After doing some investigation, I discovered that the network stack on iOS 10 has changed significantly compared to iOS 9, and I will describe in this post what has changed, why it affected SSL Kill Switch, and how I fixed it. 

__If you're looking for the latest SSL Kill Switch binary compatible with iOS 10, go to the Releases tab of [the project's page][ssl-kill-switch].__


### Getting Started

To try to understand why SSL Kill Switch stopped working on iOS 10, I used Xcode and a simple App that opens an HTTPS connection using the _NSURLSession_ API and implements SSL pinning (specifically, the [TrustKit Demo App][trustkit-demo]). And as reported, SSL Kill Switch was not able to disable TLS validation when running the App on iOS 10 and trying to use the Burp proxy to intercept the HTTPS connection.

I now needed to figure out what had changed in iOS 10's network stack, and started by grabbing the dyld shared cache from my iOS 10 device (it's in _/System/Library/Caches/com.apple.dyld/dyld\_shared\_cache\_armv7s_) and opening it in this awesome disassembler called [Hopper][hopper]. Looking at the _CFNetwork_ framework, I started checking TLS methods and symbols, to find the lowest-level TLS function that would be called during a TLS handshake:

![](/images/posts/network-hopper.png)
    
Jumping around in various system frameworks and libraries, I eventually settled for the `tls_handshake_set_callbacks` function in _libsystem\_coretls.dylib_. This function is used to setup callbacks for specific events happening during the TLS handshake; one thing that helped throughout the process is that _libsystem\_coretls.dylib_ [used to be open-source][coretls] (athough the iOS implementation might be different).

To try to understand the differences between the iOS 9 and iOS 10 network stacks, I then set a breakpoint on this function in Xcode, in order to be able to look at the stack trace, so I could see the execution flow from the high level _NSURLSession_ network call initiated by my test App, all the way down to this low-level TLS function.

    br set -n tls_handshake_set_callbacks
    Breakpoint 3: where = libsystem_coretls.dylib`tls_handshake_set_callbacks, address = 0x000000010895928c

This would allow me to understand how the network stack differs on different versions of iOS.


### iOS 9 Stack Trace

When the breakpoint is hit, Xcode displays the following stack trace on iOS 9 and 8:

![](/images/posts/network-ios9.png)

The stack trace looks like this, from top to bottom:

* `tls_handshake_set_callbacks` in _libsystem\_coretls.dylib_
* `SSLCreateContextWithRecordFuncs` in _Security_ (specifically _SecureTransport_)
* `SSLCreateContext` in _Security_ (specifically _SecureTransport_)
* `SocketStream::securitySetInfo_NoLock` in _CFNetwork_
* _[Higher level calls]_

As expected, we can see _SecureTransport_ functions (`SSLCreateContextWithRecordFuncs` and `SSLCreateContext`) being called during the TLS handshake. This is where the previous version of SSL Kill Switch would step in, by patching specific _SecureTransport_ functions to disable TLS validation. Interestingly and as we can see, some of the actual code for the TLS logic is indeed in the _libsystem\_coretls.dylib_ library, which is called by _SecureTransport_. I previously thought that the TLS implementation was all part of _SecureTransport_.


### iOS 10 Stack Trace

On iOS 10, the stack trace for the same breakpoint looks quite different:

![](/images/posts/network-ios10.png)

More specifically, from top to bottom:

* `tls_handshake_set_callbacks` in _libsystem\_coretls.dylib_
* `nw_protocol_coretls_add_input_handler` in _libnetwork.dylib_ 
* _[More libnetwork.dylib and dispatch calls...]_
* `tcp_connection_set_tls` in _libnetwork.dylib_
* `TCPIOConnection::_tlsEnable` in _CFNetwork_
* _[Higher level calls]_

As we can see, there are no _SecureTransport_ functions called during the TLS handshake! Instead, some new functions within _libnetwork.dylib_ are used, which then call  _libsystem\_coretls.dylib_, similarly to _SecureTransport_ on iOS 9.


### Conclusion 

Apple has significantly changed the network stack on iOS 10:
    
* On iOS 9, an HTTPS connection iniitiated via _NSURLSession_ involves _CFNetwork_'s `SocketStream`, _SecureTransport_ and _libsystem\_coretls.dylib_.
* On iOS 10, the same connection involves _CFNetwork_'s `TCPIOConnection`, _libnetwork.dylib_ and _libsystem\_coretls.dylib_

Without access to the source code, it is hard to tell how big of a change this is: is it some kind of refactoring / cleaning up that mainly affects the libraries' interfaces and flows, or are the implementations also very different?

Also, _libnetwork.dylib_ already existed on iOS 9, but did not have a lot of the functions that are used in iOS 10 for network connections (they are called `nw_xxx`). For instance, when trying to set a breakpoint for `nw_protocol_coretls_add_input_handler` on iOS 9, we can see that the function does not exist:

    br set -n nw_protocol_coretls_add_input_handler
    WARNING:  Unable to resolve breakpoint to any actual locations.

Because _SecureTransport_ is no longer used by higher level APIs (such as _CFNetwork_ or _NSURLSession_) on iOS 10, patching _SecureTransport_ functions like SSL Kill Switch does has no effect on an App's TLS connections. This expains why the tool compltely stopped working.

I wonder if these changes mean that Apple will eventually deprecate _SecureTransport_ and expose the iOS 10 network/TLS stack as a public API? 

### Fixing SSL Kill Switch on iOS 10

Now armed with a better understanding of the iOS 10 network stack, I started investigating how TLS validation was done in `TCPIOConnection`and _libnetwork.dylib_, in order to find a way to disable the system's default TLS validation and any code that customizes it (which is how SSL pinning is done).

When I did it for _SecureTransport_ a few years ago [for the previous versions of SSL Kill Switch][kill-switch-5], the customization mechanism for TLS validation was [well-documented][break-on-auth]  as _SecureTransport_ is a public API, and therefore easy to understand.

However, `TCPIOConnection`, _libnetwork.dylib_ and _libsystem\_coretls.dylib_ are all undocumented, private APIs, making this process a lot more time-consuming. Luckily, while playing around with some of the TLS functions in these libraries, I quickly stumbled upon an unexpected solution.

The `tls_helper_create_peer_trust` in _libcoretls\_cfhelpers.dylib_ seemed interesting, as it is used to retrieve the server's `SecTrustRef` during the TLS handshake. In the Cocoa world, a `SecTrustRef` is an object that represents the server's certificate chain and the policy to use to validate it; it is the main object to use when doing TLS validation. The [open-source implementation][tls-helper] of the function also made it a lot easier to confirm how it works and what it does:
    
    OSStatus tls_helper_create_peer_trust(tls_handshake_t hdsk, bool server, SecTrustRef *trustRef);

The `tls_helper_create_peer_trust` function generates the server's `SecTrustRef` from a `tls_handshake_t` object (which represents an ongoing TLS handshake) and puts it in the supplied `trustRef` argument. The `trustRef` object can then be used by the caller to do TLS validation and verify the server's identity.

As a random experimentation, I patched this function (using Cydia Substrate) to make it not do anything:
 
    static OSStatus replaced_tls_helper_create_peer_trust(void *hdsk, bool server, SecTrustRef *trustRef)
    {
        return errSecSuccess;
    }

With the patch applied, the `trustRef` does not contain anything at the end of the call to `tls_helper_create_peer_trust`. I was expecting this change to completely crash the App during the TLS handshake, as whatever code that is doing the TLS validation (probably inside `CFNetwork`) would get a `NULL` trust object as the server's identity (which can never happen), and wouldn't be able to process further. However, instead of crashing, it completely disabled all TLS validation and pinning, which is exactly what I was trying to do!

It is still unclear to me why this actually works; it seems like the validation logic or callback somehow is ignored when the server's `SecTrustRef` is `NULL`. This is obviously not a vulnerability, as I am injecting code in the App to trigger this behavior, but it is a bit surprising to me. If I ever have the time, I will dig further into this to try to understand what's going on. 

In the meantime enjoy the iOS 10 release!


[kill-switch-5]: /2013/08/20/ios-ssl-kill-switch-v0-dot-5-released/
[ssl-kill-switch]: https://github.com/nabla-c0d3/ssl-kill-switch2
[ios10-report]: https://github.com/nabla-c0d3/ssl-kill-switch2/issues/17
[slide-deck]: /documents/ios10_security_changes.pdf
[ats-post]: /blog/2016/08/14/ats-enforced-2017/
[hopper]: https://www.hopperapp.com/
[trustkit-demo]: https://github.com/datatheorem/TrustKit/tree/master/TrustKitDemo
[coretls]: https://opensource.apple.com/source/coreTLS/
[break-on-auth]: https://developer.apple.com/reference/security/sslsessionoption/1392670-breakonserverauth
[tls-helper]: https://opensource.apple.com/source/coreTLS/coreTLS-121.31.1/coretls_cfhelpers/tls_helpers.c.auto.html

