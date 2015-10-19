---
layout: post
title: "TrustKit, iOS 9 And The Shared Cache"
date: 2015-10-17 13:02:06 -0700
post_author: Alban Diquet
categories: ios
---

In August, Angela Chow, [Eric Castro][ec-twitter] and myself released [TrustKit][trustkit-gh] at the [Black Hat conference][trustkit-bh]: an iOS & OS X library designed to make it very easy to deploy SSL pinning within an App. 

When Apple released iOS 9 last month, it broke TrustKit; this post explains the behind-the-scene change that caused this and why it affected TrustKit.

### TrustKit on iOS 8

As explained in our [Black Hat talk][bh-slides], TrustKit worked by patching _SecureTransport_'s `SSLHandshake()` function at runtime. During the SSL handshake, this function gets called multiple times until the handshake is completed.
Throughout this calls, TrustKit forwarded them to the real `SSLHandshake()`, until it returned `noErr`, which means the handshake was succesful. 

When that specific return value was detected and before returning it to the caller, TrustKit performed SSL pinning validation and changed the return value to an error if the validation had failed.

Here is how it looked like in TrustKit:

    // Our replacement function for SSLHandshake()
    static OSStatus replaced_SSLHandshake(SSLContextRef context)
    {
        // First let's forward the call to the real SSLHandshake()
        OSStatus result = original_SSLHandshake(context);
        if (result == noErr)
        {
            // The handshake was sucessful
            // Let's perform SSL pinning validation
            
            // code starts here...
            // <code>
            // ...code ends here
        }
        return result;

Because every Apple Framework (_NSURLConnection_, _NSURLSession_, _UIWebView_, etc.) relies on _SecureTransport_ for SSL, patching `SSLHandshake()` ensured that all of the App's outgoing connections were automatically protected.

![](/images/posts/trustkit-network-stack.png)

To do the actual hooking of `SSLHandshake()`, we used [Facebook's fishhook][fishhook-gh], which "provides functionality that is similar to using _DYLD\_INTERPOSE_ on OS X."

### What Changed in iOS 9

When iOS 9 was released, I received reports that TrustKit wasn't working anymore, which was surprising to me as I had done some testing myself to ensure it worked fine.

I did some further investigation and discovered something very surprising: TrustKit would work fine on specific devices (such as the iPhone 4S or the iPad 3) but not on other devices (such as the iPhone 6), where TrustKit was successfully getting initialized but no pinning validation would actually be done.

This behavior implied that the hooking of `SSLHandshake()` was not actually working, although fishhook had been updated for iOS 9. After quite a lot of testing and googling, I luckily found the answer within a tweet from [@comex][comex-twitter] from a few months back, talking about the beta of iOS 9:

![](/images/posts/trustkit-comex-tweet.png)

To understand what this means, we first have look at how fishhook works. At a high level, fishhook modifies at runtime the process' symbol table to have the symbol you want to hook point to your own code, instead of the original C function. Then, when any code wants to call that function, it looks up its symbol within the symbol table, which then points to your replacement function. Some more technical details about this symbol rebinding technique are available on the [project's page][fishhook-gh].

The iOS 9 change [@comex][comex-twitter] described means that any code within the [dyld shared cache][shared-cache] will actually skip the symbol table and jump straight to the function itself, when calling another function in the cache. This was probably done as a performance optimization to skip the symbol lookup step.

Consequently, modifying the symbol table doesn't affect calls happening within the shared cache (such as a method within `NSURLConnection` calling _SecureTransport_'s `SSLHandshake()`, as both frameworks live in the shared cache).

This explains why TrustKit stopped working on iOS 9:

* Calls happening across Apple frameworks and libraries can no longer be hooked at all, meaning that TrustKit's `SSLHandshake()` hook would only be triggered if the App itself directly calls this function. 
* This optimization can be enabled by Apple when compiling the shared cache, which is why TrustKit stopped working only on specific devices. It seems like the iOS 9 image for the iPhone 4S has the optimization disabled, while the one for the iPhone 6 has it enabled.

Unfortunately, this also means that some of the techniques we presented at the Black Hat talk for writing secure libraries through function hooking do not work on iOS 9. This also affects other Apps and products as I have been in contact with developers who faced the exact same issue.

### What about Cydia Substrate and tweaks?

If you're developing tweaks for jailbroken iOS and hooking C functions using Substrate, this change will not affect your code. 

The reason for this is that, as explained in our talk, Substrate takes a different approach for hooking C functions: it actually modifies the instructions at the beginning of the function you want to hook, in order to insert some trampoline code to jump to your replacement function. Of course, this technique can only work on jailbroken devices (as it requires RWX memory pages). Therefore, Substrate's hooking implementation will work regardless of who is calling the function (an Apple framework or the actual App).

### What's next for TrustKit ?

I will release TrustKit 1.2.0 for iOS 9 in the next few days, which has a completely different implementation for "hooking" into the App's SSL connections to add pinning validation. I had to take the more conventional (but not as neat) approach of swizzling the App's _NSURLConnection_ and _NSURLSession_ delegates.

In the release blog post which will come soon, I will provide more details about what has changed in TrustKit 1.2.0 and how to deploy it in your App; this version also brings some significant improvements over 1.1.3.


[shared-cache]: http://www.iphonedevwiki.net/index.php?title=Dyld_shared_cache
[comex-twitter]: https://twitter.com/comex
[bh-slides]: https://github.com/datatheorem/TrustKit/raw/master/docs/TrustKit-BH2015.pdf
[ec-twitter]: https://twitter.com/_eric_castro
[trustkit-gh]: https://github.com/datatheorem/TrustKit/
[fishhook-gh]: https://github.com/facebook/fishhook/
[trustkit-bh]: https://www.blackhat.com/us-15/briefings.html#trustkit-code-injection-on-ios-8-for-the-greater-good
