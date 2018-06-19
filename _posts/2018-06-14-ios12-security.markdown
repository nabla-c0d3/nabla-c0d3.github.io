---
layout: post
title: "Security and Privacy Changes in iOS 12"
date: 2018-06-14 11:30:06 -0700
post_author: Alban Diquet
categories: ios 
---

This year and for the first time, I actually went to the Apple WWDC conference, in San Jose. The conference was quite interesting, and gave me the opportunity to meet some of the members of the Apple security team.

Here are some notes about the security and privacy changes brought by iOS 12 that I thought were interesting.

## Automatic strong passwords

The ["Automatic Strong Passwords and Security Code AutoFill" session](https://developer.apple.com/videos/play/wwdc2018/204/) describes various enhancements made to the iOS built-in password management functionality.

### Automated password generation

Starting with iOS 11, developers can "label" the username and password field in their app's login screen:

```swift
let userTextField = UITextField()
userTextField.textContentType = .username

let passwordTextField = UITextField()
passwordTextField.textContentType = .password
```

This allows iOS to automatically login the user with the credentials they previously saved in their iCloud account, if the app has been properly [associated with its web domains](https://developer.apple.com/documentation/security/password_autofill/setting_up_an_app_s_associated_domains).

With iOS 12, a strong password can automatically be generated and stored when creating a new account in an app or in Safari. This functionality can be enabled by using the `.username` and the iOS 12 `.newPassword` content types in your "Create Account" screen:

```swift
let userTextField = UITextField()
userTextField.textContentType = .username

let newPasswordTextField = UITextField()
newPasswordTextField.textContentType = .newPassword

let confirmNewPasswordTextField = UITextField()
confirmNewPasswordTextField.textContentType = .newPassword
```

iOS 12 will then prompt the user to automatically generate a strong password during the account creation flow.

Any password that was automatically generated will contain upper-case, digits, hyphen, and lower-case characters. If your backend has limitations in the characters it allows in passwords, you can define a custom password rule in your app via [the `UITextInputPasswordRules` API](https://developer.apple.com/documentation/uikit/uitextinputpasswordrules):

```swift
let newPasswordTextField = UITextField()

...

let rulesDescriptor = "allowed: upper, lower, digit; required: [$];" 
newPasswordTextField.passwordRules = UITextInputPasswordRules(descriptor: rulesDescriptor)
```

Apple has also released [an online tool](https://developer.apple.com/password-rules/) to help with writing password rules.

Similar labels can be used in a web page that Safari can leverage:

![Autofill](/images/posts/ios12-autofill.png)

### Automated 2FA SMS codes input

iOS 12 also introduces a content type for the text field that will receive 2 factor authentication codes received via SMS:

```swift
let securityCodeTextField = UITextField()
securityCodeTextField.textContentType = .oneTimeCode
```

Enabling this content type allows iOS to automatically fill in a 2FA code previously received via SMS (with a user prompt), which is pretty cool:

![Autofill](/images/posts/ios12-2fa-label.png)

### Federated authentication

iOS 12 introduces a new [`ASWebAuthenticationSession` API](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession) for automatically handling an OAuth login flow.

Given an OAuth URL (ie. where to start the authentication flow), the API will:

* Direct the user to the OAuth provider's authentication page.
* Have the user then log into their account on the provider's page. As the API uses the same cookie store as Safari, the user may already be logged into their account; if that's the case, the user will be prompted to confirm that they want to re-use their existing session in Safari, making the flow really quick. 
* Allow the user to review the OAuth permissions requested by your app, and grant access via the OAuth authorization prompt.
* Return the user to your app and provide the callback URL, which contains the user's authentication token if the flow was successful.

It was stated during the WWDC presentation that `ASWebAuthenticationSession` is now the "go-to way to implement federated authentication and it replaces [`SFAuthenticationSession`](https://developer.apple.com/documentation/safariservices/sfauthenticationsession)", which was deprecated in iOS 12.

### Credential Provider Extension

All the password management improvements described above apply to the built-in password manager in iOS and Safari, the "iCloud KeyChain". However, third-party password manager applications (such as LastPass, 1Password, etc.) can also get integrated into the password flows on iOS, via a new extension point called "Credential Provider Extension".

![Credential Provider Extension](/images/posts/ios12-autofill-extension.png)

This extension point and the corresponding APIs are all part of [the new `AuthenticationServices` framework](https://developer.apple.com/documentation/authenticationservices) available on iOS 12. This framework allows providing a UI for user to choose their password when authenticating into an app, storing a newly-created password, etc.

The framework is described in details in the ["Implementing AutoFill Credential Provider Extensions"](https://developer.apple.com/videos/play/wwdc2018/721/) presentation.

## Secure object de-serialization

At WWDC this year, a whole presentation was dedicated to secure object serialization and de-serialization: ["Data You Can Trust"][data-you-can-trust].

When an application has some logic to receive arbitrary data (for example over the Internet) and to then de-serialize the data into an object, care must be taken when implementing this logic. Specifically, if the raw data can choose any arbitrary class as the the object it gets de-serialized to, this can lead to remote code execution. This type of vulnerability affects almost every language and framework, for example [Apache and Java](https://fishbowl.pastiche.org/2015/11/09/java_serialization_bug/), [Ruby on Rails](https://codeclimate.com/blog/rails-remote-code-execution-vulnerability-explained/), or [Python's pickle module](https://blog.nelhage.com/2011/03/exploiting-pickle/).

In iOS applications, object (de-)serialization is usually implemented using:

* [The `NSCoding` protocol](https://developer.apple.com/documentation/foundation/nscoding), which allows the developer to implement the serialization logic for their own classes.
* [The `NSKeyedArchiver` class](https://developer.apple.com/documentation/foundation/nskeyedarchiver) which takes an object that implements `NSCoding` and serializes it into a specific file format called an archive, which can then be stored for example on the file system. [The `NSKeyedUnarchiver` class](https://developer.apple.com/documentation/foundation/nskeyedunarchiver) can then be used to de-serialize the object.

This approach is vulnerable to the issue described above, referred to as an "object substitution attack" in the Apple documentation: the data gets de-serialized to a different object than what was expected by the developer.

To prevent such attacks, the following APIs were introduced in iOS 6:

* `[NSKeyedArchiver decodeObjectOfClass:forKey:]`, which allows the developer to pick the class the data will get de-serialized to before it occurs, thereby preventing object substitution attacks.
* The [`NSSecureCoding` protocol][nssecurecoding] which extends the `NSCoding` protocol by adding the class method `supportsSecureCoding:`, in order to ensure that the developer is using the safe `-decodeObjectOfClass:forKey:` method to handle object serialization and de-serialization in their classes.

The ["Data You Can Trust"][data-you-can-trust] presentation this year heavily emphasized `NSSecureCoding` and `-decodeObjectOfClass:forKey:`:

![Secure Coding](/images/posts/ios12-nssecurecoding.png)

What has changed with iOS 12 is that `[NSKeyedArchiver init]` constructor is now deprecated; this was done to get developers to switch to the [`[NSKeyedArchiver initRequiringSecureCoding:]`](https://developer.apple.com/documentation/foundation/nskeyedarchiver/2962881-initrequiringsecurecoding) constructor instead, which has been made public on iOS 12 (but seems to be retro-actively available in the iOS 11 SDK). This constructor creates an `NSKeyedArchiver` that can only serialize classes that conform to the `NSSecureCoding` protocol, ie. objects that are safe to serialize and de-serialize.

## The Network framework

[The Network framework](https://developer.apple.com/documentation/network), introduced somewhere around iOS 9 or 10 has become a public API on iOS 12.

It is a modern implementation of a low-level networking/socket API. As stated in the documentation, it is meant to replace all the other low-level networking APIs available on iOS: BSD sockets, [SecureTransport](https://developer.apple.com/documentation/security/secure_transport) and [CFNetwork](https://developer.apple.com/documentation/corefoundation/cfsocket):

> "Use this framework when you need direct access to protocols like TLS, TCP, and UDP for your custom application protocols. Continue to use NSURLSession, which is built upon this framework, for loading HTTP- and URL-based resources."

![Network](/images/posts/ios12-network.png)

More details about the Network framework are available in the ["Introducing Network.framework: A modern alternative to Sockets"](https://developer.apple.com/videos/play/wwdc2018/715/) presentation.

Developers should expect the legacy network APIs (SecureTransport, etc.) to eventually get deprecated by Apple. Right now and as mentioned in the presentation, they are "discouraged APIs".

The Network framework also comes with its own set of [symbols for handling TLS connections](https://developer.apple.com/documentation/network/security_symbols), such as certificates, identities, and trust objects. They mirror the legacy SecureTransport symbols and can be used interchangeably. For example, a `SecCertificateRef`, which represents an X.509 certificate, is a `sec_certificate_t` in the Network framework. The [`sec_certificate_create()` function](https://developer.apple.com/documentation/security/2976298-sec_certificate_create) can be used to turn a `SecCertificateRef` into a `sec_certificate_t`.

Lastly, App Transport Security is not enabled for connections using the Network framework, but will be "soon" according to Apple engineers.

## Other changes

### Deprecation of UIWebView

Starting with iOS 12, the [`UIWebView API`](https://developer.apple.com/documentation/uikit/uiwebview) is now officially deprecated. Developers that need web view functionality in their application should switch to the [`WKWebView API`][wkwebview], which is a massive improvement over `UIWebView` in every aspect (security, performance, ease of use, etc.).

### Unified Random Number Generation in Swift 4.2

Swift 4.2 introduces an API for generating random number, described in the ["Random Unification" proposal](https://github.com/apple/swift-evolution/blob/master/proposals/0202-random-unification.md).

Previously, generating random numbers in Swift was done by importing C functions that are insecure in most cases (such as `arc4random()`).

### Enforcement of Certificate Transparency

Apple will be enforcing Certificate Transparency at the end of 2018 across all TLS connections on iOS. This does not require any changes in your application as the work to deploy CT has been carried out by the Certificate Authorities. More details are available in the ["Certificate Transparency policy" article](https://support.apple.com/en-us/HT205280).

The CT enforcement will be deployed "with a software update later this year".

### Enforcement of App Transport Security

With iOS 9, Apple introduced App Transport Security (ATS), a security feature which by default requires all of an app's connections to be encrypted using SSL/TLS. When ATS was first announced, it was going to be mandatory for any app going through the App Store Review process, starting on January 1st 2017. Apple later [cancelled the deadline](https://www.zdnet.com/article/apple-extends-developer-deadline-for-apps-to-use-https-secure-connections/), and no further announcements about requiring ATS have been made.

However, at WWDC this year, I learned that Apple has started reaching out to specific apps through the App Store review process, in order to ask for justifications and/or require the applications' ATS policy to be stricter, especially having `NSAllowsArbitraryLoads` (the exemption that fully disables ATS) set to `NO`.

![ATS](/images/posts/ios12-ats.png)

## So what should I do?

If you are a developer, here is a summary of the changes to implement in your application, based on all the iOS 12 features described in this article. 

### Short term

* Add support for Password Autofill to your application. The ["Enabling Password AutoFill on a Text Input View" article](https://developer.apple.com/documentation/security/password_autofill/enabling_password_autofill_on_a_text_input_view) gives a good summary of the changes you need to implement in your app.
* If your application's ATS policy still sets `NSAllowsArbitraryLoads` to `YES`, modify your policy by adding the required exemptions and domains, in order to be able to set `NSAllowsArbitraryLoads` to `NO`. More details on how to achieve this are available in [our ATS guide][ats-guide]. Sooner or later, your application will get blocked if it enables `NSAllowsArbitraryLoads`.

### Medium term

* If your application is using `NSCoding` for object serialization, switch to [`NSSecureCoding`][nssecurecoding] in order to prevent object substitution attacks.
* If you application is using the now-deprecated`UIWebView` API, switch to [`WKWebView`][wkwebview].

#### Long term

* If your application is using a low-level network API (such as BSD sockets, SecureTransport or CFNetwork) switch to the [`Network` framework][network].

[data-you-can-trust]: https://developer.apple.com/videos/play/wwdc2018/222/
[ats-guide]: /ios/ssl/2016/08/14/ats-enforced-2017/
[nssecurecoding]: https://developer.apple.com/documentation/foundation/nssecurecoding
[wkwebview]: https://developer.apple.com/documentation/webkit/wkwebview
[network]: https://developer.apple.com/documentation/network
