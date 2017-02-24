---
title: 'Netfilter: Blocking sites using TLS'
s: netfilter-blocking-sites-using-tls
date: 2016-11-30 00:29:00
tags:
---
Blocking certain websites is a task most of us need to do from time to time. A challenge that has arisen in the last years is TLS. Blocking HTTP traffic is not the most straightforward task, but it can be done using the netfilter string module.

Now, TLS is a whole different beast. We can no longer simply search for the Host field and filter on that. I've been looking for a good solution for a long time, but I haven't found anything that works out of the box. So I decided to roll my own.

Enter [xt_tls](https://github.com/Lochnair/xt_tls/), a module for netfilter which allows you to filter TLS traffic based on the SNI extension (and the server certificate in the future). You can for example make a rule like this:
```bash
sudo iptables -A OUTPUT -p tcp --dport 443 -m TLS --TLS-host "*.googlevideo.com" -j DROP
```
This rule will drop all TLS traffic to "googlevideo.com" subdomains. xt\_tls supports glob-style patterns, which should be enough for whatever you're trying to match.

**Note**: Currently the module only drops the initial client hello packet, however this should be enough, since the server will drop the connection when is doesn't receive the client hello.

Adding support for matching on the server certificate is on my todo-list, but it's proven to be quite a challenge, so I wouldn't expect anything in the near future.

TL;DR: The module is freely available on [GitHub](https://github.com/Lochnair/xt_tls/).