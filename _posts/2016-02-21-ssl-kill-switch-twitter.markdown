---
layout: post
title: "SSL Kill Switch and Twitter iOS"
date: 2016-02-21 13:02:06 -0700
post_author: Alban Diquet
categories: ios ssl
---

Some time ago, [SSL Kill Switch][ssl-kill-switch-gh] somehow stopped working on the Twitter iOS App, preventing the interception and decryption of the App's HTTPS traffic for reverse-engineering purposes. Twitter was actually one of the first iOS Apps to implement SSL pinning, and I remember using it as a test App when I started working on the first version of SSL Kill Switch, a few years ago.

I finally took the time to investigate this issue and just released [v0.10 of SSL Kill Switch][ssl-kill-switch-dl], which will work on Twitter iOS again (and any [CocoaSPDY][cocoa-spdy] App).


### Why it stopped working

SSL Kill Switch broke when Twitter updated their iOS App to use [the SPDY protocol][spdy-post] to replace HTTPS when connecting to its server endpoints. This explains why this specific App was not affected by SSL Kill Switch, although the tool was able to disable SSL pinning in every other App on the Store and even Apple daemons and services.

Twitter open-sourced [their SPDY implementation][cocoa-spdy] for iOS/OS X, CocoaSPDY, which helped me figure out what wasn't working.


### Act 1: CocoaSPDY and SSL Validation

