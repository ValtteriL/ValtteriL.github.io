---
title: "DNS records of 1% .fi domains exposed through Zone Transfers"
subtitle: "DNS misconfiguration from the 90s still common among Finnish domains"
date: 2022-01-13T16:16:31+02:00
description: "Post describing my experiment of finding out how commonly nameservers are misconfigured to allow zone transfers"
tags: [
    "dns",
    "recon",
    "finnish",
    "misconfiguration",
]
categories: [
    "Research",
]
draft: false
---

DNS Zone Transfer is a mechanism for administrators to replicate DNS datasets across DNS servers. If it is enabled for a DNS nameserver, the nameserver will gladly give all DNS data regarding a domain to anyone who asks. Enabling Zone Transfers will cause an information disclosure and can thus be considered misconfiguration.

I decided to investigate how common this nameserver misconfiguration is by doing a zone transfer query on all .fi domains I know of (in total 330k domains). This post describes the experiment.

## Effortless reconnaissance tool for hackers

When a systematic hacker targets an organization, one of the first things they do is footprinting. Footprinting consists of gathering information about the target organization that can later be used to prepare effective attacks against it. One piece of such useful information is a list of subdomains.

The list of the target's subdomains can be brute-forced or they can be amassed from public sources, such as certificate transparency logs. These techniques are effective, but they require time and effort, and the attacker cannot be sure whether they have gathered all of the subdomains. Without a full list, the attacker may miss exploitable vulnerabilities in the organization. 

There is a less known and less involved method of piecing together an exhaustive list - DNS Zone Transfer. In addition to leaking all the subdomains, it will reveal all other information in DNS records that might be useful for attackers.

## 9% of nameservers misconfigured

The experiment was done by sending an AXFR query to every nameserver of 330k .fi domains.

The results are as follows:

{{< figure src="/images/dns-zonetransfer-results.png" alt="DNS Zone Transfer results" >}}

335211 .fi domains used in total 20204 nameservers. Out of these nameservers, 1905 (9%) was misconfigured, and that resulted in the exposure of 4792 (1%) .fi domains.

Many domains use the same nameservers. Misconfiguration in a single shared nameserver will therefore result in multiple .fi domains being exposed.

In the light of these results and the minimal effort required to do it, Zone Transfer is a useful footprinting method for hackers.

## How to check if your organization is exposed

You can check whether your organization's domain is exposed using the following command (requires dig):

```bash
DOMAIN=<your domain> && for i in `dig +short NS $DOMAIN`; do dig axfr @"$i" "$DOMAIN"; done
```

Output when the domain is not exposed is as follows:

```bash
DOMAIN=molemmat.fi && for i in `dig +short NS $DOMAIN`; do dig axfr @"$i" "$DOMAIN"; done

; <<>> DiG 9.16.1-Ubuntu <<>> axfr @ns1.domainhotelli.fi. molemmat.fi
; (1 server found)
;; global options: +cmd
; Transfer failed.

; <<>> DiG 9.16.1-Ubuntu <<>> axfr @ns2.domainhotelli.fi. molemmat.fi
; (1 server found)
;; global options: +cmd
; Transfer failed.

```

If you see an output with a dump of DNS records, such as in the following output, the domain is exposed.

```bash
DOMAIN=zonetransfer.me && for i in `dig +short NS $DOMAIN`; do dig axfr @"$i" "$DOMAIN"; done

; <<>> DiG 9.16.1-Ubuntu <<>> axfr @nsztm1.digi.ninja. zonetransfer.me
; (1 server found)
;; global options: +cmd
zonetransfer.me.        7200    IN      SOA     nsztm1.digi.ninja. robin.digi.ninja. 2019100801 172800 900 1209600 3600
zonetransfer.me.        300     IN      HINFO   "Casio fx-700G" "Windows XP"
zonetransfer.me.        301     IN      TXT     "google-site-verification=tyP28J7JAUHA9fw2sHXMgcCC0I6XBmmoVi04VlMewxA"
zonetransfer.me.        7200    IN      MX      0 ASPMX.L.GOOGLE.COM.
zonetransfer.me.        7200    IN      MX      10 ALT1.ASPMX.L.GOOGLE.COM.
zonetransfer.me.        7200    IN      MX      10 ALT2.ASPMX.L.GOOGLE.COM.
zonetransfer.me.        7200    IN      MX      20 ASPMX2.GOOGLEMAIL.COM.
zonetransfer.me.        7200    IN      MX      20 ASPMX3.GOOGLEMAIL.COM.
zonetransfer.me.        7200    IN      MX      20 ASPMX4.GOOGLEMAIL.COM.
zonetransfer.me.        7200    IN      MX      20 ASPMX5.GOOGLEMAIL.COM.
zonetransfer.me.        7200    IN      A       5.196.105.14
....

```

## Exposed? Here's how to protect yourself

If your domain is exposed, the misconfiguration exists on your nameserver(s). If you manage them yourself, go ahead and disable Zone Transfer functionality for all of them.

If your nameservers are managed by a third party, ask them to disable it for you.

If for some reason the mechanism is required, only allow AXFR queries from the hosts that need it.

## Hire me to test your organization's security

I recently started doing security testing gigs for businesses. From me, you get personal service from start to finish. With the resulting report you know where the weaknesses are so you can fix them, or if your system's security is already mature, you can use the report to prove its security to your clients.

Contact me for details.