---
layout: post
title: "SSLyze v1.0.0 Released"
date: 2017-02-08 13:02:06 -0700
post_author: Alban Diquet
categories: ssl
---

I just released a new version of SSLyze, my Python tool/library which can analyze the SSL configuration of a server by connecting to it and detect various issues (bad certificates, dangerous cipher suites, lack of session resumption, etc.). 

After almost 20 releases over the past 6 years, SSLyze is now at version 1.0.0. This is a major release as I have completed a significant refactoring of SSLyze's internals,in order to clean its Python API. The API should be considered stable and is now [fully documented][sslyze-documentation]! 

Using SSLyze as a Python module makes it easy to implement SSL/TLS scanning as part of continuous security testing platform, and detect any misconfiguration across a range of public and/or internal endpoints.


### Sample Code

This sample code scans the `www.google.com:443` endpoint to detect the list of SSL 3.0 cipher suites it accepts, whether it supports secure renegotiation, and will also review the server's certificate:

    # Ensure we can connect to the server
    server_info = ServerConnectivityInfo(hostname=u'www.google.com')
    server_info.test_connectivity_to_server()

    concurrent_scanner = ConcurrentScanner()

    # Queue some scan commands
    concurrent_scanner.queue_scan_command(server_info, Sslv30ScanCommand())
    concurrent_scanner.queue_scan_command(server_info, SessionRenegotiationScanCommand())
    concurrent_scanner.queue_scan_command(server_info, CertificateInfoScanCommand())

    # Process the results
    reneg_result = None
    for plugin_result in concurrent_scanner.get_results():
        # All scan results have the corresponding scan_command and server_info as an attribute
        print(u'\nReceived scan result for {} on host {}'.format(scan_result.scan_command.__class__.__name__,
                                                                scan_result.server_info.hostname))

        # Sometimes a plugin command can unexpectedly fail (as a bug); it is returned as a PluginRaisedExceptionResult
        if isinstance(plugin_result, PluginRaisedExceptionScanResult):
            print(u'Scan command failed: {}'.format(plugin_result.as_text()))

        # Each plugin result has attributes with the information you're looking for, specific to each plugin
        # All these attributes are documented within each plugin's module
        if isinstance(plugin_result.scan_command, Sslv30ScanCommand):
            # Do something with the result
            print(u'SSLV3 cipher suites')
            for cipher in plugin_result.accepted_cipher_list:
                print(u'    {}'.format(cipher.name))

        elif isinstance(plugin_result.scan_command, SessionRenegotiationScanCommand):
            reneg_result = plugin_result
            print(u'Client renegotiation: {}'.format(plugin_result.accepts_client_renegotiation))
            print(u'Secure renegotiation: {}'.format(plugin_result.supports_secure_renegotiation))

        elif isinstance(plugin_result.scan_command, CertificateInfoScanCommand):
            print(u'Server Certificate CN: {}'.format(
                plugin_result.certificate_chain[0].as_dict[u'subject'][u'commonName']
            ))


### Getting Started

The Python API can do a lot more than this (such as scanning StartTLS endpoints, connecting through a proxy, or enabling client authentication); head to the [project's page][sslyze-gh] or the [documentation][sslyze-documentation] for more information.



[sslyze-documentation]: https://nabla-c0d3.github.io/sslyze/documentation/
[sslyze-gh]: https://github.com/nabla-c0d3/sslyze
