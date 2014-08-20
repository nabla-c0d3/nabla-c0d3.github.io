---
layout: post
title:  "STARTTLS.info Uses SSLyze!"
date:   2014-02-25 07:51:00
post_author: Alban Diquet
categories: ssl sslyze
---

Einar Otto Stangvik just released an interesting study regarding the [state of
STARTTLS for SMTP servers in Norway][starttls-norway]. He used SSLyze to perform
the scans and wrote a script to process SSLyze's XML output to analyze the data
and compute an SSL grade for each server, using SSL Labs' [SSL Server Rating
Guide][ssl-labs-rating]. He found that:

* More than 60% of the scanned mail servers do not support STARTTLS.
* 43% of the mail servers that do support STARTTLS get an F grade, meaning that
they either expose an invalid SSL certificate or they support extremely weak
cipher suites.

Although limited to email servers in Norway, this study illustrates the poor
state of email encryption. One major issue that isn't mentioned in the article
is that because so many SMTP servers expose an invalid SSL certificate, it is
impossible for SMTP clients (which can be both end-user clients and other mail servers)
to implement strict validation of the server's certificate, resulting in the SSL
connection being trivial to man-in-the-middle.

To improve the situation, Einar deployed [STARTLS.info][starttls-info], a
website to easily scan SMTP servers and rate their STARTTLS configuration,
similarly to what [SSL Labs][ssl-labs] does for HTTPS servers. This is a very
useful tool and it also nice to see a project make good use of SSLyze.


[ssl-labs]: https://www.ssllabs.com/ssltest/
[starttls-info]: https://starttls.info/
[ssl-labs-rating]: https://www.ssllabs.com/downloads/SSL_Server_Rating_Guide_2009e.pdf
[starttls-norway]: https://usikkert.no/blog/the-state-of-starttls-in-norway/

