---
layout: post
title: "Improving BenB196's elastic heartbeat reverse DNS check"
author: John Moeller
date: 2024-09-12 17:05:05 -0700
tags: [elastic, heartbeat, dns, _dns_reverse_lookup_failed]
---

We use [Elastic Heartbeat](https://www.elastic.co/guide/en/beats/heartbeat/current/heartbeat-installation-configuration.html) for basic uptime monitoring of systems and services in our environment.

We've found [BenB196's hack to monitor DNS servers with Heartbeat by asking the DNS server to resolve its own reverse DNS record](https://discuss.elastic.co/t/heartbeat-tcp-dns-query/296180). The monitor sets the tag `_dns_reverse_lookup_failed` in the case where the DNS server can't provide its own forward record given its IP address.

This was a clever, non-obvious way to use heartbeat, and a helpful way for us to be aware of DNS issues in our environment.

We recently stumbled upon a way do improve it slightly. Using heartbeat's `replace` module, we can change the monitor's status to down/failed when the `_dns_reverse_lookup_failed` tag is applied, so we don't have to go looking for the tag. All we had to do was add this as the final processor in our DNS server monitor config file:

```
  - replace:
      when:
        contains:
          tags: _dns_reverse_lookup_failed
      fields:
        - field: "monitor.status"
          pattern: "up"
          replacement: "down"
      ignore_missing: true

```

I would have replied with this info to the original thread on elastic.co, but it's closed to replies. :(

Thanks [BenB196](https://discuss.elastic.co/u/benb196/summary)!

---

