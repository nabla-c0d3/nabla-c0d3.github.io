---
layout: post
title: "SSLyze 3.0.0 Released"
date: 2020-04-05 11:30:06 -0700
post_author: Alban Diquet
categories: ssl
---

I just released a new version of SSLyze, a Python library for scanning the SSL/TLS configuration of a server: [SSLyze 3.0.0][sslyze-gh].

This has been a big effort and more than [60 000 lines of code](https://github.com/nabla-c0d3/sslyze/pull/418/files) were updated, with mainly two goals in mind:

* Making mass scans of hundreds or thousands of servers a lot more reliable.
* Making the Python API and the processing of the scan results simpler and easier.

These improvements make it a lot easier to use SSLyze as an automated SSL/TLS scanning tool, for example to continuously monitor and review the SSL configuration of your company's endpoints by running daily scans.

### Issues in previous versions

In previous versions of SSLyze, the scanning logic was often too aggressive with servers: it would open more than 20 or 30 concurrent connections, which would sometimes result in timeouts and failing connections, for servers that were not ready to handle this kind of sudden network load.When scanning a single server, running the scan again would sometimes do the trick, but that solution does not scale when running mass scans of hundreds of hosts.

This was made worse by the fact that the formatting of the scan results returned by SSLyze (both in Python and JSON) made it difficult to detect that a specific scan didn't work as expected. This could lead to the results being misinterpreted as "everything looks good" ie. SSL issues being missed.

### Making scanning more reliable

Starting with version 3.0.0, SSLyze enforces a maximum of 5 concurrent connections per server, regardless of the types of scan (cipher suites, Heartbleed) and the number of server to scans. This limit of 5 has been shown to provide a good balance between speed and success rate of the scans, and can also be lowered or increased as needed. Multiple servers are still scanned concurrently (to allow for speedy scans), but with this limit of 5 concurrent connections per individual server.

Implementing this logic required revisiting design decisions made almost a decade ago in the very first version of SSLyze. The code handling the concurrency was very complicated and used both multi-processing and multi-threading, as a naive way to speed up the scans and get around Python's [Global Interpreter Lock](https://realpython.com/python-gil/). Ultimately tho, SSLyze is an application that is mostly I/O-bound: its main functionality is to connect to servers and to send/receive data in order to test the servers' SSL configuration.

Because with I/O-bound programs the impact of the GIL on performance is tiny, I completely refactored the concurrency logic and removed any usage of the multi-processing module; everything is now done via threads using Python's modern API, the [ThreadPoolExecutor](https://docs.python.org/3/library/concurrent.futures.html). Removing the multiprocessing code also had the side-effect of speeding SSLyze's start time by half a second.

### Making the scan easier to run and process

Throughout the years, SSLyze has evolved from a command line tool to a fully-fledged Python library for SSL/TLS scanning. However, a library is only as good as its API: how easy and convenient it is to use it, in order to get the task at hand done.

In version 3.0.0, I have significantly simplified the Python API; starting a scan looks like this:

```python
# Define the server that you want to scan
server_location = ServerNetworkLocationViaDirectConnection.with_ip_address_lookup("www.google.com", 443)

# Do connectivity testing to ensure SSLyze is able to connect
try:
    server_info = ServerConnectivityTester().perform(server_location)
except ConnectionToServerFailed as e:
    # Could not connect to the server; abort
    print(f"Error connecting to {server_location}: {e.error_message}")
    return

# Then queue some scan commands for the server
server_scan_request = ServerScanRequest(
    server_info=server_info,
    scan_commands={ScanCommand.CERTIFICATE_INFO, ScanCommand.SSL_2_0_CIPHER_SUITES},
)
scanner = Scanner()
scanner.queue_scan(server_scan_request)
```

Any number of `ServerScanRequest` can be queued in order to scan multiple servers at the same time; all the available scan commands are documented [here][sslyze-scan-commands-doc]. The `Scanner` class will take care of running the scans concurrently while keeping the network load on each individual server low, in order to avoid any disruption.

Once all the `ServerScanRequest` have been queued, results of the scan can be retrieved as they get completed by doing the following:

```python
for server_scan_result in scanner.get_results():
    print(f"\nResults for {server_scan_result.server_info.server_location.hostname}:")

    # SSL 2.0 results
    ssl2_result = server_scan_result.scan_commands_results[ScanCommand.SSL_2_0_CIPHER_SUITES]
    print(f"\nAccepted cipher suites for SSL 2.0:")
    for accepted_cipher_suite in ssl2_result.accepted_cipher_suites:
        print(f"* {accepted_cipher_suite.cipher_suite.name}")

    # Certificate info results
    certinfo_result = server_scan_result.scan_commands_results[ScanCommand.CERTIFICATE_INFO]
    print("\nCertificate info:")
    for cert_deployment in certinfo_result.certificate_deployments:
        print(f"Leaf certificate: \n{cert_deployment.received_certificate_chain_as_pem[0]}")
```

Each [scan result](https://nabla-c0d3.github.io/sslyze/documentation/running-scan-commands.html#sslyze.ServerScanResult) contains the result of all the scan commands that were scheduled for a specific server:

* The server's details are available in `ServerScanResult.server_info`.
* The results of each scan command ran against the server are stored in a typed dictionary in `ServerScanResult.scan_commands_results`. As shown in the example, each result can be retrieved by passing the corresponding scan command as a key. Each result also has a different format and fields depending on the scan command. These fields are [documented][sslyze-scan-commands-doc] and also have type annotations, allowing mpypy to catch mistakes you may make when processing these results (ie. SSLyze is compatible with [PEP 561](https://mypy.readthedocs.io/en/latest/installed_packages.html#installed-packages)).
* If any of the scan command failed in any way, the error will be stored in `ServerScanResult.scan_commands_errors` and no result will be available for this command in `ServerScanResult.scan_commands_results`.

A more detailed example of using the Python API is available [here](https://github.com/nabla-c0d3/sslyze/blob/master/api_sample.py).

Lastly, it is still possible to run mass scans without the Python API, by just using SSLyze's command line. To allow processing in any language, results can be written to a JSON file using the `--json_out` option. Unlike previous versions of SSLyze, the format of the JSON results is now identical to the Python results (same field names and same types) so the documentation is the same.

### Benchmark

To test the new version, I ran the following benchmark:

* Scan the Alexa top 100 sites from a single computer.
  * Among these sites, 6 were not reachable from the U.S (most likely because they're only accessible from China) so a total of 94 sites were actually scanned.
* For each site, run 14 kinds of SSL scans/commands (Heartbleed, Robot, cipher suites, etc.).
  * That's a total of 94 * 14 = 1316 scan commands.
  * This includes for example the testing of 38 950 cipher suites in total (about 400 combinations of cipher suites and SSL/TLS versions per servers).

With the previous version of SSLyze, v2.1.4, the results were the following:

* The scan took 706 seconds total.
* 17 scan command failed out the 1316 that were run, most of them due to timeouts (ie. SSLyze being too aggressive).

With the new version of SSLyze, v3.0.0, the results were the following:

* The scan took 444 seconds total; that's almost half the time.
* Only 1 scan command failed out the 1316 that were run.

That's a pretty big improvement. Additionally, this benchmark was run against very popular sites (the Alexa top 100) that usually can handle the kind of connection spikes that old versions of SSLyze would cause. When scanning less popular sites, the new version of SSLyze will shine even more by consistently returning successful scans.

### More details and changelog

For more details, head to [the project's page][sslyze-gh] or the [Python documentation][sslyze-documentation].

[nass-gh]: https://github.com/nabla-c0d3/nassl
[sslyze-documentation]: https://nabla-c0d3.github.io/sslyze/documentation/
[sslyze-gh]: https://github.com/nabla-c0d3/sslyze/releases
[sslyze-scan-commands-doc]: https://nabla-c0d3.github.io/sslyze/documentation/available-scan-commands.html#sslyze.ScanCommand
