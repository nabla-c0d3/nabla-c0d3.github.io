---
layout: post
title: "SSLyze 5.0.0 Released"
date: 2021-11-27 11:30:06 -0700
post_author: Alban Diquet
categories: ssl
---

I just released a new major version of SSLyze, a Python library for scanning the SSL/TLS configuration of a server: [SSLyze 5.0.0][sslyze-changelog].

This major release focuses on improving the reliability of the scans, simplifying the [Python API][sslyze-documentation] and JSON output, and adding support for checking a server's TLS configuration against Mozilla's recommended configuration. The full changelog is available [here][sslyze-changelog].

### Mozilla's recommended TLS configuration and CI/CD

In this new version, SSLyze will check the server's scan results against Mozilla's recommended ["intermediate" TLS configuration](https://wiki.mozilla.org/Security/Server_Side_TLS), and will return a non-zero exit code if the server is not compliant. 

```
$ python -m sslyze mozilla.com
```
```
Checking results against Mozilla's "intermediate" configuration. See https://ssl-config.mozilla.org/ for more details.

mozilla.com:443: OK - Compliant.
```

The Mozilla configuration to check against can be configured via `--mozilla-config={old, intermediate, modern}`:

```
$ python -m sslyze --mozilla-config=modern mozilla.com
```
```
Checking results against Mozilla's "modern" configuration. See https://ssl-config.mozilla.org/ for more details.

mozilla.com:443: FAILED - Not compliant.
    * certificate_types: Deployed certificate types are {'rsa'}, should have at least one of {'ecdsa'}.
    * certificate_signatures: Deployed certificate signatures are {'sha256WithRSAEncryption'}, should have at least one of {'ecdsa-with-SHA512', 'ecdsa-with-SHA256', 'ecdsa-with-SHA384'}.
    * tls_versions: TLS versions {'TLSv1.2'} are supported, but should be rejected.
    * ciphers: Cipher suites {'TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384', 'TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256', 'TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256'} are supported, but should be rejected.
```

This can be used to easily run an SSLyze scan as a CI/CD step.

### Maximum automation with GitHub Actions

After Travis CI announced that they were going to [restrict usage for open-source projects](https://blog.travis-ci.com/2020-11-02-travis-ci-new-billing), I decided to switch to GitHub Actions for SSLyze's CI/CD.

I have been very happy with the change; one of GitHub Actions' best feature is its seamless support for the three main OSes: Linux, macOS and Windows. Previously, supporting each of them would require using a different CI/CD product (such as AppVeyor for Windows). As SSLyze has a C component and needs to work on all three OSes, this has been a huge improvement.

Using GitHub Actions, I added a bunch of CI/CD workflows to reduce how much work an SSLyze release is. These workflows might be useful to other Python projects, especially the ones with a C extension:

![](/images/posts/sslyze-5-0-0-github-actions.png)

* [Running the unit tests](https://github.com/nabla-c0d3/sslyze/blob/release/.github/workflows/run_tests.yml).
* [Ensuring that SSLyze can be installed as a Python module](https://github.com/nabla-c0d3/sslyze/blob/release/.github/workflows/test_module_setup.yml), to prevent a "bad" release (missing file or setting from the setup.py, etc.).
* [Compiling the nassl C extension to binary wheels](https://github.com/nabla-c0d3/nassl/blob/release/.github/workflows/build_wheels.yml) for all supported platforms and Python versions. SSLyze uses nassl for its scans; it's an OpenSSL wrapper implemented as a C extension for Python. Automating this has been a huge time saver, as every release of nassl requires building one binary wheel per platform (Windows x86 and x64, macOS, Linux x86 and x64) and Python version.
* [Compiling SSLyze to a Windows executable](https://github.com/nabla-c0d3/sslyze/blob/release/.github/workflows/build_windows_exe.yml), which allows users to run SSLyze on Windows without installing Python.
* [Pushing a tagged Docker image to DockerHub](https://github.com/nabla-c0d3/sslyze/blob/release/.github/workflows/release_to_docker.yml) on every release; this workflow was actually implemented by a contributor.

### Further improving the reliability of scans

One important goal with SSLyze is to ensure that it is able to scan any web server without any issues. Most TLS scanning tools (including previous versions of SSLyze) will randomly crash when pointed at specific server stacks.

Improving the reliability of scans has been an ongoing effort and with the latest release, more automated testing in CI/CD has been implemented:

* Using GitHub Actions [to spawn a server and run a full SSLyze scan](https://github.com/nabla-c0d3/sslyze/blob/5.0.0/.github/workflows/scan_apache2_server.yml) against it, for the following web servers: Apache2, Microsoft IIS and nginx.
* Within the unit tests, running an SSLyze scan against Cloudflare and Google servers.

Based on [usage statistics of web servers](https://w3techs.com/technologies/overview/web_server), the combination of Apache2, Microsoft IIS, nginx, Cloudflare and Google represents ~93% of all web servers on the Internet.

Running SSLye scans from CI/CD on all these servers helps ensuring that it can reliably scan most of the Internet without any issues.

### More details and changelog

For more details, head to [the project's page][sslyze-changelog].

[sslyze-documentation]: https://nabla-c0d3.github.io/sslyze/documentation/
[sslyze-changelog]: https://github.com/nabla-c0d3/sslyze/releases/tag/5.0.0