At a very high level, SSL Kill Switch disables SSL validation and pinning [by preventing Apps from being able to set the _SecureTransport_ callback for validating the server's certificate chain][ssl-kill-switch-post]; it instead sets a callback that skips all SSL validation, making it easy to intercept and decrypt the App's HTTPS traffic using a proxy like Burp or Charles.

However, CocoaSPDY does not rely on this callback mechanism, and instead always retrieves and validates the server certificate [at the end of the TLS handshake][spdy-validation], using the _CFStream_ APIs:
    
    - (void)_onTLSHandshakeSuccess
    {
        [...]
            bool acceptTrust = YES;
            [...]
            // Get the server's certificate chain
            SecTrustRef trust = (SecTrustRef)CFReadStreamCopyProperty(_readStream, kCFStreamPropertySSLPeerTrust);
            // Validate the certificate chain
            acceptTrust = [_delegate socket:self securedWithTrust:trust];
        [...]
            if (!acceptTrust) {
                // Close the connection if the validation failed
                [self _closeWithError:SPDY_SOCKET_ERROR(SPDYSocketTLSVerificationFailed, @"TLS trust verification failed.")];
                return;
            }

This is why SSL Kill Switch did not affect CocoaSPDY (and Twitter iOS) at all and a new strategy was needed in order to disable SSL validation. Unfortunately, I did not find a way to disable SSL validation in a generic fashion, for code that validates the certificate by retrieving it with `kCFStreamPropertySSLPeerTrust`. Hence, I had to implement a CocaSPDY-specific workaround. 

Within the library, certificate validation can be customized by creating a class that conforms to the `SPDYTLSTrustEvaluator` Objective-C protocol, and setting an instance of this class as `SPDYProtocol`'s trust evaluator, [using the `+ setTLSTrustEvaluator:` method][spdy-trust]. Then, all SPDY connections call into this trust evaluator when the server's certificate chain needs to be validated.

To disable this mechanism, I simply had to update SSL Kill Switch override the `+ setTLSTrustEvaluator:` method to force a nil trust evaluator (which CocoaSPDY sees as a trust-all evaluator):
    
    // Will contain the original setTLSTrustEvaluator method
    static void (*oldSetTLSTrustEvaluator)(id self, SEL _cmd, id evaluator);

    // Our replacement method for setTLSTrustEvaluator
    static void newSetTLSTrustEvaluator(id self, SEL _cmd, id evaluator)
    {
        // Set a nil evaluator to disable SSL validation
        oldSetTLSTrustEvaluator(self, _cmd, nil);
    }
    [...]
    
    // SSL Kill Switch's initialization function
    __attribute__((constructor)) static void init(int argc, const char **argv)
    {
        [...]
        // Is CocoaSPDY loaded in the App ? 
        Class spdyProtocolClass = NSClassFromString(@"SPDYProtocol");
        if (spdyProtocolClass)
        {
            // Disable trust evaluation
            MSHookMessageEx(object_getClass(spdyProtocolClass), NSSelectorFromString(@"setTLSTrustEvaluator:"), (IMP) &newSetTLSTrustEvaluator, (IMP *)&oldSetTLSTrustEvaluator);
            
This ensured that CocoaSPDY connections would not validate SSL certificates as soon as SSL Kill Switch was loaded in the App. However, I ran into another problem as I was still not able to intercept Twitter's HTTPS traffic using the Burp proxy.


### Act 2: SPDY Proxy-ing

After disabling SSL validation, I was still not able to see the traffic in Burp for a very simple reason: Burp does not support the SPDY protocol. Although I later realized that there are other proxy tools that do support SPDY (such as [SSLSplit][sslsplit]), I wanted to use Burp for what I needed to do. 

Luckily, I noticed that CocoaSPDY is implemented in a way that makes it very easy to deploy SPDY within any App: it leverages the powerful [`NSURLProtocol`][nsurlprotocol] API in order to transparently transform the App's outgoing HTTPS requests into SPDY requests. Hence, all it takes to turn any iOS App into a SPDY App is a few lines of code:
    
    // For NSURLConnection
    [SPDYURLConnectionProtocol registerOrigin:@"https://api.twitter.com:443"];

    // For NSURLSession
    NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    configuration.protocolClasses = @[[SPDYURLSessionProtocol class]];

This is very neat, and also gave me the idea to just disable these initialization routines in SSL Kill Switch in order to turn the App's traffic back into HTTPS, which Burp obviously supports. For example, this is how SSL Kill Switch downgrades NSURLSession connections from SPDY to HTTPS:

    static void (*oldSetprotocolClasses)(id self, SEL _cmd, NSArray <Class> *protocolClasses);
    static void newSetprotocolClasses(id self, SEL _cmd, NSArray <Class> *protocolClasses)
    {
        // Do not register protocol classes which is how CocoaSPDY works
        // This should force the App to downgrade from SPDY to HTTPS
    }
    [...]

    // SSL Kill Switch's initialization function
    __attribute__((constructor)) static void init(int argc, const char **argv)
    {
        [...]                
        MSHookMessageEx(NSClassFromString(@"NSURLSessionConfiguration"), NSSelectorFromString(@"setprotocolClasses:"), (IMP) &newSetprotocolClasses, (IMP *)&oldSetprotocolClasses);
        [...]

After disabling CocoaSPDY's initialization methods in SSL Kill Switch, I was finally able to use Burp to intercept Twitter iOS' network traffic:

![](/images/posts/killswitch-twitter.png)
    

### Try it out!

You can get the latest version of SSL Kill Switch on [the project's page][ssl-kill-switch-dl]. 
    
    
[ssl-kill-switch-gh]: https://github.com/nabla-c0d3/ssl-kill-switch2
[spdy-post]: https://www.chromium.org/spdy/
[cocoa-spdy]: https://github.com/twitter/CocoaSPDY
[ssl-kill-switch-post]: /blog/2013/08/20/ios-ssl-kill-switch-v0-dot-5-released/
[spdy-validation]: https://github.com/twitter/CocoaSPDY/blob/cc92e49759c2c906d0683f1ccd8f48cd7c3836c4/SPDY/SPDYSocket.m#L1776
[spdy-trust]: https://github.com/twitter/CocoaSPDY/blob/cc92e49759c2c906d0683f1ccd8f48cd7c3836c4/SPDY/SPDYProtocol.m#L204
[sslsplit]: https://www.roe.ch/SSLsplit
[nsurlprotocol]: https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSURLProtocol_Class/index.html
[ssl-kill-switch-dl]: https://github.com/nabla-c0d3/ssl-kill-switch2/releases

