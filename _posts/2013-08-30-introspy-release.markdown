---
layout: post
title:  "Blackbox iOS App Analysis with Introspy"
date:   2013-08-30 11:35:05
categories: ios
---

_Note: this article was originally posted on [iSEC Partners’ blog][isec-blog]._

### Blackbox iOS Pentesting

In 2013, assessing the security of iOS applications still involves a lot of manual, time-consuming tasks - especially when performing a black-box assessment. Without access to source code, a comprehensive review of these applications currently requires in-depth knowledge of various APIs and binary
formats, as well as the ability to use relatively complex, rather generic tools such as Cycript, Mobile Substrate, or a debugger.


### Introspy

To simplify this process, [a coworker][tdaniels] and I are releasing [Introspy][introspy-page] - an open-source application profiler for iOS. Introspy is designed to help penetration testers understand how an application functions at runtime by swizzling system APIs to gather detailed information on each of the calls that can then be analyzed offline.

The tool comprises two separate components: the injected dylib that performs the swizzling and traces calls to the system APIs, and an offline analysis tool for mining the traced calls. The iOS tracer can be easily installed on a jailbroken iOS device and configured via the main Settings application. The tracer hooks security-sensitive APIs called by a given application and records the calls in a database. These include function calls related to cryptography, IPC, data storage / protection, networking, and user privacy. The full list of traced calls can be found on the [project wiki][introspy-wiki]. The introspy analyzer can then extract the database of traced calls off of the device and generate an HTML report displaying all recorded calls, plus a list of potential vulnerabilities affecting the application. Additionally, there are a myriad of options to allow quick and dirty analysis from the command-line.


### Project page

We have been using this tool internally with much success for a period of time and have decided to share it with the community with the hope that it will lead to better black-box security analysis and ultimately more secure iOS applications. For a quick overview read our [GitHub page][introspy-page] or checkout the [GitHub repository][introspy-gh] for full details on how to install and configure the tool.


[isec-blog]: https://www.isecpartners.com/blog.aspx
[tdaniels]: https://twitter.com/mitchbreathely/
[introspy-page]: http://isecpartners.github.io/Introspy-iOS/
[introspy-wiki]: https://github.com/iSECPartners/Introspy-iOS/wiki
[introspy-gh]: https://github.com/iSECPartners/Introspy-iOS/
