---
layout: post
title: "SOS Reward for SecureDrop Work"
date: 2022-06-15 11:30:06 -0700
post_author: Alban Diquet
categories: appsec
---

Earlier this year, I made a submission to the [Secure Open Source Rewards program](https://sos.dev/), for some code contributions I had made to the [SecureDrop server code](https://github.com/freedomofpress/securedrop). My submission got accepted and I received a [$10,000 reward](https://sos.dev/#reward-amounts) for my work.

The SOS Rewards program is described as follow: 

> The Secure Open Source Rewards pilot program financially rewards developers for enhancing the security of critical open source projects that we all depend on. The pilot program is run by the Linux Foundation with initial sponsorship from the Google Open Source Security Team (GOSST).

If you are a developer and have made contributions to "proactively harden critical open source projects and supporting infrastructure against application and supply chain attacks", you should definitely submit your work to the program!

The [program's scope](https://sos.dev/#what-projects-are-in-scope) is broad and very different kinds of contributions might get accepted:

* Any open-source project that's popular-enough could be valid as a "critical open source project and supporting infrastructure". As part of my submission, the SecureDrop project did qualify.
* The program is not focused on security vulnerabilities. Instead, code-level contributions (refactors, test suite improvements, etc.) are what's on scope. Hence, contributions that are not specifically about fixing a vulnerability will qualify, and developers who may not have a background in security can still make useful contributions and get rewarded.

What follows is what I submitted to the program for my work on SecureDrop, as an example of a submission that got accepted.

### My Submission: Criticality

> This rewards program is limited to critical open source projects. What makes an open source project critical? It should be a popular and widely used project that has a critical impact on infrastructure and user security. Projects that come to mind are popular web frameworks or libraries, decompression libraries, crypto libraries, mail servers, databases, network services, and security or toolchain dependencies of any critical projects themselves. In the response below, please explain in as many words as you feel are needed why this project is critical.

SecureDrop is an open-source whistleblower submission system used by more than 100 news organizations in the world, including the Guardian and the New York Times. It allows whistleblowers to securely and anonymously send documents to journalists.

While this project may not be a web framework or library, I believe it does match the requirement of a "widely used project that has a critical impact on infrastructure and user security". The project is used by prominent news organizations and NGOs all around the world, and has been instrumental in uncovering important news stories exposing corruption and crime.

### My Submission: Tell us more about the work

> Please tell us what the improvement is and explain how it works, its complexity, and the security impact (including links to CLs). If the improvement required a lot of effort to complete, tell us why in detail. Include any information that may convince us that the improvement has a demonstrable, significant, and proactive impact on security. If this submission is similar to a previous submission, please let us know and tell us how this one is different.

The work was to review and significantly refactor the code in SecureDrop responsible for encrypting documents submitted by whistleblowers to journalists. The goal was to make this code simpler and more readable, improve type-checking, and make its test suite more comprehensive. While working on this, I discovered a minor security issue in the encryption code, fixed it as part of the work, and added some unit tests to prevent future regressions.

My changes were released as part of SecureDrop version 2.2.0 (entry #6174 in https://github.com/freedomofpress/securedrop/blob/release/2.2.0/changelog.md).

I split my changes into three Pull Requests on GitHub, which contain more details about the changes:

* Part 1/3 https://github.com/freedomofpress/securedrop/pull/6174 , which fixes the security issue.
* Part 2/3 https://github.com/freedomofpress/securedrop/pull/6184 , which simplifies some fixtures in the test suite.
* Part 3/3 https://github.com/freedomofpress/securedrop/pull/6160 , which simplifies the encryption code.

More details about the changes follow.

The encryption logic in securedrop leverages the GPG binary, and is called from Python; it was a very complex and confusing piece of code. This complexity:

* Made it easy for SecureDrop developers to make mistakes when calling or modifying the encryption code.
* Made it difficult for both security auditors and automated tools to review/check the code. 
Additionally, any bug in the encryption code could have a devastating impact on the overall security of the application, and hence the users/whistleblowers. This is why I decided to work on significantly simplifying it.

While working on this refactoring, I built a new test suite that I designed to be more comprehensive than the existing one. This test suite uncovered a minor security issue related to how the GPG binary works when used by the SecureDrop server:

* Each whistleblower that wants to submit documents has a GPG passphrase attached to their SecureDrop account, and stored on the SecureDrop server.
* Because of how the GPG agent was configured, GPG passphrases were automatically cached by the GPG binary, causing decryption operations to succeed even if the wrong GPG passphrase was supplied in the SecureDrop code.
* This issue was un-exploitable by itself. If an attacker were able to discover another vulnerability in the server code, they could trick the SecureDrop server into decrypting another user's documents. However, after reviewing the server code with the SecureDrop team, no such vulnerability was found, making the initial issue un-exploitable in the current code base.
* There are more details about the issue in the first PR at https://github.com/freedomofpress/securedrop/pull/6174 .
