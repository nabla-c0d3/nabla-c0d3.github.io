---
layout: post
title: "SSL Kill Switch v0.8 Released"
date: 2015-10-29 13:02:06 -0700
post_author: Alban Diquet
categories: ios
---

With the unexpectedly quick release of an [iOS 9 jailbreak][pangu-9], users reported a [boot loop issue][boot-loop-gh] when installing SSL Kill Switch. I think this was caused by underlying changes to the OS, rather than the tweak itself. Saurik subsequently updated Cydia Substrate and apparently made some changes to [MobileLoader][mobile-loader], the tweak injection mechanism, as SSL Kill Switch stopped working with following error in the logs:

    MS:Error: extension does not have filter

As opposed to previous versions of Cydia Substrate, it seems like tweaks now __must__ provide a [plist filter][mobile-loader] or they won't be loaded into any processes. Hence, I added the following filter for SSL Kill Switch:

    Filter = {
      Bundles = (com.apple.UIKit);
    };

This fixed the issue, with the side-effect that SSL Kill Switch will no longer be injected into processes that don't link _UIKit_ (ie. processes with no UI, such as system daemons like _itunesd_). If you need to use SSL Kill Switch in a daemon, you will need to modify the filter first and then recompile the tweak.

Go to the project's [Releases page][killswitch-dl] to download the new version.

[pangu-9]: http://en.pangu.io/
[boot-loop-gh]: https://github.com/nabla-c0d3/ssl-kill-switch2/issues/5
[mobile-loader]: http://www.iphonedevwiki.net/index.php/Cydia_Substrate#MobileLoader
[killswitch-dl]: https://github.com/nabla-c0d3/ssl-kill-switch2/releases
