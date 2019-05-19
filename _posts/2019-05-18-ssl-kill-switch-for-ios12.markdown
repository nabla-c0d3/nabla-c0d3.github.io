---
layout: post
title: "How SSL Kill Switch works on iOS 12"
date: 2019-05-18 11:30:06 -0700
post_author: Alban Diquet
categories: ssl ios
---

Two weeks ago, I released a new version of [SSL Kill Switch][sks-gh], my blackbox tool for disabling SSL pinning in iOS apps, in order to add support for iOS 12.

The network stack [changed significantly](https://developer.apple.com/videos/play/wwdc2018/715/) between iOS 11 and 12, and it was no surprise that the iOS 11 version of SSL Kill Switch did not work on (jailbroken) iOS 12 devices. This post describes the changes I had to make for the tool to support iOS 12.

### Strategy for disabling SSL pinning

Implementing SSL pinning in a mobile application requires customizing the validation logic done by the app on the server's certificate chain, when the app opens an SSL connection to this server. Customizing SSL validation is almost always done via some kind of callback mechanism, where the application code receives the server's certificate chain during the connection's initial TLS handshake, and then has to make a decision on the chain (whether it is "valid", or not). For example, on iOS:

* The highest level API for opening HTTPS connections is `NSURLSession`, which implements the validation callback via the [`[NSURLSessionDelegate URLSession:didReceiveChallenge:completionHandler:]`](https://developer.apple.com/documentation/foundation/url_loading_system/handling_an_authentication_challenge/performing_manual_server_trust_authentication) delegate method.
* When using the low level Network.framework (newly available in iOS 12), a block can be set as the validation callback using [`sec_protocol_options_set_verify_block()`](https://developer.apple.com/documentation/security/2976289-sec_protocol_options_set_verify_).
* This older article from the Apple documentation describes how to customize validation with other iOS network APIs: [HTTPS Server Trust Evaluation](https://developer.apple.com/library/archive/technotes/tn2232/_index.html#//apple_ref/doc/uid/DTS40012884-CH1-SECCUSTOMIZEAPIS).

Hence, the **high-level strategy for disabling SSL pinning in applications is to prevent the SSL validation callbacks from being triggered**, so that the application code that is responsible for implementing pinning is never exercised.

On iOS, it is relatively easy to prevent the `NSURLSessionDelegate` validation method from being called (and it is [how the early versions of SSL Kill Switch worked](https://github.com/nabla-c0d3/ios-ssl-kill-switch/blob/release-0.3/Tweak.xm)), but what about iOS apps that use a lower level API (such as Network.framework)? As each networking API on iOS is built on top of another, disabling the validation callbacks at the lowest level would potentially disable validation for all the higher level network APIs, which would allow the tool to work against a lot more apps.

The network stack on iOS in general has been going through a lot of changes since iOS 8, and on iOS 12, the SSL/TLS stack is built on a custom fork (I think?) of [BoringSSL](https://www.imperialviolet.org/2014/06/20/boringssl.html). This can be seen for example by setting a breakpoint on a random BoringSSL symbol when running [an app that opens a connection][tk-demo-app]:

![](/images/posts/kill-switch-ios12/SSL_CTX_new.png)

If you remember the strategy: "prevent the SSL validation callbacks from being triggered", it is likely that by targeting and patching BoringSSL, the lowest level SSL/TLS API on iOS, all the higher level APIs on iOS (including `NSURLSession`) would also have pinning validation disabled.

Let's try!

### BoringSSL's validation callback

When using BoringSSL, one way to customize SSL validation is to configure a validation callback function via the [`SSL_CTX_set_custom_verify()`](https://github.com/google/boringssl/blob/7540cc2ec0a5c29306ed852483f833c61eddf133/include/openssl/ssl.h#L2294) function:

Here is a simplified example of how it is meant to be used:

    // Define a cert validation callback to be triggered during the SSL/TLS handshake
    ssl_verify_result_t verify_cert_chain_callback(SSL* ssl, uint8_t* out_alert) {
        // Retrieve the certificate chain sent by the server during the handshake
        STACK_OF(X509) *certificateChain = SSL_get_peer_cert_chain(ssl);

        // Do custom validation (pinning or something else)
        if do_custom_validation(certificateChain) == 0 {
            // If validation succeeded, return OK
            return ssl_verify_ok;
        }
        else {
            // Otherwise close the connection
            return ssl_verify_invalid;
        }
    }

    // Enable my callback for all future SSL/TLS connections implemented using the ssl_ctx
    SSL_CTX_set_custom_verify(ssl_ctx, SSL_VERIFY_PEER, verify_cert_chain_callback);

Using a [test app with SSL pinning enabled for `NSURLSession`][tk-demo-app], I was able to confirm that `SSL_CTX_set_custom_verify()` does get called when opening a connection:

![](/images/posts/kill-switch-ios12/SSL_CTX_set_custom_verify.png)

We can also see the Apple/default iOS validation callback function passed as the third argument (in register x2): `boringssl_context_certificate_verify_callback()`. It is likely that this callback contains (among other things) logic to set things up for my test app's `NSURLSession` callback/delegate method to eventually be called with the server certificate.

And as expected, my test app's delegate method for pinning validation code does get exercised:

![](/images/posts/kill-switch-ios12/nsurlsession_delegate.png)

And I have designed my test app to have its custom/pinning validation logic always fail:

![](/images/posts/kill-switch-ios12/testapp_validation_failed.png)

Hence, if I do find a way to bypass pinning, this connection should instead succeed.

Now that we have a plan and a proper test setup (app with pinning, jailbroken device, Xcode, etc.), let's get to work!

### Tampering with BoringSSL

The first thing I tried was to replace the default BoringSSL callback set by the iOS networking stack,`boringssl_context_certificate_verify_callback()`, with an empty callback that does not check the server's certificate chain at all:

    // My "evil" callback that does not check anything
    ssl_verify_result_t verify_callback_that_does_not_validate(void *ssl, uint8_t *out_alert)
    {
        return ssl_verify_ok;
    }

    // My "evil" replacement function for SSL_CTX_set_custom_verify()
    static void replaced_SSL_CTX_set_custom_verify(void *ctx, int mode, ssl_verify_result_t (*callback)(void *ssl, uint8_t *out_alert))
    {
        // Always ignore the callback that was passed and instead set my "evil" callback
        original_SSL_CTX_set_custom_verify(ctx, SSL_VERIFY_NONE verify_callback_that_does_not_validate);
        return;
    }

    // Lastly, use MobileSubstrate to replace SSL_CTX_set_custom_verify() with my "evil" replaced_SSL_CTX_set_custom_verify()
    void* boringssl_handle = dlopen("/usr/lib/libboringssl.dylib", RTLD_NOW);
    void *SSL_CTX_set_custom_verify = dlsym(boringssl_handle, "SSL_CTX_set_custom_verify");
    if (SSL_CTX_set_custom_verify)
    {
        MSHookFunction((void *) SSL_CTX_set_custom_verify, (void *) replaced_SSL_CTX_set_custom_verify,  NULL);
    }

After implementing this as a MobileSubstrate tweak and injecting it into my test app, something interesting happened: my test app's `NSURLSession` delegate method was not called anymore (meaning it was "bypassed"), but the very first connection done by the app would fail with a new/unknown error, "Peer was not authenticated", as seen in the logs:

    TrustKitDemo-ObjC[3320:160146] === SSL Kill Switch 2: replaced_SSL_CTX_set_custom_verify
    TrustKitDemo-ObjC[3320:160146] Failed to clone trust Error Domain=NSOSStatusErrorDomain Code=-50 "null trust input" UserInfo={NSDescription=null trust input} [-50]
    TrustKitDemo-ObjC[3320:160146] [BoringSSL] boringssl_session_finish_handshake(306) [C1.1:2][0x10bd489a0] Peer was not authenticated. Disconnecting.
    TrustKitDemo-ObjC[3320:160146] NSURLSession/NSURLConnection HTTP load failed (kCFStreamErrorDomainSSL, -9810)
    TrustKitDemo-ObjC[3320:160146] Task <15E1F3B0-0B73-468A-9132-3E19048DDAE3>.<1> finished with error - code: -1200

And then in the app itself, this first connection would fail with a different error than before:

![](/images/posts/kill-switch-ios12/testapp_validation_failed2.png)

However, subsequent connections to the same server would succeed without triggering the pinning validation callback:

![](/images/posts/kill-switch-ios12/testapp_validation_succeeded.png)

Hence I had bypassed pinning for all connections except for the very first one. Almost there...

### Fixing the first connection

I needed more context to understand what the "Peer was not authenticated" error was, so I ended up pulling the shared cache (where all of Apple's libraries and frameworks are, including BoringSSL) from my iOS 12 device, as described in [this guide](https://kov4l3nko.github.io/blog/2016-05-13-disassembling-ios-system-frameworks-and-libs/).

After loading _libboringssl.dylib_ into Hopper, I was able to find the string for the "Peer was not authenticated" error (labelled as "1" in the screenshot), in a function called `boringssl_session_finish_handshake()`:

![](/images/posts/kill-switch-ios12/finish_handshake.png)

I tried to understand what this function was doing to get a better understanding of the error itself, but since I barely understand arm64 (or any) assembly, I couldn't figure it out. I tried a few other approaches (such as patching the `boringssl_context_certificate_verify_callback()` itself) but didn't find anything that worked.

As I was running out of week-end time I can allow myself to spend on this, I went for a more desperate approach. If you look again at the decompiled `boringssl_session_finish_handshake()` function, you can see two "main" code paths, conditionally triggered by an if/else statement, with the "Peer was not authenticated" error happening in the "if" code path but not in the "else" path.

A naive attempt would be to prevent the code path with this error from ever being run, ie. the "if" path. As seen in the screenshot, one condition that does trigger the "if" branch is `(_SSL_get_psk_identity() == 0x0)` (labelled as "2" in the screenshot). What if we patched this function to not return 0, in order to force the execution of the "else" code path (which doesn't trigger the "Peer was not authenticated" error)?

The MobileSubtrate patch for this looks like this:

    // Use MobileSubstrate to replace SSL_get_psk_identity() with this function, which never returns 0:
    char *replaced_SSL_get_psk_identity(void *ssl)
    {
        return "notarealPSKidentity";
    }
    MSHookFunction((void *) SSL_get_psk_identity, (void *) replaced_SSL_get_psk_identity, (void **) NULL);

After injecting this runtime patch into my test app, it worked! Even the first connection succeeded, and my app's validation callback was never triggered. I had bypassed my app's SSL pinning validation code by patching BoringSSL.

### Conclusion

This is obviously not a very clean runtime patch, and while everything seems to work fine after applying it (which is surprising), it triggers errors that can be seen in the logs whenever the app opens a connection:

    TrustKitDemo-ObjC[3417:166749] Failed to clone trust Error Domain=NSOSStatusErrorDomain Code=-50 "null trust input" UserInfo={NSDescription=null trust input} [-50]

The patch has other problems too:

* It probably messes up code related to TLS-PSK cipher suites, which is when the `SSL_get_psk_identity()` function is actually used. However, these cipher suites are rarely used, especially in mobile applications.
* The default BoringSSL callback that is part of the iOS network stack, `boringssl_context_certificate_verify_callback()`, is never called. This means that some state within the iOS networking stack is probably not getting set properly, which should lead to bugs.

Lastly, there are a few extra things I didn't have time to do:

* Double checking that my BoringSSL runtime patch does disable pinning for lower-level iOS networking APIs, such as Network.framework or CFNetwork.
* Adding support for macOS. I am pretty sure the patch itself should work as it is, but I haven't found a way of hooking BoringSSL (or any C function in the shared cache) on macOS. The tool I was using previously, Facebook's [fishhook](https://github.com/facebook/fishhook), does not seem to work anymore.

That's all! Head to [the project's repo][sks-gh] to see the code and download the tweak.

[sks-gh]: https://github.com/nabla-c0d3/ssl-kill-switch2
[tk-demo-app]: https://github.com/datatheorem/TrustKit/tree/master/TrustKitDemo/TrustKitDemo-ObjC
