---
layout: post
title: "Using tcprelay.py With Multiple Devices"
date: 2014-12-30 12:02:06 -0700
post_author: Alban Diquet
categories: ios
---

The tcprelay.py and usbmux.py Python scripts from the [usbmuxd project][usbmuxd-gh] make it possible to SSH into an iOS device over USB. This makes it easy to interact with a device without having to rely on WiFi, as described [here][ssh-wiki].

I have modified tcprelay.py to allow forwarding SSH connections to multiple devices connected via USB, instead of just one. The device's udid can be specified using the -u option:

    python tcprelay.py -t 22:5000 -u ddbc43f3247126671d2c7c279723cb2f9f93ace1

 If -u isn't supplied, tcprelay.py will use the first device available. See the [project page][project-page] for more details.

[usbmuxd-gh]: https://github.com/libimobiledevice/usbmuxd
[project-page]: https://github.com/nabla-c0d3/tcprelay
[ssh-wiki]: http://iphonedevwiki.net/index.php/SSH_Over_USB
